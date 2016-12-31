---
layout: post
title: "ARMv8 and NMI support"
date: 2016-12-31 22:29:26 -0700
comments: true
categories: 
---

Non-maskable interrupts (NMI) is a great debug feature that hardware can provide. Some great Linux kernel features that rely on NMI to work properly are:

* Backtrace from all CPUs: A number of places in the kernel rely on dumping the stacks of all CPUs at the time of a failure to determine what was going on. Some of them are [Hung Task detection](http://lxr.free-electrons.com/source/kernel/hung_task.c), [Hard/soft lockup detector](http://lxr.free-electrons.com/source/Documentation/lockup-watchdogs.txt) and spinlock debugging code.

* Perf profiling: To be able to profile code that runs in interrupt handlers, or in sections of code that disable interrupts, Perf works well only when NMI is available in the architecture.

Below is a [flamegraph](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html) generated with perf and some flamegraph magic, that shows just what happens when perf is used in an architecture like ARM that doesn't support NMI. The flamegraph is generated on an ARMv8 platform:

{% img /images/nmi/flamegraph.png %}

As you can see in the area of the flamegraph where the arrow is pointed, a large amount of time is spent in `_raw_spin_unlock_irqrestore()`. It can baffle anyone looking at this data for the first time, and make them think that most of the time is spent in this function. What's actually happenning is because perf is using a maskable interrupt in ARMv8 to profile, any section of code that disables interrupts will not be see in the flamegraph. In other words perf is unable to peek into sections of code where interrupts are disabled. As a result, when interrupts are reenabled during the `_raw_spin_unlock_irqrestore`, the perf interrupt routine then kicks in and records the large number of samples that elapsed in the interrupt-disable section. This recording accounts all these samples to the _raw_spin_unlock_restore function, hence the flamegraph anomoly. It is indeed quite sad that ARM still doesn't have a true NMI which perf would love to make use of.

BUT! [Daniel Thomspon](https://lkml.org/lkml/2016/8/19/583) has been hard at work trying to simulate Non-maskable interrupts on ARMv8. The idea is [based on using interrupt priorities](/misc/arm-irq-priortization-white-paper.pdf) and is the subject of the rest of this post.

To simulate an NMI, Daniel creates 2 groups of interrupts in his patchset. One group is for all 'normal' interrupts, and the other for non-maskable interrupts (NMI). Non-maskable interrupts are assigned a higher priority than the normal interrupt group. Inorder to 'mask' interrupts in this approach, Daniel replaces the regular interrupt masking scheme in the kernel which happens at the CPU-core level, with setting of the interrupt controller's PMR (priority mask register). When the PMR is set to a certain value, only interrupts which have a higher priority than what's in the PMR will be signaled to a CPU core, all other interrupts will be silenced (masked). By using this technique, it is possible to mask normal interrupts while keeping the NMI unmasked all the time.

Just how does he do this? So, a small primer on interrupts in the ARM world.
ARM uses the GIC [Generic interrupt controller](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dai0176c/ar01s03s01.html) to prioritize and route interrupts to CPU cores. GIC interrupt priorties go from 0 to 255. 0 being highest and 255 being the lowest. By default, the kernel [assigns priority 0xa0 (192)](http://lxr.free-electrons.com/source/include/linux/irqchip/arm-gic.h?v=4.8#L57) to all interrupts. He changes this default priority from 0xa0 to 0xc0 (you'll see why).
He then defines what values of PMR would be consider as "unmasked" vs "masked". Masked is 0xb0 and unmasked is 0xf0. This results in the following priorities (greater numbers are lower priority).
```
0xf0 (240 decimal)  (11110000 binary) - Interrupts Unmasked (enabled)
0xc0 (192 decimal)  (11000000 binary) - Normal interrupt priority
0xb0 (176 decimal)  (10110000 binary) - Interrupts masked   (disabled)
0x80 (128 decimal)  (10000000 binary) - Non-maskable interrupts
```
In this new scheme, when interrupts are to be masked (disabled), the PMR is set to 0xf0 and when they are unmasked (enabled), the PMR is set to 0xb0. As you can see, setting the PMR to 0xb0 indeed masks normal interrupts, because 0xb0(PMR) < 0xc0(Normal), however non-maskable interrupts still stay unmasked as 0x80(NMI) < 0xb0(PMR). Also notice that inorder to mask/unmask interrupts, all that needs to be done is flip bit 7 in the PMR (0xb0 -> 0xf0). Daniel largely uses Bit 7 as the mask bit in the patchset.

Daniel has tested his patchset only on the foundation model yet, but it appears that the patch series with modifications should work on the newer Qualcomm chipsets that have the necessary GIC (Generic interrupt controller) access from the core to muck with IRQ priorities. Also, currently Daniel has only implemented CPU backtrace, more work needs to be done for perf support which I'll look into if I can get backtraces working properly on real silicon first.
