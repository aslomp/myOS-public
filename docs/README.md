# myOS — Technical documentation

**Start here: [One Kernel, Several Operating-System Worlds](10-myOS-the-system-behind-the-interfaces.md).**

This is the primary English-language narrative. It walks through the architectural illustrations in the repository root and explains what the reviewed source and recorded acceptance work actually demonstrate. It also identifies where the illustrations contain examples, untested contracts or aspirational coverage.

For subsystem detail, continue with the chapters below.

This documentation explains **how myOS works and why its architecture is different**, without publishing source code or implementation listings. It is intended for operating-system engineers, infrastructure engineers, platform architects and other technical readers.

## Reading guide

1. [System architecture](01-system-architecture.md) — the single-kernel model and separation of responsibilities.
2. [Execution environments](02-execution-environments.md) — how Linux, Windows and Android software can share one native operating system.
3. [Native kernel and security](03-native-kernel-and-security.md) — process identity, memory, IPC, authority and IOMMU isolation.
4. [Drivers, storage, networking and display](04-devices-storage-and-network.md) — the hardware and service architecture.
5. [Validation and roadmap](05-validation-and-roadmap.md) — independently distinguishable capabilities, observations and remaining work.
6. [Boot and service lifecycle](06-boot-and-service-lifecycle.md) — initialization, driver readiness and environment startup.
7. [IPC, identity and shared resources](07-ipc-identity-and-shared-resources.md) — the common object model beneath distinct operating-system conventions.
8. [Android architecture and maturity](08-android-architecture-and-maturity.md) — the development branch, original AOSP components and measured boot status.

All diagrams use GitHub-rendered Mermaid. They are conceptual architecture diagrams, not internal class or function diagrams.

**Source review:** myOS master and `claude/android-x86-64-native-624c9f` as inspected on 9 October 2026. The Android branch represents development work, not features merged into master. Tested evidence and design ambitions are separated throughout.

## Illustrated architecture plates

The repository also includes detailed, independently viewable vector diagrams. These complement (rather than replace) the Mermaid process and interaction diagrams in the chapters.

- [Complete system architecture](../diagrams/system-architecture.svg)
- [Driver management and hardware authority](../diagrams/driver-architecture.svg)
- [Storage and asynchronous I/O](../diagrams/storage-io-architecture.svg)
- [Android integration and evidence status](../diagrams/android-execution.svg)

The illustrated plates use a layered, color-coded documentation style. Unlike a generic flowchart, they group participating components by responsibility and show the interfaces between execution environments, native mechanisms and services.

## How to interpret the diagrams

A box shows an architectural responsibility, not necessarily a separate kernel module or process. A line indicates conceptual dependency or authority flow and is not proof that the depicted subsystem currently passes an end-to-end acceptance test. The Android diagrams explicitly mark the branch's incomplete framework and graphics acceptance. For current implementation details and open questions, read each corresponding chapter's evidence and limitations.


9. [Windows driver-host implementation](09-windows-driver-runtime.md) — tested WDM execution, native-backed AFD/TCPIP and boundaries of original Microsoft driver support.

![Windows driver-host architecture](../diagrams/windows-driver-runtime.svg)
