# Testing, Observability and Architecture Discipline

**How myOS evaluates mechanisms without confusing a design decision with a working subsystem**

*Reviewed against myOS implementation architecture, its test-architecture and lock-ordering records, its measured performance reports and the stated technical-debt process. This chapter discusses current engineering practices, not an independent test certification.*

## 1. Three separate forms of evidence

myOS distinguishes between:

- **Production correctness rules** that must remain active whether or not a diagnostic or test build is used.
- **Test implementations** that deliberately provoke boundary conditions, races, failures and malformed inputs.
- **Execution and acceptance procedures** that demonstrate a particular binary, device, virtual machine or workload behaving correctly.

Keeping these separate matters. A safety check must not disappear simply because a production image omits stress-test instrumentation. Conversely, a feature appearing in a unit-test target is not evidence that it survives a full system boot under a workload.

## 2. The first-user-space path must remain real

A system can accidentally pass initialization tests by invoking privileged test shortcuts rather than using its production mechanisms. The myOS architecture consequently treats the real transition from the kernel into the first native user-space process, process construction, resource delegation and normal system calls as part of the testable execution path.

That is especially important for a project that hosts different executable environments. Success at parsing ELF or PE is insufficient; those processes must use real memory mappings, actual native scheduling, valid IPC and controlled resource grants.

## 3. Kernel-level correctness and stress tests

The repository documents targeted test suites for synchronization, SMP scheduling, authority boundaries, memory objects, filesystems, storage, DMA isolation and other primitives.

Multicore tests do more than verify that CPUs are online: they create shared work, exercise migration and check that one thread is neither lost nor executed twice. A specific wakeup regression tests the return path from a local device interrupt, where a thread previously could remain waiting until the next timer tick even though a wake had already occurred.

The negative cases are important. Tests must ask whether a caller **without** a capability can reach another process's memory, whether a stale handle is rejected, whether an invalid buffer remains confined, and whether cleanup occurs after a process or service dies.

## 4. Lock ordering and wait discipline

The architecture treats lock ordering as a correctness contract. Operations that allocate, block, call services or acquire other locks must not inadvertently do so while holding a lock that forbids them.

In particular, wait-table registration and event delivery require a safe relationship between the source lock, wait registration and thread blocking. A service request performed from inside the VFS must not retain a VFS lock that the service itself needs to answer.

This is a practical reason to centralize generic synchronization. Duplicated personality-specific wait loops can develop slightly different locking orders and failure behavior that are difficult to validate together.

## 5. Measured performance, with a named workload

myOS includes dated performance investigations rather than just isolated benchmark numbers. They identify the test platform, vCPU count, guest hardware, target revision, instrumentation mode and workload.

The k3s investigations are examples of measuring a complete operating system under real pressure: a node reaching Ready, container-image extraction, sustained filesystem metadata work, network activity, wake storms and the cost of APIC operations. The Proxmox idle baseline also separates user CPU time, kernel CPU time and host VM overhead.

A scheduler's CPU cost per passage and the elapsed time from **wake to execution** are different quantities. Likewise, a fast block-driver completion is not the same measurement as a filesystem transaction's end-to-end latency.

## 6. Observability should not define policy

Counters, timing categories, thread wait records, profiling and trace views help explain why a system is slow or stuck. Their presence must not become a precondition for normal scheduling, memory safety or resource lifetime.

The architecture tracks blocked threads and their wait reasons to distinguish useful sleeping from unwanted busy waiting, and records per-CPU events without placing a shared diagnostic lock on every hot path.

Instrumentation adds some cost of its own. Before/after comparisons must use the same instrumentation mode or disclose that they do not.

## 7. Technical debt is part of architecture governance

The project's technical-debt register is treated as an architectural control, not merely a backlog of low-priority tasks. A temporary limit, intentionally incomplete compatibility path, stub, hard-coded behavior or deferred semantic feature should be recorded with:

- The reason the limitation is currently accepted.
- The correct ownership boundary and intended complete behavior.
- The evidence that would justify resolving or closing it.
- The actual state after the related change, rather than an outdated historical assumption.

One current example is the separately implemented Filter Manager: successful building of a similarly named driver does not mean the original Microsoft filter manager already runs. Another is a protocol operation defined by an asynchronous device ABI but not yet consumed end to end by all device classes.

## 8. The publication rule

Public architecture documentation should use explicit labels:

| Label | What a reader may infer |
|---|---|
| Design decision | The intended ownership and invariant have been specified |
| Implemented | The relevant mechanism exists in the reviewed source |
| Tested | A named test has exercised that mechanism |
| Workload-demonstrated | Real original software or a device was observed exercising it |
| End-to-end validated | The documented complete acceptance path passed at a named revision |
| Development work not yet integrated | The change is not necessarily included in mainline |
| Open / not established | No evidence was found for the stronger claim |

An engineering infographic illustrates relationships. It is not a substitute for this evidence record. The correct public story presents **what has been built, how the parts cooperate, what has been observed, and where the evidence stops**.

For a broader source-to-documentation check, see [Architecture coverage review](19-architecture-coverage-review.md).

## 9. Intentional policy bounds versus accidental implementation limits

The architecture distinguishes three kinds of boundary. Policy limits deliberately constrain resource use and define a recoverable result. Protocol bounds follow from a particular published or versioned interface. Implementation limits are arbitrary internal array or pool sizes and must not accidentally define what an otherwise generic mechanism is capable of doing.

The engineering rule is to replace accidental limits with lifecycle-managed storage, not merely to increase a hard-coded capacity. The same principle applies to message queues, native service calls, network transactions and process bookkeeping. New allocations must have a correct owner and release path; exceeding a deliberate policy limit must be observable to the caller.
