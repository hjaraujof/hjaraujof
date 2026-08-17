# Harold Araujo — engineering detail

The long version of my [profile](README.md), for people who want specifics. No proprietary logic appears here — everything is described at the level of engineering mechanics, which is the level that transfers anyway.

Where work was shared, I say so. Commit counts are a weak signal and I have tried not to lean on them. And where I say "sole author" or "primary maintainer," you cannot verify that from this page — a reference or a future manager can, and I would rather state the limit than imply a check you can't actually run.

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

**Test framework migration.** The shared Vitest config landed six weeks *before* the first repo migrated — library first, consumers after. I then ran the first wave of Jest→Vitest migrations myself across the shared libraries and core connectors, 11 of roughly 24 repos, after which four other engineers picked up the pattern independently for the rest. Notable failures along the way:

- Vitest 4 began calling `new` on mocked constructors, breaking every arrow-factory `mockImplementation`. The obvious fix — a plain function expression — wasn't durable, because the shared ESLint config's own `prefer-arrow-callback` autofix would convert it straight back to a broken arrow. Class expressions satisfy constructability and are immune to that autofix. ~175 tests recovered across three repos.
- A mock-hoisting behaviour drifted between patch releases: passing on 4.1.3, failing on CI's 4.1.5. `vi.spyOn` is version-stable; the factory pattern wasn't.
- A leaked fake timer under a shared worker froze `setTimeout` and produced 40 cascading CI failures while all 454 tests passed locally. Fixed the leak at its source, then removed the flag that allowed cross-file state to leak at all.

## Observability platform

**The instrumentation library** (sole author, 18 consuming services): decorator-driven auto-instrumentation, OTLP protobuf export, context propagation across worker and queue boundaries, and `ParentBased` samplers after I traced a class of orphaned cross-service traces to sampling configuration. Related defects I root-caused rather than upgraded past:

- Trace context lost across SQS hops that the AWS SDK's auto-instrumentation declines to bridge — injecting only where it declines, so publish-span parentage survives.
- 1.x samplers being handed to a 2.x SDK, causing real trace loss.
- A module-scope logger construction that froze the "OTLP logs enabled" decision on a first-import-wins basis under ESM. It was dormant only because no endpoint variable was provisioned yet, and would have gone live the moment a rollout added one.

**The backend** (sole author, no other contributors): Loki, Grafana, Tempo, Mimir, and later Pyroscope behind an OpenTelemetry Collector, Terraform-provisioned on AWS with S3 as long-term storage for each backend, a private DNS zone for internal OTLP, NGINX with Let's Encrypt, and IAM-scoped collector access. I costed it at roughly **$120–130/month against an estimated $1,000–3,000/month** for equivalent commercial coverage at 20–50 hosts.

Operational reality, which is the part cost comparisons usually skip: a same-day root-cause-and-remediation cycle for a disk-full incident that moved trace and metric storage to object storage, added tail sampling and span-name normalization to stop label-length rejections, pinned the AMI to prevent a destructive instance replacement, and closed SSH in favour of session-manager access. Separately: a span-metrics resource-attribute collapse that was **discarding 77.5% of metric samples**, a multi-tenancy misconfiguration routing all logs into an unflushable placeholder tenant, and an ingestion limit where the burst — not the rate — was the binding constraint.

I also found and fixed a live secret exposure here: an SSH private key landing in instance boot logs. The remediation moved the trust boundary from "clone with an embedded deploy key" to "IAM-scoped object read," which removed a plan-time secret read entirely.

→ Full write-up: [Self-hosting observability, honestly](writing/self-hosted-observability.md)

## Identity & multi-tenant data isolation

I joined the central OAuth 2.0 service about two weeks after its first commit, wrote its original auth middleware — token validation, refresh flow, cross-subdomain cookies, sign-out — and remain that file's primary maintainer years later. I'm the second contributor over the repo's life and the leading one this year. The Ed25519 signing keyring and rotation core are a colleague's work, not mine.

Work I'd point to:

