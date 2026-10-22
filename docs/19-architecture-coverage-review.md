# Architecture Coverage Review — myOS implementation

**Audit of the public documentation against the complete architecture review**

*Reviewed: 9 October 2026. This is an assessment of public documentation coverage, not an independent execution of every test.*

## Executive result

**The original public documentation did not cover the complete architecture.** It explained the strongest system-level themes—one native kernel, Linux and Windows personalities, an Android development effort, user-mode drivers, native storage, IPC and scheduling—but omitted or underexplained major architectural decisions.

This review led to added public chapters on native security and memory authority, networking and containers, ring-3 system services and registry persistence, VFS interception and change notification, native boot and platform integration, storage correctness, and engineering validation.

The expanded publication now gives a technical reader an entry point into **every major architectural domain** covered by the architecture review. That is not the same as reproducing all design decisions in equal implementation detail: the engineering review includes historical milestones, accepted decisions, measurements, alternative designs, limitations and some explicitly proposed work.

## Coverage map

The status labels below indicate *documentation depth*, not whether every feature has passed an acceptance test.

| Architecture domain | Public documentation | Depth | Remaining detail worth documenting |
|---|---|---|---|
| Native objects, memory, threads and capabilities | [System architecture](01-system-architecture.md), [Security and memory](12-kernel-security-memory-and-authority.md) | Explained | Complete native object lifetime and delegation matrix |
| First userspace bootstrap and authority grants | [Boot and loading](16-boot-loading-and-platform-foundations.md) | Summary | Exact bootstrap invariants and recovery transitions |
| User-space executable loading and format boundary | [Execution environments](02-execution-environments.md), [Boot and loading](16-boot-loading-and-platform-foundations.md) | Explained | All executable construction and failure/rollback paths |
| Native console, shell and terminal projection | [Boot and loading](16-boot-loading-and-platform-foundations.md) | Summary | Full terminal and boot-console state machine |
| VirtIO transport, device protocols, interrupts | [Drivers, storage and display](04-devices-storage-and-network.md), [Boot and loading](16-boot-loading-and-platform-foundations.md) | Explained | Transport memory-ordering and recovery details |
| Native driver ABI and device catalog | [Drivers, storage and display](04-devices-storage-and-network.md) | Explained | Per-class device-host acceptance evidence |
| Graphics, boot screen, DRM and input | [Drivers, storage and display](04-devices-storage-and-network.md), [Boot and loading](16-boot-loading-and-platform-foundations.md) | Summary | Display ownership and presentation integration in full depth |
| Hypervisor host ports / guest-agent path | [Boot and loading](16-boot-loading-and-platform-foundations.md) | Summary | Service lifecycle across transports and disconnection |
| Clocks, deadlines and native power service | [Scheduler](11-one-scheduler-multiple-environments.md), [System services](14-user-space-system-services-and-registry.md) | Explained | Clock-source failover and precise firmware constraints |
| Generic block devices and GPT partition views | [Storage correctness](17-storage-correctness-and-persistence.md) | Explained | Partition recovery policy and malformed-media test matrix |
| ext4 allocation, directory operations and journaling | [Storage correctness](17-storage-correctness-and-persistence.md) | Summary | Transaction ordering, recovery, on-disk interoperability and unsupported feature matrix |
| Physical memory layout, usercopy and protected stacks | [Security and memory](12-kernel-security-memory-and-authority.md) | Explained | Detailed page-table/TLB consistency proof |
| ASLR, KASLR, KPTI and speculative mitigations | [Security and memory](12-kernel-security-memory-and-authority.md) | Explained | CPU-specific measurements and mitigations still unverified on hardware |
| Native capability and cross-process authority | [Security and memory](12-kernel-security-memory-and-authority.md) | Explained | Formal authority invariants for every resource type |
| Testing architecture and kernel lock ordering | [Engineering validation](18-testing-observability-and-architecture-discipline.md) | Explained | Complete matrix of individual regression suites |
| Scheduler SMP, lifecycle, waits and reclamation | [Scheduler](11-one-scheduler-multiple-environments.md) | In depth | More comparative workload traces if available |
| Linux personality and syscall ABI | [Execution environments](02-execution-environments.md) | Summary | Complete Linux ABI contract and unsupported-case reference |
| Linux fork/exec, dynamic linking, signals and terminals | [Execution environments](02-execution-environments.md), [Boot and loading](16-boot-loading-and-platform-foundations.md) | Summary | Full thread/signal/exec transition diagrams |
| File-backed memory, page cache and writeback | [Storage correctness](17-storage-correctness-and-persistence.md) | Explained | Shared-memory coherence and remaining double-caching cases |
| Linux credentials, seccomp and namespaces | [Security and memory](12-kernel-security-memory-and-authority.md), [Networking and containers](13-networking-containers-and-resource-groups.md) | Explained | Specific namespace ABI and security edge cases |
| Native resources and Linux cgroup v2 | [Networking and containers](13-networking-containers-and-resource-groups.md) | Explained | Controller-by-controller evidence and accounting cases |
| Native networking, IPv4/IPv6, TCP/UDP and link model | [Networking and containers](13-networking-containers-and-resource-groups.md) | Summary | Protocol and packet-path detail from extensive Phase X record |
| Container virtual links, domains and notifications | [Networking and containers](13-networking-containers-and-resource-groups.md) | Explained | Complete CNI/flannel and container-network acceptance matrix |
| Filtering, NAT, nftables translation and ring-3 netctl | [Networking and containers](13-networking-containers-and-resource-groups.md) | Summary | Supported expressions, protocol extensions and live tool compatibility |
| Native message endpoints, ALPC and Linux IPC | [IPC and shared resources](07-ipc-identity-and-shared-resources.md) | In depth | Cancellation and completion cases from the long IPC ADR |
| Native wait-on-address and readiness | [IPC and shared resources](07-ipc-identity-and-shared-resources.md), [Scheduler](11-one-scheduler-multiple-environments.md) | Explained | Per-ABI behavior and test traceability |
| Windows PE runtime and original services | [Execution environments](02-execution-environments.md), [Main story](10-myOS-the-system-behind-the-interfaces.md) | In depth | Exact NT/Win32 API support and current desktop blockers |
| Windows WDM driver hosting | [Windows driver host](09-windows-driver-runtime.md) | In depth | Original Microsoft-driver support evidence per image |
| Native service calls and policy services | [System services](14-user-space-system-services-and-registry.md) | Explained | Full operation, lending and lifecycle semantics |
| Concurrent ring-3 runtime services | [System services](14-user-space-system-services-and-registry.md) | Explained | Failure/backpressure and startup-fanout traces |
| Intentional policy limits vs accidental ceilings | [Engineering validation](18-testing-observability-and-architecture-discipline.md) | Explained | Specific bounds currently under reconsideration |
| Windows registry, persistence and read-only views | [System services](14-user-space-system-services-and-registry.md) | Explained | Detailed generation protocol and crash-replay evidence |
| Supervised restarts and system-service manifest | [System services](14-user-space-system-services-and-registry.md) | Explained | Every service's effective restart rule |
| Native VFS operation interception | [VFS interception and notifications](15-vfs-interception-and-file-notifications.md) | In depth | Outstanding operation classes not intercepted |
| VFS change notifications; Linux and Windows adapters | [VFS interception and notifications](15-vfs-interception-and-file-notifications.md) | In depth | Buffer overflow, rename and cancellation test matrix |
| Technical debt and evidence governance | [Engineering validation](18-testing-observability-and-architecture-discipline.md) | Explained | Publication-level material-debt ledger |

