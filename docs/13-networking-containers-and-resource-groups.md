# Native Networking, Virtual Networks and Container Isolation

**Why containers and operating-system compatibility use the same native networking machinery**

*Source: myOS master architecture, networking Phase X, virtual link and network-domain records, packet filtering, seccomp, PID namespace and cgroup v2 decisions, and the documented k3s workload evidence. Review date: 9 October 2026.*

## 1. One network stack, multiple API surfaces

The myOS network architecture intentionally separates actual packet processing from the Linux-specific vocabulary used to configure it. Ethernet and IP processing, address ownership, route selection, native socket transport and network domains are generic kernel services.

Linux tools use sockets, network-device identifiers, rtnetlink messages and familiar configuration files. A Windows client expects NT and Windows networking interfaces. These are ways of describing or controlling shared resources, **not separate TCP/IP implementations**.

The implemented native stack includes Ethernet framing, ARP, IPv4 and IPv6 facilities, ICMP, neighbor discovery, router discovery/SLAAC, UDP and TCP. Device-specific adapters expose network packets and state to generic interfaces. Host tools and container software are layered above that architecture.

## 2. Network interfaces, link devices and network domains

Three abstractions are important:

- A **link device** talks to the underlying packet-carrying device or software link.
- A **network interface** provides address, status, routing and related interface behavior.
- A **network domain** is the native boundary for a set of network interfaces, sockets, configuration state and associated authority.

A Linux network namespace is a translation onto native domains, not an unrelated second network stack. The personality is responsible for Linux namespace creation and interface naming, while native resource rights still limit the operations available in the target domain.

An existing socket is associated with a network domain. Moving a process into a different Linux-visible namespace does not automatically retroactively transfer existing sockets into that domain.

## 3. Container networking from end to end

The architecture was driven by real requirements of unmodified container software rather than by hypothetical network features. k3s, containerd, runc, Flannel and CNI tools need Linux-visible network devices, bridges, virtual Ethernet pairs, routing changes, filtering and address assignment.

The native implementation provides virtual links, switches and domain lifecycle primitives. Linux rtnetlink translates Linux device-management requests into those operations.

Consider a container sending data outward:

1. A process sees its own Linux namespace and configured virtual interface.
2. The compatibility layer resolves its socket and interface state onto native network resources.
3. A native virtual link carries frames to a native software switch or routing path.
4. The native filtering and flow-tracking mechanisms evaluate the applicable configuration.
5. The selected physical or virtual network driver transmits the packet through the common device service.

No step requires the kernel network core to understand a Linux rtnetlink structure, a Windows firewall rule format or an Android-specific interface name.

Some tunnel and overlay abstractions in the architecture have explicitly limited support; the existence of a virtual-link representation is not proof of a complete VXLAN, overlay or Kubernetes networking feature set.

## 4. Native filtering and protocol adapters

Packet filtering is built from neutral concepts: hook locations, ordered rule lists, packet fields, match conditions, actions and counters. A Linux nftables/iptables adapter translates Linux configuration commands and transactions into these generic structures.

Rules can be attached to receive, forwarding, local-input, local-output and transmit paths. The architecture covers matching over generic packet metadata and actions such as counting, permitting, denying, redirecting or translating traffic, with bounded or rate-based behavior where implemented.

This design matters for Windows integration. A future or partially implemented Windows filtering interface should project its semantics onto the **same authoritative native dataplane**, rather than adding an independent Windows-only packet filter.

The source architecture also distinguishes supported Linux compatibility requests from messages, expressions or netfilter subsystems that currently return an unsupported result. It is therefore incorrect to say that the full Linux kernel networking control plane or Microsoft Windows Filtering Platform is already implemented.

## 5. Connection state and translation

A stateful firewall needs more than a packet matcher. Connection tracking must understand how packets belong to the same flow, when that association becomes stale, and how translation affects subsequent packets. myOS separates generic flow-tracking and translation mechanisms from the Linux adapter that describes the rules.

This creates a reusable basis for NAT and connection-related policy but does not mean every Linux conntrack tool, nft expression or kube-proxy scenario has been validated end to end.

## 6. Network change notifications

An application may subscribe to configuration changes without polling interface state. The native model emits a neutral configuration-change event scoped to the appropriate network domain. Linux projects those changes into rtnetlink notifications. A Windows networking API can use the same underlying source while choosing its own event and buffer conventions.

This follows the same overall system principle as file-change notification: the *thing that changes* produces a neutral event; the personality translates the event for its caller.

## 7. Containers: process and resource isolation

Linux-compatible containers exercise much more than networking. Their process startup and runtime environment depend on namespace rules, mounts, signal delivery, filesystem permissions, seccomp and resource controls.

### PID namespaces

Native process identity does not change when a Linux process is presented as PID 1 in a newly created namespace. The Linux personality owns the mapping between namespace-visible PIDs and native processes, including the lifecycle rules required by fork, clone, wait and process inspection.

### Seccomp

Linux seccomp filters use Linux-defined syscall numbers, result actions and instruction semantics. myOS implements those Linux contracts in the personality rather than exposing the Linux filter format as a generic native kernel policy language. A native resource right is still needed for operations that require authority; passing a seccomp filter cannot grant it.

### Cgroup v2 over native resource groups

The Linux cgroup filesystem presents hierarchy, controls, accounting values and limits expected by real container software. Native resource groups are the underlying accounting and authority model.

Not every Linux controller or statistic exists. A field with no supported measurement must not be represented as a fabricated number, and an accepted control must not be presented as enforced unless its underlying native mechanism applies it.

This distinction is especially important for memory limits: ownership, charging, migration and reclamation must agree with actual memory objects and native lifecycle. A Linux cgroup file is only a view of this shared enforcement system.

## 8. k3s as an integration workload

The repository's k3s records capture not just “the process started” but a sequence of observed system-level behaviors: service startup, node readiness, container runtime activity, networking, filesystem operations and system pods. In one historical QEMU/KVM run the node reached Ready after approximately 15 seconds, and a separate Proxmox baseline captured scheduler and kernel overhead for a running set of pods.

Those are meaningful cross-subsystem tests, but they do not demonstrate every Kubernetes networking backend, security policy or cloud deployment configuration. The performance records are revision- and environment-specific.

## 9. Architectural takeaway

The network design is important because it prevents the Linux personality from accidentally becoming the network owner merely because k3s was the demanding first consumer. Network-domain identity, filtering, resource rights, device transfer and flow state are native. Linux, Windows and Android-facing conventions can be supported **without duplicating the network stack**.