- **A tracing policy enforced by a test.** After wiring OpenTelemetry into a service handling passwords, client secrets, refresh tokens, and OAuth codes, I wrote a structural test that fails the build if any newly-traced class captures argument values without individual security review — plus fixes that stopped an OAuth state nonce and exception messages from reaching the trace backend.
- **A Redis wedge.** A task deployed without its Redis host variable silently fell back to localhost; every connection was refused and an auth endpoint hung about 60 seconds until the load balancer cut it. Fixed with a bounded reconnect strategy and a fail-closed boot assertion, then generalized into a shared timeout primitive replacing ad-hoc `Promise.race` across every Redis-touching route.
- **A defect class, not a bug.** Proxy HTML error pages were breaking JSON parsing and rendering raw error text onto the login screen. I'd seen the same class fixed in two sibling services; this one had no shared client to port, so I built the parsing layer fresh across 24 call sites — including an error type whose classification survives serialization boundaries where `instanceof` doesn't, and a bounded diagnostic excerpt that never reaches the browser. Two latent bugs surfaced during the conversion: a form parsing a body before checking status, and a client treating an unparseable 200 as confirmed success.
- **Tenant isolation at the database.** Most recently, Postgres row-level security: a non-superuser role that cannot bypass RLS, an ORM extension scoping every tenant query through per-transaction settings, an additive migration backfilling the tenant column under forced policies, and a CI gate that proves isolation against a live database rather than trusting application code. Currently on a branch, not yet merged.

## Distributed document processing

**The shared batch state machine** every connector depends on for document processing under a distributed lease. I authored its ownership protocol and its idle-cost work: claim-and-lease ownership, heartbeat renewal that only renews still-in-flight documents, adaptive idle backoff that self-tunes polling cadence, terminal-versus-retryable error classification, bounded retry and bounded memory. Adjacent work on the same class — busy-tick floor, run-loop jitter — is another engineer's.

**The bulk-import sidecar** (founder, dominant author): an authenticated Socket.IO channel, a CPU-sized worker-thread pool so long spreadsheet parses never block the event loop, and about a dozen document-type processors on one generic pipeline resolved through DI factories. The interesting parts are the failure-ordering invariants:

- A mixed batch returns *partially completed* with a generated failed-rows workbook, rather than failing whole — because a forced re-upload would duplicate the rows that already landed.
- Saved document IDs are captured before the terminal status write, so a failure there can't erase the record of what actually persisted.
- A non-critical diagnostic archive write is isolated so its failure can't overwrite an already-completed record with a failure.

Bridging trace context and cached secrets across the worker-thread boundary needed explicit carriers, since `AsyncLocalStorage` doesn't survive a structured-clone boundary.

