# QEMU Cheatsheet

<!--
QEMU (Quick Emulator) is a generic and open-source machine emulator and virtualizer.
It can be used to run operating systems and programs for one machine (the "guest") on a different machine (the "host").
When used with KVM (Kernel-based Virtual Machine), QEMU can achieve near native performance for guests matching the host architecture.

This cheatsheet provides common commands for creating, managing QEMU virtual machines, and disk images.
It primarily focuses on `qemu-system-x86_64` for emulating x86_64 architecture.
For other architectures, use the appropriate QEMU binary (e.g., `qemu-system-aarch64` for ARM64, `qemu-system-riscv64` for RISC-V).

General command structure:
`qemu-system-ARCH [options]`
-->

## Basic VM Launch & Core Options

### Essential Options
```bash
# qemu-system-x86_64 \
#   --enable-kvm \    # Crucial: Enable KVM hardware virtualization for performance (requires KVM kernel modules & CPU support)
#   -m <RAM_size> \   # Allocate RAM to the VM (e.g., -m 4G, -m 2048M, or -m 2048 for Megabytes)
#   -smp <cores>[,sockets=<n>,threads=<n>,maxcpus=<n>] \ # Configure CPU cores, sockets, threads
#                     # Example: -smp 4 (4 cores)
#                     # Example: -smp cores=2,threads=2,sockets=1 (2 cores, 2 threads each)
#   -cpu <cpu_model> \ # Specify CPU model to emulate.
#                      # `-cpu host` passes through host CPU features (good for KVM if guest OS supports it).
#                      # `-cpu max` enables all features QEMU supports.
#                      # `-cpu qemu64` for a generic qemu64 CPU. List models with `qemu-system-x86_64 -cpu help`.
#   -machine <type> \ # Specify machine type (e.g., -machine q35 (modern), -machine pc (older i440FX)).
#                     # Use `type=accel=kvm` for KVM: -machine type=q35,accel=kvm
#   -name "<VM_Name>" \ # Set a name for the VM window (useful for identification in window managers or VNC)
#   -daemonize \      # Run QEMU as a daemon (background process)
#   -monitor stdio \  # Provide QEMU monitor on the console (Ctrl+A, C to switch to monitor, Ctrl+A, X to quit QEMU)
#                     # Or -monitor telnet::4444,server,nowait (monitor on telnet port 4444)
#   -serial stdio \   # Redirect guest serial port to stdio (useful for headless/debug)
#   -parallel none    # Disable parallel port
```

### Example: Basic VM Boot from ISO (for OS Installation)
```bash
# This command starts a VM, booting from an ISO image, typically for installing an OS onto a virtual disk.
# Assumes my_vm_disk.qcow2 is already created (see Disk Image Management).
qemu-system-x86_64 \
  --enable-kvm \
  -machine type=q35,accel=kvm \
  -cpu host \
  -m 4G \
  -smp 2 \
  -name "My OS Installer VM" \
  -boot order=d \               # 'd' for CD-ROM first, 'c' for hard disk, 'n' for network
  -cdrom /path/to/os_install.iso \ # Attach ISO as CD-ROM
  -drive file=my_vm_disk.qcow2,format=qcow2,if=virtio,cache=writeback \ # Attach disk using virtio-blk
  -vga virtio \                 # Use VirtIO VGA for better performance (guest drivers needed)
  -display gtk,gl=on \          # Use GTK display with OpenGL acceleration (if supported)
  -netdev user,id=net0 \        # Basic user-mode networking (NAT)
  -device virtio-net-pci,netdev=net0 # Attach VirtIO NIC for the user network
```
<!-- After OS installation, change `-boot order=c` to boot from the hard disk. -->

### Example: Starting an Existing VM (from Disk)
```bash
qemu-system-x86_64 \
  --enable-kvm \
  -machine type=q35,accel=kvm \
  -cpu host \
  -m 4G \
  -smp 4 \
  -name "My Existing VM" \
  -boot order=c \
  -drive file=my_vm_disk.qcow2,format=qcow2,if=virtio,cache=writeback \
  -vga virtio \
  -display gtk,gl=on \
  -netdev user,id=net0,hostfwd=tcp::10022-:22 \ # User networking + forward host port 10022 to guest port 22 (SSH)
  -device virtio-net-pci,netdev=net0
```

