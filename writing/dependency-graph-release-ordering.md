# Publishing in dependency order

*August 2026*

Here is a topology that sounds reasonable and is a trap.

You have a fleet of TypeScript repositories — call it fifty — and about a dozen of them are internal libraries published to a private registry. Every library is versioned by CI: merge to the release branch, and a pipeline computes the next version, publishes it, and writes its own version-bump commit. No human touches a `version` field. Consumers depend on exact pins and get re-pinned when a new build lands.

Each piece of that is a good idea. CI-owned versions stop two people racing the same number. Exact pins stop a floating range from changing your build overnight. Publish-on-merge keeps latency low.

Together they create a hard ordering constraint that nothing in the setup enforces: **when a base library and its consumers change in the same batch of work, the base must merge, publish, and become resolvable in the registry before a consumer's install runs.** Violate that and you get one of three outcomes, none of which announce themselves clearly:

1. The consumer freezes a stale pin and silently ships against the old behaviour.
2. The consumer's tree resolves *two* versions of the same package — say a `dev` and a `qa` build — and you get a type error that reads like nonsense, or a runtime bug that depends on which copy a given import path reached.
3. The frozen-lockfile install fails outright, which is the best case, because at least it's loud.

Doing this by hand means one person holding a mental topological sort of a fifty-repo graph and clicking merge buttons in the right order for an hour. That works until it doesn't.

## The machine

What I built is, at its core, a topological sort with a CI dispatcher bolted to it.

