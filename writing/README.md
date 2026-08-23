# Writing

Technical write-ups on platform work — mostly the failures, since those are where the reasoning is visible.

- [When your safety check is the thing that's broken](false-green-gates.md) — two CI validators that exited 0 having validated nothing, an ESM entry-point guard that never matched under `.bin` symlinks and shims, and the registry-propagation check I deleted for a related reason.
- [Publishing in dependency order](dependency-graph-release-ordering.md) — a dozen interdependent packages that each publish on merge, why that topology has an ordering constraint nothing enforces, and the six failures that produced the guards now in place.
- [Self-hosting observability, honestly](self-hosted-observability.md) — the cost case for running Loki/Grafana/Tempo/Mimir yourself, and the operational bill that arrives with it, including three bugs that were silently discarding data.
- [I designed it, someone else ran it](designed-it-someone-else-ran-it.md) — a full-resync orchestrator I built and handed over, the four things production taught the colleague who operated it, and why every one of them was about downstream behaviour under real volume.
- [Where tracing stops being automatic](where-tracing-stops-being-automatic.md) — the boundaries auto-instrumentation doesn't cross: a queue hop, a worker thread, a preload ordering, a first-import-wins latch in the module graph, and two sampling defects that delete data while everything reports healthy.
- [A missing measurement is not a passing one](missing-measurement-not-passing.md) — the availability SLI whose denominator gave my two busiest services a permanent, authoritative-looking zero; why span status cannot see a 4xx; and how each traffic gate was derived from the band it has to resolve rather than copied from the last alert. The rules are public, so the reasoning is checkable.
- [What happens if the write after this one fails](failure-ordering.md) — three invariants in a bulk-import pipeline that are failure-ordering decisions rather than features, and are invisible to both a feature review and a coverage number.
- [The rule needed to be executable](executable-rules.md) — a tracing policy enforced by a test rather than by review vigilance, and the guardrail of my own that was the reason I stopped looking.
- [Test the crawler against the resource, not a fixture](crawlers-against-real-resources.md) — why a cloud-crawler fixture only ever proves the crawler agrees with your recollection, and what provisioning a real resource per service costs.

These describe work on a private codebase, except the CloudGraph provider work and the SLO rules, which are public and linked. No proprietary logic, secrets, account identifiers, or endpoints appear in any of them; cost figures refer to infrastructure I provision, compared against published vendor pricing at the time. The mechanisms generalize — that is why they are written down.
