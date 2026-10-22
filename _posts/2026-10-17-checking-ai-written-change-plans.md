---
layout: post
title: "Don't trust the AI's plan. Trust a checker small enough to read."
description: "AI agents can now write the plans that change production systems. planproof checks each plan with a small, rule-based checker and explains the result in plain English. What it is, why it can be trusted, the evidence, and the decades of public work it builds on."
image: /assets/img/posts/planproof-card.png
mermaid: true
redirect_from: /planproof/
---

AI agents are getting good at operations work. Give one a ticket like *"upgrade nodes n1 and n2"* and it will write a step-by-step plan to do it. Most of the time the plan is fine. Sometimes it isn't, and the mistakes are the kind that look fine at a glance.

That matters because change is where outages come from. Google's SRE book puts it bluntly: ["roughly 70% of outages are due to changes in a live system"][sre]. In 2017 a single mistyped input to a routine capacity command took [Amazon S3 in its busiest region down for hours][s3].

So who checks the AI's plan? Reading every plan line by line doesn't scale, and it's easy to miss things. Trusting the AI isn't an answer either. This post describes a third option I built and tested: **a small checker, with no AI in it, that checks every plan and explains the result in plain English.** It's called [planproof][repo]. There's also [an 11-minute video][video] of this post.

**In short:**

- **What:** an AI writes a change plan; a small rule-based checker simulates it step by step and either proves it safe, refutes it with the exact step and reason, or says it can't tell. A person reads a one-page report and approves.
- **Why trust it:** the checker is about 1,700 lines of plain code, with no AI and nothing random in it. Every rule is tested in both directions, and its predictions were compared with a real cluster.
- **The evidence:** of 25 deliberately bad plans, the checker stopped 20, and the other 5 are exactly the kind it tells the reader to check. Three AI models reviewing raw plans caught 12 of 18 bad ones; given planproof's report, all 18.
- **Not new:** every part comes from published work going back to the 1960s. What's new is putting those parts together for AI-written ops plans, and measuring the result.

**Update, 21 October 2026:** proving parts of the checker in Lean (the next step mentioned under "What it doesn't do") found two corner-case bugs in its model of Kubernetes, which no test had caught. After a drain, it could forget that a pod may still be on another node. During a rolling update, it forgot that the old pods are still running on their nodes. Each let it prove a plan that could take a deployment to zero. Both are fixed. No result in this post changed: none of the tested plans hit either case with a Proved verdict. [The details][leanfindings].

**Update, 22 October 2026:** the Lean work is finished. A copy of the checker is now proved correct for its five core rules, and the proofs found a third bug of the same kind, also fixed. [The follow-up post][leanpost] compares the proved version with the one described here.

## The idea in one picture

Think of a maths student and a strict examiner. The student (the AI) can be brilliant, creative and sometimes wrong. The examiner doesn't care how the student came up with the answer. They check each line against a fixed rulebook. You don't need to trust the student; you need to trust the examiner, and the examiner's job is small enough to check.

That's how proof assistants such as [Lean][lean] work, and it's the whole design of planproof:

```mermaid
graph LR
  T([Ticket]) --> A["AI agent<br/>writes a plan"]
  A -- "plan" --> K["Checker (the kernel)<br/>no AI, ~1,700 lines"]
  S[("Snapshot of<br/>the cluster")] --> K
  P[("Policy")] --> K
  K -- "Refuted: step + reason" --> A
  K -- "Proved + plain-English report" --> H([Person approves])
  H --> E["Executor<br/>runs only proved plans"]
  E --> C[("Cluster")]
```

1. **The AI writes a plan.** A goal ("node n1 drained and cordoned") and the steps to get there.
2. **The checker checks it** against a snapshot of the cluster and the team's policy. If something's wrong, it says which step breaks which rule, and the AI can fix the plan and try again.
3. **A person reads a one-page report** in plain English: what the plan will do, what was checked, and what was *not* checked. They answer one question: is this goal what the ticket asked for?
4. **A separate executor runs only proved plans**, and stops if the cluster has changed since the check.

## How the checker thinks

The checker plays the plan out like a board game.

**The board** is a snapshot of the cluster: which machines (nodes) exist, which copies of each service (pods) run on them, how much room each node has, and the minimum number of copies each service must keep running (its *disruption budget*).

**Each step moves pieces** by fixed rules. "Drain node n1" moves every pod off n1. Because the checker can't know exactly where the scheduler will put them, it assumes the worst case: pods that are moving count as down until a "wait" step says they're back.

**After every move, it checks the rules** (13 of them). Did any service drop below its minimum? Is there room for the moved pods? Did the plan stop new pods landing on a node before emptying it? Is the plan within the policy's limits?