## Corrections to the public diagrams' implied claims

The root PNG illustrations are architectural explanations, not an accurate Boolean compatibility matrix. Their contents must be read with the documented evidence qualifications:

1. **Filesystem filter compatibility:** the original Microsoft Filter Manager has recorded successful startup and selected minifilter I/O acceptance. Complete compatibility across arbitrary filters and filesystem operations remains unverified.
2. **General Windows driver support:** WDM image hosting and real IRP tests are demonstrated; universal NT inbox driver, WDF, filesystem, USB and graphics driver operation is not.
3. **Fully unified filesystem caching:** a generic cache and shared file-backed objects exist, but some mmap/file-cache paths can still occupy separate frames.
4. **All filesystem formats:** ext4 and selected ring-3 read-only providers have evidence. Named formats elsewhere in a conceptual storage diagram are not automatically implemented.
5. **Android desktop:** authentic AOSP boot reached enforcing SELinux and original services, but the audited run stopped before framework and GUI startup.
6. **Scheduler policy:** per-CPU queues, balancing and bounded wakeup preference exist. General CPU hotplug, complete NUMA-aware placement and broad real-time/NT priority semantics are not established.
7. **Network control:** a native dataplane and Linux control interfaces exist, with significant translation moved to a ring-3 service. This does not establish full Windows filtering or every Linux netfilter extension.

## Material architectural insights that were missing before this audit

Several mechanisms deserve to be mentioned alongside scheduler, storage, IPC and drivers in any comprehensive technical description:

- **Native service calls** allow controlled ring-0 to ring-3 delegation on behalf of a blocked thread, with explicit capabilities and cancellation/lifetime control.
- **Resource groups** are native authority and accounting constructs. Linux cgroup v2 is one compatible view, not their owner.
- **One network dataplane** serves native sockets and virtual networking; Linux netlink and nftables conventions are adapters and can be translated outside the kernel.
- **One filesystem change source** supplies both Linux inotify and Windows directory changes, while interception is a different, earlier authorization mechanism.
- **The Windows registry service** owns mutation and journaled persistent state, while consumers get read-only views.
- **The system-service manifest** governs activation, upgrade inclusion, capabilities and restart policy across installed systems.
- **Native executable loading** delegates image grammar and OS-specific runtime construction to user space rather than growing a Linux/PE loader inside the kernel.
- **Architectural limits are classified.** Resource policy and protocol bounds are deliberate; arbitrary fixed-size data structures must not become accidental operating-system limits.

## Remaining depth gaps

The expanded documentation **does not reproduce every line of the comprehensive source record**. That would produce another internal implementation manual and would conflict with the public repository's purpose.

Material next-level areas, should more public depth be required, include:

- A subsystem-by-subsystem security and authority matrix with individual denial tests.
- An exact ext4 crash consistency and unified caching analysis with workload measurements.
- A complete native packet-path and container-networking explanation, including what each original network tool actually exercises.
- An interactive desktop/input/display architecture, including ownership transitions and original Windows graphics dependencies.
- A reproducible hardware and hypervisor compatibility table distinguishing QEMU, Proxmox, Hyper-V and actual devices.
- A structured evidence index linking every public status claim to a dated test or recorded original workload.

## Bottom line

The public repository now explains the major architecture decisions and the philosophy behind them. It is appropriate as a public **technical architecture narrative**, with honest limitations. It should **not** be labelled an exhaustive line-by-line specification of every subsystem or a complete independent acceptance audit.

The source of truth for low-level invariants, historical choices, test conditions and remaining debt remains the internal myOS architecture and test documentation.
