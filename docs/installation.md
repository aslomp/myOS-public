# Installing myOS (Red Unicorn)

myOS is an experimental x86-64 operating system. Start with a disposable virtual machine or a dedicated USB drive. Do not use a disk containing important data.

> [!WARNING]
> **Physical hardware and data-loss risk**
>
> Do not boot the installation ISO directly on a normal computer containing valuable data. The ISO installer automatically selects a supported target disk and may overwrite its partition table and filesystems without interactive confirmation. Its existing checks are not a guarantee against accidental data loss.
>
> QEMU/KVM, Proxmox and Hyper-V with disposable virtual disks are the recommended evaluation environments. Direct installation onto internal physical disks is not recommended at this stage.
>
> **The prepared USB route is the recommended way to evaluate myOS on physical hardware.** Unlike the installation ISO, it boots an already installed system and does not grant Linux general raw-disk access. The boot policy selects the USB system and data volumes by filesystem UUID and restricts Linux block-device authority to the selected data partition. Internal disks are not made available to ordinary Linux applications, including Linux root, through that authority. This is a deliberate protection against accidental writes, not a guarantee against defects in privileged native services or drivers. Back up important data before physical testing.
>
> Experimental kernel and driver code can cause filesystem corruption, incorrect DMA operations, system freezes, device malfunction and firmware or power-management problems. Permanent hardware damage is less likely than data loss or system instability, but cannot be ruled out. Ring-3 driver isolation and Secure Boot do not eliminate these risks.
>
> Use dedicated test hardware and never rely on this preview for production workloads or irreplaceable data.

## 1. Obtain the ISO

