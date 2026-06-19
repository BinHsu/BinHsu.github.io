---
title: "AWS Blocks is the platform team you don't have"
subtitle: "AWS shipped an infrastructure-from-code toolkit that derives your AWS resources straight from TypeScript. It's a real gift to a single-app team on AWS — and the wrong layer the moment you have a platform team or a multi-cloud stance. Here's the boundary, mapped against a platform I actually run."
date: 2026-06-19 04:00:00
tags: [aws, infrastructure-from-code, platform-engineering, cdk, architecture]
asset_path: /assets/img/posts/2026-06-19-aws-blocks-is-the-platform-team-you-dont-have
thumbnail-img: /assets/img/posts/2026-06-19-aws-blocks-is-the-platform-team-you-dont-have/two-contexts.png
share-img: /assets/img/posts/2026-06-19-aws-blocks-is-the-platform-team-you-dont-have/two-contexts.png
---

AWS shipped [Blocks](https://docs.aws.amazon.com/blocks/latest/devguide/what-is-blocks.html) —
an infrastructure-from-code toolkit. You write a TypeScript backend, instantiate a
`KVStore` here and a `Database` there, and Blocks derives the AWS infrastructure from
that code: a DynamoDB table, an Aurora cluster, the IAM to wire them, all following
AWS best practices. The whole app runs on your laptop with no AWS account, and the
same code deploys to AWS unchanged.

The first question everyone asks is the wrong one: *do I still need to write Terraform?
Does this replace my platform team?* That frames Blocks on a technical axis — IFC versus
IaC, new versus old. Wrong axis. The thing that decides whether Blocks is a gift or a
liability isn't how advanced it is. It's the shape of the team holding it. The right
question is: **do you have a platform team, and is your future single-cloud?**

![Where AWS Blocks fits: a 2x2 of org shape against cloud strategy]({{ page.asset_path }}/two-contexts.png)
*The sort in one picture. Blocks' sweet spot is the bottom-left — single-cloud, no
platform team. Every other quadrant is where one of the two boundaries bites; a
multi-cloud landing zone with a platform team sits in the opposite corner.*

## What it actually is

A **Block** is an npm package that bundles three things for one capability: the cloud
resource, the runtime API, and a local implementation. You instantiate it in a single
entry file — the IFC (infrastructure-from-code) layer — and AWS Blocks derives the
infrastructure from those instantiations. No separate IaC files.

Each Block maps to a specific AWS service:

| Block | AWS service behind it |
|---|---|
| `KVStore` / `DistributedTable` | Amazon DynamoDB |
| `Database` | Amazon Aurora Serverless v2 (Postgres) |
| `DistributedDatabase` | Amazon Aurora DSQL |
| `FileBucket` | Amazon S3 |
| `AuthCognito` | Amazon Cognito |
| `Agent` / `KnowledgeBase` | Amazon Bedrock |
| `Realtime` | Amazon API Gateway WebSocket |
| `AsyncJob` / `CronJob` | Amazon SQS / EventBridge + AWS Lambda |

The mechanism that makes one codebase work everywhere is Node.js conditional exports.
The same `import { KVStore }` resolves to a different implementation depending on context:

| Context | What `KVStore` becomes |
|---|---|
| `npm run dev` (local) | an in-memory / filesystem store, no cloud |
| CDK synthesis (deploy) | a DynamoDB table in a CloudFormation template |
| Lambda (runtime) | an AWS SDK DynamoDB client |

```ts
// aws-blocks/index.ts — the IFC layer. Illustrative, from the AWS Blocks docs.
const scope = new Scope('my-app');
const todos = new KVStore(scope, 'todos', {});   // ← this line IS the infrastructure

export const api = new ApiNamespace(scope, 'api', (context) => ({
  async createTodo(title: string) {
    await todos.set(title, { title, done: false });
  },
}));
```

That one `new KVStore(...)` line is a local store in development, a DynamoDB table at
deploy, and an SDK call in production. You never write the table, the IAM policy, or the
API wiring. Underneath it's all AWS CDK, and there's an escape hatch — a CDK layer where
you drop in raw constructs for anything without a Block. The headline features are real:
local-first development with no AWS account, end-to-end TypeScript types from the database
to the UI with no codegen, and AI steering files shipped in the npm package so a coding
agent gets the patterns right on the first try.

## Where this is genuinely good

Take the obvious case seriously, because it's strong. A solo developer or a small
full-stack team, building one product, all-in on AWS, with no one whose job is to own
infrastructure. Before Blocks, that team either hand-rolls Terraform they half-understand,
or ships click-ops they can't reproduce. Blocks hands them platform-team-quality defaults
for free: least-privilege IAM per Block, encryption at rest, secrets in Parameter Store,
APIs fronted by API Gateway — the things a good platform engineer would set up, set up
by the toolkit. Local-first means they iterate without a cloud bill. The AI steering
files mean an agent scaffolds correct code instead of plausible-looking wrong code.

For that team, Blocks isn't a lock-in trap. It's the platform engineer they were never
going to hire. That's a real category, and Blocks serves it well.

## The two boundaries

I run the other kind of system: a multi-cloud-intended landing zone with a platform
layer I own and standardize across teams. Mapping Blocks against it, two structural
boundaries show up immediately — and neither is a maturity gap that a future version
closes. They're inherent to the model.

**It's bound to AWS by construction.** Every Block *is* an AWS service. The IFC layer
derives AWS resources; the backend deploys to Lambda and API Gateway. There is no
portability seam — no point where a Block resolves to "DynamoDB on AWS, or Cosmos DB
elsewhere." That's not a missing feature; it's the design. So for a multi-cloud or
cloud-portable strategy, Blocks isn't something you "adopt one Block at a time." It's a
non-starter. The whole value comes from collapsing the abstraction onto concrete AWS
services, and that collapse is exactly what a portability strategy refuses to do.

**It collapses the layer a platform team owns.** Blocks' core promise — *you don't manage
infrastructure; we derive it following best practices* — is precisely the layer a
dedicated platform engineer exists to own. When there's a golden path, a landing zone,
multi-account governance, a shared control plane, that layer is deliberately centralized:
one place defines how every team gets a database, so the standard holds. Blocks inverts
that. Each app team derives its own infrastructure from its own code, outside the
platform's control plane. From the platform's side, that's shadow infrastructure —
resources it didn't provision, can't see in its Terraform state, and can't hold to the
standard. Blocks doesn't *compose* with a platform layer. It *bypasses* it. Used inside an
org that has one, it doesn't remove work; it fragments the standard the platform team was
built to protect — the same [lost-single-control-surface](/after-jenkins-gha-feels-fragmented/)
problem in a different layer.

## The principle

Infrastructure-from-code isn't a more advanced IaC. It's a trade. It buys velocity by
collapsing the infrastructure layer into application code — and that trade has a sign that
flips on context. When no one needs to own that layer, the collapse is pure win: less to
learn, less to maintain, best practices for free. When someone's whole job is to own that
layer — for portability, for governance, for a fleet of teams that must look the same —
the same collapse is pure cost: it scatters the thing they were centralizing.

So the axis that sorts Blocks was never technical sophistication. It's org shape and cloud
strategy. **An abstraction that hides the infrastructure layer is a gift to a team that
doesn't have to own that layer, and a liability to a team whose job is to own it.** Blocks
doesn't replace your platform team. It's what you reach for when you don't have one.

## One honest limit on this read

I evaluated Blocks the way an architect evaluates a tool that lands in a space they
already work in: I read the model end to end and mapped it against a platform I run. I
haven't deployed it to production, and this isn't a field report on its runtime behavior
— it's a read on *where it fits*, drawn from its design and my own. The fit boundary is an
architectural claim, and architectural claims are the ones most worth stating early and
correcting in public. If I run it and the boundary moves, I'll say so.

---

*This site is the lab side of my work. The polished portfolio lives at [binhsu.org](https://binhsu.org).*
