# Self-hosting observability, honestly

*August 2026*

I run the observability stack for a fifty-plus service fleet on hardware I provision myself: Loki for logs, Mimir for metrics, Tempo for traces, Pyroscope for continuous profiling, Grafana in front, an OpenTelemetry Collector taking OTLP and fanning out to the three backends. Terraform provisions it on AWS, S3 holds long-term storage for each backend, a private DNS zone carries internal telemetry, and NGINX with Let's Encrypt terminates TLS for the UI.

The number that got it approved: **roughly $120–130 per month**, against a costed **$1,000–3,000 per month** for equivalent commercial coverage at 20–50 hosts. That is a real spread, and I stand behind the comparison.

This post is about the rest of the invoice — the part that isn't denominated in dollars. I think self-hosting was the right call. I also think most "we saved 90% by self-hosting" posts stop writing at the point where the interesting work starts.

## What the money actually buys

An order-of-magnitude saving is not an argument on its own, because the two options are not the same product. A vendor sells you somebody else's on-call for the telemetry pipeline and defaults a product team tuned across thousands of customers. What I bought instead is control over retention and cardinality policy, and the absence of per-host pricing shaping what I am willing to instrument at all.

That last one turned out to matter more than the invoice. Adding continuous profiling as a fourth pillar was a Terraform change on a Tuesday. Under per-host pricing it would have been a procurement conversation, and I probably would not have bothered.

What I took on in exchange was operational surface, and the rest of this is what that surface looked like in practice.

## The disk filled up

The clearest single day of this project was an incident where the box ran out of disk. Traces and metrics were being written to local block storage, and the retention arithmetic that had been fine at one ingest volume stopped being fine at another.

Fixing it properly took a coordinated set of changes in one day, which I want to enumerate because each one is a different category of problem:

- **Architecture.** Traces and metrics moved off local disk to S3-backed object storage. That's the fix that makes the failure mode structurally unlikely rather than merely deferred.
- **Cardinality.** Span names were being generated with high-variability segments, producing label values long enough that the metrics backend rejected them. Normalizing span names stopped the rejections, and tail sampling capped the trace volume reaching storage in the first place.
- **Infrastructure safety.** While editing the Terraform I noticed the instance's AMI reference would trigger a replacement on the next apply — a destructive change to the box I was in the middle of stabilizing. Pinning it with an ignore rule prevented an unforced outage.
- **Security.** Access moved from open SSH ingress to session-manager only, which had been on the list and was easier to justify while everything was already being touched.

The shape of that day is what self-hosting actually asks of you: the incident is yours, the root cause spans storage architecture and telemetry semantics simultaneously, and nobody is going to page a vendor's SRE team on your behalf.

## Data we were losing without knowing

Three of these, and they are the difference between the dashboards rendering and the dashboards being true.

**77.5% of metric samples were being discarded.** Span metrics were generated from a stream where resource attributes had collapsed, so distinct series were merging and the vast majority of samples were dropped as duplicates. Nothing errored. Dashboards drew lines. The lines were wrong. I found it by reconciling what the collector reported ingesting against what the backend reported storing — which is now something I check deliberately rather than incidentally.

**All logs were going to a tenant that could not be flushed.** Multi-tenancy was misconfigured such that logs were routed to a placeholder tenant, which accepted writes and never flushed them. Logs appeared to be shipping. They were being written into a hole.

**The burst was the binding limit, not the rate.** Ingestion was being throttled while the configured rate limit looked generous, because the burst size was the constraint that actually bound. My commit message at the time was, roughly, *"the burst binds, not the rate"* — which is the kind of thing you only learn by reading the limiter's source after your first three guesses were wrong.

All three failed silently, and they failed silently by construction. A telemetry pipeline's job is to absorb data and keep the application running, so every component in it is built to drop rather than block. Nothing in that chain has an incentive to tell you it dropped something. The consequence is that the pipeline needs its own monitoring, which is a recursion you have to accept: I run collector self-telemetry and stack self-health dashboards, plus alerts for "the pipeline has gone quiet" with floor thresholds so a genuinely idle window does not page anyone.

## Alerting is a product, not a config file

The first version of my alerting was rules firing into a channel. That's not alerting, that's a firehose with extra steps.

What it became: about twenty-five rule definitions split across SLO burn-rate, pipeline-health, and ingestion-health concerns, routed through a notification topic into a function that formats them as structured messages, plus a separate daily digest that summarizes trace errors with links to exemplar traces — production and staging sectioned separately, development excluded entirely.

The iteration that mattered was subtractive. Paging got scoped to production only. Floor thresholds stopped low-traffic windows from generating "pipeline silent" alarms. A counter-reset bug that had been inflating error counts got fixed. Every one of those changes reduced the number of notifications and increased the proportion that meant something.

The failure mode I was avoiding is specific: an alert nobody acts on trains people to ignore the channel, and the channel is where the real one arrives.

## The secret in the boot log

One finding here was security, not reliability, and it's worth including because it's the kind of thing self-hosting puts on your plate.

The instance bootstrapped by cloning its own configuration at boot, using a deploy key. The key was being written into the boot log. Anyone with log access had the key.

The narrow fix is to stop echoing it. The fix I shipped changed the trust model: the instance now fetches its configuration and secrets from a dedicated object-storage bucket using an instance role, which meant the provisioning plan no longer needed to read the secret manager at all. The trust boundary moved from "possession of a long-lived key embedded in bootstrap" to "an IAM role scoped to one bucket read."

That's a better system, and I only got there because the first fix felt like it was treating the symptom.

## Where this stands

I would do it again, and I am doing it — the stack is what the fleet reports to today, and the profiling pillar went in recently enough that I am still tuning its memory limits.

The condition I would attach is narrower than "have an on-call rotation." Someone has to be willing to go looking for silent data loss on a day when nothing is broken. Two of the three bugs above were invisible from the UI and neither raised an alert; I found both by reconciling what the collector reported ingesting against what the backends reported storing, because I decided to check, not because anything told me to. A vendor's defaults would not have caught them either — but a vendor's support engineer might have, and that is a real difference in kind. If nobody on your side is going to do that reconciliation, buy the product and spend the attention somewhere it pays better.

Current state: Mimir 3.1.2 and Tempo 3.0.2, roughly 25 alert rules split across SLO-burn, pipeline-health and ingestion-health, every deployed config validated on PR against a positive control. The ingest-to-storage reconciliation is still a manual check I run rather than an automated one, which is the next thing on the list.
