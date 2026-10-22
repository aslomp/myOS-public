# Windows drivers on myOS — verified architecture and scope

**Review basis:** Windows driver architecture and recorded acceptance results, updated 22 October 2026. This documentation review did not independently execute QEMU.

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
| WDM test driver | Own WDK-built test driver | Hosted DriverEntry, device objects, IRPs, cancellation and lifecycle reported tested |
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

## Original Microsoft Filter Manager — implemented and tested

myOS runs the **original Microsoft Filter Manager**, loaded from the user's Windows installation, in its ring-3 Windows driver-host environment. This is not a reimplementation or a simulated driver state.

Windows-session acceptance has demonstrated successful initialization through the original Windows driver-loading mechanism and recognition of the service as running. Separate end-to-end minifilter acceptance has demonstrated that native filesystem activity is delivered through the original Filter Manager to a hosted minifilter, including file open, read, write and close-related operations.

The native VFS remains responsible for the underlying filesystem operation, while the Windows filtering contract is provided by the original Microsoft component. This is a functioning integration, not an outstanding architectural proposal.

As with any compatibility implementation, these specific tests do not establish universal support for every possible third-party minifilter or filesystem callback. That distinction must not be confused with the **already demonstrated execution and operation of the original Filter Manager**.

Other Windows driver classes, including framework-based, storage and graphics drivers, have separate compatibility and validation requirements.

## Sources

The review used technical implementation reviews and dated driver-host acceptance records. Historical design sections remain in some of these documents, so later implementation and measured-status sections take precedence.

## NDIS status — 22 October 2026

The current implementation includes Windows NDIS miniport compatibility over native device and networking mechanisms. Windows-visible adapter and statistics interfaces are separate from execution of an original miniport binary.

A recorded QEMU test dated 16 October 2026 used the original Intel e1000e Windows driver. It reported a bound device, a 1 Gbit/s link, four successful ping replies and 11 passing NIC acceptance checks. These are results for that particular tested configuration, not a claim of universal driver compatibility or a new acceptance run.
