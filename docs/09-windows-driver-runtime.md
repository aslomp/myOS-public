# Windows drivers on myOS — verified architecture and scope

**Review date:** 9 October 2026. This review considered implementation evidence and recorded acceptance checks; it did not execute QEMU independently.

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

The technical review documents a QEMU run of a **WDK-built WDM test driver**, including entry/unload, duplicate and concurrent loading, device names and links, synchronous/deferred IRPs, cancellation, reload and recovery from a crashed host. The Windows network-driver host acceptance script covers the myOS-built **TCP/IP compatibility driver** and **Winsock compatibility driver**, including TCP loopback bind/listen/connect/accept, data exchange, nonblocking behavior, concurrent connections, UDP datagrams and unload.

These are substantial Windows driver capabilities. The script specifies the checks and the audit reports successful runs; this documentation review has not independently repeated them.

## Exact driver distinctions

| Driver or group | Origin | Current evidence |
|---|---|---|
| a WDM test driver | Own WDK-built test driver | Hosted DriverEntry, device objects, IRPs, cancellation and lifecycle reported tested |
| TCP/IP compatibility driver | Own WDK-built network driver | Runs using native myOS networking in host acceptance |
| Winsock compatibility driver | Own WDK-built Winsock kernel-side driver | Native-backed TCP/UDP AFD contract exercised by host acceptance |
| Minifilter test driver | Compatibility validation workload | Reported filesystem-operation callbacks through the original Microsoft Filter Manager |

| Original Microsoft Filter Manager | Original Microsoft component from an installed Windows environment | Successfully initialized; reported end-to-end tests exercise hosted minifilter callbacks from native file operations |
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

The architecture uses Microsoft's original **Filter Manager**, rather than a separately developed replacement. Recorded Windows-session acceptance demonstrates successful driver initialization and Windows service visibility. Additional minifilter acceptance exercises selected native filesystem operations, including file opens, reads, writes and close-related notifications, through the hosted filtering path. Support for every filter callback, filesystem operation and third-party minifilter has not been demonstrated.

The project should not present a broad list of Windows inbox drivers, KMDF/UMDF coverage, or Windows GPU drivers as running without distinct evidence.

## Sources

The review used technical implementation reviews and dated driver-host acceptance records. Historical design sections remain in some of these documents, so later implementation and measured-status sections take precedence.

## NDIS status — 22 October 2026

The current implementation includes Windows NDIS miniport compatibility over native device and networking mechanisms. Windows-visible adapter and statistics interfaces are separate from execution of an original miniport binary.

A recorded QEMU test dated 16 October 2026 used the original Intel e1000e Windows driver. It reported a bound device, a 1 Gbit/s link, four successful ping replies and 11 passing NIC acceptance checks. These are results for that particular tested configuration, not a claim of universal driver compatibility or a new acceptance run.
