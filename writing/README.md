# Writing

Technical write-ups on platform work — mostly the failures, since those are where the reasoning is visible.

- **[When your safety check is the thing that's broken](false-green-gates.md)** — two CI validators that exited 0 having validated nothing, an ESM entry-point guard that never matched under `.bin` symlinks and shims, and the registry-propagation check I deleted for a related reason.
- **[Publishing in dependency order](dependency-graph-release-ordering.md)** — a dozen interdependent packages that each publish on merge, why that topology has an ordering constraint nothing enforces, and the six failures that produced the guards now in place.
- **[Self-hosting observability, honestly](self-hosted-observability.md)** — the cost case for running Loki/Grafana/Tempo/Mimir yourself, and the operational bill that arrives with it, including three bugs that were silently discarding data.

All three describe work on a private codebase. No proprietary logic, secrets, account identifiers, or endpoints appear in any of them; cost figures refer to infrastructure I provision, compared against published vendor pricing at the time. The mechanisms generalize — that is why they are written down.
