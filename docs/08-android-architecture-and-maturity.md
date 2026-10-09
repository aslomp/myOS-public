# Android on myOS: architecture and demonstrated maturity

**Development branch:** `claude/android-x86-64-native-624c9f`  
**Review basis:** Android architecture audit dated 9 October 2026

## Objective

Run the original **AOSP x86-64 user space** alongside Linux and Windows on the same native myOS kernel, without a second kernel, VM, Android framework replacement or extensive patches to the upstream platform. The target includes real Android initialization, system services, package installation, launching an interactive application inside the existing Plasma desktop, and correct package removal.

## Layered design

```mermaid
flowchart TB
  subgraph Android["Isolated Android execution environment"]
    INIT["Original AOSP init and system services"]
    BINDER["Binder services and framework processes"]
    GFX["Original SurfaceFlinger and hardware composer"]
  end
  subgraph Contracts["Android-facing contracts"]
    LINUX["Linux system-call and filesystem interfaces"]
    MAC["SELinux policy and security labels"]
    IPC["Binder ABI and process-domain control"]
    DEV["eBPF, cgroups and graphics interfaces"]
  end
  subgraph Shared["Shared native myOS kernel"]
    SYS["Scheduling · memory · IPC · VFS"]
    NET["Network and resource authority"]
    HARD["Device ownership and drivers"]
  end
  INIT --> LINUX
  INIT --> MAC
  BINDER --> IPC
  GFX --> DEV
  LINUX --> SYS
  MAC --> SYS
  IPC --> SYS
  DEV --> NET
  DEV --> HARD
```

The Android-specific interfaces are primarily an extension of the existing Linux-facing environment. New reusable needs should be implemented as **native, environment-independent mechanisms** rather than embedding Android-specific behavior into every native subsystem.

## Original platform components

The branch uses prebuilt Android images, including original AOSP platform partitions and system binaries. Android's first-stage boot, APEX management, policy loading, core services and graphical composition should be performed by the original components. A native installation mechanism provides the storage and bootstrap contracts the image requires. The Android kernel from the source image is **not** used.

## Key architectural dependencies

| Area | Required behavior |
|---|---|
| Filesystem and images | Read-only platform partitions, EROFS, writable Android data, file labels and by-name device presentation |
| Initialization | Correct boot parameters, mount operations, system properties and service dependencies |
| Security | Original SELinux policy, enforcing mode, domain isolation and rights enforcement |
| IPC | Binder identity, object and transaction semantics backed by native communication and process authority |
| Runtime | Process/thread scheduling, memory mappings, runtime memory protection and process groups |
| Networking | Android's expected eBPF and network-control behavior over shared native networking |
| Display | Original compositor and composer operating through a virtual DRM output |
| Host integration | Display and input bridging into the existing Linux desktop without taking over its GPU stack |

## Observed progress at audit time

```mermaid
flowchart LR
  FIRST["AOSP init"] --> POLICY["SELinux enforcing"]
  POLICY --> CORE["APEX and selected<br/>original system services"]
  CORE --> BPF["bpfloader"]
  BPF -.-> FRAME["Zygote / system_server"]
  FRAME -.-> UI["SurfaceFlinger / Android apps"]
```

The audit reports execution of init, ueventd, apexd, logd, servicemanager, vold, and selected keystore and hardware-backed security service interfaces. The reviewed boot sequence then encounters a **bpfloader failure**. There was no observed start of zygote, system_server, SurfaceFlinger, netd or adbd at that milestone.

Separately, the graphics bridge is documented as implemented and compiling in a development context; no end-to-end booted Android display had been measured. The latest branch head reviewed by the audit had not itself passed a full acceptance and regression run. These are separate evidence levels and should not be conflated.

## Integration risks

The branch audit flags unresolved or insufficiently validated boundaries involving globally mutable settings, peer-domain isolation for selected devices, namespace-related process identification, Android memory objects and security labels, native scheduler regressions and eBPF execution. Some code paths were not boot-tested. These issues are especially important because regression can affect Linux and Windows even before Android reaches interactive startup.

## Next proof milestones

1. Full native kernel and existing Linux/Windows regression suite on the Android branch.
2. Successful original Android network initialization beyond bpfloader.
3. Original zygote and system_server boot.
4. Original graphics composition shown through the desktop bridge.
5. Working adb package install, app start, interaction and uninstall.
6. Repeated tests under enforcing SELinux, including negative cross-environment isolation cases.

**Conclusion:** the third-environment architecture is materially implemented and early boot progress is demonstrated, but Android desktop and application compatibility remain **unfinished integration goals**.
