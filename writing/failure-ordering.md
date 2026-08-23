# What happens if the write after this one fails

*August 2026*

A customer uploads a spreadsheet with four thousand product rows. Thirty-eight of them have a price the schema rejects. The import runs for eleven minutes, saves 3,962 documents, and then has to tell the user something.

Almost every default answer to that is wrong, and the wrongness is not in the parsing or the validation. It is in the order of the writes that record what happened.

I founded and own the sidecar service this runs in — 289 of its commits against 28 for the next human contributor, about 29,800 lines of non-test TypeScript, deployed to ECS. It exists so a twenty-minute parse stops occupying a request in the customer-facing portal. Underneath it is a 1,642-line processor base parameterised over workbook, document and upload types, driving roughly a dozen pipelines — products, prices, costs, media, purchase orders and their worksheets, transfer orders, stock adjustments, vendors, employees — through one lifecycle: download, parse, map, schema-validate, dedupe, dispatch to the internal data API, persist status.

The consolidation is the part I would defend in a design review, and it is not what this is about. Three invariants in that lifecycle are written down, each because the wrong order silently loses or duplicates rows. All three came out of one question: what happens if the write after this one fails.

## Partial success has to be a terminal state

The default for a batch with 38 bad rows out of 4,000 is to fail the batch. It is the easiest thing to implement, it is what a transaction would do, and it produces a data-integrity bug one step later.

Failing the batch tells the user to fix their file and upload it again. The 3,962 rows that already persisted are still there. So the re-upload either duplicates them or forces a dedupe pass that the user cannot see the result of, and the second attempt has the same failure shape as the first — some rows land, some do not, upload again.

So a mixed batch terminates as *partially completed* and generates a failed-rows workbook containing only the rows that did not land, with their errors. The user fixes 38 rows and uploads 38 rows. The successful 3,962 are never re-sent, because they were never rolled back.

That has a second-order requirement which is easy to miss: re-processing an upload is refused unless the upload is still in its pre-processed state. Without that, a retry against a partially-completed upload re-dispatches the rows that already landed, which is the duplication the workbook exists to prevent, arriving by a different door.

## Capture the ids before the write that reports them

The lifecycle ends with a status write: this upload is complete, partially complete, or failed, and here are the ids of the documents that were saved.

If you assemble that record from the saved ids and write it in one step, then a failure in that write loses the ids of documents that are already sitting in the destination system. The documents exist. Nothing in the import service knows they do. Every recovery path from that point is either a duplicate import or a manual reconciliation.

So the saved ids are captured before the terminal status write, not as part of it. A failure in the status write leaves a record that says which documents persisted, and it can be repaired without going back to the customer. The write that reports an outcome is a worse place to hold the only copy of that outcome than the step that produced it.

## An unimportant failure must not overwrite an important success

The last thing the pipeline does is archive a diagnostic bundle — the parse report, the row-level errors, the timing. It is useful when someone asks why an import took eleven minutes. It is not part of the contract with the user.

In the first version it was inside the same completion path as the status write, which means an S3 timeout on a diagnostic artifact could turn an import that had fully succeeded into a record marked failed. A non-critical write was able to demote a critical result.

It is now isolated: its failure is logged and cannot reach the status of the upload. The general form of that rule is that a write's blast radius should not exceed its importance, and the place to check it is the error handler, not the happy path.

## The same question in a different service

A fan-out router in the same fleet republishes each document to every sibling service's queue, with the siblings discovered at runtime through service discovery. A document is counted as failed only when *all* queues fail — reaching four of five is recorded as success, because four consumers have the document and the fifth has a retry path.

Two smaller decisions in it came from the same question. An API response with an *absent* collection key is treated as transient: do not cache it, retry. An empty collection is cached normally, because empty is an answer and absent is a missing answer. And one config that is documented as absent in normal operation is classified as expected steady state rather than an error span — recording it as an error would poison the collector's keep-all-errors sampling policy on every single boot, so a correct-looking error classification would have degraded the sampling for the whole service.

## Why these are hard to review

None of the three invariants is visible in a feature. There is no ticket for "partial success is a terminal state", and a reviewer reading the diff sees an error handler, a re-order of two awaits, and a `try` around an archive call. All three would pass review as written either way, and all three are the difference between a recoverable import and a support conversation about duplicated rows.

They are also invisible to a coverage number, which is why I ran a deliberate quality pass on the suite instead of a coverage pass: 125 weak `.toBeDefined()` assertions replaced with specific value checks, and 285-plus tests added for external-failure paths — S3 timeouts, upstream API errors, partial saves — across 46 files in one change. Graded against a written rubric, the suite went from 70 to 90. The coverage percentage barely moved, because the lines were already executed. What changed is that the failure paths are now exercised, and a re-order of those two writes now breaks a test.

One related trap, since it belongs to the same class. Trace context and cached secrets do not cross the worker-thread boundary on their own: `AsyncLocalStorage` does not survive a structured clone, so both need explicit carriers. A pipeline that looks correct in the main thread can lose its trace context on the boundary and report nothing wrong.

The rule I use now is a single question, asked at every write in a terminal path: if this one fails, what does the previous one leave behind, and can somebody reconstruct the truth from it? Ordering is a design decision with a data-integrity consequence, and it costs nothing at the time you make it.