**At the end, it compares the result with the goal.** The plan must change exactly what its goal says, and nothing else. This is what catches an agent that "also quietly updated the cart service".

Here's what it says about a plan that drains two nodes without waiting in between ([full report][packet]):

```text
CAUTION: DO NOT APPROVE THIS PLAN.
During step 4, no `shop/cart` pods are ready. The disruption budget for
`shop/cart` needs at least 1.
During step 4, only 1 `shop/checkout` pod is ready. The disruption budget
for `shop/checkout` needs at least 2.
```

The plan looks reasonable: it cordons both nodes, drains them and waits at the end. The problem only shows up in the *middle* of the plan, which is exactly where people and AI models both tend not to look.

When the checker can't decide, for example because an autoscaler might change the numbers under it, it says **Unknown** rather than guessing. It's built to be *sound*: it would rather refuse a safe plan than approve an unsafe one.

## Why you can trust the checker

This is the heart of it. The AI is the clever, unpredictable part, so it's kept outside. The part that decides is kept small and plain:

- **Small enough to read.** About 1,700 lines of Go, with a hard limit of 1,800 enforced on every change. A competent engineer can review all of it in an afternoon.
- **No AI and nothing hidden.** It uses only Go's standard library, with no network, clock or randomness. The same plan and snapshot always get the same answer: I re-checked all 247 stored AI runs and got the identical verdict every time.
- **Every rule tested both ways.** For each rule, a plan it must refuse and a nearby plan it must accept, so "refused" can't just mean "broken". There are also 25 deliberately bad plans, and fuzzing.
- **Checked against a real cluster.** A model of Kubernetes is only useful if it matches Kubernetes. So I ran each kind of step on a real cluster and measured the lowest number of ready pods while it happened:

| Step run on a real cluster | Predicted lowest ready pods | Real lowest ready pods |
| --- | --- | --- |
| Restart, new pod first | 4 and 4 | 4 and 4 |
| Restart, one down at a time | 4 and 3 | 4 and 3 |
| Scale down | 2 and 4 | 2 and 4 |
| Scale up | 2 and 4 | 2 and 4 |
| Cordon, drain, uncordon | 4 and 3 | 4 and 3 |

Two services were measured in each run. The real cluster never did worse than predicted, and matched the prediction exactly every time ([raw results][kindtest]).

This principle is more than 50 years old. A 1972 US Air Force study set the requirements for the part of a system that enforces security: always used, tamper-proof, and [simple enough to be analysed and tested][refmon]. Saltzer and Schroeder called it ["economy of mechanism"][saltzer]. planproof applies it to AI.

## What the person still does

The checker checks a plan against *its own goal*. It can't know whether the goal is what the ticket asked for. So the report always says so first, under "NOT CHECKED":

```text
The kernel does not check that this goal is the correct goal for the
request. You must check this.
```

That's deliberate. People tend to over-trust automated checks, experts included, and [training doesn't fix it][automationbias]. So the report states what wasn't checked as plainly as what was.

This isn't a hypothetical. In the trials, Claude Haiku 5.5 was asked to roll out an image identified by a 64-character fingerprint, and twice copied it with 65 characters. The plan was internally consistent, so the checker proved it: a safe rollout of an image nobody asked for. The grader caught it, as a careful reader of the goal would. I then tightened the checker so a malformed fingerprint is refused outright, and on the re-run the same mistake was stopped.

The report itself is written in a controlled language modelled on [ASD-STE100 Simplified Technical English][ste], the standard aircraft maintenance manuals use: a fixed word list, one meaning per word, short sentences. A linter rejects any report template that uses a word not on the list.

## The evidence

Every number here comes from scripts in the repository, and a check on every change fails if a published table drifts from what the scripts produce. That check found one mistyped number in my own write-up. The full results are in [the repo's results folder][results].

### 1. AI models writing plans

Four models wrote plans for 13 tickets: 9 that can be done safely, and 4 that can't (the policy forbids it, there's no room, or an autoscaler is in the way). A ticket passes only if the final plan is proved *and* does what the ticket asked, or, for the impossible ones, if the model says no safe plan exists.

| Model | Solvable tickets solved | Impossible tickets correctly declined | Proved plans that weren't what was asked | Cost of 39 runs |
| --- | --- | --- | --- | --- |
| Claude Opus 5.5 | 27/27 | 12/12 | 0 | $0.68 |
| Claude Sonnet 5.5 | 27/27 | 12/12 | 0 | $0.32 |
| Claude Haiku 5.5 | 25/27 | 12/12 | 0 (2 before the fix described above) | $0.04 |
| Qwen2.5 1.5B (on a laptop) | 0/9 | (declined everything) | 0 | free |

Each Claude model ran every ticket three times; the local model once.

