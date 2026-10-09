# myOS — One Kernel, Several Operating-System Worlds

myOS is an independently developed operating system for x86-64. A single native kernel provides processes, memory, IPC, storage, networking and hardware authority. Linux and Windows execution environments run original software on that foundation; a separate Android development branch is bringing original AOSP user space onto the same architecture.

**[Read the complete, source-checked architecture story](docs/10-myOS-the-system-behind-the-interfaces.md)** — a continuous technical explanation of the diagrams, the design decisions, original workload evidence and known implementation limits.

![myOS complete system architecture](myOS%20Complete%20System%20Architecture%20Diagram.png)

For detailed technical chapters, start at the [documentation index](docs/README.md).

## Why this repository exists

This repository documents the design and observed implementation of myOS for technical readers without publishing source code. Diagrams explain responsibilities; the accompanying text distinguishes implemented and demonstrated features from development work and unfinished integration.

The Windows driver host, Linux driver hosting, shared IPC model, native VFS and Android boot progress receive particular attention in the source-checked story.

*Reviewed on 9 October 2026 against myOS master, the Android development branch and the Windows-driver development branch. Not every box in an illustrative infographic is a working feature.*
