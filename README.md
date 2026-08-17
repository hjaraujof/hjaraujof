# Harold Araujo

**Senior Software Engineer — Backend, Platform & Observability**

Viña del Mar, Chile (UTC-4) · [LinkedIn](https://www.linkedin.com/in/harold-araujo/) · haroldj.araujof@gmail.com

Senior engineer, 12+ years, currently operating at platform scope across a 58-repo TypeScript fleet: instrumentation, CI and release engineering, and the dependency graph underneath them. Python and TypeScript/Node.js on AWS and Azure, with a bias for test automation, migration discipline, and writing things down so other people can move.

Open to Senior/Staff backend or platform roles, IC track, remote (UTC−4).

## Currently

Platform engineer for a **58-repo** TypeScript service fleet — a dozen internal packages, a connector platform integrating external ERP and commerce systems, and the CI, release, and observability machinery underneath them. Six themes, in [more depth here](https://github.com/hjaraujof/hjaraujof/blob/main/PROFILE.md):

- **Dependency-graph & release engineering.** Built the orchestrator that sequences fleet-wide merges by dependency order — wave partitioning with cycle detection, then a gate that blocks consumers when a parent's publish fails. Root-caused the duplicate-module-instance and phantom-dependency failures a package-manager migration exposed. → [write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/dependency-graph-release-ordering.md)
- **Toolchain centralization.** Founded the shared config package that **32 repos** extend for ESLint, TypeScript, and Vitest — the single base config that made a 14-repo ESM migration a one-file change. Ships two enforcement CLIs. → [write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/false-green-gates.md)
- **Observability platform.** Sole author of the OpenTelemetry instrumentation library adopted by **18 services**, and sole author of the self-hosted Loki/Grafana/Tempo/Mimir + Pyroscope backend it reports to — Terraform on AWS, **~$125/month** against a costed **$1–3k/month** SaaS alternative. → write-ups on [the backend](https://github.com/hjaraujof/hjaraujof/blob/main/writing/self-hosted-observability.md) and [the instrumentation](https://github.com/hjaraujof/hjaraujof/blob/main/writing/where-tracing-stops-being-automatic.md)
- **Identity & tenant isolation.** Wrote the original auth middleware for the central OAuth 2.0 service and still maintain it — second-most-active contributor to that repo overall, most active this year. Most recently: Postgres row-level security for tenant isolation, with a CI gate that proves isolation against a live database instead of trusting application code.
- **Distributed processing.** Designed the lease-and-claim batch state machine the connector fleet shares (heartbeat renewal, adaptive backoff, terminal-vs-retryable classification), and the bulk-import sidecar behind it — **12 upload families** on one pipeline, with partial-batch success that never forces a duplicate re-upload.
- **Incident response.** Patched CVE-2025-29927 across every consuming repo the week it landed. When a compromised axios maintainer account published two malicious releases (1.14.1 and 0.30.4, dropping a RAT through `plain-crypto-js`), pinned exact versions across **nine services the same day** — the registry unpublishing those two did not close the risk, because the account could publish another straight into any caret range.
- **How I decide.** Before committing the fleet to a TypeScript base config I ran a `tsc` dry-run harness across **24 repos** to separate mechanical errors from real ones, then shipped the more conservative variant and documented why the aggressive one was rejected. Breaking changes go out with written migration plans — the UUID identity cut ran to 592 lines — and when an adversarial review finds defects in my own branch, the findings get tagged and answered one at a time.

## Writing

Technical write-ups on the work above — the reasoning, not the résumé version:

- **[When your safety check is the thing that's broken](https://github.com/hjaraujof/hjaraujof/blob/main/writing/false-green-gates.md)** — a validator that exited 0 having validated nothing, and why a check that can silently pass is worse than no check at all.
- **[Publishing in dependency order](https://github.com/hjaraujof/hjaraujof/blob/main/writing/dependency-graph-release-ordering.md)** — what breaks when a dozen interdependent packages each publish on merge, and the guards each failure taught me.
- **[Self-hosting observability, honestly](https://github.com/hjaraujof/hjaraujof/blob/main/writing/self-hosted-observability.md)** — the cost case for running your own LGTM stack, and the operational bill that comes with it.
- **[Where tracing stops being automatic](https://github.com/hjaraujof/hjaraujof/blob/main/writing/where-tracing-stops-being-automatic.md)** — the boundaries auto-instrumentation doesn't cross, and two sampling defects that delete data while everything reports healthy.

## Open source

Contributor to **[CloudGraph](https://github.com/cloudgraphdev)**, an open-source GraphQL Cloud Security Posture Management engine, 2021–2023. This is the part of my work anyone can verify without taking my word for it:

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