The small local model was not a fair competitor: I used it because it fits a laptop GPU. But it shows the point. It proposed unsafe or unrelated changes on almost every ticket, such as scaling every service including the protected one, and **the checker refused every one**. The safety comes from the checker, not from the model being good. Haiku also repaired its own plan from the checker's reply in 7 of 39 runs ([full results][trials]).

### 2. Deliberately bad plans

I wrote 25 bad plans of the kinds an agent might produce: draining without cordoning, skipping a wait, breaking a limit, changing a protected service, a wrong rollback, a goal that doesn't match the steps.

```mermaid
pie showData
  title 25 deliberately bad plans
  "Stopped by the checker" : 20
  "Safe, but not what was asked: the person's check" : 5
```

None was missed. The 5 the checker proves are safe plans that do the wrong thing, like scaling to 5 when the ticket said 4. That's the one check the report tells the reader to make ([table][seeded]).

### 3. Compared with a traditional policy gate

My previous project, [safe-ops-agent-k8s][safeops], guards an AI agent with a policy gate: [Open Policy Agent][opa] rules applied to each single change. I ran both checkers on the same plans, using safe-ops' own code unchanged ([comparison][comparison]).

- **They check different things.** 13 of the 25 bad plans drain nodes or change images, which the single-change gate can't express at all. planproof checks whole multi-step plans, including the moments in between.
- **Where both apply, they agree on the hazards both model:** too many replicas, touching a protected service, taking a critical service to zero.
- **The gate allowed two plans with real problems:** scaling a service an autoscaler controls (planproof: "Unknown") and a scale-down the ticket didn't ask for. planproof proves that one too, but flags it for a person's approval.
- **It found a gap in planproof too.** planproof's policy can't say "this service may go to zero, that one never", which the gate can. It's recorded as a [known gap][gaps].
- **Neither can tell whether a goal matches the ticket.** Only a person can.

### 4. Does the report help a reviewer?

I gave 12 plans (6 good, 6 bad) to three AI models acting as reviewers, three times each. Each plan was reviewed twice, separately: once as the raw plan with the cluster's facts, and once as planproof's report. These are AI reviewers, not people: a study with people would be the real test, and I haven't run one.

| AI reviewer | Bad plans rejected, raw plan | Bad plans rejected, with the report | Good plans wrongly rejected |
| --- | --- | --- | --- |
| Claude Haiku 5.5 | 12/18 | **18/18** | 0 raw, 1 with report |
| Claude Sonnet 5.5 | 12/18 | **18/18** | 0 |
| Claude Opus 5.5 | 12/18 | **18/18** | 0 |

Reading raw plans, all three models approved the same two bad plans every time. One was the drain-without-waiting plan above: they wrote that it "respects the disruption budgets", and it doesn't. The other was the 65-character fingerprint. Both are things the checker works out mechanically and a reader has to work out in their head ([results][review]).

## What it doesn't do

- **It's a demo, not a production system.** It shows how such a check can work; it isn't meant to run real clusters.
- **The checker is only as good as its model of Kubernetes.** The model is deliberately simple and pessimistic, and it matched a real cluster in every test. Proving it in Lean has started: three of the core rules are now proved for every possible plan, and that work found the two bugs in the update above.
- **A person still checks the goal.** That's by design, and the evidence shows it's the one gap left.
- **It covers seven operations** on nodes and deployments: cordon, uncordon, drain, scale, change image, restart, wait. Anything else isn't supported.
- **The evidence is small.** 13 tickets, 25 bad plans, 12 review items, and AI reviewers rather than people.

## Built on decades of public work

None of this is my invention. planproof puts together ideas that have been published, tested and used for a long time. What's mine is applying them to AI-written ops plans, and measuring what that gives.

