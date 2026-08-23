# I designed it, someone else ran it

*August 2026*

I built a full-resynchronisation tool in January: 32 files, about 16,000 lines, 403 tests, four analysis documents. Then I handed it over and barely touched it again — six commits on that path since, two of which are fleet-wide migrations that happened to land there.

A colleague has run it for the seven months since, across 48 commits. This is what production taught him that my design did not anticipate, and what I think that says about the limits of designing something you are not going to operate.

## The operation

A full resync re-pulls a client's entire dataset from a vendor system through a connector, a queue, a consumer service, and into its final collection. The mechanism is deliberately destructive: you delete the connector's cached and pending state so its inbound workflows re-pull everything on their next poll.

Doing that by hand is unsafe in three independent ways.

There are roughly **forty document types with real dependency edges** between them. A catalogue style depends on brand, vendor, classification, attribute set and tax category. A sales receipt depends on customer, employee, style and tax category. Trigger a type before its dependencies have landed and its mappers read reference data that isn't there yet — no error, just rows silently written with fields missing.

Several types run to **hundreds of thousands of records**, and the consumer downstream has a fixed per-instance throughput ceiling. Fire everything at once and you don't get a slow sync, you get a flooded consumer and a queue that takes hours to drain.

And a full run takes **many hours**. A closed laptop, a dropped connection or a stray Ctrl-C must not cost you the work already completed, because re-pulling from the vendor's API is both slow and rate-limited.

## What I designed

I would still make all four of these.

**The dependency graph is data, not call order.** Phase tiers and per-type dependencies live in a config module and a YAML file rather than being implicit in the sequence somebody wrote the calls in. The config validates itself at construction: unknown type references, phase-membership consistency, and cycles via a depth-first recursion stack. The alternative — ordering encoded in imperative code — means the graph is only knowable by reading the whole execution path, and a new document type gets added in the wrong place by someone who read the code correctly and still guessed wrong.

**Completion comes from three independent signals, because no single one can be trusted.** The tool polls the connector's own state records, the consumer's state records, and the final collection count, and treats a type as done only when all three stop moving for a stability window. This is the decision I was least sure about at the time and am most sure about now. Every individual system in that chain has a completion signal, and every one of them is a statement about that system's local view — the connector says it dispatched, which is not the same as the consumer having accepted, which is not the same as the rows existing.

**Run state persists per document type.** A sync interrupted at hour five resumes from hour five. The state file records what was triggered, what completed, and what was in flight.

**The destructive trigger is gated behind an interactive confirmation.** The operation deletes cached state to force a re-pull; that is not something anyone should do because of a mistyped flag.

## What production taught the person running it

All four of the following are his work, not mine.

**A completion poller is a load generator against the thing it is watching.** The tool polled the consumer's state tables every ten seconds per document type. For a single high-volume type, those reads all land on one storage partition — the same partition the consumer is writing its state transitions to. Under real volume the poller throttled the consumer it was waiting for, so the count never stabilised and the phase timed out. From outside, that is indistinguishable from a genuine stall. He raised the interval to thirty seconds.

This is a category of bug I did not have a slot for. I had thought about the tool's effect on the *vendor* (rate limits) and on the *consumer* (throughput ceiling). I had not thought about the tool's effect on the consumer *by way of observing it*. Monitoring as a source of load is obvious once stated and was not in my design vocabulary.

**Pass and fail are not enough terminal states.** He added a third: a run that correctly enqueued everything but whose downstream has not finished draining. My design had a binary outcome, which meant a correct run against a slow tail reported failure — and a reported failure on a six-hour destructive operation is expensive, because the honest response to it is to check everything by hand. Separating "I did my job and the downstream is still working" from "something is wrong" is the difference between a tool an operator trusts and one they double-check.

**Gating has to follow the dependency graph, not the leaves.** Some clients don't opt into certain transactional document types, so those should be skipped. His first cut skipped the eleven leaf types. But the mid-tier types those leaves depend on were still being synced, which meant work being done for data nobody would consume. The fix broadened the gate to dependent and published types — which is to say the gating logic needed the same dependency graph the execution logic already had, and initially didn't consult it.

**Resume logic that trusts its own state file is fragile in exactly the case it exists for.** His resume path now classifies each stale in-progress type by reading *live* downstream counts rather than believing the state file: still-pending means keep waiting, zero-pending with recorded successes means it finished while the process was away. An earlier version of that check read the connector's count instead of the consumer's, which is the wrong side of the pipeline, and it was caught in review rather than by the tests.

He also wrote the pre-scaling manager, the detached-run wrapper and the high-volume type extraction — three files with none of my lines in them.

## What held

The module boundaries. The tool is now about 20,869 lines against my 16,000, and every one of those four hardening efforts landed *inside* the structure — a new manager alongside the orchestrator, extra states in the reconciler, a broader predicate in the gating step, a different data source in the resume classifier. Nobody rewrote the orchestrator, replaced the phase model, or moved the dependency graph back into code.

That is the actual verdict on the design, and it is a more useful one than my own opinion of it. A design that needs four significant additions in seven months but no restructuring is a design that put its seams in roughly the right places.

The line-blame split is now about 73/27 in my favour and moving toward him, which is what should happen to a tool somebody else runs.

## What I'd do differently

The pattern in all four gaps is the same, and I didn't see it until I lined them up: **every one is about the downstream's behaviour under real volume.** The poller contending with the consumer, the tail that drains slower than the run, the wasted work behind an un-consulted graph, the state file diverging from live counts. Not one of them is about the vendor, the dependency ordering, or the tool's internal structure — the parts I could and did reason about from a design document.

That is not a coincidence. Volume-dependent downstream behaviour is the class of thing that does not exist at design time. You cannot test it in staging because staging doesn't have hundreds of thousands of records, and you cannot reason about it from the code because it emerges from the interaction of two systems under load.

What I could have built is the instrumentation that would have surfaced it sooner. The tool measured its own progress carefully — per-type counts, phase timing, reconciliation. It did not measure **its own effect on the systems it was driving**: no metric for the consumer's throttle rate during a run, no comparison of consumer write latency inside a sync window versus outside it. Three of the four findings above would have shown up as a graph rather than as a false timeout somebody had to diagnose.

The next tool I build that drives production systems hard gets that instrumentation before it ships: the consumer's throttle rate during a run, and its write latency inside a sync window against outside it. I did not build it here, and someone else paid the diagnosis cost.
