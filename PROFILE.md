# Harold Araujo — engineering detail

The long version of my [profile](README.md), for people who want specifics. No proprietary logic appears here — everything is described at the level of engineering mechanics, which is the level that transfers anyway.

Where work was shared, I say so. Commit counts are a weak signal and I have tried not to lean on them. And where I say "sole author" or "primary maintainer," you cannot verify that from this page — a reference or a future manager can, and I would rather state the limit than imply a check you can't actually run.

Everything described here is carried by a single engineering team of under ten people, which is the scale every ownership claim below should be read against — "own" never means having a team underneath me. There is no on-call rotation either: production alerts route by service ownership, which for the observability backend, the release machinery and the bulk-import sidecar means me, and incidents get worked the day they land rather than handed to a rota. Two numbers you will not find here are request volume and uptime attainment. I would be reconstructing both rather than reading them, so they are absent — the service-level objectives I defined are in the observability section, with their thresholds and the gap in their calibration stated.

---

## Dependency-graph & release engineering

A dozen internal packages under one private registry scope, consumed by 50+ repos, each package CI-versioned: merging publishes a prerelease and the pipeline writes its own version-bump commit. That topology has a hard ordering constraint — when a base library and its consumers change together, the base must merge, publish, and *actually land in the registry* before a consumer's install runs, or the consumer freezes a stale pin, resolves two instances of the same package, or fails a frozen-lockfile install outright.

