# When your safety check is the thing that's broken

I maintain a shared devtooling package that about thirty repositories extend for their ESLint, TypeScript, and test configuration. It also ships two small command-line validators that run in CI:

- one fails the build if any internal dependency is declared with a caret or tilde instead of an exact version,
- one fails the build if a test-helper package's peer dependency has drifted from the schema package it was built against.

Both are the kind of tool you write once, wire into a pipeline, and stop thinking about. Which is the problem, because for some period of time — I can't say exactly how long — **both of them exited 0 having validated nothing.**

## The bug

The validators are ESM. In CommonJS, the idiom for "run `main()` only when this file is executed directly, not imported" is:

```js
if (require.main === module) main()
```

ESM has no `require.main`, and the widely-cited replacement compares the process's entry path against the module's own URL:

```js
if (process.argv[1] === fileURLToPath(import.meta.url)) main()
```

That works when you run `node ./dist/validate.js`. It does not work when you run the tool the way a real project runs it — by its bin name, through the symlink or shim that a package manager installs into `node_modules/.bin`.

The reason is a detail I had never had cause to care about: **`process.argv[1]` is not resolved through symlinks, and `import.meta.url` is.** npm installs a bin as a symlink; pnpm installs a small shell shim. Either way, Node receives the un-realpathed path in `argv[1]` while `import.meta.url` reports the realpathed one. The two strings never match. The guard's condition is always false. `main()` never runs.

And a validator that never runs is a validator that never fails. Exit code 0. Green check. A pipeline step whose entire purpose was to block a class of bad dependency declaration, reporting success while inspecting nothing.

The fix is one line:

```js
if (fs.realpathSync(process.argv[1]) === fileURLToPath(import.meta.url)) main()
```

## Why no test caught it

I had unit tests. None of them could have caught this.

A test that imports the validator module and calls its exported functions exercises all the logic and passes. A test that runs `node dist/validate.js` against a fixture also passes, because that invocation happens to satisfy the un-realpathed comparison. The bug only exists in the layer between the package manager and Node — the symlink or shim that a real consumer's CI actually traverses, and that no ordinary test goes near.

So the regression test I wrote doesn't import anything. It builds the package, installs it into throwaway fixture projects under **both** an npm-symlink layout and a pnpm-shim layout, invokes the binary **by name**, and asserts it fails on a deliberately bad manifest. It is slower and uglier than a unit test. It is also the only kind of test that can observe the failure.

Testing the module was testing a different program than the one consumers run. If a tool's correctness depends on how it gets invoked, the test has to invoke it that way.

## The second one

A few months earlier I deleted a different check for a related reason.

The same fleet publishes about a dozen internal packages, each versioned by CI: merging triggers a pipeline that computes the next version, publishes it, and writes its own bump commit. Consumers then re-pin to whatever landed. Because publishing is asynchronous, the release tooling had a step called something like `verifyRegistryPropagation`, which compared the local `package.json` version against what the registry served, and warned or failed if they disagreed.

It was checking the wrong side of the transaction. The publish *pipeline* owns version mutation — it suffixes prereleases, it bumps remotely in parallel mode. The local manifest is not the authority and was never expected to match. So the check produced false failures when everything was fine, and told dependent repositories nothing about whether a parent's publish had actually succeeded — the question it appeared to answer.

I removed it, 107 lines, and replaced it with something that tracks the publish pipeline's own terminal result, correlated by the commit SHA on the destination branch. Failures, timeouts, and "no pipeline found" now propagate into the gate that decides whether a dependent may proceed.

Both cost the same thing, which is not damage — it is the trust you spend on a green tick. People merge on that tick.

## What changed after

Two concrete practices, and one I am still arguing with myself about.

Every check I add to CI now ships with a fixture it must reject, and I watch it reject that fixture before I believe the check exists. One config-validation workflow does this in the open: the job runs a deliberately invalid file through the validator and asserts a non-zero exit, immediately beside the real invocation. It costs about four lines. It is the only evidence that a validator is a validator rather than a step that prints something.

The second is a question I ask before writing a check at all: what is this reading, and is that thing authoritative for the claim? `verifyRegistryPropagation` fails that question on inspection — nothing about a local manifest testifies to a remote publish. I did not ask it at the time.

The unresolved one is scope. The peer-lockstep validator exists only because the version validator inspected `dependencies` and was silently blind to peer and dev pins, so a helper package built against one schema version and resolved against another passed clean. Splitting them made each fail loudly on a narrow condition, which I think is right. It also means there are now two bins to keep alive, two entry-point guards, and the same realpath bug latent in both until I fixed them together. I do not have a principle that resolves that; I have two validators and a test that spawns both.

Current state: both bins have the realpath guard, the fixture-spawning suite covers npm-symlink and pnpm-shim layouts, and the suite runs in CI on the package itself. Neither validator has silently passed since.
