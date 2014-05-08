---
layout: post
title: "Spinlock implementation in Linux"
date: 2014-05-07 22:42:45 -0500
comments: true
categories: 
---
This article tries to clarify how spinlock is implemented in the face of premption. Future articles may speak about handling for interrupts, and how to use the right spin_lock/unlock function for the right purpose. The focus of this article will be more on basic concepts than details, as details tend to be forgotten more easily and shouldn't be too hard to look up.

Fundamentally somewhere in `include/linux/spinlock.h`, a decision is made on which spinlock header to pull based on whether SMP is enabled or not:
```
#if defined(CONFIG_SMP) || defined(CONFIG_DEBUG_SPINLOCK)
# include <linux/spinlock_api_smp.h>
#else
# include <linux/spinlock_api_up.h>
#endif
```

We'll go over how things are implemented in both the SMP (Symmetric Multi-Processor) and UP (Uni-Processor) cases.

For the SMP case, `__raw_spin_lock*` functions in `kernel/locking/spinlock.c` are called when one calls some version of a `spin_lock`.

Following is the definition of the most basic version defined with a macro:

```
#define BUILD_LOCK_OPS(op, locktype)                                    \
void __lockfunc __raw_##op##_lock(locktype##_t *lock)                   \
{                                                                       \
        for (;;) {                                                      \
                preempt_disable();                                      \
                if (likely(do_raw_##op##_trylock(lock)))                \
                        break;                                          \
                preempt_enable();                                       \
                                                                        \
                if (!(lock)->break_lock)                                \
                        (lock)->break_lock = 1;                         \
                while (!raw_##op##_can_lock(lock) && (lock)->break_lock)\
                        arch_##op##_relax(&lock->raw_lock);             \
        }                                                               \
        (lock)->break_lock = 0;                                         \
}                                                                       \
```

The function has several imporant bit. First it disables preemption on line 5, then tries to *atomically* acquire the spinlock on line 6. If it succeeds it breaks from the `for` loop on line 7, leaving preemption disabled for the duration of crticial section being protected by the lock. If it didn't succeed in acquiring the lock (maybe some other CPU grabbed the lock already), it enables preemption back and spins till it can acquire the lock keeping *preemption enabled during this period*. Each time it detects that the lock can't be acquired in the `while` loop, it calls an architecture specific relax function which has the effect executing some variant of a `no-operation` instruction that causes the CPU to execute such an instruction efficiently in a lower power state. We'll talk about the `break_lock` usage in a bit. Soon as it knows the lock is free, say the `raw_spin_can_lock(lock)` function returned 1, it goes back to the beginning of the `for` loop and tries to acquire the lock again.

What's important no note here is the reason for keeping preemption enabled (we'll see in a bit that for UP configurations, this is not done). While the kernel is spinning on a lock, other processes shouldn't be kept from preempting the spinning thread. The lock in these cases have been acquired on a *different CPU* because (assuming bug free code) its impossible the current CPU which is trying to grab the lock has already acquired it, because preemption is disabled on acquiral. So it makes sense for the spinning kernel thread to be preempted giving others CPU time.
It is also possible that more than one process on the current CPU is trying to acquire the same lock and spinning on it, in this case the kernel gets continuously preempted between the 2 threads fighting for the lock, while some other CPU in the cluster happily holds the lock, hopefully for not too long.

That's where the `break_lock` element in the lock structure comes in. Its used to signal to the lock-holding processor in the cluster that there is someone else trying to acquire the lock. This can cause the lock to released early by the holder if required.


Now lets see what happens in the UP (Uni-Processor) case.

Believe it or not, its really this simple:
```
#define ___LOCK(lock) \
  do { __acquire(lock); (void)(lock); } while (0)

#define __LOCK(lock) \
  do { preempt_disable(); ___LOCK(lock); } while (0)

// ....skipped some lines.....
#define _raw_spin_lock(lock)                    __LOCK(lock)
```

All that needs to be done is disabled preemption and acquire the lock. The code really doesn't do anything other than disable preemption. The references to the `lock` variable are just to suppress compiler warnings as mentioned in comments in the source file.

There's no spinning at all here like the UP case and the reason is simple: in the SMP case, remember we had agreed that while a lock is *acquired* by a particular CPU (in this case just the 1 CPU), no other process on that CPU should have acquired the lock. How could it have gotten a chance to do so with preemption disabled on that CPU to begin with?

Even if the code is buggy, (say the same process tries to acquires the lock twice), its still impossible that 2 *different processes* try to acquire the same lock on a Uni-Processor system considering preemption is disabled on lock acquiral. Following that idea, in the Uni-processor case, since we are running on only 1 CPU, all that needs to be done is to disable preemption, since the fact that we are being allowed to disable preemption to begin with, means that no one else has acquired the lock. Works really well!

Hope this post made a few things more clear, there's a lot more to spinlocking than I wished I could write. A good reference is [Rusty's Unreliable Guide To Locking](https://www.kernel.org/pub/linux/kernel/people/rusty/kernel-locking/).

