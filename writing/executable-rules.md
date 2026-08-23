# The rule needed to be executable

*August 2026*

The service handles passwords, client secrets, refresh tokens and OAuth codes. I wrote its original auth middleware — token validation, refresh flow, cross-subdomain cookies, sign-out — about two weeks after the repository's first commit, and I still maintain that file. The Ed25519 signing keyring and the rotation core in the same service are a colleague's work, not mine.

Adding distributed tracing to it was the ordinary kind of task. One decorator on a class, and every method in it produces a span with timing and a parent link. The same decorator, with one more option, also records the arguments each method was called with.

That option is the entire problem. Method arguments in this service are the passwords, the client secrets, the refresh tokens and the OAuth codes. Turning it on anywhere is a credential-exfiltration path into a trace backend that has a completely different access model from the database — longer retention, broader read access, and a UI that is designed to show you request payloads.

So the rule is: trace everything, export no arguments. That rule was easy to hold for the eleven classes I instrumented in one afternoon. It is not holdable at all six months later, when someone adds the twelfth class, copies the decorator from the eleventh, and has never heard of the rule.

## A rule that a person has to remember is a rule that expires

The usual answer is a code-review convention and a line in a contributing guide. I have written both, and neither survives the departure of the person who wrote them, because a convention is only enforced by whoever happens to remember it while reading a diff.

So the rule became a test. It enumerates the traced classes in the service and fails the build when one of them captures argument values without an individual security review recorded for it. Adding the twelfth class does not require the author to know anything: the build tells them there is a decision to make, and the decision has to be written down before the pipeline goes green.

The property that makes this work is that the test does not check for a mistake. It checks for an unreviewed state. A developer who needs argument capture on one method can have it — they take the review, the review is recorded, the build passes. Nobody gets it by accident, and nobody gets it because the reviewer was tired.

Two live leaks turned up while I was wiring it: an OAuth state nonce, and exception messages, both reaching the trace backend. The nonce is the more interesting one — nothing about it looks like a credential, it is short, opaque and single-use, and it is exactly the value an attacker needs to complete a cross-site request forgery against the authorization flow.

## And then the guardrail lied to me

The other half of this is a different service and a worse story.

I run structured adversarial reviews against my own branches before they merge. One of them produced nine validated findings. Four were defects in my own previous three commits, two of them putting credentials into telemetry, and the reasoning I had used to justify the design was itself the bug.

One finding was about a rule I had already made executable. I had written an invariant to enforce that error-response bodies are excerpted with a bound before they reach a log or a span, because an unbounded body interpolation is how a proxy's HTML error page ends up rendered on a login screen — and how a response containing something sensitive ends up in a log line. The invariant was a regex over the source, and it matched a literal expression. A call site that assigned the response to a differently-named variable and interpolated that instead sailed straight past it.

The finding that mattered was not the unbounded interpolation. It was that I had believed the code was clean *because my own check said so*. The guardrail was the reason I stopped looking.

A second finding in the same review had the same shape from the other direction. I had inspected one branch, declared it safe, and recorded that as a deliberate non-finding — while the diff had changed the sibling branch, which had no error handling at all. My note said "checked and fine" about a line the change had not touched.

## Why one of them could be trusted

The tracing test and the excerpt regex look like the same kind of artifact. They are not, and the difference is worth naming because it decides whether a guardrail can be trusted.

The tracing test enumerates a set: which classes carry the decorator, and which of those have a recorded review. Both halves are facts about the program, and a new class cannot be invisible to the enumeration — it either has the decorator or it does not.

The excerpt regex matched text. It was a lexical check wearing a semantic hat, and its coverage was the set of spellings I had thought of while writing it. Every call site the author spells differently is outside the check, and the check cannot tell you that: it silently reports zero violations either way. Its silence carried no information, and I read the silence as a pass.

That is the same failure I have written about in [a validator that exited 0 having validated nothing](false-green-gates.md), arriving by a different route. There the check never ran. Here it ran and inspected a smaller program than the one I thought it was inspecting. Both produce a green tick that a reasonable person merges on.

The practice that came out of it: when a check is lexical and cannot enumerate its own subject, it ships with a fixture it must reject — a call site spelled a different way, asserted to fail — and I watch it fail before I believe the check exists. If a check cannot be made to enumerate and cannot be made to fail on demand, it is a linter hint, and it does not get to be the reason I stop reading a diff.

Current state: the tracing policy is enforced by the structural test, argument capture is off everywhere in that service except where a recorded review says otherwise, and the nonce and the exception messages no longer reach the backend. The excerpt rule now has a bounded-excerpt helper at every call site I converted — 24 of them — with an error type whose classification survives a serialization boundary where `instanceof` does not. The regex is still a regex. It has a fixture it must reject, and it is no longer the only thing I rely on, which is the most I can honestly claim for it.
