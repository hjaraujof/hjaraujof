# Where tracing stops being automatic

*August 2026*

Auto-instrumentation buys you a trace that covers one process handling one synchronous request. Every boundary past that — a queue message, a worker thread, a process start, a bundler, a module import — is a place where context either survives because somebody made it survive, or silently doesn't.

I own the OpenTelemetry instrumentation library for a fifty-service fleet: decorator-driven span creation, OTLP export, and the context plumbing underneath. Adding spans was never the work. The work was finding the boundaries where spans stop connecting, and every one of them failed the same way — no error, no warning, and a trace that looks plausible until you notice it should have had a parent.

## The queue hop

A publisher enqueues a message, a consumer picks it up, and the consumer's trace starts as a new root. The request that caused it is somewhere else entirely, unlinked. This is the well-known one, and the AWS SDK's auto-instrumentation does bridge it — for some services, some of the time.

The naive fix is to inject trace context into every outgoing message. That produces a second defect: where the auto-instrumentation *already* injected, you overwrite the parent it established, and the publish span detaches from its own trace. So the injection has to be conditional — only where the SDK instrumentation declines to act, preserving the publish-span parent everywhere it doesn't.

The same shape recurs on an SNS-to-SQS fan-out used for cache invalidation. Two hops, two chances for the chain to break, and the second one is not covered by the first fix.

## The worker thread

The context manager is `AsyncLocalStorage`. It follows async continuations within a process, which is exactly why it works and exactly why it stops at a `worker_threads` boundary: the payload crosses through structured clone, and an ALS store is not clonable state.

What I built for this is a transport-agnostic inject/extract pair — a function that serializes the active context plus baggage into a plain object, and one that restores it on the far side. Plain object, so the same pair works for a worker payload, a job-queue entry, or a message body. Cached secrets had to make the same crossing for the same reason, which is how I found it.

Worth saying plainly: this is not an OpenTelemetry defect. The context manager is documented as not spanning process or thread boundaries. It is a boundary the abstraction visibly does not cover, and it reads as covered because everything inside one thread just works.

## Process start

The SDK patches `http` and `undici` when it initializes. If you load it with `import './instrumentation'` at the top of your entry file, module resolution has already bound the unpatched modules by the time the patch runs. You get spans for your own decorated code and none for any outbound HTTP call — which looks like a partially-working setup rather than a misordered one.

So the SDK loads through a `node -r` preload, and the ordering is load-bearing. I have that written in a comment above the file, because the failure gives you a working-looking trace with a hole in it, and nobody debugging six months later would guess that the fix is where the import happens rather than what it imports.

## The module graph

This is the one I would not have predicted.

Our shared instrumentation package constructed its logger at module scope, and the "are OTLP logs enabled" decision was made at construction time. Under ESM, module evaluation happens once, on first import — so whichever module imported the package first froze that decision for the entire process. Import order, in a fleet of services with different entry points, is not a thing anyone reasons about.

It was dormant. No endpoint variable was provisioned anywhere yet, so every service resolved the same answer and nothing looked wrong. It would have gone live the moment a logs rollout added the variable, at which point some services would have exported logs and some silently would not, determined by import order. I found it while repinning the package for an unrelated reason and fixed it by deferring construction behind a lazy proxy, so the decision moves to first use rather than first import.

A sibling of the same class: instrumenting the SQS client the publisher actually resolves, rather than the hoisted copy. When a package manager leaves two instances of a client in the tree, you can patch one and have the application use the other, and the patch reports success. I wrote about how those duplicate instances arise [in more detail elsewhere](dependency-graph-release-ordering.md) — the tracing symptom is just the most confusing way to discover it.

## Sampling

Two failures here, both of which delete data while every component reports healthy.

A bare ratio-based sampler on a child service re-rolls the dice per trace. When the parent has already been sampled in, the child re-rolls and drops it, so you get traces that begin at a service boundary with their origin missing. `ParentBased` respects the upstream decision; the fix is a one-line sampler change and the symptom is a wall of orphaned traces.

The second was a major-version mismatch: 1.x samplers being handed to a 2.x SDK. The types were compatible enough to build and the sampler was ignored at runtime. Nothing errored.

Because of those two, environment validation now checks the propagator configuration explicitly and warns through the diagnostic bridge when it is set to something unexpected. An unset value is fine — the SDK defaults sensibly. A *wrong* value silently breaks propagation with no other signal, which is the case worth surfacing.

## The bundler

A portal built to a standalone server lost its logging transport chain and cloud SDK from the deployable, because the bundler's file tracing only follows what it can resolve statically and a transport loaded by name is invisible to it. Locally the logger worked. In production it was dead on arrival.

The fix is an explicit include list, which is unremarkable. The reason I mention it is that the same class also produced duplicated spans in that app, and both bugs live in the gap between "my code is instrumented" and "the artifact I deployed contains the instrumentation." That gap is not somewhere I would have thought to look.

## The blackout

Five services had zero telemetry despite instrumentation being deployed and configured. I had two plausible explanations and both were wrong.

The first was that the collector endpoints in the secret store were unreachable. They were — but provably for a different reason than I assumed: they were stored as twenty-one individual per-key entries, and the reader parses a single JSON blob, so it could never have read any of them regardless of network or permissions.

The second was that the configuration service would inject the endpoints at startup. It does inject them, and it does so *after* the point where the preloaded instrumentation reads `process.env`. Correct mechanism, useless ordering.

The actual fix injects the endpoints into every rendered task definition, at all four render sites, non-clobbering, with a warning when an existing value drifts from the expected one. It deliberately does not set the shared OTLP variable — setting that would have switched on the full trace SDK for services that were only supposed to be part of a logging rollout, which is a silent scope expansion I would rather not discover later.

Ruling out two reasonable hypotheses took longer than the fix and is most of what I would want someone to take from this. Both were the kind of explanation that survives a code read and dies on inspection of what the process actually does at startup.

## What must never reach a span

Tracing an authorization service means the decorator sits one careless argument away from exporting a password, a client secret, or a refresh token. So the tracing covers every controller, service and repository without capturing argument values, and there is a structural test that fails the build if a newly-traced class captures arguments without individual security review. Two things it did not catch and I fixed by hand: an OAuth state nonce being recorded on an install span, and exception messages carrying secrets into the trace backend.

One interaction that took me a while to see. A configuration key that is documented as absent in steady state was throwing on every boot, and the throw was being recorded as a span error. The collector's sampling policy keeps all errors — so a routine, expected absence was poisoning the error sample on every service start. Classifying that one case as expected steady state, while still recording every other fetch failure as an error, fixed the sampling more than any sampler change did.

## Where this stands

Traces and correlated logs across the fleet, propagation surviving queue, worker-thread and fan-out boundaries, `ParentBased` sampling, the policy test running in CI, and the preload ordering documented where someone will find it.

Still manual: I reconcile what the collector reports ingesting against what the backends report storing, by hand, because that reconciliation is how I found two separate silent data-loss bugs on the backend side. And I have no automated detection for a *broken chain* — a trace that should have a parent and doesn't. Every one of the failures above was found by noticing something, not by an alert. That is the gap I would close next.
