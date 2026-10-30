---
layout: post
title: "What CPU pinning and huge pages buy you on Kubernetes, measured"
description: "I gave a latency benchmark its own CPU core on Kubernetes and its tail latency got worse. The kubelet only keeps other pods off a pinned core; the host still uses it. Here is what fixed it, stage by stage, with numbers."
date: 2026-10-30 00:00:00 +0000
redirect_from: /cpu-pinning/
---

Kubernetes can give a pod its own CPU cores. Turn on the static CPU manager, request whole CPUs, and the kubelet moves every other pod off them. That's the standard advice for latency-sensitive workloads, and it sounds like the end of the story.

So I measured it. I gave a latency benchmark a core of its own, put a noisy neighbour on the rest of the machine, and compared it with an unpinned run sharing CPUs with that neighbour. The pinned pod's tail latency was **about five times worse**.

This post is about why, and what it took to turn pinning into a win: a 3.6× better tail than the unpinned run. It's the first project in a lab I'm building, [billy-kube][repo], which takes the low-latency tuning used on trading systems and measures what each knob does on Kubernetes, one knob at a time.

**In short:**

- **The static CPU manager isolates pods from pods, not from the host.** A pinned core that sits idle between wake-ups attracts device interrupts, host threads and deep sleep states. Pinning alone made the tail worse.
- **Limiting C-states alone changed nothing.** The deep sleep states were the first symptom I found, not the cause.
- **Boot-time isolation fixed it.** With `isolcpus`, `nohz_full` and `irqaffinity` on the core the kubelet hands out, the pinned pod's p99.9 wake-up latency fell to **76 µs against 272 µs** for the unpinned run.
- **Huge pages are a steady 15–18%** off memory latency, whatever the host tuning.
- **Check what else ran on the CPU, not just the cpuset.** Every run passed its "is it really pinned?" check. Only per-CPU counters showed the problem.

## The setup

One bare-metal laptop: an Intel i7-1165G7 with 4 cores and 8 threads, Linux 6.8, and a three-node [kind][kind] cluster (Kubernetes 1.32). kind runs each node as a container, so every node shares the laptop's kernel and CPUs. That turns out to matter.

One worker is the latency node. Its kubelet runs the **static CPU manager**, which gives *Guaranteed* pods (requests equal to limits) that ask for whole CPUs an exclusive set of cores. The kubelet keeps core 0 (CPUs 0 and 4, its two hyper-threads) for itself and system daemons. A taint keeps ordinary pods off the node.

| Core | CPUs | Used for |
| --- | --- | --- |
| 0 | 0, 4 | Reserved by the kubelet (housekeeping) |
| 1 | 1, 5 | The pinned benchmark pod |
| 2, 3 | 2, 6, 3, 7 | The shared pool: the noisy neighbour |

Two benchmarks run in each pod:

- **cyclictest** sets a timer for every 200 µs, sleeps until it fires, and records how late it woke, for 60 seconds. That lateness is *wake-up latency*: how long it takes the CPU to start running your code when its event arrives. It's the closest simple stand-in for "a market data packet arrived; how long until the handler runs?"
- **memlat** follows a chain of pointers through 256 MiB of memory in random order, so every step is a cache miss and, with normal pages, usually a TLB miss too. It reports the time per step.

A **noise pod** runs `stress-ng` (CPU, cache and memory stress) on the shared pool for the whole run.

Each run is one of two variants:

- **baseline:** a *Burstable* pod with no CPU pinning, sharing all CPUs with the noise pod. It ends up on CPU 0.
- **tuned:** a Guaranteed pod with 2 whole CPUs, which the kubelet pins to CPUs 1 and 5 and moves the noise pod off. It also gets 512 MiB of 2 MiB huge pages, so memlat runs with both page sizes.

A run only counts if a script confirms, from the pods' actual cpusets, that the tuned pod has exclusive CPUs and the baseline really shares them with the noise. Every number below is the median of three runs.

## Pinning made it worse