| Idea in planproof | Where it comes from |
| --- | --- |
| Trust a small checker, not the clever part | The LCF theorem prover ([Gordon, Milner, Wadsworth, 1979][lcf]); the "de Bruijn criterion" that a proof should be checkable by a small program ([Barendregt and Wiedijk, 2005][debruijn]); [Lean][lean] |
| Keep the checker small, always used, testable | The reference monitor ([Anderson, 1972][refmon]); economy of mechanism ([Saltzer and Schroeder, 1975][saltzer]); the verified seL4 kernel ([Klein et al., 2009][sel4]) |
| The AI ships its work; the checker verifies it cheaply | Proof-carrying code ([Necula, 1997][pcc]); certifying algorithms ([McConnell et al., 2011][certifying]) |
| Language models guess, a verifier checks, feedback goes back | LLM-Modulo frameworks ([Kambhampati et al., 2024][llmmodulo]); ReAct ([Yao et al., 2022][react]) |
| Check each step's before and after | Hoare logic ([Hoare, 1969][hoare]) |
| "Change exactly the goal, nothing else" | The frame problem ([McCarthy and Hayes, 1969][frame]) |
| Simulate on a simplified, worst-case picture | Abstract interpretation ([Cousot and Cousot, 1977][cousot]) |
| Validate a plan against a model of the world | Plan validation in AI planning, VAL ([Howey, Long and Fox, 2004][val]) |
| Check the executor's actual actions afterwards | Translation validation ([Pnueli, Siegel and Singerman, 1998][tv]) |
| A Lean model, tested against the real code (next step) | How AWS built Cedar ([2024][cedar]) |
| Never go below minimum capacity, at any moment | The [AWS S3 2017 post-mortem][s3]; Kubernetes [disruption budgets][pdb] |
| A report in controlled, plain language | [ASD-STE100][ste]; US [plain-language guidelines][plain] |
| Say plainly what wasn't checked | Research on automation bias ([Parasuraman and Manzey, 2010][automationbias]) |

Each entry, with what I took and where planproof falls short of the original, is in the repository's [prior-art page][priorart].

## What I'd take away

- **Don't make the AI trustworthy. Make it not matter.** Put a small, plain checker between the AI and production, and let the AI be as clever as it likes on the other side.
- **Check plans, not just single changes.** The dangerous moment is often in the middle of a sequence, not in any one step.
- **Make the human's job small and specific.** "Is this goal what we asked for?" is a question a person can answer well. "Is this 40-line plan safe?" isn't.
- **Look for prior art first.** Most of the hard thinking was done decades ago, in security and theorem proving. It just hasn't been pointed at AI agents much yet.

The code, specification, tests and every result are in [the repo][repo]. The [video version][video] covers the same ground, with the checker's output on screen.

---

*This post was drafted with AI (Claude) from the project's requirements, test results and sources, then edited and checked by me.*

[repo]: https://github.com/billykong/planproof
[leanfindings]: https://github.com/billykong/planproof/blob/main/docs/prd/areas/l-lean.md#findings
[leanpost]: {% post_url 2026-10-22-proving-the-plan-checker-in-lean %}
[video]: https://youtu.be/7fQ-RVWXN7E
[results]: https://github.com/billykong/planproof/tree/main/results
[trials]: https://github.com/billykong/planproof/blob/main/results/agent-trials.md
[seeded]: https://github.com/billykong/planproof/blob/main/results/seeded.md
[comparison]: https://github.com/billykong/planproof/blob/main/results/safeops-comparison.md
[review]: https://github.com/billykong/planproof/blob/main/results/review-proxy.md
[kindtest]: https://github.com/billykong/planproof/blob/main/results/kind-test.txt
[packet]: https://github.com/billykong/planproof/blob/main/examples/drain-two-nodes/packet-no-wait.txt
[gaps]: https://github.com/billykong/planproof/blob/main/docs/known-gaps.md
[priorart]: https://github.com/billykong/planproof/blob/main/docs/prior-art.md
[safeops]: {% post_url 2026-10-15-defence-in-depth-for-autonomous-infrastructure-agents %}
[sre]: https://sre.google/sre-book/introduction/
[s3]: https://aws.amazon.com/message/41926/
[lean]: https://lean-lang.org/
[refmon]: https://csrc.nist.gov/files/pubs/conference/1998/10/08/proceedings-of-the-21st-nissc-1998/final/docs/early-cs-papers/ande72.pdf
[saltzer]: https://doi.org/10.1109/PROC.1975.9939
[lcf]: https://doi.org/10.1007/3-540-09724-4
[debruijn]: https://doi.org/10.1098/rsta.2005.1650
[sel4]: https://trustworthy.systems/publications/papers/Klein_EHACDEEKNSTW_09.abstract
[pcc]: https://doi.org/10.1145/263699.263712
[certifying]: https://doi.org/10.1016/j.cosrev.2010.09.009
[llmmodulo]: https://arxiv.org/abs/2402.01817
[react]: https://arxiv.org/abs/2210.03629
[hoare]: https://doi.org/10.1145/363235.363259
[frame]: https://plato.stanford.edu/entries/frame-problem/
[cousot]: https://doi.org/10.1145/512950.512973
[val]: https://strathprints.strath.ac.uk/2550/
[tv]: https://doi.org/10.1007/BFb0054170
[cedar]: https://arxiv.org/abs/2407.01688
[pdb]: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
[ste]: https://www.asd-europe.org/standards-specifications/simplified-technical-english/
[plain]: https://centerforplainlanguage.org/learning-training/tools-training/
[automationbias]: https://doi.org/10.1177/0018720810376055
[opa]: https://www.openpolicyagent.org/docs/latest/
