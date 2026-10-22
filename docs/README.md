# myOS — Public Technical Architecture

**One independent native kernel. Multiple operating-system execution environments.**

The documentation is written in English for operating-system engineers, platform architects and technically experienced readers. It explains mechanisms, architectural boundaries, real workload evidence and known limitations **without source-code listings**.

## Start here

**[One Kernel, Several Operating-System Worlds](10-myOS-the-system-behind-the-interfaces.md)** is the central illustrated narrative. It explains the root architecture PNGs in an intentional order, then distinguishes implementation, demonstrated workloads and development progress.

**[Architecture Coverage Review](19-architecture-coverage-review.md)** checks the publication against the complete comprehensive architecture review, mapping subject areas to chapters and identifying outstanding detailed coverage.

## Core technical chapters

1. [System architecture](01-system-architecture.md) — the native kernel and the responsibility boundaries.
2. [Execution environments](02-execution-environments.md) — original Linux, Microsoft Windows and AOSP user spaces.
3. [Native kernel and security](03-native-kernel-and-security.md) — common objects, IPC, memory, capabilities and DMA.
4. [Drivers, storage, networking and display](04-devices-storage-and-network.md) — shared device and service paths.
5. [Validation and status](05-validation-and-roadmap.md) — evidence levels, workload validation and implementation limitations.
6. [Boot and service lifecycle](06-boot-and-service-lifecycle.md) — native initialization, service startup and failure handling.
7. [IPC, identity and shared resources](07-ipc-identity-and-shared-resources.md) — message transports, waits, lifetime and authority.
8. [Android architecture and maturity](08-android-architecture-and-maturity.md) — development effort and actual AOSP milestones.
9. [Windows driver runtime](09-windows-driver-runtime.md) — WDM execution and original Microsoft-driver limitations.
10. [Complete illustrated myOS story](10-myOS-the-system-behind-the-interfaces.md) — an integrated public architecture explanation.
11. [Native scheduler and SMP](11-one-scheduler-multiple-environments.md) — per-CPU runqueues, waits, timing and measured costs.

## Deep-dive coverage added from the complete architecture record

12. [Kernel security, memory and native authority](12-kernel-security-memory-and-authority.md) — address isolation, mitigations, rights and resource groups.
13. [Native networking, containers and resource groups](13-networking-containers-and-resource-groups.md) — packet dataplane, Linux network conventions, namespaces and cgroup v2.
14. [Ring-3 services, registry and boot policy](14-user-space-system-services-and-registry.md) — native service calls, supervision, registry consistency and persistence.
15. [VFS interception and cross-environment file notifications](15-vfs-interception-and-file-notifications.md) — one change-event source, Linux inotify and Windows directory changes.
16. [Boot, executable loading and platform integration](16-boot-loading-and-platform-foundations.md) — early native boot, capability grants, graphics/input and host ports.
17. [Storage correctness and persistence](17-storage-correctness-and-persistence.md) — block boundaries, GPT, ext4, journals, page cache and ring-3 providers.
18. [Testing, observability and architecture governance](18-testing-observability-and-architecture-discipline.md) — regression, instrumentation, classified limits and technical debt.
19. [Architecture coverage review](19-architecture-coverage-review.md) — completeness assessment and remaining deep-detail gaps.

## Illustrated architecture plates

These layered vector illustrations complement the existing PNGs in the repository root. The PNGs illustrate conceptual relationships; their captions and the related chapters explain the exact current maturity and evidence boundaries.

- [Native system architecture](../diagrams/system-architecture.svg)
- [Driver management and hardware](../diagrams/driver-architecture.svg)
- [Windows driver runtime](../diagrams/windows-driver-runtime.svg)
- [Storage and asynchronous I/O](../diagrams/storage-io-architecture.svg)
- [Native scheduler and SMP](../diagrams/native-scheduler-smp.svg)
- [Android execution integration](../diagrams/android-execution.svg)

Mermaid interaction diagrams are also embedded in selected chapters. Neither type of diagram is a substitute for current source and validated test evidence.

## Source and review scope

**Review date:** 9 October 2026. The documentation separates implemented components from demonstrated end-to-end behavior and identifies where validation is still missing.

The public chapters are intended as **architecture explanations**, not a second source-code reference manual, not proof of universal Linux/Windows/Android compatibility, and not a claim that every architecture decision has equal public depth.