## Disk Image Management (`qemu-img`)

<!-- `qemu-img` is a command-line tool for creating, converting, and modifying virtual disk images. -->

### Creating Virtual Disks

#### `qcow2` (QEMU Copy On Write 2) Format
<!-- Flexible format supporting snapshots, compression, encryption, smaller image sizes (thin provisioning). Recommended. -->
```bash
# Create a basic qcow2 disk image (e.g., 30GB)
#   -f qcow2: Specify the format.
#   <image_name>.qcow2: Name of the disk image file.
#   <size>G: Size of the virtual disk in Gigabytes (e.g., 20G, 50G). Can also use M for Megabytes.
qemu-img create -f qcow2 my_vm_disk.qcow2 30G

# Create a qcow2 image with options:
#   -o compat=<level>: Compatibility level (e.g., 1.1 for older QEMU/hypervisors).
#   -o compression_type=<type>: Type of compression (e.g., zstd, zlib). zstd often offers good balance.
#   -o preallocation=<mode>: Preallocate metadata ('metadata') or full space ('full').
#                            'off' (default) is thin provisioned. 'full' is like a raw image in size.
# Example: 100GB disk, qcow2 format, compat 1.1, zstd compression, metadata preallocation
qemu-img create -f qcow2 -o compat=1.1,compression_type=zstd,preallocation=metadata my_large_disk.qcow2 100G
```

#### `raw` Format
<!-- Simple format, just raw bytes. Better performance in some cases, larger initial file size (fully allocated).
     Less features than qcow2 (no snapshots at image level). -->
```bash
qemu-img create -f raw my_vm_disk_raw.img 30G
```

### Getting Disk Image Information
<!-- Displays format, size, backing file (if any), snapshots, etc. -->
```bash
qemu-img info my_vm_disk.qcow2
```

### Resizing a Disk Image
<!-- WARNING: After increasing image size, you must use tools within the guest OS to extend the partition and filesystem.
     Shrinking is more complex and risky. -->
```bash
# Increase image size by 10GB (e.g., from 30G to 40G)
qemu-img resize my_vm_disk.qcow2 +10G
# Set image size to a specific value
# qemu-img resize my_vm_disk.qcow2 50G
```

### Converting Disk Images
<!-- Convert between formats (e.g., VMDK to qcow2, raw to qcow2). -->
```bash
# qemu-img convert -f <source_format> -O <output_format> <source_image_file> <output_image_file>
#   -f: Source format (optional, qemu-img often detects it).
#   -O: Output format.
#   -p: Show progress.
# Example: Convert a VMDK to qcow2
# qemu-img convert -f vmdk -O qcow2 source_disk.vmdk converted_disk.qcow2 -p
# Example: Convert raw to qcow2
qemu-img convert -O qcow2 raw_disk.img compressed_disk.qcow2 -p
```

### Creating Snapshots (qcow2 only)
<!-- Snapshots capture the state of a disk image at a specific point in time. Requires qcow2 format. -->
```bash
# Create a snapshot of a running or offline VM's disk
qemu-img snapshot -c <snapshot_name> <vm_disk_image.qcow2>
# Example: qemu-img snapshot -c pre_update_snap my_vm_disk.qcow2

# List snapshots in an image
qemu-img snapshot -l <vm_disk_image.qcow2>

# Apply (revert to) a snapshot
# qemu-img snapshot -a <snapshot_name> <vm_disk_image.qcow2>
# WARNING: This reverts the disk image to the snapshot state. Current state will be lost unless you snapshot it first.

# Delete a snapshot
# qemu-img snapshot -d <snapshot_name> <vm_disk_image.qcow2>
```
<!-- For live VM snapshots, it's often better to use the QEMU monitor (`savevm`, `loadvm`) which can also save RAM state. -->

## Attaching Drives & Boot Options

