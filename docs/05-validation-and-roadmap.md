# Validation, capability maturity and remaining work

myOS is an actively developed, capability-based operating system. Its main architectural result is a **single native kernel** that manages hardware and resources while original Linux, Windows and emerging Android environments supply their respective application-facing contracts. The system has demonstrated substantial original workloads; it is not yet a general-purpose production-certified operating system.

## How to read capability claims

| Evidence level | Meaning |
|---|---|
| Implemented | A mechanism or interface exists; operation is not implied |
| Test written | A reproducible test exists, but may not have been run |
| Reference-passing | A test agrees with the selected upstream reference |
| Differentially conformant | The tested scenario agrees on both the reference and myOS |
| Workload-demonstrated | An original workload or device ran in a recorded environment |
| Stress-validated | Selected concurrency, lifecycle and fault scenarios survived repeated tests |

A capability is never called universally compatible simply because one application launched or its syscall handler exists.

## Current capability assessment

| Area | Present capability | Remaining limits |
|---|---|---|
| Native kernel | SMP scheduling, virtual memory, process/thread identity, IPC, common object lifetime, filesystems and device authority form a functioning foundation | Whole-system reliability, long-duration stress, isolation and broad hardware validation continue |
| Linux execution | Original applications, graphical environments and container workloads have run. Detailed Linux 7.2.11 x86-64 ABI conformance work covers filesystem, memory, thread, signal, credential, namespace, mount, event and debugging behavior | Many fine-grained behaviors and reference comparisons remain incomplete or divergent |
| Windows execution | Original Windows processes, management tools and services execute; NT objects, ALPC, WNF, access tokens and driver interfaces have substantial implementations | Desktop completion, original drivers, service dependencies and concurrency still have open integration cases |
| Device and storage | Common block, filesystem, networking, display and ring-3 driver infrastructure; selected original drivers demonstrated | Controller, driver, crash recovery, I/O durability and hardware coverage vary |
| Android integration | AOSP userspace integration on the common native foundation, including Android-specific interface work | Full framework, display and application lifecycle are not established by the documented acceptance evidence |
| Cross-environment operation | One scheduler, resource model and hardware authority support multiple execution environments | Simultaneous workload, abuse, isolation and recovery matrices require continuing validation |

## Linux compatibility: what the catalogue really means

The reference programme enumerates **385 non-x32 x86-64 system-call entries** from a fixed Linux 7.2.11 configuration. The catalogue includes obsolete and configuration-disabled entries. It is an inventory, not a count of fully implemented or validated syscalls.

A complete behavioral claim requires checking the accepted and rejected flags, argument widths, error precedence, permissions, lifecycle across fork/exec/close/exit, cross-process observations and concurrency. A case passing on upstream Linux is not yet a myOS pass. A case passing on both covers the tested contract, not every reachable behavior. Some security decisions intentionally differ from stock Linux and remain declared policy rather than silently relabeled conformant.

Coverage must separate actual syscall invocation, implementation branch execution, upstream contract coverage and cross-syscall interactions. In particular, access controls such as Landlock and discretionary permissions need enforcement tests when *other* filesystem, networking and process calls invoke them.

## Windows compatibility and evidence

Original Windows applications, native services and selected driver workloads demonstrate important NT-compatible execution paths. The ongoing refinement of ALPC, WNF, handles, access tokens and graphics prerequisites targets precise object lifetime, error behavior and synchronization. A recorded driver or GUI success does not establish that every driver, desktop path or Windows release operates correctly. Known Windows GUI and acceptance-test issues remain open.

## Shared native architecture

A compatibility requirement is satisfied in the execution environment when it concerns Linux- or Windows-specific ABI policy; reusable resource, scheduling, memory, VFS and hardware operations belong in the common kernel. Recent work on hardware debugging, thread context, directory-node identity, file-description sharing and native memory objects is part of the same design, not a second kernel or a duplicated compatibility stack.

## Open acceptance work

The most important next step is a repeatable, versioned end-to-end test matrix: original Linux and Windows workloads, the Android boot/framework milestones, multicore concurrency, resource exhaustion, permission boundaries, driver restart and storage durability. Tests must distinguish the reference kernel from myOS and preserve both positive results and known divergences. Subsystem fuzzing and static analysis strengthen assurance but do not prove the absence of defects.

Historic demonstrations cited elsewhere in the documentation describe what ran in those particular configurations. They are evidence, not a live automated production-readiness score.

**Project status:** operational research/development operating system with growing compatibility, ongoing integration and incomplete whole-system acceptance. There is no claim of formal security certification, universal ABI equivalence or general-production readiness.
