# Harold Araujo

**Senior Software Engineer: Backend, Platform and Observability**

Viña del Mar, Chile (UTC-4) · [LinkedIn](https://www.linkedin.com/in/harold-araujo/) · haroldj.araujof@gmail.com

I have 14 years in backend and platform work. Today I work at platform scope across a 58-repo TypeScript fleet that one team of under ten engineers carries. My part is the instrumentation, the CI and release machinery, and the dependency graph under them. Open to Senior or Staff backend and platform roles, IC track, remote.

## What I built there

- **Observability.** Sole author of the OpenTelemetry library that 18 services use, and of the self-hosted Loki, Grafana, Tempo and Mimir backend behind it. [Write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/self-hosted-observability.md)
- **SLOs.** Availability and latency objectives as multi-window burn-rate alerts, with [the rules in public](https://github.com/hjaraujof/lgtm-stack-experiment/blob/main/config/mimir/rules/demo/slo-burn.yaml). [Write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/missing-measurement-not-passing.md)
- **Release engineering.** An orchestrator that merges across the fleet in dependency order and blocks consumers when a parent's publish fails. [Write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/dependency-graph-release-ordering.md)
- **Shared tooling.** One config package that 32 repos extend, and the Vitest preset that 24 repos consume. [Write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/false-green-gates.md)
- **Bulk processing.** A spreadsheet-import sidecar whose processor base drives 12 import pipelines. [Write-up](https://github.com/hjaraujof/hjaraujof/blob/main/writing/failure-ordering.md)
- **Monorepo consolidation.** Ported standalone services into one monorepo, and split a destructive schema migration into expand, switch and contract steps with a dual write.

The full account, with the numbers and who built what, is in [PROFILE.md](https://github.com/hjaraujof/hjaraujof/blob/main/PROFILE.md).

## Writing

- [When your safety check is the thing that's broken](https://github.com/hjaraujof/hjaraujof/blob/main/writing/false-green-gates.md)
- [Publishing in dependency order](https://github.com/hjaraujof/hjaraujof/blob/main/writing/dependency-graph-release-ordering.md)
- [Self-hosting observability, honestly](https://github.com/hjaraujof/hjaraujof/blob/main/writing/self-hosted-observability.md)
- [Where tracing stops being automatic](https://github.com/hjaraujof/hjaraujof/blob/main/writing/where-tracing-stops-being-automatic.md)
- [I designed it, someone else ran it](https://github.com/hjaraujof/hjaraujof/blob/main/writing/designed-it-someone-else-ran-it.md)
- [A missing measurement is not a passing one](https://github.com/hjaraujof/hjaraujof/blob/main/writing/missing-measurement-not-passing.md)
- [What happens if the write after this one fails](https://github.com/hjaraujof/hjaraujof/blob/main/writing/failure-ordering.md)
- [The rule needed to be executable](https://github.com/hjaraujof/hjaraujof/blob/main/writing/executable-rules.md)
- [Test the crawler against the resource, not a fixture](https://github.com/hjaraujof/hjaraujof/blob/main/writing/crawlers-against-real-resources.md)

## Open source

I contributed to [CloudGraph](https://github.com/cloudgraphdev), an open-source GraphQL engine for cloud security posture, from 2021 to 2023. Its CLI has 889 stars. Anyone can check this part of my work.

| Repo | What I did |
|------|------------|
| [cloudgraph-provider-azure](https://github.com/cloudgraphdev/cloudgraph-provider-azure) | Most of the Azure resource coverage: 30+ service crawlers. #2 contributor overall. |
| [cloudgraph-provider-aws](https://github.com/cloudgraphdev/cloudgraph-provider-aws) | Crawlers for CloudFront, DynamoDB, CloudFormation, Elastic Beanstalk and VPC, and the IAM policy analysis. |
| [cli](https://github.com/cloudgraphdev/cli) · [sdk](https://github.com/cloudgraphdev/sdk) | Azure provider wiring and a Dgraph teardown command. |

## Side projects

- **Unattended PR reviewer.** A Claude Agent SDK loop on a systemd timer that reviews open pull requests, posts only verified findings and checks pushed fixes again. 160+ production runs.
- **Algorithmic trading platform** (private, solo-built, Python). One async broker interface over Alpaca and Binance, a backtester, and an immutable TimescaleDB trade journal. Two strategies paper-trade today.
- [lgtm-stack-experiment](https://github.com/hjaraujof/lgtm-stack-experiment). A self-contained LGTM observability stack: Docker Compose for local use and a single-host AWS Terraform deploy.
- [english_tutor](https://github.com/hjaraujof/english_tutor). A local English tutor on llama.cpp and faster-whisper. No data leaves the machine.

## Stack

TypeScript and Node.js · Python · NestJS · Next.js · Prisma · AWS · Terraform and OpenTofu · OpenTelemetry · Grafana, Loki, Tempo, Mimir · PostgreSQL and TimescaleDB · Vitest

<sub>Most of my work is in private org repos. Ask me about any of it.</sub>
