# Windows drivers on myOS — verified architecture and scope

**Review scope:** source branch claude/windows-fltmgr-cpp-driver-071154, commit 42785b21717803277cf4b8dbb40c30b6c2086242, inspected 9 October 2026. The branch is separate from master. This review examined source documents and acceptance checks; it did not execute QEMU independently.

![Windows driver host architecture](../diagrams/windows-driver-runtime.svg)

## What exists

myOS includes a **Windows kernel-driver host running in ring 3**, an NT driver ABI runtime, and a shared native driver manager capable of starting software-driver hosts. The host loads relocatable PE/COFF Windows kernel-driver images, resolves imported NT kernel routines against its compatibility runtime, calls driver initialization and handles device objects and I/O requests. This is executable WDM driver hosting, not merely simulated Windows service status.

### Architectural ownership

| Layer | Responsibility |
|---|---|
| Original Windows user space and NT personality | Driver service requests, NT object names, handles and Windows status translation |
| Shared native driver manager | Catalog, host selection, starting/restarting and lifecycle |
| Ring-3 Windows driver host | PE driver loading, NT imports, DriverEntry, unload, IRP handling |
| Native myOS kernel | Processes, memory objects, waits/IPC, networking and hardware authority |
| Hardware or native class services | Common devices and network transport, independent of driver family |

The Windows driver host does not introduce a second kernel. Software drivers can run without owning a physical PCI device; network drivers use the native network mechanisms.

## What is reported as tested

The branch audit documents a QEMU run of the WDK-built **wdmtest.sys**, including entry/unload, duplicate and concurrent loading, device names and links, synchronous/deferred IRPs, cancellation, reload and recovery from a crashed host. The Windows network-driver host acceptance script covers the myOS-built **tcpip.sys** and **afd.sys**, including TCP loopback bind/listen/connect/accept, data exchange, nonblocking behavior, concurrent connections, UDP datagrams and unload.

These are substantial Windows driver capabilities. The script specifies the checks and the audit reports successful runs; this documentation review has not independently repeated them.

## Exact driver distinctions

| Driver or group | Origin | Current evidence |
|---|---|---|
| wdmtest.sys | Own WDK-built test driver | Hosted DriverEntry, device objects, IRPs, cancellation and lifecycle reported tested |
| tcpip.sys | Own WDK-built network driver | Runs using native myOS networking in host acceptance |
| afd.sys | Own WDK-built Winsock kernel-side driver | Native-backed TCP/UDP AFD contract exercised by host acceptance |
| testflt.sys | Own WDK-built minifilter test | Builds; functioning filter chain not established by reviewed acceptance |
| Own fltmgr.sys | Own WDK-built file-filter manager | Builds, but explicitly **technical debt**: not the desired original Filter Manager |
| Original Microsoft fltmgr.sys | Microsoft binary from user installation | Not shown loading and working in reviewed evidence; unresolved NT imports remain |
| Npfs, CNG, KSecDD, nsiproxy representations | Native-backed NT personality | Presented as NT driver objects; not necessarily original driver binary execution |
| Native and hosted Linux hardware drivers | Existing myOS driver infrastructure | Shared manager and device authority; independent of Windows software-driver hosting |

**Important:** existence of a .sys artifact, success building with WDK, NT namespace representation, and successful execution of the original Windows driver are four different claims.

## Driver request and execution

~~~text
 Original Windows service or client
              |
          NT personality
       NT driver/service names
              |
       Shared driver manager
       catalog + host lifecycle
              |
       Windows driver host (ring 3)
       +-----------------------+
       | PE image / NT imports |
       | DriverEntry and IRPs  |
       | unload / cancellation |
       +-----------------------+
              |
      Native service interfaces
       IPC / waits / memory / net
              |
       Native kernel authority
~~~

## Original Microsoft Filter Manager

The branch README explicitly says its independently implemented Filter Manager is not to be extended as the final solution; the required target is Microsoft's own **fltmgr.sys** running under the Windows driver host. That authentic binary requires a much larger NT kernel API surface and correct file-filter integration. The inspected branch does not establish that this original driver has started and passed minifilter tests.

The project should not present a broad list of Windows inbox drivers, KMDF/UMDF coverage, or Windows GPU drivers as running without distinct evidence.

## Sources

The review used the private myOS branch's Windows driver ABI audit, Windows driver documentation, Windows-driver README, WDK build report and driver-host acceptance script. Historical design sections remain in some of these documents, so later implementation and measured-status sections take precedence.