### Using `-hda`, `-hdb`, `-cdrom` (Legacy IDE)
<!-- These options emulate IDE devices. Simple but not performant. -->
```bash
# -hda <disk_image>: Primary master IDE disk.
# -hdb <disk_image>: Primary slave IDE disk.
# -hdc <disk_image>: Secondary master IDE disk.
# -hdd <disk_image>: Secondary slave IDE disk.
# -cdrom <iso_image>: Attach an ISO as an IDE CD-ROM.
# Example:
# qemu-system-x86_64 ... -hda system.qcow2 -cdrom install.iso ...
```

### Using `-drive` (Modern & Flexible)
<!-- The `-drive` option is more versatile and allows specifying interface types like VirtIO for better performance. -->
```bash
# -drive file=<path>,format=<fmt>,if=<interface>,index=<n>,media=<type>,cache=<mode>,...
#   file=<path_to_image_file>
#   format=<qcow2|raw|vmdk|etc.> (Optional if QEMU can guess)
#   if=<interface_type>:
#     `ide`: Emulates an IDE drive.
#     `scsi`: Emulates a SCSI drive (often needs a SCSI controller like `lsi53c895a` or `virtio-scsi-pci`).
#     `virtio`: VirtIO block device (recommended for performance, requires guest OS drivers).
#     `none`: Don't attach to any guest-visible interface (used with other options like `-device`).
#   index=<number>: Specifies drive index for the interface.
#   media=<disk|cdrom>: Type of media.
#   cache=<writethrough|writeback|none|directsync|unsafe>: Cache mode.
#          `writeback` often gives good performance but risks data loss on host crash if not handled carefully.
#          `none` can be good for raw images or if host caching is sufficient.

# Example: VirtIO block device (recommended for primary disk)
#   -drive file=my_os_disk.qcow2,format=qcow2,if=virtio,index=0,media=disk,cache=writeback
# This defines the drive. To make it visible, you also need a virtio-blk-pci device:
#   -device virtio-blk-pci,drive=drive0 # Assuming you give the drive an id=drive0
# A simpler way for VirtIO if index is clear:
# qemu-system-x86_64 ... -drive file=my_os_disk.qcow2,format=qcow2,if=virtio ...

# Example: CD-ROM using SATA (AHCI) interface
#   -drive file=install.iso,index=2,media=cdrom,if=none,id=cdrom0 \
#   -device ich9-ahci,id=ahci \
#   -device ide-cd,bus=ahci.2,drive=cdrom0
# Simpler for CD-ROM:
#   -drive file=install.iso,media=cdrom,format=raw # QEMU often figures out interface
```

### Boot Options
```bash
# -boot order=<boot_sequence>
#   'c': Boot from first hard disk.
#   'd': Boot from first CD-ROM.
#   'n': Boot from network (PXE).
#   'once=X': Boot from X for the next boot only, then revert to default.
# Example: Boot from CD-ROM first, then hard disk
#   -boot order=dc
# Example: Boot from hard disk
#   -boot order=c
# Example: Next boot only from CD-ROM, then default to hard disk
#   -boot once=d,menu=on # menu=on shows interactive boot menu
```

## Display & Graphics

```bash
# -display <type>: Specify display backend.
#   `gtk`: Use GTK window (often default, good features like USB redirection GUI).
#     `gl=on`: Enable OpenGL acceleration for the display window (can improve rendering performance of the VM window itself).
#   `sdl`: Use SDL window.
#   `curses`: Text-based display in terminal (for console-only guests).
#   `none`: No display output (for headless VMs).
#   `vnc=<host>:<display_num>[,password]`: Enable VNC server.
#     Example: `-display vnc=:0` (VNC server on localhost:5900)
#     Example: `-display vnc=0.0.0.0:1,password` (VNC on all interfaces, port 5901, password prompt on client)
#             For VNC password: use QEMU monitor `change vnc password` then set it.
#
# -vga <vga_type>: Specify emulated VGA card.
#   `std`: Standard VGA card with Bochs VBE extensions (widely compatible).
#   `cirrus`: Cirrus Logic GD5446 (older, good for legacy guests).
#   `vmware`: VMware SVGA-II compatible (useful if guest has VMware drivers).
#   `qxl`: QXL paravirtualized graphics card (good for SPICE display server, offers 2D acceleration).
#   `virtio`: VirtIO VGA/GPU (recommended for modern guests with VirtIO drivers, offers good performance and features like dynamic resolution).
#            Often requires `-display gtk,gl=on` or similar for host-side acceleration to be effective.
#   `none`: No VGA card (for headless VMs or if using serial console exclusively).

# Example: GTK display with VirtIO VGA and OpenGL acceleration
#   -display gtk,gl=on -vga virtio
# Example: Headless VM with serial console output to stdio
#   -nographic -serial stdio
#   Or: -display none -vga none -serial stdio
```

