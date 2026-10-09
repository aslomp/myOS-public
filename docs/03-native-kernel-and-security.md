# Native kernel, isolation and security

The native kernel is the common operating-system authority. It provides process and thread identity, scheduling, virtual memory, synchronization, IPC, files, networks and device-resource access. The execution environments translate their APIs to these facilities.

```mermaid
flowchart TB
  E["Linux / Windows / Android entry points"]
  P["Personality contract and policy checks"]
  C["Native capabilities and resource identity"]
  subgraph N["Kernel mechanisms"]
    S["Scheduler and threads"]
    M["Address spaces and memory objects"]
    I["Channels, endpoints and completion"]
    F["VFS and file-backed pages"]
    NET["Network domains and packet processing"]
    D["Device claims and DMA authority"]
  end
  E --> P --> C
  C --> S
  C --> M
  C --> I
  C --> F
  C --> NET
  C --> D
```

## Process and memory model

Processes use native address spaces and memory objects. Executable loading and mapping are separated from the core ownership model. Multicore scheduling, synchronization, wakeups and virtual-memory translation are kernel services, not implementations duplicated by each personality. File-backed memory and page-cache integration are intended to avoid unnecessary independent copies of identical file contents.

## Communication and synchronization

Native channels, endpoints, readiness, wait mechanisms and completion facilities form the underlying transport. Linux sockets, pipes, event facilities and Windows ALPC/NT wait contracts project their respective semantics onto native primitives. This allows the transport and resource lifetime to be reviewed independently from an operating-system-specific message format.

## Capability-oriented authority

A process can operate on a resource only within the rights available to it. Device claims, mappings and interrupt delegation are specific examples. Driver termination must release ownership rather than leave unrestricted access behind.

## Hardware-backed DMA isolation

```mermaid
flowchart LR
  DR["User-space driver"] --> CLAIM["Device claim and authorized DMA mapping"]
  CLAIM --> DMA["Native DMA authority"]
  DMA --> DOMAIN["IOMMU domain / isolation group"]
  DOMAIN --> DEVICE["Device DMA requests"]
  DOMAIN --> MEM["Authorized physical-memory pages"]
```

Intel VT-d and AMD-Vi support are documented as exercised on emulated IOMMU hardware. The **isolation group** is the real hardware boundary: devices that hardware cannot distinguish may share a translation domain, even when owned by different driver processes. On configurations without an IOMMU, software ownership checks cannot provide the same protection from malicious device DMA. Physical-hardware coverage is not established by the emulated tests.

## Protection and policy

The documented design includes address-space protections and mitigation work, security-aware process authority, and environment-specific access-control enforcement. Android additionally requires enforcement of its original SELinux policy. Protection properties must be evaluated end-to-end; adding an API does not automatically prove containment across every environment.

## Current risks

The Android branch audit identifies pending environment-isolation issues involving writable machine-wide configuration, some globally visible metadata, incomplete peer-domain checks for selected devices, and unverified regression behavior. These are material integration risks, not generic theoretical caveats. The kernel should be described as architecturally security-conscious, **not certified secure**.

---

## Capability and memory authority in depth

A native resource has an identity and lifecycle independent of any environment. Its handle confers a specific set of operations; possession of a copied identifier or an operating-system-specific name is not, by itself, authorization. When a resource is shared, the question is whether its rights were deliberately delegated.

### Kernel memory versus driver-accessible memory

```text
       Ring-3 driver                 Native kernel                Hardware
       -------------                 -------------                --------
       requests a claim ----------> validates device owner
       requests mapped pages -----> checks handle + rights
                                     pins allowed pages
                                     programs IOMMU domain -----> DMA translation
       receives device address <--- authorized mapping
       submits hardware work -----------------------------------> reads/writes
       closes or terminates ------> unmaps, releases claim -----> DMA blocked
```

IOMMU enforcement is crucial because otherwise a device can perform DMA independently of the CPU's process page tables. The smallest hardware isolation unit may be a *group*, not one PCI function. Members of a group can potentially access the group's mappings even if different driver processes own those members. Software ownership and hardware separation therefore require different claims and threat assessments.

### Scheduling, waiting and completion

The native scheduler decides when threads run; native waits and endpoints communicate *why* a thread can proceed. A Linux futex, an NT wait and an Android scheduling requirement may use shared infrastructure but still need distinct return semantics. Lost wakeups, stale completion identity and incorrect cancellation can deadlock original service infrastructures long before visible applications appear.

### Mandatory access control

Android's original SELinux policy is enforced through its Linux-facing security semantics while generic authority remains native. SELinux labels and domain decisions are not a replacement for capabilities, nor may capability possession silently defeat a mandatory policy where the Android contract requires it. The Android audit documents open isolation and label-correctness issues; these are operational boundaries that need tests with hostile or malformed peers.

### Mitigation expectations

Address-space isolation, CPU mitigations, W^X policy, memory lifetime, service privilege reduction and IOMMU protection are distinct dimensions. None implies all others. Validation should explicitly include unauthorized DMA, permission escalation through handle transfer, namespace escape attempts and device-owner death, rather than only confirming that successful requests work.
