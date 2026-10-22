# One Scheduler, Multiple Operating-System Environments

**The native execution engine of myOS — architecture, synchronization, multicore behavior and measured cost**

*Source review: myOS implementation on 9 October 2026. The measurements below come from a documented 1 October 2026 Proxmox experiment, not a benchmark rerun for this article. The Android and Windows-driver development work are discussed as execution-environment integrations, not as claims of full application compatibility.*

![myOS native scheduler and SMP architecture](../diagrams/native-scheduler-smp.svg)

## 1. Why the scheduler belongs at the center of the myOS story

myOS presents different operating-system contracts to Linux ELF applications, original Microsoft Windows PE processes, native services and, in a separate development effort, original Android user space. Those contracts differ in their process-visible identifiers, APIs, thread startup, synchronization and error handling. **The actual CPU time, however, belongs to one native scheduling system.**

This is the architectural difference between *hosting operating-system interfaces* and *running several guest operating systems*. myOS does not hand Linux threads to a hidden Linux scheduler or Windows threads to a Windows NT scheduler. Every native runnable thread competes for processor time through the same kernel mechanism. Environment-specific semantics are projected onto that mechanism.

This matters far beyond process dispatch. A Windows service waiting for an ALPC reply, a Linux container waiting for network I/O, and a user-space device driver waiting for an interrupt are all participants in the same scheduling and wakeup infrastructure. The scheduler is therefore where the three core design principles meet:

- **Mechanism belongs to the native kernel.** Thread state, CPU execution, timing and wakeup scheduling have one authority.
- **Semantics belong to the execution environment.** Linux signals, NT alerts, completion APIs and Android-specific expectations keep their correct meanings without creating new CPU schedulers.
- **Policy and authority are explicit.** User-space driver management and resource ownership do not imply arbitrary control over native thread scheduling.

The scheduler is not presented here as a clone of Linux CFS or the Windows priority scheduler. Its current dispatch policy is primarily round-robin, extended with a bounded preference for briefly running, recently awakened threads. That is different from a complete Windows priority-class model or a real-time admission scheduler.

## 2. One native thread model, multiple callers

An executable may originate from different formats or run different runtime libraries, but once myOS has created the thread it is a native schedulable entity. The scheduler tracks its state, saved CPU context, memory address-space association, ownership, accounting and last CPU. It also preserves architectural thread state such as floating-point/vector registers and FS/GS bases while applying the relevant process memory and security transitions.

A user process can contain several threads. A driver host itself is just another supervised user-space process with threads. Native kernel threads and each CPU's idle thread also participate in execution management.

The key thread states are:

| State | Meaning |
|---|---|
| Ready | Eligible to run, placed in a CPU's local ready list |
| Running | Currently executing on a CPU |
| Blocked | Waiting for an operation, event or deadline |
| Exiting | Must not be dispatched again; retirement is pending |

An exited thread is not a permanently stored fifth scheduling state. It leaves the live scheduler tables, and its resources are reclaimed safely afterward.

**Lifetime is part of scheduling correctness.** An old thread identifier must not accidentally refer to a newly created thread in a reused slot. myOS uses generation-aware identity. A ready-list entry must not be present twice, and a thread being migrated must not execute simultaneously on two processors. These invariants are guarded in the actual implementation and are exercised by dedicated stress tests.

### Why the CPU must not free its current stack

The scheduler runs on a kernel stack belonging to the context it is switching away from. Consequently, a terminating thread cannot immediately release its own stack while switching away; executing code would still depend on that memory. myOS first transfers the outgoing thread to a per-CPU departure state, then retires it only after the CPU has demonstrably switched to another stack. A dedicated reaper subsequently releases its remaining resources.

A related rule protects migration: another CPU must not steal a thread until its previous CPU has stopped using that thread's stack. This is an example of why correct scheduling is as much about *ownership and lifetime* as about choosing the next runnable thread.

## 3. Multicore architecture: local queues instead of one global scheduling lock

