# myOS — One Kernel, Several Operating-System Worlds

**One native kernel. Multiple operating-system environments. Original applications. Shared hardware.**

myOS is an independently developed, capability-based operating system for x86-64, designed to execute original Linux and Windows software on a single native kernel, with Android integration under active development.

It is not a Linux distribution, a Windows compatibility layer running on Linux, or a hypervisor running separate guest operating systems.

Instead, myOS implements the underlying operating-system mechanisms itself and exposes the interfaces expected by different software ecosystems.

**[Explore the architecture](docs/10-myOS-the-system-behind-the-interfaces.md) | [Technical documentation](docs/README.md)**

![myOS complete system architecture](myOS%20Complete%20System%20Architecture%20Diagram.png)

## The idea

Traditional operating systems tightly couple their kernel architecture with the interfaces expected by applications.

myOS separates these concerns.

Processes, threads, virtual memory, scheduling, inter-process communication, device ownership and hardware resources are managed by a single independent kernel.

Linux, Windows and Android environments translate their respective operating-system contracts onto these shared native mechanisms.

The fundamental design principle is:

**Implement generic mechanisms once. Implement operating-system-specific semantics where necessary.**

This allows different operating-system environments to coexist without requiring separate kernels, virtual machines or duplicated hardware ownership.

## Architecture highlights

| Component | Approach |
|---|---|
| Kernel | Independently developed, capability-based Rust kernel |
| Execution | Native x86-64 processes and shared scheduling |
| Linux | Linux system-call compatibility and original ELF applications |
| Windows | NT system-call compatibility and original Windows components |
| Android | AOSP integration through a separate development effort |
| Drivers | Native hardware authority with user-space driver hosting |
| Memory | Shared native memory management and address-space mechanisms |
| IPC | Native communication primitives supporting environment-specific contracts |
| Storage and networking | Shared infrastructure exposed through different operating-system interfaces |

The design emphasizes kernel independence, architectural consistency and reuse of native mechanisms.

## Engineering progress

**[Implementation update — 22 October 2026](docs/05-validation-and-roadmap.md#implementation-update--22-october-2026):** a dated technical-status update for Windows execution, NDIS and IPC, distinguishing implemented mechanisms from independently validated behavior.

myOS is an active engineering project, not a finished general-purpose operating system.

Documented development and validation include:

- Linux user-space execution and substantial real-world workloads, including desktop and container infrastructure.
- Original Windows services and applications running through native compatibility mechanisms.
- Native multiprocessor scheduling, memory management and inter-process communication.
- Device management and driver-hosting mechanisms.
- Android integration and original AOSP user-space development.

Capabilities differ in maturity. Implemented functionality, validated workloads, experimental features and incomplete integration are distinguished in the technical documentation.

**[Read the implementation evidence and known limitations](docs/05-validation-and-roadmap.md)**

## Technical documentation

The repository contains detailed architectural explanations, diagrams, design decisions, implementation observations and validation results.

**Start here:**

- [Complete illustrated architecture](docs/10-myOS-the-system-behind-the-interfaces.md)
- [System architecture](docs/01-system-architecture.md)
- [Execution environments](docs/02-execution-environments.md)
- [Native kernel and security](docs/03-native-kernel-and-security.md)
- [Drivers, storage and networking](docs/04-devices-storage-and-network.md)
- [Native scheduler and SMP](docs/11-one-scheduler-multiple-environments.md)
- [Architecture coverage review](docs/19-architecture-coverage-review.md)

**[Browse all technical chapters](docs/README.md)**

## Why publish the architecture?

myOS is an independent operating-system engineering effort.

This repository makes its architecture, design decisions, technical challenges and measured progress publicly accessible for study, discussion and technical evaluation.

The purpose is to share the engineering behind the system, rather than simply present screenshots or feature claims.

The implementation source code is not published in this repository.

## Author

**Arend Slomp**  
Independent software engineer and creator of myOS.

[LinkedIn — Arend Slomp](https://www.linkedin.com/in/arendslomp/)

## Copyright and permissions

Copyright © 2026 Arend Slomp.

The documentation may be copied and redistributed for non-commercial purposes, provided that the original repository is clearly credited and linked.

Commercial sale or exploitation of the documentation requires prior written permission.

**[Full copyright and permissions](COPYRIGHT.md)**

---

*myOS is under active development. Architecture documentation describes the design and evidence available at the stated review dates; it does not imply universal application or driver compatibility.*