## Networking

### User Mode Networking (Default, Easy NAT)
<!-- Easiest to set up, provides NAT for the guest. Guest gets IP like 10.0.2.15.
     Host is not directly accessible from guest by default, and guest is not directly accessible from host/network. -->
```bash
# Simple user mode network:
# -netdev user,id=mynet0 \
# -device virtio-net-pci,netdev=mynet0 # Use VirtIO NIC for performance

# User mode with port forwarding (e.g., host SSH to guest):
# Forward host port 10022 to guest port 22 (SSH).
# -netdev user,id=mynet0,hostfwd=tcp::10022-:22 \
# -device virtio-net-pci,netdev=mynet0
# Now, `ssh user@localhost -p 10022` on the host will connect to guest's SSH.
```

### Tap Networking (Bridged/Advanced)
<!-- Connects guest directly to host network via a tap interface, making it appear as another machine on the network.
     More complex to set up, requires host network configuration (bridge, routing). `sudo` often needed for QEMU. -->
```bash
# Requires a pre-configured tap interface (e.g., tap0) on the host, often added to a bridge.
# 1. Create tap0 on host: `sudo ip tuntap add dev tap0 mode tap user $(whoami)`
# 2. Bring tap0 up: `sudo ip link set tap0 up`
# 3. Add tap0 to a bridge (e.g., br0): `sudo brctl addif br0 tap0` (assuming br0 exists and is configured)
#
# Then, in QEMU command:
# -netdev tap,id=mynet0,ifname=tap0,script=no,downscript=no \
# -device virtio-net-pci,netdev=mynet0
#   `ifname=tap0`: Use host tap interface named tap0.
#   `script=no,downscript=no`: QEMU should not run helper scripts to configure the tap interface
#                              (as we've presumably configured it manually or via a bridge helper).
```
<!-- Bridged networking setup is OS-dependent and can be complex. Refer to QEMU/OS documentation. -->

## USB Redirection & Passthrough

### Redirecting Host USB Device (Legacy Method)
<!-- This method is older and might have limitations. -->
```bash
# Find USB device details on host: `lsusb`
# Example: Bus 001 Device 005: ID 1234:5678 Vendor Name Product Name
#
# -usb \
# -device usb-host,hostbus=<bus_num>,hostaddr=<dev_num>
# Or by vendor/product ID:
# -device usb-host,vendorid=0x1234,productid=0x5678
# Example:
# qemu-system-x86_64 ... -usb -device usb-host,vendorid=0x1234,productid=0x5678
# Might require root/proper permissions on /dev/bus/usb.
```

### Using USB Controller and Redirecting (More Flexible for GTK/SPICE)
<!-- Emulate a USB controller and then redirect devices via monitor or UI. -->
```bash
# Add a USB controller (e.g., XHCI for USB 3.0)
# -device qemu-xhci,id=xhci
#
# Then, use QEMU monitor or UI (like GTK's "USB Device Selection") to redirect specific host USB devices.
# QEMU Monitor command: `usb_add host:<vendor_id>:<product_id>` or `usb_add host:<bus>.<addr>`
```