With no host tuning (the laptop's defaults: power-saving governor, turbo on, every sleep state allowed):

| Variant | p50 | p99 | p99.9 |
| --- | --- | --- | --- |
| baseline (shared with noise) | 58 µs | 66 µs | 363 µs |
| tuned (exclusive core) | 56 µs | 218 µs | 1690 µs |

The pod with a core to itself had a p99 three times worse, and a p99.9 more than four times worse, than the one sharing CPUs with `stress-ng`.

### First suspect: C-states

When a CPU has nothing to do, it doesn't spin. It drops into a sleep state, called a **C-state**, to save power. Deeper states save more and take longer to wake from. On this laptop the kernel lists their exit latencies: C1 1 µs, C2 253 µs, C3 1048 µs.

Those last two numbers look a lot like the tuned p99 and p99.9. The kernel's per-CPU idle counters agreed: during one tuned run, CPU 1 entered C2 and C3 more than 3,000 times, while CPU 0, kept busy by the noise, entered them 12 times each. A busy neighbour keeps a core awake. An exclusive core running a benchmark that mostly sleeps is idle almost all the time, and the hardware treats it that way.

cyclictest normally guards against this itself, by holding `/dev/cpu_dma_latency` at 0, which tells the kernel not to use deep states. That device isn't inside a container by default.

### The fix that didn't fix it

So I applied the usual runtime tuning: the `performance` governor, turbo off, and every C-state deeper than C1 disabled through sysfs.

| Variant | p50 | p99 | p99.9 |
| --- | --- | --- | --- |
| baseline | 59 µs | 66 µs | 178 µs |
| tuned | 58 µs | 283 µs | 1981 µs |

CPU 1 now entered C2 and C3 **zero** times, and the tuned tail didn't move. The C-states had been real, but they weren't the whole story.

## What was actually on the "exclusive" CPU

The kubelet's promise is narrow. It keeps *the latency node's pods* off the pinned CPUs. It says nothing about the host: device interrupts, kernel work, the container runtime, the other kind nodes (which share this kernel), or the desktop. To the host scheduler, an idle CPU is the best place to put spare work.

Everything below can be read without root. I took a snapshot before and after 40 seconds of a tuned run, and sampled which threads ran on CPU 1 every two seconds:

```bash
cp /proc/interrupts /tmp/irq.before; cp /proc/softirqs /tmp/softirq.before
for s in /sys/devices/system/cpu/cpu1/cpuidle/state*; do echo "$(cat $s/name) $(cat $s/usage)"; done
ps -eLo psr,comm --no-headers | awk '$1 == 1'   # threads last run on CPU 1
cat /proc/irq/*/effective_affinity_list | sort | uniq -c   # where device IRQs may fire
```

In those 40 seconds, CPU 1 handled:

- **Device interrupts:** 2,368 from the NVMe disk and 735 from the Wi-Fi card.
- **Inter-processor interrupts:** 4,745 function calls, 4,387 reschedules and 368 TLB shootdowns, sent by other CPUs.
- **12,647 scheduler softirqs**, against 1,040 on the busy CPU 0. That's the load balancer, repeatedly checking whether to move work onto the idle core, and often doing it.
- **Threads from** containerd, the kubelet, the API server, etcd, CoreDNS, snapd, a VPN daemon, the desktop and my terminal.

Each one interrupts cyclictest or delays its wake-up. The pod was alone among *pods* and very much not alone on the CPU.

## Isolating the core at boot

Keeping the host off a CPU takes kernel boot parameters. Each one removes a different source of noise:

| Parameter | What it does |
| --- | --- |
| `isolcpus=domain,1,5` | Takes CPUs 1 and 5 out of scheduler load balancing. At boot, every task's allowed CPUs become the rest, so host threads never land there unless something pins them (as the kubelet does). |
| `isolcpus=managed_irq,…` | Keeps kernel-managed interrupts off the isolated CPUs where the driver allows it. |
| `irqaffinity=0,4` | Sends all other interrupts to the housekeeping core by default. Match it to the kubelet's reserved CPUs. |
| `nohz_full=1,5` | Stops the periodic scheduler tick on those CPUs while they run a single task. |
| `rcu_nocbs=1,5` | Moves the kernel's RCU callback work off those CPUs. |

On Ubuntu, these go in a GRUB drop-in, so undoing them means deleting one file and running `update-grub`:

```bash
# /etc/default/grub.d/90-isolation.cfg
GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT isolcpus=domain,managed_irq,1,5 nohz_full=1,5 rcu_nocbs=1,5 irqaffinity=0,4"
```

The kubelet has to agree. Its `reserved-cpus` is the housekeeping core, `0,4`, so the first core it hands to a pinned pod is core 1, the isolated one. Isolate only that core. A shared-pool pod that lands on a `domain`-isolated CPU stays there, because nothing balances it off again.

With the runtime tuning plus these boot parameters:

| Variant | p50 | p99 | p99.9 |
| --- | --- | --- | --- |
| baseline | 59 µs | 70 µs | 272 µs |
| tuned | 59 µs | **72 µs** | **76 µs** |

The pinned pod's p99.9 fell from about 2 ms to 76 µs. The three runs were 76, 77 and 76 µs, the tightest spread of anything I measured. Over the same 40 seconds, CPU 1 now had no device interrupts, no scheduler softirqs, one reschedule IPI, and no threads except cyclictest and its own kernel threads.

The whole story in one table, as p99.9 wake-up latency:

| Host tuning | Baseline | Tuned | Tuned vs baseline |
| --- | --- | --- | --- |
| None | 363 µs | 1690 µs | 4.7× worse |
| Runtime: governor, turbo off, C-states | 178 µs | 1981 µs | 11× worse |
| Runtime + boot isolation | 272 µs | 76 µs | **3.6× better** |

## Huge pages: the boring, reliable win

memlat measures something else: the cost of reaching memory. Every memory access goes through the TLB, a small cache of virtual-to-physical address translations. With normal 4 KiB pages, 256 MiB is 65,536 pages, far more than the TLB holds, so a random walk misses it almost every step and the CPU walks the page tables. With 2 MiB huge pages it's 128 pages.

| Host tuning | 4 KiB p50 | 2 MiB p50 | Saving |
| --- | --- | --- | --- |
| None | 151 ns | 125 ns | 17% |
| Runtime | 170 ns | 140 ns | 18% |
| Runtime + boot | 165 ns | 140 ns | 15% |

Huge pages cut 15–18% off every stage, independent of everything else. On Kubernetes they're a node resource (`hugepages-2Mi`) you request like memory, after reserving them on the host.

Two more things show up in that table. Turbo off costs about 10% of memory latency, because the core runs at a lower clock. And isolation helps memory as well: the worst memlat step on the pinned core fell from about 20 µs to about 0.5 µs, because nothing interrupts the walk any more.

## What this doesn't show

- **One laptop.** A laptop has one NUMA node, a desktop running alongside, and power management tuned for battery. A server will have different numbers; I'd expect the same mechanism.
- **kind shares the host kernel.** On a real node there's no second Kubernetes node on the same kernel, but there's still the container runtime, the kubelet, monitoring agents and device interrupts. The argument for host isolation is the same.
- **The p50 of about 58 µs is the benchmark, not the CPU.** cyclictest ran as a normal (`SCHED_OTHER`) task, and the kernel lets those wake up to 50 µs late by default to batch timers (*timer slack*). A real-time priority run would remove that floor. That's the next experiment.
- **A rare 3.6 ms maximum remains** on the isolated core. I don't know what it is yet.
- **Wake-up latency is a proxy.** It isn't packet-to-handler latency. Network latency comes later in the lab, starting with Cilium against the default CNI.

## Takeaways

1. **The static CPU manager keeps pods off pods.** It doesn't keep the host off. On any node with system daemons, a pinned core that idles attracts host threads, interrupts and sleep states.
2. **Isolation is a host job.** `isolcpus=domain,managed_irq`, `nohz_full`, `rcu_nocbs` and `irqaffinity` on the core the kubelet hands out, with `reserved-cpus` set to the housekeeping core and C-states limited. Without it, pinning can make things worse.
3. **Measure the CPU, not the config.** Every run passed a check that the pod really was pinned. Only `/proc/interrupts`, `/proc/softirqs`, the idle counters and a thread sample showed why it was slow.
4. **Huge pages are the easy win.** A steady 15–18% for random memory access, with no host tuning needed.

The manifests, scripts, raw results and the summariser behind every table are in the [billy-kube][repo] repository, project 01.

[repo]: https://github.com/billykong/billy-kube
[kind]: https://kind.sigs.k8s.io/
