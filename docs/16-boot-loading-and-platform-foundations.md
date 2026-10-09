# From Firmware to Running Processes

**Native boot, executable loading, platform discovery and the separation between the kernel and user-space policy**

*Reviewed against the master architecture records for the native bootstrap, executable loading, VirtIO, interrupts, boot policy, console, timers, ACPI and power. This is not a new machine-by-machine boot certification.*

## 1. The first process defines the boundary

myOS does not ask the kernel to understand every executable format. The kernel uses a deliberately minimal native bootstrap image to establish the first userspace process. From that point, native user-space loading services recognize the executable formats and construct processes using format-neutral kernel operations.

That boundary is more important than it may initially seem. It means the kernel never needs to parse Windows PE dynamic linking, Linux ELF interpreter details or the runtime structures of either environment merely to create an address space. It knows how to provide **memory, a thread, a process and capabilities**.

The initial userspace process receives a deliberately limited set of resources. Every later process is constructed from delegated authority, not through a privileged rule such as “every Linux root process may read every disk.”

## 2. Loading is a two-phase operation with explicit grants

A process launcher can prepare an image, allocate its address space and map the required program regions before giving it execution rights. Construction and commitment are distinct from running it.

The loader recognizes the external format, validates its structure and requests the native operations necessary to build the process. An executable's requested memory protection is subject to the native write/execute policy. A parent can hand a child a narrowed resource right without providing unrestricted authority over the parent or other processes.

Linux-specific interpreter setup, Windows PE loader behavior and runtime environment blocks are therefore responsibilities of their respective runtimes, not independent kernel loaders.

## 3. Boot policy is a service responsibility

The normal system bootstrap uses a native user-space launcher and service supervisor. That supervisor chooses which subsystem and required services to start, while the kernel remains unaware of whether the final user interface will be a Linux shell, a graphical desktop or Windows system components.

The same native bootstrap can provide a recovery path when a normal execution environment is unavailable. Keeping that recovery path native is significant: a failure inside Windows or Linux startup need not deprive the entire machine of a basic control plane.

Boot dependency order is determined by actual resource requirements rather than by pretending that one compatibility environment is the machine's init system.

## 4. Native console, display and input

A native console represents a controlled byte stream. Linux terminal semantics—line editing, terminal flags, terminal control and signal conventions—are implemented by Linux-facing adapters on top of that neutral object.

A native display service can own the physical or virtual display resource and present controlled surfaces or buffers to clients. A Linux graphics consumer may see DRM/KMS-style interfaces while the ownership of actual hardware stays with the native service. Input drivers can likewise publish neutral events rather than injecting Windows or Linux-specific structures into the hardware layer.

This layering explains why a user-mode GPU driver can participate in a Linux desktop without making the Linux personality responsible for all graphics resources.

The reviewed design covers VirtIO graphics and a Wayland/Plasma path; it does not establish complete WDDM support or a fully running Android graphical framework.

## 5. Hardware discovery and bus protocols

Platform discovery supplies CPU topology, buses, interrupts and resources. The kernel discovers PCI devices and ACPI-provided platform structures, while separate driver hosts and services implement more specific device behavior.

VirtIO itself is layered. The PCI transport and queue protocol sit above generic device claims, memory mappings, interrupts and DMA authority. Individual block, network, input or display drivers consume those facilities. A filesystem never needs to know what kind of bus transported its block requests.

Hyper-V VMBus integration follows the same principle. Platform hypercalls and interrupt mechanisms are narrowly exposed by the native kernel; the VMBus protocol can execute in an authorized ring-3 host.

## 6. Time, power and firmware boundaries

The native clock and timer design supports scheduling deadlines and efficient time reads. Early hardware time sources remain available before user-space services exist, but long-term power policy does not have to live in the kernel.

The user-space power service decides when to synchronize filesystems and how to interpret a requested reset or poweroff. A separate ACPI-oriented mechanism service obtains narrow rights to necessary firmware tables and I/O registers.

Documented QEMU/OVMF poweroff and reboot behavior is a real validation milestone. It must not be extended into a claim that full suspend, hibernation and every Windows power-control interface are complete.

## 7. The general principle

The kernel provides *the authority to construct an operating system*, not the implementation of every possible runtime, device protocol or policy. Each additional execution environment should require a richer adapter, not another physical-machine owner.

The service manifest and its upgrade rules complete this picture: required native system services must be installed and started consistently, rather than existing only in a development boot configuration.

## 8. One boot screen across ownership changes

The architecture has a shared boot-status model that begins when UEFI presents an initial display and continues across bootloader, kernel and ring-3 display-service ownership. Progress represents **observed milestones**, not elapsed time or a guessed percent complete. The model is monotonic and works when storage, networking and other initialization steps overlap.

Rendering stays with whichever component currently controls the display. The system does not introduce a separate all-purpose graphics stack merely to draw a splash screen. The handover happens when the relevant user environment becomes ready or the first display client actually presents a frame; an application that attaches but never renders must not erase the boot display prematurely. A panic can transfer the console back to fault reporting.

## 9. Hypervisor host ports and the guest agent

A virtual machine can expose named host communication ports without making the operating-system kernel understand the hypervisor's higher-level protocol. myOS has a **transport-neutral host-port abstraction** using shared byte rings and event notification. A ring-3 VirtIO serial driver can publish such ports, while the Linux personality presents the conventional character devices expected by an original QEMU guest-agent process.

The host-port mechanism belongs to native user-space services and drivers; the Linux device naming belongs to the Linux personality. The design has host protocol tests and a named guest-agent acceptance path. This distinction is another concrete example of original software consuming native mechanisms without becoming the machine's hardware owner.
