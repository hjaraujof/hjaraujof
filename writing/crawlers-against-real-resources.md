# Test the crawler against the resource, not a fixture

*August 2026*

Before the platform work I spent two years at a cloud-security startup, on [CloudGraph](https://github.com/cloudgraphdev) — an open-source GraphQL engine for cloud security posture management, whose CLI has 889 stars. I built out most of the Azure provider's resource coverage: 30-plus service crawlers across AKS, Active Directory, Event Grid and Event Hub, Data Factory, Security Center, App Service, SQL and networking. I am the second contributor to that repository overall and the first by feature commits. On the AWS provider I added crawlers for CloudFront, DynamoDB, CloudFormation, Elastic Beanstalk and VPC, and wrote the IAM policy and permissions-boundary analysis.

All of that is public and checkable, which is why I am writing about it rather than about the private service downstream of it.

A crawler in this kind of engine is a small, dull piece of code. It takes credentials, pages through one cloud service's resources, and normalizes each one into a graph entity with edges to the entities it relates to. Perhaps three hundred lines. The interesting question is how you know it is right.

## A fixture is a copy of your own belief

The cheap answer is a fixture: capture one API response, commit it, assert that the crawler turns it into the entity you expect.

That test passes forever and proves one thing — that the crawler agrees with your recollection of the API on the day you wrote it down. Every way a real response can differ from your recollection is outside the test:

- A field that is optional in a way the documentation does not stress, absent on a resource in a different tier or a different region.
- A list call that returns a summary shape while the detail call returns the full one, so the field you need is only on the second request.
- Pagination that terminates differently when the page is exactly full.
- Tags, which every cloud provider models differently from its own other services.
- An SDK minor version that changes a wrapper shape without changing the wire format.

None of those produce an error. The crawler runs, returns entities, and the entities are missing something. Which is the failure mode that matters here, because of what sits downstream: a posture-management engine's job is to answer "is anything in this account exposed". A crawler that silently drops a resource, or returns it without the attribute a rule keys on, causes a *clean report*. The customer reads an empty findings list and concludes they are safe.

I have written about the same shape in a different context — [an availability objective that could not see its two busiest services](missing-measurement-not-passing.md) and therefore reported them as flawless. A missing measurement rendering as a passing one is the general failure, and a security scanner is the worst possible place for it.

## So the resource has to be real

The practice I settled on, and the reason I am writing this down: for every AWS service I added a crawler for, I also wrote the Terraform that provisions a real instance of that service, so the crawler could be exercised against an actual resource rather than a recorded one.

The Terraform lives beside the crawler. Provision, crawl, assert the entity, destroy. When Azure or AWS changes a response shape, the run fails against reality instead of passing against a stale recording. When I had misread the pagination contract, I found out because the second page did not appear, not because a reviewer knew that service well.

It also changes what a crawler review is worth. Reviewing a crawler by reading it means checking it against the reviewer's memory of the same API — two people's recollections instead of one. Reviewing it with a provisioning plan attached means the pipeline has already checked the part neither of us can hold in our heads.

## What it costs, because it is not free

This discipline has a real bill and I would rather state it than pretend the practice is universally applicable.

A provisioned resource costs money for as long as it exists, and some services are slow. A managed Kubernetes cluster or a managed SQL instance takes minutes to create and minutes to tear down, which is a long time to hold a pipeline and a long time for a flaky create to cost you a run. Some resources cannot be created without organization-level configuration that a test account does not have. And a destroy that fails leaves a bill behind, so the teardown path needs as much care as the create path.

The consequence is that this is a per-service judgement, not a blanket rule. Cheap, fast resources get provisioned in the loop. Expensive or slow ones get provisioned deliberately, less often, and their crawlers carry more risk in exchange. Writing the Terraform is worth it either way, because the alternative to a slow real test is not a fast real test — it is a fixture that agrees with you.

## The separation that made the analysis testable

One more piece of the architecture matters here, and it is the reason the split was worth maintaining. Every crawler in both providers sits behind one crawler and SDK abstraction, so the resource-graph shape and the crawler contract are the same whichever cloud you are pointed at. Thirty-plus Azure services and the AWS services I added all produce entities of the same kind.

Downstream of that graph sat the private analysis service I worked on at the same company: it consumes a pre-crawled resource graph rather than calling AWS itself. That is a deliberate boundary. The crawl is where reality — API drift, partial responses, throttling, credentials — enters the system, and it is inherently non-deterministic. The analysis on top of it is pure evaluation over a fixed graph, so it can be tested exhaustively with no cloud account involved at all.

Put another way: the non-determinism is concentrated in the layer that has real resources to test against, and excluded from the layer that has none. The crawler needs a live cloud to be believed, and the evaluator needs nothing but the graph.

If I were starting a crawler-based system now, the provisioning plan would be a requirement of the first crawler rather than a habit that arrived with the sixth — and I would still make the per-service call about which ones run on every pipeline, because the ones that are too slow to run often are usually the ones whose API you understand least.
