# Harold Araujo

**Senior Software Engineer: Backend, Platform and Observability**

Viña del Mar, Chile (UTC-4) · [LinkedIn](https://www.linkedin.com/in/harold-araujo/) · haroldj.araujof@gmail.com

Fourteen years of backend and platform work, told in three tenses.

## Present tense

I work at platform scope across a 58-repo TypeScript fleet that one team of under ten engineers carries. My part is the instrumentation, the CI and release machinery, and the dependency graph under them.

- **I run the observability.** I am the sole author of the OpenTelemetry library that 18 services use, and of the self-hosted Loki, Grafana, Tempo and Mimir backend behind it. [Write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/self-hosted-observability.md)
- **I keep the SLOs honest.** Availability and latency objectives are multi-window burn-rate alerts, and [the rules are public](https://github.com/hjaraujof/lgtm-stack-experiment/blob/main/config/mimir/rules/demo/slo-burn.yaml). [Write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/missing-measurement-not-passing.md)
- **I release in dependency order.** My orchestrator merges across the fleet and blocks consumers when a parent's publish fails. [Write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/dependency-graph-release-ordering.md)
- **I maintain the shared tooling.** One config package serves 32 repos, and one Vitest preset serves 24. [Write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/false-green-gates.md)
- **I own the bulk imports.** A spreadsheet-import sidecar drives 12 import pipelines from one processor base. [Write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/failure-ordering.md)
- **I am consolidating the fleet.** Standalone services move into one monorepo, and a destructive schema migration ships as expand, switch and contract steps with a dual write.

After hours, the tense stays present:

- **An unattended PR reviewer.** A Claude Agent SDK loop on a systemd timer reviews open pull requests, posts only verified findings and checks pushed fixes again. It has 160+ production runs.
- **An algorithmic trading platform** (private, solo-built, Python). One async broker interface over Alpaca and Binance, a backtester, and an immutable TimescaleDB trade journal. Two strategies paper-trade today.
- [lgtm-stack-experiment](https://github.com/hjaraujof/lgtm-stack-experiment). A self-contained LGTM observability stack: Docker Compose for local use and a single-host AWS Terraform deploy.
- [english_tutor](https://github.com/hjaraujof/english_tutor). A local English tutor on llama.cpp and faster-whisper. No data leaves the machine.

## Past tense

- **2021 to 2023, cloud security.** I built most of the Azure coverage of [CloudGraph](https://github.com/cloudgraphdev), an open-source GraphQL engine whose CLI has 889 stars. I also wrote most of the private IAM analysis service behind its findings.
- **2012 to 2021, full stack.** I started in PHP, moved to Python, then to Node.js. I shipped an ERP, a clinical-laboratory system, and an editorial API on Stripe and AWS.

The open-source part is the part that anyone can check:

| Repo | What I did |
|------|------------|
| [cloudgraph-provider-azure](https://github.com/cloudgraphdev/cloudgraph-provider-azure) | Most of the Azure resource coverage: 30+ service crawlers. #2 contributor overall. |
| [cloudgraph-provider-aws](https://github.com/cloudgraphdev/cloudgraph-provider-aws) | Crawlers for CloudFront, DynamoDB, CloudFormation, Elastic Beanstalk and VPC, and the IAM policy analysis. |
| [cli](https://github.com/cloudgraphdev/cli) · [sdk](https://github.com/cloudgraphdev/sdk) | Azure provider wiring and a Dgraph teardown command. |

## Future tense

I will join a team as a Senior or Staff backend or platform engineer: IC track, remote.

The full account of all three tenses, with the numbers and who built what, is in [PROFILE.md](https://github.com/hjaraujof/hjaraujof/blob/main/PROFILE.md).

## Writing

Mostly about failures, because the reasoning shows there.

- [When your safety check is the thing that's broken](https://github.com/hjaraujof/hjaraujof/blob/main/writing/false-green-gates.md)
- [Publishing in dependency order](https://github.com/hjaraujof/hjaraujof/blob/main/writing/dependency-graph-release-ordering.md)
- [Self-hosting observability, honestly](https://github.com/hjaraujof/hjaraujof/blob/main/writing/self-hosted-observability.md)
- [Where tracing stops being automatic](https://github.com/hjaraujof/hjaraujof/blob/main/writing/where-tracing-stops-being-automatic.md)
- [I designed it, someone else ran it](https://github.com/hjaraujof/hjaraujof/blob/main/writing/designed-it-someone-else-ran-it.md)
- [A missing measurement is not a passing one](https://github.com/hjaraujof/hjaraujof/blob/main/writing/missing-measurement-not-passing.md)
- [What happens if the write after this one fails](https://github.com/hjaraujof/hjaraujof/blob/main/writing/failure-ordering.md)
- [The rule needed to be executable](https://github.com/hjaraujof/hjaraujof/blob/main/writing/executable-rules.md)
- [Test the crawler against the resource, not a fixture](https://github.com/hjaraujof/hjaraujof/blob/main/writing/crawlers-against-real-resources.md)

## Stack, conjugated

**I use**

- Languages and frameworks: TypeScript, Node.js, SQL, NestJS, Next.js, Fastify, GraphQL, Prisma, Zod, Socket.IO
- Data: PostgreSQL, DynamoDB, MongoDB, SQLite, Redis
- Cloud and delivery: AWS (ECS, Cognito, S3, SQS, CodeArtifact), Terraform and OpenTofu, Docker, GitHub Actions, Nx, pnpm
- Observability: OpenTelemetry, Pino, Grafana, Loki, Tempo, Mimir, Pyroscope, Prometheus rules
- Quality: Vitest, TDD, contract tests
- AI tooling: Claude Code, MCP, multi-agent orchestration

**I am building with**

Python, pytest, FastAPI, TimescaleDB, Claude Agent SDK, llama.cpp, faster-whisper

**I used**

Azure, Kubernetes as a deploy target, GitLab CI, Bitbucket Pipelines, Jest, Loopback, Angular, React, Django, PHP

<sub>Most of my work is in private org repos. Ask me about any of it.</sub>