## QEMU Monitor Console
<!-- Interactive console for managing the running VM. -->
```bash
# Accessing the monitor:
#   If started with `-monitor stdio`: Press Ctrl+A, then C. (Ctrl+A, X to quit QEMU).
#   If started with `-monitor telnet::4444,server,nowait`: `telnet localhost 4444`.

# Common monitor commands:
#   `help`: List available commands.
#   `info status`: Show VM status (running, paused).
#   `info version`: Show QEMU version.
#   `info block`: Show information about block devices.
#   `info network`: Show network stack state.
#   `system_powerdown`: Send ACPI power down signal to guest.
#   `system_reset`: Reset the VM.
#   `quit` or `q`: Quit QEMU.
#   `stop`: Pause the VM.
#   `cont`: Continue a paused VM.
#   `screendump <filename.ppm>`: Save current VM display as an image.
#   `savevm <snapshot_name>`: Save a VM state snapshot (disk and RAM) if VM was started appropriately (e.g., not with -snapshot option).
#   `loadvm <snapshot_name>`: Restore a VM state snapshot.
#   `delvm <snapshot_name>`: Delete a VM state snapshot.
#   `eject <device_name>`: Eject media (e.g., `eject ide1-cd0` for a CD-ROM).
#   `change <device_name> <filename>`: Change media (e.g., `change ide1-cd0 /path/to/new.iso`).
```

## Other Useful Options

### Sound Card
```bash
# Emulate an Intel HD Audio controller (common)
# -device ich9-intel-hda -device hda-duplex
# Other options: -soundhw es1370,sb16,ac97,hda (older way to specify multiple)
```

### Serial/Parallel Ports
```bash
# Redirect guest serial port to host stdio (already shown)
# -serial stdio
# Redirect to a PTY (pseudo-terminal)
# -serial pty # QEMU will print the PTY device (e.g., /dev/pts/X)
# Redirect to a file
# -serial file:output.log
# Disable parallel port
# -parallel none
```

### BIOS/Firmware Files
```bash
# -bios <bios_file.bin>: Specify a custom BIOS file.
# -pflash <uefi_firmware.fd>: Specify a UEFI firmware file for flash memory (for UEFI VMs).
# Often QEMU finds default BIOS/UEFI files. For UEFI, you might need separate files for code and vars.
# Example for UEFI with separate code/vars (often needed for secure boot or specific setups):
#   -drive if=pflash,format=raw,readonly=on,file=/path/to/OVMF_CODE.fd \
#   -drive if=pflash,format=raw,file=/path/to/OVMF_VARS.fd
```

### `-snapshot` Option (Ephemeral VM)
<!-- Start VM in snapshot mode: any changes made to disk images are temporary and written to temp files.
     The original disk images are not modified. Useful for safe testing. -->
```bash
# qemu-system-x86_64 ... -snapshot -hda my_vm_disk.qcow2 ...
# All changes to my_vm_disk.qcow2 are discarded when the VM powers off.
```
<!-- This is different from qcow2-level snapshots managed by `qemu-img` or the monitor's `savevm`. -->

## Performance Tips
*   **Use KVM:** Always use `--enable-kvm` (or `-enable-kvm`) if your host CPU supports virtualization (Intel VT-x, AMD-V) and KVM modules are loaded. This provides near-native speed.
*   **VirtIO Drivers:** Use VirtIO devices (`virtio-blk-pci` for disks, `virtio-net-pci` for network, `virtio-vga` or `virtio-gpu-pci` for graphics, `virtio-balloon` for memory management) in the guest OS. Install guest VirtIO drivers for best performance.
*   **Disk Cache Options:** Experiment with `-drive ... cache=<mode>`. `cache=none` or `cache=writethrough` can be safer but slower. `cache=writeback` is faster but has risks on host crash if not using barriers properly.
*   **CPU Model:** `-cpu host` passes through most host CPU features, generally good with KVM.
*   **Memory Allocation:** Allocate enough RAM (`-m`) to the guest but don't over-allocate (starving the host).
*   **Multi-core:** Use `-smp` to assign multiple cores if your guest OS and workload benefit from it.
*   **Display:** For GUI guests, `-vga virtio -display gtk,gl=on` (or `sdl,gl=on`) can improve graphics responsiveness. For headless, use `-nographic` or `-display none`.