The SMP implementation discovers processors through the platform's hardware description and brings secondary CPUs into long mode. Each online CPU receives its own scheduling state and idle thread. Its normal scheduling operation uses its **own runqueue and lock**, avoiding a single global queue lock on the high-frequency dispatch path.

The model is deliberately simple to reason about: each CPU owns a local thread table, ready list and currently executing context. CPU-wide thread placement and migration are separate, less frequent operations.

### How work is distributed

New threads are initially placed using a deterministic round-robin choice among online CPUs. That initial placement is **not** currently a NUMA-locality-aware placement policy.

When a CPU runs out of local ready work, it can try to steal a runnable thread from another online CPU. In choosing a candidate, it examines runqueue load and prefers the most heavily loaded available CPU. The design uses bounded, nonblocking attempts to acquire another CPU's runqueue lock, specifically to avoid waiting indefinitely while already holding its own local lock.

There is a second balancing path: periodic **load rebalancing** can move work even if the receiving CPU's own ready list is not empty. Without it, permanently busy local queues could leave a much more heavily loaded processor underutilized elsewhere. Both paths avoid migrating a thread that is still on its previous CPU's active stack.

Stealing and rebalancing are intentionally throttled. Continuously probing every runqueue can create precisely the shared-lock pressure the local-queue architecture was designed to avoid.

### What the test suite establishes

Dedicated SMP scheduler tests exercise at least three relevant properties: all online CPUs eventually execute runnable work; intentionally imbalanced work migrates to another CPU; and individual test threads do not disappear or execute twice. A separate regression targets local-CPU wakeups that formerly could wait for the next timer tick.

The presence of these checks is useful evidence of engineering rigor. It is not a claim that the entire scheduler has been formally verified or that every supported hardware topology has passed stress testing on the latest revision.

## 4. Preemption, runqueue fairness and bounded wakeup preference

When a runnable thread is selected, myOS restores its native execution state and gives it a time slice. A CPU with runnable work retains timer-based preemption so compute-heavy work cannot indefinitely monopolize the processor. Threads that yield, block or exit relinquish execution earlier.

On each CPU, regular ready-list selection is FIFO/round-robin. However, the implementation addresses a practical problem: interactive or service threads often wake, perform only a very small amount of work and block again. Always placing such a thread behind a long series of compute-bound threads could add a full slice of latency per competitor.

myOS consequently has a **bounded wakeup preference** for threads whose previous CPU burst was short. These threads can be inserted ahead of the ordinary ready tail, but the number of consecutive boosted dispatches is limited. After a bounded burst, ordinary runnable work receives a turn. This is a responsiveness optimization over round-robin, *not* a comprehensive priority scheduler.

The practical relevance is immediate: a Windows service responding to an ALPC request and a user-space device host responding to I/O can be highly latency-sensitive despite doing little CPU work. The scheduler must move their threads promptly without starving a sustained Linux workload.

## 5. Sleeping correctly is as important as running correctly

The simplest scheduler explanation is “blocked threads sleep until another thread wakes them.” That description hides the most dangerous race: **an event can arrive between the last condition check and the act of going to sleep**.

If the wake occurs too early and is forgotten, the thread may sleep indefinitely even though the data it awaited is already available. This can look like a broken network driver, a stalled Windows service, a frozen GUI or an intermittent boot failure.

myOS uses one personality-neutral, interruptible wait protocol:

1. Prepare the scheduler's blocking transition.
2. Register interest in the relevant event sources.
3. Recheck the condition while registration and the source state are coordinated.
4. Arm an optional deadline.
5. Commit to blocking only when the condition is still unmet; otherwise cancel the pending block.

The protocol also accounts for process termination, environment-specific interruption and failure to allocate registration memory. If wait registration or deadline allocation cannot be established safely, the thread is **not** allowed to sleep in a state from which no valid wake can reach it.

A wake moves the sleeping thread back toward runnable execution. Linux may translate the outcome into signal or syscall semantics. Windows may translate it into a wait result or ALPC completion. The scheduler itself does not need to understand the full Linux signal model or NT object API.

### Targeted wakeups and prompt rescheduling