**A fan-out router** in the same fleet: sibling services discovered at runtime through service discovery, each document republished to every sibling's queue, and a document counted as failed only when *all* queues fail — reaching four of five is recorded as success. Two details I'm fond of: distinguishing an API response with an *absent* collection key (transient, don't cache, retry) from a genuinely *empty* one (cache normally); and deliberately classifying one documented-absent config as expected steady state rather than an error span, because recording it as an error would poison the collector's keep-all-errors sampling policy on every boot.

## Product & integration engineering

Before the platform work, and still alongside it.

**A customer-facing Next.js portal** (largest single repo by my commit count): I designed and built its document-upload subsystem end to end — abstract validator, controller, and service base classes extended by concrete flows, S3 upload, async job validation, and real-time progress over a Socket.IO event channel on a custom server. I also migrated its authentication from a hand-rolled token scheme to a standard library and built the permission layer over it (cached permission fetch held in session state, guard-based route authorization), and later instrumented a standalone build — which required fixing Next.js output-file tracing that was silently dropping the logging transport chain and cloud SDK from the deployable, a bug that would have shipped a dead production logger.

**The connector platform** — roughly fourteen services integrating NetSuite, QuickBooks Online, Shopify, Teamwork, Yotpo, SFTP batch partners and a logistics provider, on one shared base library.

My deepest work here is the Teamwork integration, the largest of them: per-document-type state machines and mappers for purchase orders, catalog styles, sales orders, employees and stock adjustments, each with an explicit idempotency key threaded through the pipeline, plus a later gen2 rewrite alongside the original. Those retry, dedup and continuation patterns were proven there first and then generalized into the base library every connector now extends — which is the sequence I'd defend as the right one: solve it concretely in the hardest case, then lift it.

Two pieces of the surrounding contract layer I'd point to. A **schema registry**: each repo declares its types as Zod schemas, vendor-namespaced, published to a runtime configuration service — so every consumer gets a generated, versioned, runtime-validated contract instead of an interface someone hand-copied and forgot to update. And a **drift-guarded fake API**, whose route manifest is generated from the live controllers rather than maintained by hand, so a consumer's tests cannot quietly pass against a surface that no longer exists.

**Inside the monorepo:** I built the pipeline that imported standalone repositories into the Nx workspace with git history preserved, applying each one's config and dependency changes ahead of the import. I also own the shared SQS queue library there (tenant-scoped consumers, quarantine and retry handling, FIFO support) and the S3-backed storage service, both consumed fleet-wide. This is where most of my last six months has gone.

**Cross-cutting migrations** I led: a required-UUID identity field added to every operational document — a breaking change across 12+ services, sequenced schema-first with generation as a safety net during rollout, deterministic v5 IDs so reprocessing stays idempotent, and legacy services excluded with written rationale; four overlapping libraries consolidated into one namespaced package with subpath exports, migrated across 27 consumers in dependency-ordered batches; and a structured-logging replacement with a wide-event child-logger API, jittered retry, transport health checks, and a stderr fallback.

## How I make and document decisions

I don't have headcount authority and don't want it, so the influence I have is whatever my written reasoning earns. In practice that looks like four habits.

**Measure before committing the fleet.** The TypeScript base config decision could have been made by reading release notes. Instead I built a `tsc` dry-run harness and ran it across 24 repos, bucketing every resulting error into mechanical or genuine, which is how I found that the aggressive variant's cost fell almost entirely on 19 CommonJS-emitting repos for no real safety gain. The rejected option and the reason are written down next to the one that shipped.

**Breaking changes ship with a plan, not an announcement.** The required-UUID identity cut went out with a 592-line migration document: phase order (schema first, then a generation safety net in the data layer, then connectors in parallel), the backward-compatibility strategy for the rollout window, deterministic v5 IDs so reprocessing stays idempotent, and an explicit list of legacy services excluded with the rationale for each. The four-library consolidation had an eight-batch rollout plan across 27 consumers, ordered by dependency. The observability stack has a capacity audit with a lettered work breakdown I worked through over several weeks.

**Decisions that constrain deployment get an ADR.** Readiness gates and container start ordering, the immutability boundary for release manifests, and which repository is allowed to hold per-client desired state — all written as decision records, because the alternative is re-litigating them in review comments every quarter.

**Adversarial review of my own branches, and the findings answered individually.** I run structured reviews against my own work before it merges, tag what comes back, and address each finding in its own commit. One of those commit bodies reads: *"An adversarial review of this branch produced 9 validated findings. Four were defects in the previous three commits, two of them putting credentials into telemetry, and the reasoning I used to justify the design was itself the bug."* Another records that my own guardrail was why I believed a false claim — the regex I wrote to enforce an invariant matched a literal expression, so a differently-named variable slipped past it.

That last habit is the one I'd point to. Reviewing your own design hard enough to find that your justification was the defect is uncomfortable and it is cheaper than the alternative.

## What I'm not claiming

The fleet's monorepo is led by a colleague; I contribute libraries and service conversions to it rather than owning its architecture. The Bitbucket-era CI repository that preceded my GitHub Actions work is also a colleague's, though the merge-worker lineage between them is visible in both repos' code. The vendor-specific integration logic in several connectors — mapping, rate limiting, retry mechanics — belongs to other engineers; my work in those repos is the modernization layer on top. And in the configuration service, the rules and rollout-evaluation engines are a colleague's; I own the consumer SDK, the tracing, and the release pipeline.

I'd rather say this up front than have someone find it in a `git shortlog`.