Check the [release page](https://github.com/aslomp/myOS-public/releases) for the official installation ISO, referred to below as `myos.iso`. The architecture repository does not contain the operating system sources. If no ISO is available, these instructions describe the intended release workflow.

Use the installation ISO rather than a recovery image. It includes the boot payload, system files, release manifest, driver development kit (DDK), and USB preparation utility with its supporting files. Verify the published download checksum when available.

## 2. Install from ISO

### Proxmox VE

Create a disposable VM:

| Setting | Recommendation |
| --- | --- |
| Machine | q35 |
| Firmware | OVMF (UEFI), with EFI disk |
| CPU | x86-64 |
| RAM | 4 GiB suggested (at least 2 GiB) |
| System disk | Empty VirtIO Block disk (virtio0); 64 GiB suggested if adding Windows and Android |
| Network | VirtIO |
| Entropy | VirtIO RNG |
| Display | VirtIO graphics, where supported |
| CD/DVD | Official myOS ISO |

For the Proxmox configuration above, use VirtIO Block as the installer's target disk rather than VirtIO SCSI, SATA or IDE. This recommendation does not apply to Hyper-V's supported synthetic SCSI controller. Secure Boot may remain enabled when the firmware trusts the release bootloader.

**Warning: the installer automatically repartitions the first VirtIO block disk. It does not offer an interactive disk-selection or confirmation screen. Attach only an empty disposable target disk.**

1. Boot the VM from the ISO.
2. Wait for the installer to create the GPT, EFI System Partition and ext4 system volume and copy the operating system.
3. When installation completes and the VM reboots, detach the ISO or boot from the installed disk.
4. The base installation works offline. Network access may be needed for additional packages.

### QEMU / KVM

On an x86-64 Linux host with QEMU, KVM and OVMF, create a new virtual disk:

```sh
qemu-img create -f qcow2 myos-disk.qcow2 64G
cp /usr/share/OVMF/OVMF_VARS_4M.fd myos-vars.fd
qemu-system-x86_64 \
  -machine q35 -enable-kvm -cpu host -smp 4 -m 4096 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.fd \
  -drive if=pflash,format=raw,file=myos-vars.fd \
  -drive if=virtio,format=qcow2,file=myos-disk.qcow2 \
  -device virtio-rng-pci -nic user,model=virtio-net-pci \
  -device virtio-vga -serial stdio \
  -cdrom myos.iso -boot once=d
```

Adjust the OVMF paths for your distribution. After installation, use the same command without the CD-ROM and one-time boot arguments. Preserve the virtual disk and firmware variable store.

### Hyper-V

Hyper-V SCSI storage is supported by myOS. Create a Generation 2 VM with the following configuration:

| Setting | Recommendation |
| --- | --- |
| Generation | 2 (UEFI) |
| Secure Boot | May remain enabled (with compatible firmware trust configuration) |
| Memory | 2048 MiB minimum; more for additional environments |
| Processors | 4 virtual processors recommended |
| System disk | Empty VHDX attached to the SCSI controller |
| DVD drive | Official myOS ISO attached to the SCSI controller |
| Network | Synthetic network adapter connected to a virtual switch |

Use a release prepared for Hyper-V. Its storage and display devices differ from VirtIO. Set the DVD drive first in the firmware boot order for installation, then boot from the VHDX after installation. Use only a disposable empty target disk. The exact automatic disk-selection behavior should be verified for the release being installed.

## 3. Create a bootable USB drive

The USB utility creates an installed operating system on removable media. Do not simply copy the ISO to the stick. An x86-64 Linux host is required; no myOS source checkout, Rust or Cargo is needed.

Install the host tools (Debian/Ubuntu example):

```sh
sudo apt install xorriso mtools dosfstools e2fsprogs cpio fdisk util-linux fakeroot gcc binutils lld make python3 xz-utils flex bison curl
```

### Extract the matching USB utility from the ISO

```sh
ISO="$(realpath myos.iso)"
mkdir -p myos-usb-tools
xorriso -osirrox on -indev "$ISO" \
  -extract /mkusb.sh myos-usb-tools/mkusb.sh \
  -extract /tools myos-usb-tools/tools
cd myos-usb-tools
bash mkusb.sh --help
```

Keep the extracted script and tools together. The DDK itself is read directly from the ISO when needed. Older ISOs without these files cannot use this extraction workflow.

### Select and build hardware drivers

Run the following inventory command under Linux on the **target computer**, then transfer the resulting text file to the build machine if different:

```sh
bash mkusb.sh inventory > inventory.txt
```

List the drivers available in this release and build the selected bundle:

```sh
bash mkusb.sh build-drivers --iso "$ISO" --list
bash mkusb.sh build-drivers --iso "$ISO" \
  --hardware inventory.txt --driver usb-storage --out ./drivers
```

Review uncovered hardware, especially the storage controller needed to reach the USB disk. Repeat `--driver NAME` for other supported drivers. The utility uses the ISO's precompiled DDK and obtains the pinned original Linux source archive, checking its checksum. An existing matching archive can be provided with `--linux-tarball FILE`. Rebuild the bundle when changing ISO releases. This builds drivers, not a Linux kernel or myOS itself.

### Write and verify the USB drive

Use a dedicated USB drive (16 GB or larger is recommended for the base system). The entire target drive will be erased.

```sh
lsblk -o NAME,SIZE,MODEL,TRAN,RM,MOUNTPOINTS
```

Replace `/dev/sdX` below with the actual whole USB disk, never a partition or the host's system disk. Unmount its mounted partitions first.

```sh
sudo bash mkusb.sh --iso "$ISO" --device /dev/sdX \
  --drivers ./drivers --check-device
sudo bash mkusb.sh --iso "$ISO" --device /dev/sdX --drivers ./drivers
sudo bash mkusb.sh verify --target /dev/sdX
```

The USB utility checks that the target is a whole writable device, rejects detected host system disks and active swap disks, refuses fixed non-USB disks by default, and refuses mounted targets unless explicitly requested to unmount them. It requires confirmation of the exact device path before destructive writes. The `--check-device` operation does not write anything. Review the confirmation and do not bypass the device checks. The default USB layout has a 1024 MiB EFI partition, a 6 GiB system partition and a separate data partition using the remaining space. For larger workloads choose a larger system partition using `--system-size` and a sufficiently large drive. Windows extraction requires substantial additional free space on its target filesystem; a large data partition alone does not guarantee that space.

To create a disk image instead of writing a physical drive:

```sh
bash mkusb.sh --iso "$ISO" --image stick.img --size 16G --drivers ./drivers
bash mkusb.sh verify --target stick.img
```

Boot the target computer using its one-time UEFI USB boot menu. The system runs from the prepared USB disk; it does not install onto the internal disk. Its normal runtime policy does not enable `disk-admin`; the Linux root is selected by UUID and raw block access is narrowed to the USB data partition. If that data volume cannot be found, Linux receives no storage authority rather than unrestricted access. Hardware support varies by release; an xHCI USB controller is recommended.

### Physical hardware tested

The USB boot route has been exercised on the developer's physical ASUS laptop, not only in virtual machines. The recorded test hardware includes:

| Component | Tested machine |
| --- | --- |
| Platform | ASUS x86-64 laptop, AMD Renoir/Cezanne platform |
| Integrated graphics | AMD Radeon Renoir, PCI ID `1002:1636` |
| Discrete graphics | NVIDIA GeForce GTX 1660 Ti Mobile (present; not an acceleration claim) |
| Wired networking | Realtek RTL8111/8168 family |
| USB controller | AMD xHCI |

Bare-metal UEFI boot from USB, wired networking and SSH have been demonstrated. On 15 November 2026, the original AMDGPU driver hosted by myOS produced a visible 1920 × 1080 Plasma desktop; touchpad and USB mouse input were also confirmed. This is evidence for that hardware configuration, not a claim of universal laptop support or hardware-accelerated Mesa rendering. Exact CPU model, RAM capacity and laptop model are not confirmed here.

## 4. Install the Windows environment

Windows is optional. myOS runs original Microsoft user-space components on its own kernel; it does not install or boot a separate Windows kernel. Microsoft installation media are not distributed with myOS. Obtain your own authorized x86-64 Windows installation media and comply with its licensing terms.

Allow ample free space: a Windows 11 image may require roughly 16 GiB for extracted files plus over 8 GiB for temporary extraction. Additional free space is advisable.

Boot the installed myOS system. Attach the Windows installation ISO to the VM as a secondary CD/DVD, or make the ISO file available to the running system. Open the myOS Linux shell with administrative permissions.

```sh
install-windows --list
```

Choose the desired x86-64 image index from the actual output, then run (replace N):

```sh
install-windows --index N
```

If automatic media discovery is ambiguous, specify the source explicitly:

```sh
install-windows --source /path/to/windows.iso --list
install-windows --source /path/to/windows.iso --index N
```

The source may also be a mounted installation directory or a WIM/ESD file. The installer may obtain Python, wimlib, 7-Zip and hivex through the Alpine package manager if missing; network access is then required.

The Windows root is installed under `/windows`. Reboot myOS after installation. Its boot policy detects the installation manifest and starts the Windows environment. The first startup completes deployment initialization and can require another automatic restart. Allow this to finish.

To try an original Windows executable from the Linux shell:

```sh
/windows/System32/notepad.exe
```

This requires the Windows runtime to be active and a suitable graphical session for graphical output. Desktop and application compatibility remain experimental. Do not use `--force` casually: reinstalling replaces the Windows root, including its registry state.

## 5. Install the Android environment

Android is also optional and runs as a separate user-space environment on the native myOS kernel, not as a guest kernel. The current installer prepares a release-pinned AOSP/Cuttlefish image, its ramdisks, partition images and the myOS configuration layer.

### Install from the running myOS system

On releases that include the Android installer, boot myOS, open its Linux shell as an administrator and run:

```sh
install-android --verify
```

The command installs the required Alpine tools, downloads the release-pinned Android image, checks its size and SHA-256, prepares the Android files and installs them under `/android`. An Internet connection and a writable target filesystem are required. Allow at least 8 GiB of free temporary workspace (by default `/var/tmp/myos-android`) plus space for the installed environment. For a larger Android user-data image, use `--userdata-mb 2048`, for example. Existing user-data images are preserved.

Reboot myOS when installation completes.

### Alternative: prepare Android on a Linux host using the ISO

Extract the matching installer and its dependencies from the release ISO:

```sh
ISO="$(realpath myos.iso)"
mkdir -p myos-android-tools
xorriso -osirrox on -indev "$ISO" \
  -extract /tools myos-android-tools/tools
cd myos-android-tools
sudo apt install bash curl unzip python3 e2fsprogs coreutils findutils util-linux
```

The kit includes a prebuilt EROFS extractor; Rust, Cargo and the private myOS source tree are not required. Set a private working directory outside the extracted kit and your home directory, with sufficient free space:

```sh
sudo env ANDROID_ROOT=/var/tmp/myos-android \
  bash tools/install-android.sh --dir /path/to/target-root/android --verify
```

Replace target-root with the actual offline, writable myOS root filesystem. Do not point it at the host's root or the EFI partition. The installer's separate `--disk` mode expects a specific development volume layout and is not suitable for arbitrary USB or ISO-installed GPT disks.

Once a valid `/android/myos-install.manifest` exists on the installed myOS root, boot myOS to start the Android environment. Android applications and ADB depend on the release's Android integration and networking configuration. Full Android boot and universal APK compatibility are not guaranteed.

## 6. Troubleshooting

- **ISO does not boot:** check x86-64 UEFI, firmware boot settings and, if Secure Boot is enabled, bootloader trust.
- **No installation disk:** use an empty VirtIO Block disk on QEMU/Proxmox or a supported Hyper-V SCSI VHDX with the matching release. Check the release's installer disk-selection behavior.
- **USB root volume missing:** verify the media, xHCI controller and usb-storage bundle. Check MYOS-USB.txt on the EFI partition.
- **No graphical desktop:** the release may support only limited display hardware or software rendering. Use the console to diagnose startup.
- **Windows executable not recognized:** ensure the Windows installation completed and myOS was rebooted.
- **Android does not start:** check the installed manifest and whether the release supports the prepared image.

For bug reports, include the myOS release identity, hypervisor or physical hardware, driver selections and relevant console logs. Remove personal information from logs before posting.

See [validation and known limitations](05-validation-and-roadmap.md) for the distinction between implemented mechanisms and demonstrated workloads.
