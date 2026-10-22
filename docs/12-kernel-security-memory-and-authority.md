# Kernel Security, Virtual Memory and Native Authority

**How one operating system enforces protection beneath several compatibility interfaces**

*Reviewed 9 October 2026. Topics include address-space layout, memory safety boundaries, stack protection, mitigations, cross-process authority and resource management. This chapter describes architecture and limitations, not certification.*

## 1. The security boundary is native, not Windows or Linux

The fundamental security invariant of myOS is that **identity is not authority**. A Linux user ID, a Linux process capability set or a Windows access token has meaning within the relevant application contract, but none of these identifiers can create an unrestricted native right. An interface may refuse an operation in addition to a native authority check; it must not override that check to grant access.

This distinction is necessary because myOS uses one kernel. Without it, a process that happened to receive an elevated Windows or Linux identity could potentially cross into native device or memory operations that were never meant to be accessible.

An operation can therefore require both the environment's permission and an explicit native resource right. A Linux root process does not automatically become the owner of every native device; Windows service identity does not imply unrestricted machine authority. The same distinction applies to cross-process memory, filesystem node rights, machine reset and networking domains.

## 2. The virtual-memory model

Processes run in native address spaces. Backing memory is represented by independently owned memory objects; mappings place that memory into one or more authorized address spaces. A mapping can have its own permitted access mode, while object rights govern whether a process was allowed to establish the mapping at all.

These are different notions: a process can know an address without having the authority to map or inspect the underlying memory object. Sharing memory requires authorized object or process rights, not guessing a process ID or a virtual address.

### Address-space transitions

A context switch activates the correct address space and security state before the new thread resumes. The generic scheduler preserves architectural thread state, including vector state and the base registers used by runtime libraries for thread-local storage.

The protected kernel address-space layout includes a physical-memory mapping for legitimate privileged access, a relocated kernel image and a dedicated region for kernel-thread stacks. The address of the kernel image and physical-memory mapping is not treated as a permanent fixed public constant.

### Memory object lifetime

The same memory object may be referenced by multiple mappings and handles. Closing one reference does not immediately free pages that another process still maps. Pinned memory, including memory registered with a device I/O channel, has separate lifetime obligations. The system must also prevent a stale identifier from acquiring authority over newly allocated storage.

A lazily committed memory object can be created with no physical backing and populated explicitly by an authorized owner. Unlike a normal anonymous zero-fill-on-demand mapping, accessing an uncommitted hole is an error. That property is important when one ring-3 writer publishes a shared state image to read-only consumers: readers must not silently instantiate private zero pages before the writer commits them.

## 3. Kernel attack-surface reduction

### SMEP and SMAP

Supervisor Mode Execution Prevention and Supervisor Mode Access Prevention are CPU-enforced safeguards around user memory. Their existence does not eliminate every vulnerability: the kernel still needs carefully controlled copying between address spaces.

The user-memory copy boundary is sensitive to concurrent changes. One thread may check a user range while another thread in the same process modifies a mapping. myOS therefore treats faults, page-table changes and controlled exception recovery as part of the memory-access contract rather than assuming a range checked earlier will remain valid forever.

### Guarded kernel stacks

Every live thread relies on a kernel stack for privileged execution and context switching. A dedicated virtual stack mapping with an unmapped guard page is intended to turn an overflow into a diagnosable fault rather than a silent overwrite of adjacent kernel memory. Stack lifetime is coordinated with scheduling: a terminating thread cannot reclaim the stack still used by the processor executing its final switch.

### User ASLR and kernel KASLR

User-space ASLR randomizes placement of relevant Linux program components and mappings. Kernel ASLR relocates the kernel image at boot and chooses non-fixed high-half locations for important mappings. These are different defenses against address predictability; neither is a substitute for memory safety or correct authorization.

### KPTI and speculative-execution mitigations

The architecture includes a CPU-vulnerability model that chooses relevant measures based on platform capability and exposure. Kernel Page Table Isolation limits what kernel address-space content is visible in user-mode translation contexts. The entry and return paths integrate mitigation choices for Meltdown- and Spectre-family hazards.

The design explicitly documents gaps, trade-offs and platform requirements. In particular, mitigation support or measurements on a guest without a particular CPU feature cannot establish that feature's behavior on physical hardware. This architecture should not be described as a formally validated defense against all microarchitectural attacks.

### Fast time reads without surrendering kernel authority

A native clock page supplies neutral timing data. The Linux execution environment can expose its familiar vDSO clock interface on top of that page, avoiding a costly kernel transition for frequent time queries. The kernel remains the authority for the underlying time source; the Linux format is presentation.

## 4. Capability delegation and cross-process operations

Authority travels through deliberately transferred resource references. Capability rights are scoped by object type, and delegation can narrow rights rather than expanding them. This applies when launching a process, supplying a driver with a device, presenting one storage volume to a filesystem provider or allowing another process to inspect state.

Cross-process memory copying is especially sensitive. The ability to inspect a process ID or learn its address is not authority to read or write it. An authorized process reference must identify the target, specify the permitted operation and remain valid throughout the transfer.

A filesystem node is likewise a boundary. A process delegated a subtree should not be able to escape it merely through pathname tricks, symbolic links or renaming. Linux discretionary permissions add another refusal layer on top of native node authority.

## 5. Resource groups are a native concept

A resource group is not fundamentally a Linux cgroup. Native myOS tracks and constrains resources through generic group authority. Linux cgroup v2 presents one environment-specific filesystem and management convention over that system. Other execution environments can participate in accounting without adopting Linux filenames or process-ID semantics.

Resource-group rights distinguish inspecting usage, creating subgroups, modifying limits and assigning processes. Memory charges follow actual object and page ownership; diagnostic fields are deliberately omitted or returned according to their supported semantics rather than invented to satisfy a Linux-looking table.

For the networking and container application of these concepts, see [Native Networking and Container Isolation](13-networking-containers-and-resource-groups.md).

## 6. Isolation is also a hardware property

A driver may be confined in ring 3 while still programming a real device. CPU memory isolation does not prevent the device from issuing DMA requests. Native device claims, permitted page mappings and IOMMU domains protect this separate attack path. The smallest hardware separation boundary may cover a group of devices, not a single device function.

Physical IOMMU protection has documented tests on emulated Intel and AMD implementations; do not generalize this to every physical board.

## 7. Evidence and limits

| Mechanism | Reviewed status | Qualification |
|---|---|---|
| Native capability authority | Implemented across native resources and personalities | A compatibility identity never grants native rights on its own |
| Process and memory objects | Implemented and exercised | Universal single-copy file caching is **not** established |
| Cross-process operations | Dedicated authority and regression tests documented | Test coverage is not a security certification |
| User ASLR / kernel KASLR | Implemented architecture | Randomization is not memory safety |
| SMEP / SMAP / guarded stacks | Implemented architecture and targeted protection work | Platform conditions and edge cases still matter |
| KPTI and mitigation selection | Implemented and tested on documented CPU configurations | Some feature-specific paths remain unverified on physical hardware |
| Native resource groups | Implemented; Linux cgroup projection uses them | Not a complete Linux cgroup-controller implementation |

The central message is not that any protection is perfect. It is that **a single native security authority survives the presence of different operating-system personalities**.
