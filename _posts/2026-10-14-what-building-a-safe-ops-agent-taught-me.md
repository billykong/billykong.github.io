---
layout: post
title: "Eight things building a safe AI ops agent taught me"
description: "I built an AI agent that operates Kubernetes but can't change anything on its own, and logged about 150 findings along the way. Here are the lessons the design didn't predict."
---

Over the last few weeks I built [safe-ops-agent-k8s][repo]: a reference implementation of an AI agent that operates a Kubernetes cluster without being trusted. It investigates incidents, runs code it writes, reads tools, and proposes changes. It can't change anything itself.

The design rests on one idea: **treat the model's output as untrusted, and enforce every control somewhere the agent can't reach.** So the agent runs generated code in a disposable gVisor sandbox with no network and no credentials. It can only *propose* a change, as a `ChangeRequest` record; a small controller with no AI in it computes the facts (dry run, diff, blast radius, rollback check), a policy in OPA decides allow, escalate or deny, and a human approves anything that matters. Gateways hold every API key and tool secret. A kill switch stops everything in about three seconds. All of it runs on a laptop, on kind inside WSL2, and every claim in the [requirements][prd] names the test that proves it.

That's the design. This post is about what happened when I built it. Each project keeps a findings log, and by the end there were about 150 entries: things that surprised me, broke, or passed for the wrong reason. Most of them say one of eight things.

## 1. Controls fail silently, and look healthy while they do

The most common finding was a control that appeared to be on and wasn't.

- I applied default-deny network policies on day one, and they were accepted without complaint. The cluster's default network plugin then failed every rule sync, because the stock WSL2 kernel lacks a module it needs, so **all traffic flowed**. Nothing reported an error.
- After moving to Cilium, the sandbox controller turned out to generate its own policy for each sandbox template, allowing egress to the whole internet. Policies are additive, so it quietly overrode mine. Hubble showed it: SYN packets to the internet `FORWARDED`.
- A check that sandboxes couldn't reach cluster DNS passed. It passed for the wrong reason: gVisor's network stack bypassed Cilium's service translation entirely, so the packets were classed as internet traffic and dropped. The rule I thought I was testing never matched anything.
- A Pod Security test that "a privileged pod is denied" passed because ordinary validation rejected the pod first. Pod Security never saw it.

**The fix is a habit:** pair every "this is blocked" check with "a similar allowed thing works", and assert *which layer* refused. Without the pair, "blocked" can just mean "the network is broken". ([P1 findings][p1])

## 2. Defaults are wider than the job

Almost every component came with defaults broader than I needed, and none of them said so.

- Service-account tokens on this cluster were valid for **a year**. A copy of the agent's token would have kept working long after anyone noticed. Projecting tokens explicitly with a one-hour lifetime fixed it.
- Kata Containers, out of the box, turned off syscall filtering inside the guest, let a pod annotation change the guest kernel's command line, and ran the VM outside every host resource limit, with no memory, CPU or swap cap at all.
- The Gateway API controller I added for model serving exposed a port on the node, and reads every Secret in the cluster. The autoscaler's chart could scale any resource and also read every Secret.

"Installed with the official chart" is not a security posture. Every add-on deserves its own review of what it's allowed to do. ([P1][p1], [P3][p3], [P5][p5] findings)

## 3. Kubernetes' own guards have blind spots

Admission policies were the backbone of the approval gate, and they have limits worth knowing before you rely on them.

- **They can't protect themselves.** Kubernetes never runs admission policies on admission configuration, so anything allowed to edit policies can delete the guard. That one can only be held by RBAC hygiene, tested on every run, plus an alert on any change.
- **They can't see reads.** A stolen copy of the agent's token kept *listing* proposals after the kill switch said stop. What ended it was recreating the service account: tokens carry the account's UID, so the API server refuses them once the UID is gone.
- **They have no clock.** An approval can't be expired in admission; the controller has to judge expiry itself.
- **RBAC can't split fields inside `status`.** Anyone allowed to approve could also have rewritten the plan the controller applies, until a policy limited each field to its owner.
- **Fail-closed cuts both ways.** One policy read an optional request field directly. When the field was absent, the expression errored, and `failurePolicy: Fail` turned that into denying every legitimate request. It would have taken the agent down on first use.

Test the *allowed* path of a fail-closed control as hard as the denied one. ([P2 findings][p2])

## 4. Isolation technology has sharp edges

The sandbox is the layer people trust most, so I attacked it on purpose: fork bombs, memory bombs, persistence, credential hunts, DNS exfiltration.

