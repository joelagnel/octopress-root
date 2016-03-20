---
layout: post
title: "TIF_NEED_RESCHED: why is it needed"
date: 2016-03-20 01:44:32 -0700
comments: true
categories: 
---

`TIF_NEED_RESCHED` is one of the many "thread information flags" stored along side every task in the Linux Kernel. One of the flags which is vital to the working of preemption is `TIF_NEED_RESCHED`. Inorder to explain why its important and how it works, I will go over 2 cases where `TIF_NEED_RESCHED` is used.

Preemption
----------
Kernel Preemption is the process of forceably grabbing CPU from a user or kernel context and giving it to someone else (user or kernel). It is the primary means for timesharing a CPU between competing tasks (I will use task as terminology for process).
In Linux, the way it works is a timer interrupt (called the tick) interrupts the task that is running and makes a decision about whether a task or a kernel code path (executing on behalf of a task like in a syscall) is to be preempted. This decision is based on whether the task has been running long-enough and something higher priority woke up and needs CPU now, or is ready to run.

These things happen in `scheduler_tick()`, the exact path is *TIMER HARDWARE INTERRUPT* -> `scheduler_tick` -> `task_tick_fair` -> `entity_tick` -> `check_preempt_tick`.

Here's a small bit of code in `check_preempt_tick`
```
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
        unsigned long ideal_runtime, delta_exec;
        struct sched_entity *se;
        s64 delta;

        ideal_runtime = sched_slice(cfs_rq, curr);
        delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
        if (delta_exec > ideal_runtime) {
                resched_curr(rq_of(cfs_rq));
                /*
                 * The current task ran long enough, ensure it doesn't get
                 * re-elected due to buddy favours.
                 */
                clear_buddies(cfs_rq, curr);
                return;
        }
```
Here you see a decision is made that the process ran long enough based on its runtime and if so call `resched_curr`. Turns out `resched_curr` sets the `TIF_NEED_RESCHED` for the current task! This informs whoever looks at the flag, that this process should be scheduled out soon.

Even though this flag is set at this point, the task is not going to be preempted yet. This is because preemption happens at specific points such as exit of interrupts. If the flag is set because the timer interrupt decided that something of higher priority needs CPU now, then at the exit of the timer interrupt (interrupt exit path), `TIF_NEED_RESCHED` is checked, and because it is set - `schedule()` is called causing context switch to happen to another process of higher priority, instead of just returning to the process that the timer interrupted like it normally would.
While I haven't nailed down the exact line of code in the interrupt exit path to user land that does this, I do believe the interrupt exit path to kernel space does a call to `preempt_schedule_irq` (see `arch/x86/entry/entry_64.S`) that checks for `TIF_NEED_RESCHED` and if it is set, then calls the context-switch code in the scheduler. These are just details though and what's important to understand is the `TIF_NEED_RESCHED` caused the preemption to eventually happen.

Critical sections in kernel code where preemption is off
--------------------------------------------------------
One nice example of a code path where preemption is off is the `mutex_lock` path in the kernel. In the kernel, there is an optimization where if a mutex is already locked and not available, but if the lock owner (the task currently holding the lock) is running on another CPU, then the mutex temporarily becomes a spinlock (which means it will spin until it can get the lock) instead of behaving like a mutex (which sleeps until the lock is available). The pseudo code looks like this:
```
mutex_lock() {
  disable_preempt();
  if (lock can't be acquired and the lock holding task is currently running) {
	while (lock_owner_running && !need_resched()) {
		cpu_relax();
	}
  }
  enable_preempt();
  acquire_lock_or_sleep();
}
```

The lock path does exactly what I described. `cpu_relax()` is arch specific which is called when the CPU has to do nothing but wait. It gives hints to the CPU that it can put itself into an idle state or use its resources for someone else. For x86, it involves calling the [halt instruction](https://en.wikipedia.org/wiki/HLT).

What I noticed is the Ftrace latency tracer complained about a long delay in the preempt disabled path of mutex_lock for one of my tests, and I made some [noise](http://www.spinics.net/lists/linux-rt-users/msg15022.html) about it on the mailing list. Disabling preemption for long periods is generally a bad thing to do because during this duration, no other task can be scheduled on the CPU. However, Steven [pointed out that](http://www.spinics.net/lists/linux-rt-users/msg15025.html) for this particular case, since we're checking for need_resched() and breaking out of the loop, we should be Ok. What would happen is, the scheduling timer interrupt (which calls `scheduler_tick()` I mentioned earlier) comes in and checks if higher priority tasks need CPU, and if they do, it sets `TIF_NEED_RESCHED`. Once the timer interrupt returns to our tightly spinning loop in mutex_lock, we would break out of the loop having noticed `need_resched()` and, re-enable preemption as shown in the code above. Thus the long duration of preemption doesn't turn out to be a problem as long tasks that need CPU are prioritized correctly. `need_resched()` achieved this fairness.

Next time you see `if (need_resched())` in kernel code, you'll have a better idea why its there :). Let me know your comments if any.

