# myOS — Public technical documentation

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