**The orchestrator.** A declared dependency graph drives wave partitioning (Kahn's algorithm) with DFS cycle detection; each wave dispatches its repos as parallel CI runs, and a dependent gate blocks, skips, or prompts when a parent's publish fails or is skipped. I wrote the large majority of this: by `git shortlog -sn`, 169 of 185 commits in the reusable-workflow library, and roughly two-thirds of the CLI that drives it.

**The version pipeline**, built without release-please or changesets: branch-aware semver and prerelease derivation, atomic branch-plus-tag pushes with collision retry, and a JSON-schema config validator. Roughly 28 repos consume it as reusable workflows.

**Failures that taught me something,** each of which became a guard rather than a patch:

- A stale registry dist-tag resolved *behind* a consumer's pin and silently downgraded it — shipped before anyone noticed. The repin tool now refuses any resolution below the current pin, using a real semver-precedence comparator rather than `sort -V`, which orders prereleases wrong.
- A postinstall hook failing under `set -e` aborted the repin mid-run, leaving nested workspace manifests stale while the root advanced — so a consumer's tree resolved both a `dev` and a `qa` build of the same package at once. Rewrote it to edit every manifest directly, then run one lockfile-only install at the end.
- A repo detected as "changed" every run cascaded a re-merge trigger to all 17 of its dependents without ever merging itself, so one subtree never converged.
- A `build_only` flag inverted in a dispatch call let merges land but never publish, quietly breaking the child version-bump chain fleet-wide.
- I deleted a registry-propagation check — 107 lines — after realizing it was a false oracle: it diffed local manifests against the registry, but the publish pipeline owns version mutation, so it was checking the wrong side of an async change. Replaced with SHA-correlated tracking of the pipeline's terminal state.

**Resolution-mechanics bugs** an npm→pnpm migration surfaced, where npm's flat tree had been masking undeclared dependencies and accidentally collapsing duplicates:

- A shared library imported `@opentelemetry/api` in source while declaring it neither a dependency nor a peer — it worked only because consumers hoisted it. Declared explicitly and pinned to the same range a sibling package uses, specifically so the package manager dedupes to one copy: it holds a global provider registry, and two copies silently break trace correlation everywhere.
- A repository class arrived by two import paths — directly, and inherited through a base class that resolved its own copy. Two instances, so TypeScript's private-field nominal check reported two incompatible types with the same name.
- Prisma's generated client landed where nothing read from it, because a custom generator output pointed at the hoisted directory while the client re-exports from its own nested stub.
- One repo had a committed, tokenless `.npmrc` shadowing the one CI's registry login writes — a 401 in exactly one service while four siblings built fine.

**Supply-chain response.** Two events, and the second is the one I'd rather be judged on. A published CVE in a framework's authorization middleware got patched and rolled across every consuming repo the week it landed — necessary, unremarkable. Then a maintainer account for a widely-used HTTP client was compromised and two malicious releases went out. The registry unpublished them within hours, which makes the naive read "crisis over." I pinned exact versions across nine services the same day anyway, because the *account* was the vulnerability: while it stayed compromised, every caret range in the fleet was a standing invitation for the next publish. Exact pinning for security-sensitive packages is now the standing response rather than a one-off.

→ Full write-up: [Publishing in dependency order](writing/dependency-graph-release-ordering.md)

## Toolchain & configuration centralization

I founded the shared devtooling package the fleet extends: a parameterized ESLint flat-config factory, a Prettier config, one TypeScript base, and a Vitest preset. **32 repos declare it; 30 wire their ESLint config to it.** Two Next.js applications roll their own ESLint and TypeScript configs and adopt only the Vitest preset — so this covers the services and libraries, not literally everything.

The TypeScript base mattered more than it sounds. Before committing to a variant for the fleet's move to NodeNext, I ran a real `tsc` dry-run harness across 24 repos — not estimates — to bucket errors into mechanical and genuine, measuring roughly 2,400 bare imports and a 23-consumer blast radius. That produced a deliberate choice to *omit* `verbatimModuleSyntax`: its mechanical import-rewrite cost fell almost entirely on the fleet's 19 CommonJS-emitting repos for no real safety gain. Because the base lived in one package, the subsequent ESM migration across 14 independently-versioned repos was a one-file change plus coordinated re-pinning, executed in about a week.

The package also ships two enforcement CLIs: one that fails a build if any internal dependency carries a range instead of an exact pin, and one that pins test-helper packages in lockstep with the schema package they peer against — written after I found the first validator was blind to peer and dev dependencies.

Both were, for a while, doing nothing at all. See the write-up.

→ Full write-up: [When your safety check is the thing that's broken](writing/false-green-gates.md)

**Test framework migration.** I wrote the shared Vitest preset six weeks *before* the first repo migrated — library first, consumers after — and 24 repos now consume it via `mergeConfig`. It sets the coverage provider, a JUnit reporter wired to CI test reporting, and heap logging; it deliberately does *not* pin isolation or pool settings, because two repos legitimately needed to override them. Eleven other Vitest repos keep standalone configs and don't consume it, which is fine — it was never a mandate.

Of the fleet's 27 Jest→Vitest migrations, **I performed 15** — the shared libraries, the connector base, the data API, the portal, the sidecar — and five colleagues did the other 12. Two of mine were finishing other people's: one repo had jest removed with no replacement, leaving it with no test runner at all, and another carried an abandoned half-migration whose 42 residual type errors I cleared before turning its CI gate on.

Notable failures along the way:

- Vitest 4 began calling `new` on mocked constructors, breaking every arrow-factory `mockImplementation`. The obvious fix — a plain function expression — wasn't durable, because the shared ESLint config's own `prefer-arrow-callback` autofix would convert it straight back to a broken arrow. Class expressions satisfy constructability and are immune to that autofix. ~175 tests recovered across three repos.
- A mock-hoisting behaviour drifted between patch releases: passing on 4.1.3, failing on CI's 4.1.5. `vi.spyOn` is version-stable; the factory pattern wasn't.
- A leaked fake timer under a shared worker froze `setTimeout` and produced 40 cascading CI failures while all 454 tests passed locally. Fixed the leak at its source, then removed the flag that allowed cross-file state to leak at all.
- A dead, unawaited `vi.importActual`/`vi.doMock` pair surfaced as an intermittent teardown error in roughly one run in four — while every test still reported passing, which is why it had been written off as noise.

**Not just unit tests.** Five distinguishable kinds exist in this fleet and my share differs sharply by kind, so it's worth separating them.

The bulk is ordinary unit testing with mocked boundaries, and that's where my near-total-ownership repos sit — 112 of 121 test files in the import sidecar, 109 of 112 in the devops CLI, 38 of 43 in the instrumentation library, 12 of 12 in the shared utility package. Across the eight repos I own most of, 326 of 459 test files trace to me. That figure is honestly skewed: it's pulled up by those four and pulled down by two genuinely shared repos where I'm a large minority or tied, not dominant.

Beyond that: **integration suites that use real components rather than mocks** — ten pipeline tests in the sidecar running parse → validate → transform → dispatch end to end, plus upload and socket tests against a real Socket.IO connection and real workbook services. **Contract tests**, which I built solely: a route-aware fake of the data API whose manifest is *generated from the live controllers*, so route drift fails a test instead of passing silently. **Characterization suites** written against a real Postgres to pin existing behaviour before converting two services into the monorepo. And **infrastructure tests** — alerting rules for the metrics backend have their own rule-unit specs, plus a reconciliation script, because an alert that cannot fire is worse than no alert.

**How I treat flake.** Not by re-running until green. Four examples where the commit records a measured before and after: a suite taken from 3-of-8 passing to **43 of 43 across ten consecutive runs** by scoping non-parallel execution to only the database-backed project rather than the whole suite; the teardown bug above measured at **0 of 12 runs** after the fix against roughly 1-in-4 before; the constructable-mock break recovering 29 tests in one package and ~175 across three; and a deliberate quality pass replacing **125 weak `.toBeDefined()` assertions** with value checks and adding **285+ external-failure tests** across 46 files, graded against a written rubric from 70 to 90. Coverage percentage barely moved on that last one — the failure paths went from unexercised to exercised, which is the part that matters.

I'd add one honest note: the global DynamoDB mock that fixed a worker-pool flake in the largest connector, and the majority of that connector's and the data API's test files, are colleagues' work.

## Observability platform

**The instrumentation library** (sole author, 18 consuming services): decorator-driven auto-instrumentation, OTLP protobuf export, context propagation across worker and queue boundaries, and `ParentBased` samplers after I traced a class of orphaned cross-service traces to sampling configuration. Related defects I root-caused rather than upgraded past:

- Trace context lost across SQS hops that the AWS SDK's auto-instrumentation declines to bridge — injecting only where it declines, so publish-span parentage survives.
- 1.x samplers being handed to a 2.x SDK, causing real trace loss.
- A module-scope logger construction that froze the "OTLP logs enabled" decision on a first-import-wins basis under ESM. It was dormant only because no endpoint variable was provisioned yet, and would have gone live the moment a rollout added one.

**The backend** (sole author, no other contributors): Loki, Grafana, Tempo, Mimir, and later Pyroscope behind an OpenTelemetry Collector, on AWS with S3 as long-term storage for each backend, a private DNS zone for internal OTLP, NGINX with Let's Encrypt, and IAM-scoped collector access. I costed it at roughly **$120–130/month against an estimated $1,000–3,000/month** for equivalent commercial coverage at 20–50 hosts.

**All of it is OpenTofu/Terraform, and the infrastructure code is the part I'd hand someone.** Split by concern rather than kept as one file — compute, S3 backends with lifecycle rules and public-access blocks, instance bootstrap, alerting, the Slack alert bridge, and the profiling box each get their own. Specific decisions worth naming:

- **State and secrets stay out of the plan.** Instance bootstrap fetches its configuration and secrets from a dedicated bucket via an instance role, which removed the plan-time secret-manager read entirely. That change came out of finding a deploy key in the boot log; the fix moved the trust boundary rather than stopping the echo.
- **`ignore_changes` on the AMI reference.** I noticed mid-incident that the next apply would have replaced the instance I was stabilising. Pinning it turned an unforced outage into a no-op — arguably the highest-value three lines in the repo.
- **Plan-only CI on pull requests**, and separately a validation job for every deployed config file, not just the Terraform — with a positive control, a deliberately invalid file the job must reject, so the validator is observed failing rather than assumed working.
- **Capacity as a first-class change.** Adding continuous profiling as a fourth pillar was an isolated VPC-internal box rather than more load on the CPU-bound main instance — a Terraform change on a Tuesday, which is the whole argument for owning the infrastructure code.

**Service-level objectives.** I wrote the fleet's SLO layer — windowed recording rules for availability and latency, then multi-window multi-burn-rate alerts on the Google SRE workbook pattern. A **99% availability** budget: a fast burn at **14.4×** the budget on the 1h *and* 5m windows pages, a slow burn at **6×** on the 6h *and* 30m windows opens a ticket, and both windows must breach so a brief spike cannot page. The latency objective is **95% of inbound requests under 250ms** over 30 minutes. The thresholds were the easy part; the measurement underneath them took two corrections and still carries one open gap.

The numerator was wrong twice, and both times it measured instrumentation depth rather than failure. Availability first ran on span *status* across every span kind, so one failing request marked every decorated method span it touched, and a deeply instrumented service looked less available than a shallow one. The latency SLI had the same defect: one service emits roughly 500 internal spans per inbound request, so its measured fast-ratio read 0.92 against a real user-visible 0.78, and a second service breached for 45 contiguous minutes on internal spans alone while serving 100% of inbound requests under 250ms. Adding instrumentation was moving the SLO with no user-visible change. Both are now restricted to server spans, and availability keys on HTTP status code rather than span status.

- **A 401 is not an availability breach, and it also cannot be invisible.** Under OpenTelemetry semantics a 4xx leaves server span status unset, so a sustained authentication failure across most of a service's traffic read as a healthy zero — invisible to every rule keyed on span status, and NestJS does not log 4xx either. It gets its own SLI, a sustained 401/403 ratio, deliberately excluded from the error budget because a service that authenticates a caller and says no is working correctly.
- **Sample-count gates derived rather than copied.** 300 requests per 30 minutes for the latency ratio, where the standard error is about 1.3 points against a 5-point band; 150 for the auth ratio, because that band is 10 points wide and copying 300 across would have excluded the one service the alert exists to catch. An earlier gate keyed on request *rate* flapped for weeks before I worked out the instability was sample count.

The open gap is coverage. Restricting the availability denominator to requests that carry an HTTP status code left the two highest-traffic services emitting no series at all, because their framework's own tracing sets neither status code nor route. Dividing by all server spans instead would have handed them an authoritative-looking 0% error rate, so the honest denominator silently produces nothing for exactly the two services most worth watching — and that silence now raises its own ticket. The trade-off is written into the rule: re-enabling the instrumentation that would supply the status code reintroduces a duplicate-span defect that was costing tens of thousands of errors a day.

The 1% error budget is still marked provisional, applied uniformly and never validated against a measured per-service baseline. The multi-window structure and the gates I will defend; attainment against a budget I have not finished calibrating I will not. A sanitized copy of the rules is public in [lgtm-stack-experiment](https://github.com/hjaraujof/lgtm-stack-experiment/blob/main/config/mimir/rules/demo/slo-burn.yaml), thresholds, gates and reasoning comments intact.

Operational reality, which is the part cost comparisons usually skip: a same-day root-cause-and-remediation cycle for a disk-full incident that moved trace and metric storage to object storage, added tail sampling and span-name normalization to stop label-length rejections, pinned the AMI to prevent a destructive instance replacement, and closed SSH in favour of session-manager access. Separately: a span-metrics resource-attribute collapse that was **discarding 77.5% of metric samples**, a multi-tenancy misconfiguration routing all logs into an unflushable placeholder tenant, and an ingestion limit where the burst — not the rate — was the binding constraint.

I also found and fixed a live secret exposure here: an SSH private key landing in instance boot logs. The remediation moved the trust boundary from "clone with an embedded deploy key" to "IAM-scoped object read," which removed a plan-time secret read entirely.

→ Full write-up: [Self-hosting observability, honestly](writing/self-hosted-observability.md)

## Identity & authorization

I joined the central OAuth 2.0 service about two weeks after its first commit, wrote its original auth middleware — token validation, refresh flow, cross-subdomain cookies, sign-out — and remain that file's primary maintainer years later. I'm the second contributor over the repo's life and the leading one this year. The Ed25519 signing keyring and rotation core are a colleague's work, not mine.

Work I'd point to:

- **A tracing policy enforced by a test.** After wiring OpenTelemetry into a service handling passwords, client secrets, refresh tokens, and OAuth codes, I wrote a structural test that fails the build if any newly-traced class captures argument values without individual security review — plus fixes that stopped an OAuth state nonce and exception messages from reaching the trace backend.
- **A Redis wedge.** A task deployed without its Redis host variable silently fell back to localhost; every connection was refused and an auth endpoint hung about 60 seconds until the load balancer cut it. Fixed with a bounded reconnect strategy and a fail-closed boot assertion, then generalized into a shared timeout primitive replacing ad-hoc `Promise.race` across every Redis-touching route.
- **A defect class, not a bug.** Proxy HTML error pages were breaking JSON parsing and rendering raw error text onto the login screen. I'd seen the same class fixed in two sibling services; this one had no shared client to port, so I built the parsing layer fresh across 24 call sites — including an error type whose classification survives serialization boundaries where `instanceof` doesn't, and a bounded diagnostic excerpt that never reaches the browser. Two latent bugs surfaced during the conversion: a form parsing a body before checking status, and a client treating an unparseable 200 as confirmed success.
- **What I'm not claiming here.** The fleet's Postgres row-level-security tenancy model — the tenant-context library, the leakage-test gate, the per-transaction query scoping — is a colleague's design and program, not mine. I converted one service into the monorepo *under* that model and fixed two things doing so (making interactive transactions work with the RLS session settings, and the leakage suite's CI bootstrap), but the isolation architecture is his.

## Distributed document processing

**The shared batch state machine** every connector depends on for document processing under a distributed lease. I authored its ownership protocol and its idle-cost work: claim-and-lease ownership, heartbeat renewal that only renews still-in-flight documents, adaptive idle backoff that self-tunes polling cadence, terminal-versus-retryable error classification, bounded retry and bounded memory. Adjacent work on the same class — busy-tick floor, run-loop jitter — is another engineer's.

**The bulk-import sidecar** is the service I've owned most completely: I founded it in August 2024 and wrote 289 of its commits over two years, against 28 for the next human contributor. It's ~29,800 lines of non-test TypeScript across 200+ files with 117 test files, deployed to ECS, and it exists so that long spreadsheet imports stop blocking the customer-facing portal.

The architecture is a DI container, an authenticated Socket.IO server, and a CPU-sized worker-thread pool. Three parts I'd defend in a design review:

**The generic processor base.** A 1,642-line class parameterised over workbook, document and upload types, driving roughly a dozen import pipelines — products, prices, costs, media, purchase orders and their worksheets, transfer orders, stock adjustments, vendors, employees — through one lifecycle: download, parse, map, schema-validate, dedupe, dispatch to the internal data API, persist status. Before it, each upload type wired its own parser, mapper, validator and config by hand. After, adding a spreadsheet type is a set of factory registrations rather than a new vertical. That consolidation is the thing I'd point to, not the line count.

**Two trust levels on one channel.** The portal's own backend connects with a shared API secret; end-user browser sessions connect with a Cognito JWT. Same Socket.IO server, different principals, and the distinction matters because job progress is per-user data. Socket.IO's `connectionStateRecovery` buffers progress events for a client that drops and reconnects mid-job, so a user who loses their connection during a twenty-minute import doesn't come back to a blank screen and re-submit.

**Failure ordering, which is really a data-integrity story.** Three invariants, each written down because the wrong order silently loses or duplicates rows:

- A mixed batch returns *partially completed* with a generated failed-rows workbook, rather than failing whole — because a forced re-upload would duplicate the rows that already landed. Re-processing is refused unless the upload is still in its pre-processed state.
- Saved document IDs are captured before the terminal status write, so a failure there can't erase the record of what actually persisted.
- A non-critical diagnostic archive write is isolated so its failure can't overwrite an already-completed record with a failure.

Bridging trace context and cached secrets across the worker-thread boundary needed explicit carriers, since `AsyncLocalStorage` doesn't survive a structured-clone boundary.

I also ran a deliberate test-quality pass on it rather than a coverage-number pass: 125 weak `.toBeDefined()` assertions replaced with specific value checks, and 285+ tests added for external-failure paths — S3 timeouts, upstream API errors, partial saves — across 46 files in one change. I graded the suite against a written rubric before and after, 70 to 90. Coverage percentage barely moved; what changed is that the failure paths are now exercised.

**Worth reading together with the portal work below.** The upload subsystem I built in the customer-facing portal (validator, controller and service base classes; S3 upload; progress over a Socket.IO channel) and this sidecar are two halves of one path — the interface and the engine behind it, across two services. I built both ends.

**A fan-out router** in the same fleet: sibling services discovered at runtime through service discovery, each document republished to every sibling's queue, and a document counted as failed only when *all* queues fail — reaching four of five is recorded as success. Two details I'm fond of: distinguishing an API response with an *absent* collection key (transient, don't cache, retry) from a genuinely *empty* one (cache normally); and deliberately classifying one documented-absent config as expected steady state rather than an error span, because recording it as an error would poison the collector's keep-all-errors sampling policy on every boot.

## Product & integration engineering

Before the platform work, and still alongside it.

**A customer-facing Next.js portal** (largest single repo by my commit count): I designed and built its document-upload subsystem end to end — abstract validator, controller, and service base classes extended by concrete flows, S3 upload, async job validation, and real-time progress over a Socket.IO event channel on a custom server. I also migrated its authentication from a hand-rolled token scheme to a standard library and built the permission layer over it (cached permission fetch held in session state, guard-based route authorization), and later instrumented a standalone build — which required fixing Next.js output-file tracing that was silently dropping the logging transport chain and cloud SDK from the deployable, a bug that would have shipped a dead production logger.

**The connector platform** — roughly fourteen services integrating NetSuite, QuickBooks Online, Shopify, Teamwork, Yotpo, SFTP batch partners and a logistics provider, on one shared base library.

My deepest work here is the Teamwork integration, the largest of them: per-document-type state machines and mappers for purchase orders, catalog styles, sales orders, employees and stock adjustments, each with an explicit idempotency key threaded through the pipeline, plus a later gen2 rewrite alongside the original. Those retry, dedup and continuation patterns were proven there first and then generalized into the base library every connector now extends — which is the sequence I'd defend as the right one: solve it concretely in the hardest case, then lift it.

**The full-sync orchestrator.** Re-synchronising a client's entire dataset touches roughly forty document types with real dependency edges between them, several running to hundreds of thousands of records, against a downstream consumer with a fixed throughput ceiling. Done by hand it is unsafe three separate ways: trigger out of dependency order and mappers read reference data that isn't there yet; fire everything at once and you flood the consumer; run for six hours and a dropped connection costs you the entire run.

I designed and built the tool that does it — 32 files and roughly 16,000 lines in the initial implementation, with 403 tests and four analysis documents. Four decisions I would still defend. The dependency graph and its phase tiers live in **data**, validated at startup for unknown type references and for cycles via a depth-first recursion stack, rather than being implicit in the order somebody wrote the calls. Completion is detected by polling **three independent signals** — connector state, consumer state, and the final collection count — because no single system in that chain has a completion signal you can trust on its own. Run state persists per document type, so a six-hour sync interrupted at hour five resumes instead of restarting. And the trigger is gated behind an interactive confirmation, because the mechanism is deleting cached state to force a re-pull, which is not something to do on a mistyped flag.

An honest note on where it stands: I built it and have barely touched it since — six commits on that path, two of them incidental fleet-wide migrations. A colleague has substantially extended it and now owns it operationally, including reactive backpressure with fleet pre-scaling, export gating for document types a client hasn't opted into, and a three-state terminal outcome separating "still draining" from "complete" and "failed". Those are his, not mine. Line-blame currently sits near 73/27 in my favour and is moving toward him, which is what should happen to a tool somebody else runs.

Two pieces of the surrounding contract layer I'd point to. A **schema registry**: each repo declares its types as Zod schemas, vendor-namespaced, published to a runtime configuration service — so every consumer gets a generated, versioned, runtime-validated contract instead of an interface someone hand-copied and forgot to update. And a **drift-guarded fake API**, whose route manifest is generated from the live controllers rather than maintained by hand, so a consumer's tests cannot quietly pass against a surface that no longer exists.

**Inside the monorepo:** I converted two standalone services into the Nx workspace with git history preserved — characterization tests locking in existing behaviour before refactoring, and an ADR for the container start-ordering decision one of them forced. The conversion tooling itself is a colleague's; I used it. I own the S3-backed storage service there. On the shared SQS queue library I'm the second contributor, not its author. This is where most of my last six months has gone.

**Cross-cutting migrations** I led: a required-UUID identity field added to every operational document — a breaking change across 12+ services, sequenced schema-first with generation as a safety net during rollout, deterministic v5 IDs so reprocessing stays idempotent, and legacy services excluded with written rationale; four overlapping libraries consolidated into one namespaced package with subpath exports, migrated across 27 consumers in dependency-ordered batches; and a structured-logging replacement with a wide-event child-logger API, jittered retry, transport health checks, and a stderr fallback.

## Cloud security posture management (2021–2023)

Before the platform work, at a cloud-security startup — the open-source engine, and the private analysis service behind its IAM findings.

**The open-source half** is [CloudGraph](https://github.com/cloudgraphdev), a GraphQL cloud security posture management engine whose CLI has 889 stars. I built out most of the Azure provider's resource coverage — 30-plus service crawlers across AKS, Active Directory, Event Grid and Event Hub, Data Factory, Security Center, App Service, SQL and networking — and I'm the #2 contributor overall there, #1 by feature commits. On the AWS provider I added crawlers for CloudFront, DynamoDB, CloudFormation, Elastic Beanstalk and VPC, wrote the IAM policy and permissions-boundary analysis, and wrote the Terraform that provisions each service I added so its crawler could be exercised against a real resource rather than a fixture. Anyone can check all of that.

**The private half** is the blast-radius analysis service, a NestJS/TypeScript microservice of roughly 11,200 non-test lines, about 9,300 of them mine — the network-reachability engine is a colleague's. It reimplements a slice of AWS's own IAM policy evaluation rather than calling the Policy Simulator.

One evaluator handles identity-based, resource-based, inline, managed, permission-boundary and trust policies: `Action`/`NotAction` and `Resource`/`NotResource` resolution, ARN pattern matching, and roughly forty IAM condition operators — the string and ARN families, numeric and date, `Bool`, the `IfExists` variants, and `IpAddress` by actual CIDR containment. Explicit Deny beating Allow is a set difference per principal rather than a special case. Principals are classified across AWS accounts, roles, users, assumed-role sessions, SAML sessions, OIDC/web-identity sessions and service principals, and `Principal: "*"` expands against every ARN known from the crawl, so public exposure falls out of the evaluation instead of needing its own detector. Two documented gaps in that expansion: it enumerates no session principals and does not consult the RAM API.

Every role's assume-role policy runs through the same evaluator, with reachability computed across `sts:AssumeRole`, `AssumeRoleWithSAML` and `AssumeRoleWithWebIdentity` — so "who can become this role" is answered by the machinery that answers "what can this role do". On top of that sits the multi-degree traversal the product is named for: from one starting ARN the assessment walks outward degree by degree, merging and deduplicating findings per parent node, and checks a cancellation flag as it goes so a long run can be abandoned rather than waited out. It ships as an SQS-driven worker and a REST API in one process, with Redis in front of S3 for results, and deploys through GitLab CI to Kubernetes with a review app per merge request.

It consumes a pre-crawled resource graph rather than calling AWS itself. The crawling is the provider work above — the same system, a different piece of it.

## How I make and document decisions

I don't have headcount authority and don't want it, so the influence I have is whatever my written reasoning earns. In practice that looks like four habits.

**Measure before committing the fleet.** The TypeScript base config decision could have been made by reading release notes. Instead I built a `tsc` dry-run harness and ran it across 24 repos, bucketing every resulting error into mechanical or genuine, which is how I found that the aggressive variant's cost fell almost entirely on 19 CommonJS-emitting repos for no real safety gain. The rejected option and the reason are written down next to the one that shipped.

**Breaking changes ship with a plan, not an announcement.** The required-UUID identity cut went out with a 592-line migration document: phase order (schema first, then a generation safety net in the data layer, then connectors in parallel), the backward-compatibility strategy for the rollout window, deterministic v5 IDs so reprocessing stays idempotent, and an explicit list of legacy services excluded with the rationale for each. The four-library consolidation had an eight-batch rollout plan across 27 consumers, ordered by dependency. The observability stack has a capacity audit with a lettered work breakdown I worked through over several weeks.

**Decisions that constrain deployment get an ADR.** Readiness gates and container start ordering, the immutability boundary for release manifests, and which repository is allowed to hold per-client desired state — all written as decision records, because the alternative is re-litigating them in review comments every quarter.

**Reviewing other people's work, at volume.** Across the fleet I've approved roughly **4,150 pull requests** — 3,702 recorded in Bitbucket merge trailers before the migration, 448 on GitHub since — on a team of under ten engineers. Only 23 of the GitHub ones were formally assigned to me, which is the part I'd point at: almost all of it is review I picked up rather than review that was routed to me. Concentrated where a bad merge is most expensive — the shared libraries, the connector base, and the CI and release repos. That is most of how the conventions in this document actually propagated; a config standard nobody enforces at review time is a suggestion.

**Adversarial review of my own branches, and the findings answered individually.** I run structured reviews against my own work before it merges, tag what comes back, and address each finding in its own commit. One of those commit bodies reads: *"An adversarial review of this branch produced 9 validated findings. Four were defects in the previous three commits, two of them putting credentials into telemetry, and the reasoning I used to justify the design was itself the bug."* Another records that my own guardrail was why I believed a false claim — the regex I wrote to enforce an invariant matched a literal expression, so a differently-named variable slipped past it.

That last habit is the one I'd point to. Reviewing your own design hard enough to find that your justification was the defect is uncomfortable and it is cheaper than the alternative.

## What I'm not claiming

The fleet's monorepo is led by a colleague; I contribute libraries and service conversions to it rather than owning its architecture. The Bitbucket-era CI repository that preceded my GitHub Actions work is also a colleague's, though the merge-worker lineage between them is visible in both repos' code. The vendor-specific integration logic in several connectors — mapping, rate limiting, retry mechanics — belongs to other engineers; my work in those repos is the modernization layer on top. And in the configuration service, the rules and rollout-evaluation engines are a colleague's; I own the consumer SDK, the tracing, and the release pipeline.

I'd rather say this up front than have someone find it in a `git shortlog`.
