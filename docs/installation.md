# Installing myOS (Red Unicorn)

myOS is an experimental x86-64 operating system. Start with a disposable virtual machine or a dedicated USB drive. Do not use a disk containing important data.

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

Do not use VirtIO SCSI, SATA or IDE as the standard installer's target disk. Disable Secure Boot on the test VM if the image is not trusted by its firmware.

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

Use a Generation 2 UEFI VM and a release explicitly prepared for Hyper-V. Its storage and display devices differ from VirtIO. Do not assume the standard ISO installer supports a Hyper-V SCSI disk; consult the corresponding release notes.

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

Review the confirmation and do not bypass the device checks. The default USB layout has a 1024 MiB EFI partition, a 6 GiB system partition and a separate data partition using the remaining space. For larger workloads choose a larger system partition using `--system-size` and a sufficiently large drive. Windows extraction requires substantial additional free space on its target filesystem; a large data partition alone does not guarantee that space.

To create a disk image instead of writing a physical drive:

```sh
bash mkusb.sh --iso "$ISO" --image stick.img --size 16G --drivers ./drivers
bash mkusb.sh verify --target stick.img
```

Boot the target computer using its one-time UEFI USB boot menu. The system runs from the prepared USB disk; it does not install onto the internal disk. Hardware support varies by release; an xHCI USB controller is recommended.

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

- **ISO does not boot:** check x86-64 UEFI and firmware boot settings.
- **No installation disk:** use an empty VirtIO Block disk.
- **USB root volume missing:** verify the media, xHCI controller and usb-storage bundle. Check MYOS-USB.txt on the EFI partition.
- **No graphical desktop:** the release may support only limited display hardware or software rendering. Use the console to diagnose startup.
- **Windows executable not recognized:** ensure the Windows installation completed and myOS was rebooted.
- **Android does not start:** check the installed manifest and whether the release supports the prepared image.

For bug reports, include the myOS release identity, hypervisor or physical hardware, driver selections and relevant console logs. Remove personal information from logs before posting.

See [validation and known limitations](05-validation-and-roadmap.md) for the distinction between implemented mechanisms and demonstrated workloads.