- **gVisor ignores the pod's process limit.** Guest processes are goroutines in gVisor's kernel, not host processes: during a fork bomb the host saw 14 of 512 pids used, and the sandbox crashed anyway. The fix was an `RLIMIT_NPROC` set by the executor, sized to fit the memory budget.
- **Host swap made memory limits soft.** A pod limited to 256 MiB wrote 1 GiB without an error. (A testing trap on the way: allocating a `bytearray` doesn't use memory until you write to it, so "1 GiB allocated" proves nothing.)
- **PID 1 is special.** A process running as PID 1 gets no default signal handling and doesn't reap zombies. This bit three separate times: the sandbox's executor, the agent (which ignored SIGTERM and leaked its sandbox), and the workflow worker.
- gVisor's apparent CPU overhead on a pure-Python loop turned out to be a Spectre mitigation the host applied to some processes and not others.

None of these came from reading documentation. They came from attack demos and benchmarks. ([P1 findings][p1])

## 5. Data leaves through more than the network

I expected exfiltration to be a firewall problem. Mostly it wasn't.

- **A DNS allowlist allows IP addresses.** Sandboxes that may install packages can resolve only PyPI's names. But PyPI sits on a CDN shared with thousands of other sites, and through PyPI's address a sandbox reached `www.python.org`, `www.fastly.com` and a news site's origin. Pinning the TLS server name fixed that. Domain fronting still works, because the CDN routes on an HTTP header inside the encrypted stream. That one is written down as an open gap.
- **The model provider's own tools bypass every firewall.** Features like web fetch run on the provider's side. The only place to stop them is the gateway, reading the request body.
- **An approved tool's arguments are a channel.** A DNS lookup tool can carry data in the name it looks up. Argument rules in the tool gateway closed the wide channels; an incident ID with six digits still carries about 20 bits a call to the tracker.

The pattern: only the model's output can be relied on to be hostile, so check it where it turns into an effect, a tool call, a request or a packet, in the component that understands what it means. ([P1][p1], [P3][p3] findings)

## 6. Measuring an AI system is harder than building one

The eval suite replays scenarios through the whole pipeline on every pull request. Getting it to measure the right thing took longer than building it.

- **A gate needs a baseline, not a perfect score.** One scenario fails by design: the mock model obeys an injected instruction, and the policy gate holds. Requiring 100% would have kept CI red or got the scenario deleted.
- **The policy sees the proposal, not the request.** I wrote a deliberate regression that quietly turned "scale to 30" into 20. Thirty would have been denied as over the namespace's ceiling; twenty went through the policy as an ordinary large scale-up. Only the eval, which knows what was asked, caught it.
- **A spent token budget looks like a cautious agent.** After a day of runs the agent's hourly budget ran out mid-eval, it escalated four scenarios, and the gate read that as a regression.
- **Any sender could write any metric**, including the "safety held" figure on the dashboard, until the collector dropped every metric not named as the agent's.
- **Rules judge effects; they don't read intent.** Of five deliberately bad proposals seeded among real work, the automated rules caught three. The two they missed did something harmless that didn't match what they said they'd do. A small local model used as a judge agreed with the deterministic checks on only five of eight scenarios, and passed the injection scenario.

Catching a mismatch between intent and effect needs a capable reader, human or model, and that reader needs measuring too. ([P4][p4], [P5][p5] findings)

## 7. The environment fights back

- My scripts ran against whichever cluster was current. Creating a cluster for another project switched it, and a test run there could have reported PASS for controls that don't exist. Every command is now pinned to its cluster.
- Large image pulls kept failing with "connection reset". A packet capture showed the resets arriving one hop past the Windows host: the home router's NAT. I ended up writing a fetcher that resumes downloads with HTTP range requests and verifies each layer's hash. One 1.7 GiB layer took 12 minutes and 13 resumes.
- A WSL restart broke gVisor's default platform. A node restart left Cilium mislabelling a few pods, which silently broke their traffic.
- GPU time-slicing shares time, not memory. A second model replica measured the first one's memory as its own, computed a negative cache size, and crashed.

Rehearse from scratch, pin every command to its target, and put a health check at the start of every drill. ([P1][p1], [P4][p4], [P5][p5] findings)

## 8. Treat evidence like code

- I added up test counts by hand twice, and was wrong both times. Counts now come from the output of a real run.
- Rehearsing CI on a throwaway cluster found a dependency that a long-lived cluster had hidden for weeks.
- A required CI check can't sit behind a path filter, or docs-only pull requests wait forever for a check that never runs.

Generate evidence; don't transcribe it. ([P2][p2], [P4][p4] findings)

## What's still open

The red-team in this project uses a mock model that obeys every instruction it reads. That's deliberate: the question is which layer holds when the model fully complies, and for 13 of 14 attacks the log names one. But it also means I haven't yet measured how the real model behaves under injection. That's next.

The human approver is the last line for the proposals the rules can't judge, and I haven't measured real approvers yet either. Ten known weaknesses are reproduced by the tests and written up with options in [known gaps][gaps]. Writing them down was part of the point: a documented limitation is defensible, and an undocumented one looks like an oversight.

The code, the tests, and all 150-odd findings are in [the repo][repo]. Next I'll write up the architecture as a whole.

---

*This post was drafted with AI (Claude) from the project's findings logs, then edited and checked by me.*

[repo]: https://github.com/billykong/safe-ops-agent-k8s
[prd]: https://github.com/billykong/safe-ops-agent-k8s/tree/main/docs/prd
[gaps]: https://github.com/billykong/safe-ops-agent-k8s/blob/main/docs/known-gaps.md
[p1]: https://github.com/billykong/safe-ops-agent-k8s/blob/main/projects/01-sandboxed-code-exec/README.md#findings-so-far
[p2]: https://github.com/billykong/safe-ops-agent-k8s/blob/main/projects/02-approval-gate/README.md#findings
[p3]: https://github.com/billykong/safe-ops-agent-k8s/blob/main/projects/03-tool-access/README.md#findings
[p4]: https://github.com/billykong/safe-ops-agent-k8s/blob/main/projects/04-observability/README.md#findings
[p5]: https://github.com/billykong/safe-ops-agent-k8s/blob/main/projects/05-local-inference/README.md#findings