An event waking a thread on another CPU needs that CPU to reconsider its runqueue. The implementation uses **targeted reschedule IPIs**, avoiding an unnecessary broadcast to every processor for the normal cross-CPU case. An idle-polling CPU can sometimes observe the kick without an IPI.

A separate regression covers the same-CPU interrupt case: if an interrupt wakes a local thread, returning directly to the interrupted context without a scheduler passage can postpone execution until the next regular tick. myOS can request a rescheduling passage on the interrupt-return boundary instead.

The distinction between **wakeup processing cost** and **wake-to-run latency** is important. Processing the transition to Ready may take less than a microsecond in a particular measurement, but the elapsed time until the awakened thread actually runs can also include queueing, interrupt delivery and the scheduler's policy.

## 6. Timers, deadlines and idle behavior

A CPU that is completely idle should not consume cycles handling periodic scheduling interrupts that have no useful work to schedule. myOS supports **tickless idle**, under a clock capable of advancing independently of the scheduling tick.

When idle and no deadline is pending, the CPU can stop its local scheduling timer and enter hardware idle. If a deadline exists, it arms a **one-shot timer** for the required bound. An external device interrupt, local deadline or reschedule IPI can bring it back into useful execution.

A CPU running work is different: it still needs preemption. Yet a short sleep deadline may occur before the current thread's normal slice expires. The scheduler can program a one-shot interrupt for that earlier deadline while preserving the remaining slice and the original slice-end limit. This prevents a series of unrelated short deadlines from indefinitely extending a compute-heavy thread's slice.

### Deadline slack and coalescing

Timeouts can allow a specified degree of slack. When deadline windows overlap, multiple expirations can be handled at the same interrupt rather than each requiring separate timer programming and delivery. The source stores the earliest hard deadline so the common scheduler path need not search a full deadline list on every dispatch.

These mechanisms matter in heavily multithreaded workloads. A container environment such as k3s generates a large number of sleeps, futex waits, poll operations and timer deadlines even when application CPU use is low. Timer programming and wakeup overhead can dominate the kernel cost of such a seemingly idle system.

**Boundaries:** tickless behavior applies to idle CPUs, not a blanket elimination of scheduling interrupts. Advanced energy-aware scheduling, general-purpose CPU hotplug and a comprehensive NUMA placement policy should not be presented as completed features.

## 7. Thread context, process boundaries and security

The scheduler is responsible not just for choosing *which* thread runs, but for ensuring it resumes in the correct address space and architectural state.

The implementation tracks the address-space root associated with a thread. Dispatch activates the appropriate mapping context and integrates with the kernel's address-space isolation and CPU-mitigation work. Per-thread floating-point/vector context is preserved, as are the processor's FS and user-GS base values used by different runtime environments. The implementation also avoids unnecessary vector-state restores in cases where the correct state still resides on the same CPU.

Process termination propagates into thread lifecycle: a thread belonging to a terminating process must not return to user mode merely because it is still sitting in a ready queue. Controlled exceptions exist for native critical sections that must finish releasing an owned resource before termination can complete.

This matters especially when myOS runs original Windows system components and drivers in ordinary processes. The source of a thread does not change who owns its address space, when it can run, or how its resources are reclaimed.

## 8. Measuring a real scheduler under real workloads

The repository contains a reproducible performance study using an **idle but running k3s installation** on a Proxmox virtual machine. The documented test used **2 virtual CPUs**, 3 GiB RAM, virtual storage/network devices, and a real set of Kubernetes system pods. It first stabilized the system, then gathered a **60-second measurement window**. This is a measured configuration, not a general statement about the hardware-independent speed of myOS.

### Recorded baseline — 1 October 2026

