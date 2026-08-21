# Harold Araujo

**Senior Software Engineer — Backend, Platform & Observability**

Viña del Mar, Chile (UTC-4) · [LinkedIn](https://www.linkedin.com/in/harold-araujo/) · haroldj.araujof@gmail.com

Senior engineer, 12+ years, currently operating at platform scope across a 58-repo TypeScript fleet: instrumentation, CI and release engineering, and the dependency graph underneath them. Python and TypeScript/Node.js on AWS and Azure, with a bias for test automation, migration discipline, and writing things down so other people can move.

Open to Senior/Staff backend or platform roles, IC track, remote (UTC−4).

## Currently

Platform engineer for a **58-repo** TypeScript service fleet — a dozen internal packages, a connector platform integrating NetSuite, QuickBooks Online, Shopify, Teamwork and Yotpo, and the CI, release and observability machinery underneath all of it. In [more depth here](https://github.com/hjaraujof/hjaraujof/blob/main/PROFILE.md):

- **Dependency-graph & release engineering.** Built the orchestrator that sequences fleet-wide merges by dependency order — wave partitioning with cycle detection, then a gate that blocks consumers when a parent's publish fails. Root-caused the duplicate-module-instance and phantom-dependency failures a package-manager migration exposed, and converted standalone services into an Nx monorepo **with git history preserved**. Same discipline under fire: CVE-2025-29927 rolled fleet-wide the week it landed, and exact-version pins across **nine services the same day** a maintainer account was compromised. → [write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/dependency-graph-release-ordering.md)
- **Toolchain centralization.** Founded the shared config package that **32 repos** extend for ESLint, TypeScript, and Vitest — the single base config that made a 14-repo ESM migration a one-file change. Chose that config only after a `tsc` dry-run across **24 repos** separated mechanical errors from real ones, and recorded why the aggressive variant was rejected. Ships two enforcement CLIs. After I moved the first **11 of ~24 repos** off Jest, four other engineers adopted the pattern for the rest without me. → [write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/false-green-gates.md)
- **Observability platform.** Sole author of the OpenTelemetry instrumentation library adopted by **18 services**, and sole author of the self-hosted Loki/Grafana/Tempo/Mimir + Pyroscope backend it reports to — Terraform on AWS, **~$125/month** against a costed **$1–3k/month** SaaS alternative. Found a resource-attribute collapse in it that was silently discarding **77.5% of metric samples** while every dashboard still rendered. → write-ups on [the backend](https://github.com/hjaraujof/hjaraujof/blob/main/writing/self-hosted-observability.md) and [the instrumentation](https://github.com/hjaraujof/hjaraujof/blob/main/writing/where-tracing-stops-being-automatic.md)
- **Identity & tenant isolation.** Wrote the original auth middleware for the central OAuth 2.0 service and still maintain it — second-most-active contributor to that repo overall, most active this year. Most recently: Postgres row-level security for tenant isolation — a role that cannot bypass RLS, every query scoped per transaction, and a CI gate that proves **two-tenant** isolation against a live database instead of trusting application code.
- **Bulk and distributed processing.** Founded the spreadsheet-import sidecar that keeps long parses off the portal — **289 commits to the next contributor's 28**. Its **1,642-line** generic processor base drives **12 import pipelines** that previously each wired their own parser, mapper and validator by hand. Socket.IO progress on two trust levels (shared secret for the portal, JWT for browsers) with buffered replay for clients reconnecting mid-job, and failure ordering so a partial batch never forces a duplicate re-upload. Separately: the ownership protocol and idle-cost work in the lease-and-claim batch state machine the connector fleet shares, and the S3-backed storage service used across it.
- **Integration platform.** Built the fleet's deepest connector — per-document state machines and mappers with idempotency keys threaded through — then generalized its retry and dedup patterns into the base library every connector now extends. Designed and built its **full-sync orchestrator** (~16,000 lines, 403 tests): a **~40-type dependency graph** held as validated data rather than call order, completion detected from three independent signals because no single system's can be trusted, and per-type state so a six-hour run resumes instead of restarting. Plus a Zod schema-registry contract layer, and a fake API generated from live controllers so consumer tests can't silently go stale.
- **Making silent failures loud.** Traced a **~60-second** auth-endpoint hang — long enough for the load balancer to cut the request — to a task deployed without its Redis host variable, silently defaulting to localhost. Fixed with a fail-closed boot assertion, then generalized into a shared timeout primitive across every Redis-touching route. Separately: found an SSH private key leaking into instance boot logs, and closed it by moving the trust boundary from an embedded deploy key to an IAM-scoped read.
- **Review and migration discipline.** I run adversarial review against my own branches. One pass returned nine findings — four were defects in my own previous three commits, two of them putting credentials into telemetry, and the reasoning I had used to justify the design was itself the bug. Breaking changes ship with a written plan first: the mandatory-UUID cut sequenced schema, then a generation safety net, then **12+ services** in parallel, with excluded legacy services named and justified.

## Writing

Technical write-ups on the work above — the reasoning, not the résumé version:

- **[When your safety check is the thing that's broken](https://github.com/hjaraujof/hjaraujof/blob/main/writing/false-green-gates.md)** — a validator that exited 0 having validated nothing, and why a check that can silently pass is worse than no check at all.
- **[Publishing in dependency order](https://github.com/hjaraujof/hjaraujof/blob/main/writing/dependency-graph-release-ordering.md)** — what breaks when a dozen interdependent packages each publish on merge, and the guards each failure taught me.
- **[Self-hosting observability, honestly](https://github.com/hjaraujof/hjaraujof/blob/main/writing/self-hosted-observability.md)** — the cost case for running your own LGTM stack, and the operational bill that comes with it.
- **[Where tracing stops being automatic](https://github.com/hjaraujof/hjaraujof/blob/main/writing/where-tracing-stops-being-automatic.md)** — the boundaries auto-instrumentation doesn't cross, and two sampling defects that delete data while everything reports healthy.
- **[I designed it, someone else ran it](https://github.com/hjaraujof/hjaraujof/blob/main/writing/designed-it-someone-else-ran-it.md)** — what production taught the colleague who operated a tool I built, and why design-time reasoning misses it.

## Open source

Contributor to **[CloudGraph](https://github.com/cloudgraphdev)**, an open-source GraphQL Cloud Security Posture Management engine, 2021–2023 — its CLI has **889 stars**. This is the part of my work anyone can verify without taking my word for it:

| Repo | What I did |
|------|------------|
| [cloudgraph-provider-azure](https://github.com/cloudgraphdev/cloudgraph-provider-azure) | Built out most of the Azure resource coverage — 30+ new service crawlers (AKS, Active Directory, Event Grid/Hub, Data Factory, Security Center, App Service, SQL, networking, …); #2 contributor overall |
| [cloudgraph-provider-aws](https://github.com/cloudgraphdev/cloudgraph-provider-aws) | New service crawlers (CloudFront, DynamoDB, CloudFormation, Elastic Beanstalk, VPC) and IAM policy / permissions-boundary analysis |
| [cli](https://github.com/cloudgraphdev/cli) · [sdk](https://github.com/cloudgraphdev/sdk) | Provider wiring for Azure, entity-mutation generation strategies, Dgraph teardown command |

## Side projects

- **Algorithmic trading platform** (private, solo-built) — Python: a unified async broker interface over Alpaca and Binance, an event-driven backtester with walk-forward validation, Monte Carlo, deflated Sharpe and CVaR, Kelly-criterion sizing using Ledoit-Wolf covariance shrinkage, and an immutable TimescaleDB trade journal with WAL fallback wired into every bot. Two strategies paper-trade live today; the rest are in development. Happy to walk through the design and the validation methodology.
- **[lgtm-stack-experiment](https://github.com/hjaraujof/lgtm-stack-experiment)** — self-contained, org-agnostic LGTM observability stack: Docker Compose for local dev plus a single-host AWS Terraform deploy, 13 Grafana dashboards, Mimir alerting rules, NGINX + Let's Encrypt.
- **[english_tutor](https://github.com/hjaraujof/english_tutor)** — fully local English tutor, no data leaves the machine: llama.cpp serving Qwen 2.5 on a Pascal GPU, faster-whisper ASR, FastAPI backend; grammar review plus speaking-fluency feedback.

## AI-assisted engineering

I run an unattended PR reviewer against live repositories: a Claude Agent SDK loop on a systemd timer that discovers open pull requests, runs a staged adversarial review with reachability-gated findings, posts only verified findings as inline comments, re-verifies pushed fixes, and honors human `/waive` overrides — **160+ production runs** with per-run cost telemetry and task-tier model routing. Underneath it sits reusable agent-loop infrastructure: a billing guard, a task→model router, and a qualification ladder that stops ordinary automation from being over-built into agents. I also maintain a fleet-scale integration of [graphify](https://github.com/Graphify-Labs/graphify) (**104K★**) across ~60 repositories — auto-refresh git hooks, freshness-gated query hints, and drift surveys.

## Stack

**Languages:** TypeScript / Node.js · Python · SQL · PHP (earlier career)
**Backend:** NestJS · Next.js · GraphQL · Prisma · Fastify · Zod
**Cloud & infra:** AWS (ECS, Cognito, DynamoDB, S3, SQS, CodeArtifact) · Azure · Terraform · Docker
**Observability:** OpenTelemetry · Pino · Grafana · Loki · Tempo · Mimir · Prometheus
**Data:** PostgreSQL / TimescaleDB · DynamoDB · MongoDB · SQLite
**Quality:** Vitest · Jest · pytest · TDD · GitHub Actions · Bitbucket Pipelines
**AI tooling:** Claude Code · Claude Agent SDK · MCP · multi-agent orchestration

---

<sub>Most of my work lives in private org repos. Happy to go deep on any of the above.</sub>
