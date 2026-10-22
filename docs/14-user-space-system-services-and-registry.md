# User-Space System Services, Boot Policy and the Windows Registry

**Why service lifecycle, policy and durable Windows state do not belong inside the generic kernel**

*Review date: 9 October 2026. Covers native service calls, concurrent service runtimes, Windows policy, registry persistence, supervised restarts and system-service coordination.*

## 1. The native kernel is an authority, not the location of every service

myOS is designed to run substantial system responsibilities in ring 3. Drivers, power policy, filesystem providers and compatibility-oriented process services do not necessarily need to execute with kernel privilege. The challenging part is making them cooperate without introducing unbounded custom request systems or granting them excessive rights.

The service architecture therefore separates four responsibilities:

| Responsibility | Owner |
|---|---|
| Native resources and protection | Kernel |
| Starting services, granting startup authority and supervision | Native boot and service infrastructure |
| Environment-specific semantics | The relevant Linux or Windows adapter or policy service |
| Handling potentially blocking work | Concurrent ring-3 service runtimes |

The outcome is more than reducing ring-0 source size: it is preserving one set of generic communication and authority mechanisms while moving policy and format-specific logic to where it can fail independently.

## 2. Native service calls: a reusable boundary between ring 0 and ring 3

A channel transports messages, but privileged code sometimes needs to ask a user-space service to perform work **on behalf of the original requesting thread**. A request may need an answer, cancellation, bounded lifetime and controlled sharing of resources.

myOS has a native service-call mechanism with these requirements. A call can carry operation arguments and explicitly delegated capabilities, and can be linked to other calls where the lifecycle requires it. The service may hold a call while doing other work; the original client can wait for completion, time out, be terminated or lose the service.

The important difference from a loose collection of custom message queues is ownership: the kernel tracks the request's identity, authority and lifecycle while ring-3 code supplies the operation's policy or format semantics.

### Call authority

A service does not inherit all privileges of the caller. A resource can be owned or lent for a limited scope, with the allowed rights determined by the call and its native capability. A resource handed to a service is a real resource reference, not a trusted process identifier or fabricated service message.

### Failure and cancellation

If the service disappears, a pending client must receive a meaningful failure rather than remain asleep forever. If the caller dies, outstanding references and lent authority must be released. Holding a request is distinct from leaking it.

### Current scope

The source design includes a generic native service-call implementation and documented consumers, including Windows-facing policy work. It also records functionality still residing in existing specific call paths. The existence of the mechanism is **not proof that every kernel-to-service interaction has already been migrated**.

## 3. Concurrent service execution

A service handling only one request at a time can become the bottleneck for an entire environment. For example, launching one Windows process may require waiting for original Windows subsystem services, while unrelated processes continue to request work.

The personality runtimes therefore use shared user-space concurrency infrastructure to serve independent requests without placing the whole service behind one long wait. The runtime infrastructure is native and neutral; the Windows and Linux services provide their own protocol and construction semantics.

Concurrency creates correctness requirements of its own: no request may be delivered twice, a service's orderly shutdown must resolve outstanding requests, and slow clients must not exhaust an unrelated service's available execution capacity.

## 4. The boot manifest as the source of truth

A system that depends on user-space services needs reproducible startup and upgrade behavior. The architecture defines one service manifest/catalog as the authoritative inventory of essential services, their artifacts, required resources, authority and activation dependencies.

This means installation media, a boot from an existing disk and an upgrade must agree on which services belong to the installed operating system. A service required after an upgrade cannot be present only in a development startup script and absent on the user's installed machine.

This is also a security mechanism: startup grants are explicit, so a new service does not silently inherit privileged capabilities merely because its executable is included in the distribution.

## 5. Supervision and crash recovery

The native boot service supervises the registered system processes. Restart behavior is declared per service, with explicit rules for never restarting or restarting after failure, plus bounded retry windows and delays where applicable.

Recovery is intentionally not magical. A crashed service may be restarted, but the restart cannot necessarily reconstruct a lost in-memory transaction or continue an arbitrary previous RPC. The architecture distinguishes service process restart, persistent state recovery and client reconnection.

When a repeatedly failing service exhausts its restart allowance, the supervisor must release retained endpoint authority and report a stable failure rather than create an endless restart storm.

## 6. Windows registry: one writer, many read-only consumers

The registry is a strong example of the system's architectural discipline. The original Windows components expect a Windows registry hierarchy, key/value APIs, hive semantics and efficient reads. Those are Windows-specific requirements; they are not reasons for the native kernel to own a general-purpose Windows database.

The architecture documents a move toward a dedicated ring-3 registry service as the single mutable authority over Windows registry state. Windows processes receive read-only views of appropriate registry images for fast queries. Writes are routed to the authorized service.

This choice is backed by an observation: the measured Windows initialization workload made vastly more registry reads than writes. Forwarding every read to one ring-3 service would add unnecessary context switching and contention while providing no security benefit over read-only mapped images.

### Consistency

The read-only image uses a generation convention so readers do not accept a partially published mutation. The service commits changes and publishes consistent state; a reader can detect when a generation changed.

### Persistence

Original Microsoft hive files serve as import sources. The running registry has its own journaled persistent state owned by the registry service, rather than silently editing installation-media hive files.

A write-ahead journal records accepted mutations before the in-memory publication. Journal replay reconstructs the state after a restart; incomplete or corrupted trailing records are excluded. An explicit flush represents a durability boundary, while ordinary acknowledgements follow the filesystem's normal persistence guarantees.

### Least privilege

The registry writer receives access to its state directory, not unrestricted access to every Windows process or all files. Windows processes do not receive writable capabilities to their shared registry images.

### Evidence boundary

The source describes an acceptance scenario involving changes, an explicit flush, forced VM termination, reboot and comparison. This documents the test contract and implementation rationale; it should not be generalized into a claim of complete compatibility with every Windows registry operation and failure mode.

## 7. Power management provides the same pattern

The power service owns policy: orderly reboot, poweroff and reset, including the decision to synchronize storage first. A separate authorized ACPI provider performs hardware-specific operations. The native kernel supplies only restricted machine, firmware and I/O-port primitives.

Poweroff and reboot have documented QEMU/OVMF tests. Suspend and hibernation are not yet complete, and Windows power APIs are not fully delegated to the native service. The architecture deliberately keeps those limitations explicit.

## 8. Why these services matter to myOS

They demonstrate a consistent method for adding large compatibility surfaces:

**Keep shared mechanisms in the native kernel, execute long-running policy and format-specific work in ring 3, and transfer only explicit authority between them.**

The same principle explains drivers, filesystem providers, Windows policy, registry state, power control and system startup.