A declared config describes which internal packages depend on which. From that, wave partitioning (Kahn's algorithm) produces batches: everything in wave *n* depends only on things in waves before it, so a wave's members can merge in parallel. Kahn's leaves any cycle behind as nodes with nonzero in-degree, so detecting that one exists is free; a separate depth-first pass walks those leftovers to report the exact cycle path instead of just its existence, which is the difference between a usable error and "there is a cycle somewhere in 50 repos." A dependency cycle in a publish graph is a design defect to report and stop on, not something to resolve at runtime.

Each wave dispatches its repositories as remote CI runs and waits. A **dependent gate** sits between waves: if a parent's publish failed, or was skipped, its dependents don't proceed. They block, skip, or prompt depending on configuration, but what they don't do is merge on top of a library version that doesn't exist yet.

The version computation underneath is deliberately hand-built rather than delegated to `release-please` or `changesets`: branch-aware semver and prerelease derivation, atomic branch-plus-tag pushes with retry on collision, and a schema validator for the release config so a malformed file fails at PR time rather than mid-release.

None of that is novel computer science. Kahn's algorithm is from 1962. The engineering is in what happens when the graph misbehaves.

## What the graph did instead

Every one of these was a production incident, and every one produced a guard rather than a patch.

**A stale tag silently downgraded a consumer.** The repin tool asked the registry what a dist-tag currently resolved to and wrote that into the manifest. A tag pointing behind a consumer's existing pin — because of a rollback, or a channel that hadn't caught up — regressed a library from a 2.x line back to 1.x. It shipped. Nobody noticed until someone read a diff.

The tool now refuses any resolution lower than the current pin, and compares with a real semver-precedence comparator rather than `sort -V`. That detail matters: `sort -V` orders prereleases wrong, so a naive version sort will happily tell you `1.0.0-dev.2` is newer than `1.0.0`. The gated path exits non-zero rather than warning, because a warning in a pipeline nobody is watching is a no-op.

**A postinstall hook corrupted a multi-workspace update.** The repin ran `pnpm add` per package, which fires lifecycle scripts. Under `set -e`, a failing root postinstall aborted the script before it reached the nested-workspace loop — so the root manifest advanced while nested manifests kept stale pins. That's how a tree ends up resolving a `dev` and a `qa` build of the same package simultaneously, which surfaced as a type error that looked like a compiler bug.

The rewrite edits every manifest directly with `jq` — no install, no lifecycle scripts — then runs a single lockfile-only, script-free install at the end. Separating "change the intent" from "realize the intent" removed the whole class.

**A nested lockfile nobody knew about.** A frozen-lockfile install kept failing in CI while the manifest looked right. The repo had a standalone package with its own `package.json` *and its own lockfile*, outside the workspace globs. The sync step only ever regenerated the root lockfile. Now every lockfile in the tree is discovered and synced.

**A repo that cascaded forever without moving.** One package was detected as changed on every run and duly triggered a re-merge for all seventeen of its dependents — but it had no merge worker wired on its own branch, so it could never merge itself. Seventeen repositories churned every run and the subtree never converged. The change-detection logic and the participation config had drifted apart, and nothing reconciled them.

**An inverted boolean.** The follow-up publish dispatch passed `build_only: true` to consumers' pipelines. That flag means *skip version and publish*. So merges landed, publishes never fired, and the child version-bump chain quietly stopped fleet-wide. One word, and the entire point of the orchestration was disabled while every step reported success.

**A check that was checking the wrong thing.** There was a step comparing the local `package.json` version against the registry, meant to confirm a publish had propagated. But the publish pipeline owns version mutation — it suffixes prereleases and bumps remotely — so the local manifest was never expected to match. It produced false failures during healthy releases and told dependents nothing about whether the parent had actually published. I deleted it and replaced it with SHA-correlated tracking of the pipeline's own terminal state. ([More on that pattern.](false-green-gates.md))

## The other half: resolution mechanics

Ordering is only half the problem. The other half is that the same fleet migrated package managers repo-by-repo over about a year, from a flat hoisted `node_modules` to a strict symlinked store. A flat tree is forgiving in a specific and dangerous way: it exposes your transitive dependencies as though you had declared them, and it collapses what should be two copies of a package into one by accident. A strict store stops doing you those favours, and every one becomes a hard failure at the moment a repo switches.

- A shared library imported an OpenTelemetry package in its source while declaring it neither a dependency nor a peer. It worked because consumers happened to hoist it. Declaring it wasn't enough — I pinned it to the same range a sibling library uses, *specifically so the resolver dedupes to a single copy*, because that package holds a global provider registry. Two copies don't crash; they silently break trace correlation across every service.
- A repository class arrived at one service by two paths: imported directly, and inherited through a base class that resolved its own copy. Two instances of the same package, so TypeScript's private-field nominal check reported two incompatible types with identical names.
- An ORM's generated client landed somewhere nothing read from, because a custom generator output pointed at the hoisted directory while the client re-exports from its own nested stub.
- One repo had a committed, tokenless registry config file shadowing the one CI's login writes — a 401 in exactly one service while four siblings built fine. The fix was to untrack it rather than delete it, so local development kept working and CI's file was the only one in play.

Each was diagnosed to the specific resolution behaviour responsible rather than papered over with broader hoisting, which is the tempting move and the one that hides the next instance.

## What I got wrong

The dependency graph is a hand-maintained config file, and the phantom-cascade incident happened exactly where that file and reality disagreed — a repo listed as a graph participant that structurally could not merge. Deriving the graph from the manifests would have made that state unrepresentable. I did not do it because the config predated the orchestrator and I built on what was there. It is still a config file.

The guards I would keep. Publishing is irreversible, so a repin nobody can un-publish is a worse outcome than a pipeline that refuses to proceed; that is why every guard fails closed, and the one resolver I let simply take the tag's answer silently downgraded a production dependency.

What I did not predict is the ratio. The topological sort took an afternoon — Kahn's algorithm is from 1962 and the wave partitioner is about eighty lines. An inverted boolean, a lockfile sitting outside the workspace globs, and a symmetric diff that could not distinguish "destination is legitimately ahead" from "merge residue" cost weeks between them, spread across incidents months apart. Every one of those sat in a seam between two systems that were each behaving correctly on their own terms. I still have no way to find that class before it fires, and that is the part of this I would most like to solve.