| Metric | Recorded value | Correct interpretation |
|---|---:|---|
| Mean scheduler passage cost | **3.76 µs** | CPU time attributed to scheduler passage, not the latency of an application syscall |
| Mean wake processing cost | **0.76 µs** | CPU time to handle a wake, **not** wake-to-run latency |
| Scheduler passages | **3,534/s** | Includes passages that need not switch to another thread |
| Context switches | **2,162/s** | Actual switches observed |
| Wakeups | **2,009/s** | Scheduler wake events |
| Timer interrupts | **962/s** | Observed local timer IRQ rate summed over the VM |
| Guest idle CPU time | **95.3%** | Share of combined guest CPU time in the 60-second interval |
| Guest kernel CPU time | **2.9%** | Share of combined guest CPU time |
| Earliest-deadline lookup | **~85 ns** | Measured lookup operation within that profiling setup |

The source experiment attributes roughly 3.49 CPU-seconds out of 120 available CPU-seconds to the guest kernel, and 114.4 seconds to idle. The measurement instrumentation was enabled and has its own cost.

These figures are useful because they are attached to a named workload, platform, time window and revision. They are **not** a like-for-like benchmark versus Linux or Windows, and they must not be extrapolated to every CPU count or workload.

### What was optimized

The performance history documents repeated measurements and architectural fixes, including eliminating global wake storms, narrowing wakeups to relevant waiters, reducing costly local-APIC operations, moving from a global scheduler lock to per-CPU queues, introducing accurate deadline tracking and avoiding unnecessary idle-timer work. Such changes improved the whole system because they removed unnecessary work beneath the execution environments.

The data also shows an important profiling lesson: a low CPU percentage in a virtual machine does not guarantee low host overhead. Interrupts, virtualization exits, timer programming and locks all have observable costs that must be measured separately.

## 9. What the scheduler proves about the overall architecture

A scheduler is easy to hide behind a high-level operating-system diagram. In myOS it deserves to be visible because it demonstrates that the architecture has a real shared execution foundation.

Consider three simultaneous operations: a Linux process is waiting on a network socket; an original Microsoft service is waiting for an ALPC reply; and a Windows-format driver hosted in ring 3 is waiting on an I/O completion. Their interfaces and completion conditions are different. Yet a successful wake ultimately creates the same kind of native scheduling event. The CPU they run on is chosen by the same native scheduler; a terminating thread is retired by the same lifecycle machinery; and a runnable thread cannot be executed concurrently on two CPUs.

The actual application services, IPC semantics and driver interfaces remain environment-specific where required. **CPU execution, wait scheduling and resource lifetime do not become separate operating systems.**

This is the right way to describe the scheduler's role publicly: a demonstrated, independently developed SMP scheduling architecture that serves all native threads, coupled with testable wakeup behavior and reproducible performance observations.

## 10. Evidence and limits

| Claim | Evidence class | Important qualification |
|---|---|---|
| Per-CPU ready queues and multicore scheduling | Current source and dedicated SMP tests | Full CPU-hotplug lifecycle not implemented |
| Idle stealing and periodic balancing | Current source and stress cases | NUMA-aware placement not currently the initial policy |
| Round-robin and bounded wakeup preference | Current source | Not a complete Linux CFS or Windows priority-class implementation |
| Targeted cross-CPU wakeups | Current source and wake test coverage | Wake-to-run latency differs from wake processing cost |
| Shared, interruption-aware wait protocol | Current source and regression work | Environment-specific interruption semantics remain in their respective interfaces |
| Tickless idle and one-shot deadlines | Current source and profiling | Periodic preemption still exists for active CPUs |
| 3.76 µs passage and 0.76 µs wake processing | Dated Proxmox k3s baseline | Historic measurements on 2 vCPUs, not new independent benchmarks |
| Windows and Android workloads under this scheduler | Architectural integration | Windows has original-service evidence; Android remains a partially validated Android implementation |

*Review basis:* Native scheduler and SMP implementation; generic wait, wake and timer facilities; dedicated scheduler wake and SMP-stress test definitions; and the dated k3s idle-baseline performance report. No source listings have been copied into this public chapter. No new runtime or hardware test was performed as part of this review.

---

**The essential message:** myOS does not need one scheduler per operating-system compatibility layer. It runs their threads on **one native SMP scheduling engine**, which is exactly what makes it one operating system rather than multiple systems stitched together.
