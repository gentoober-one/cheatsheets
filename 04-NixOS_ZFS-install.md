# NixOS on ZFS Installation Guide

<!--
**WARNING: ADVANCED USERS ONLY.** This guide details installing NixOS on ZFS.
It involves destructive disk operations like `blkdiscard` and `parted`.
**Proceed with extreme caution.** It is strongly recommended to:
  - Test this guide in a Virtual Machine first.
  - Back up any important data from the target disk(s) before starting.
  - Understand each command and its implications.
ZFS has specific hardware considerations for optimal performance and reliability, such as ample RAM (and ECC RAM is highly recommended).
This guide assumes you are comfortable with command-line interfaces, disk partitioning, and ZFS concepts.
-->

## Phase 1: Initial Setup & Prerequisites

<!-- These steps are performed from the NixOS live installer environment. -->

### 1.1 List Available Disks
<!-- Identify your target disk(s). This command helps by listing disks by their stable IDs, filtering out EUI-named entries. -->
```bash
find /dev/disk/by-id/ | grep -v "eui"
```

### 1.2 Set Target Disk Variable(s)
<!-- Set the DISK variable to the path(s) of your target disk(s).
     For a single disk setup, use its full path.
     For multiple disks (e.g., for a ZFS mirror), list them space-separated within the quotes.
     Example for single disk: DISK="/dev/disk/by-id/your-disk-id"
     Example for multiple disks: DISK="/dev/disk/by-id/disk1-id /dev/disk/by-id/disk2-id"
-->
```bash
# Replace with your actual disk ID(s)
DISK='/dev/disk/by-id/ata-QEMU_HARDDISK_QM00001'
# For multiple disks (mirror example):
# DISK="/dev/disk/by-id/disk-id-1 /dev/disk/by-id/disk-id-2"
```

### 1.3 Create a Temporary Mount Point
<!-- This script will use a temporary directory to mount filesystems during installation. -->
```bash
MNT=$(mktemp -d)
echo "Temporary mount point: ${MNT}"
```

### 1.4 Enable Nix Flakes Functionality (User Environment)
<!-- Flakes are a modern way to manage Nix configurations and dependencies.
     This enables it for the current user in the live environment. -->
```bash
mkdir -p ~/.config/nix
echo "experimental-features = nix-command flakes" >> ~/.config/nix/nix.conf
# Apply immediately by restarting the daemon (if applicable) or sourcing profile.
# Often, just opening a new shell or proceeding works if nix commands are called later.
```

### 1.5 Install Necessary Tools
<!-- Ensure git, jq (for JSON processing, though not explicitly used later in this script), and parted (for partitioning) are available. -->
<!-- These are installed into the live environment using nix-env. -->
```bash
# Check and install git if not present
if ! command -v git &> /dev/null; then
  echo "Installing git..."
  nix-env -f '<nixpkgs>' -iA git
fi

# Check and install jq if not present
if ! command -v jq &> /dev/null; then
  echo "Installing jq..."
  nix-env -f '<nixpkgs>' -iA jq
fi

# Check and install parted (provides partprobe) if not present
if ! command -v partprobe &> /dev/null; then
  echo "Installing parted..."
  nix-env -f '<nixpkgs>' -iA parted
fi
```

## Phase 2: Disk Partitioning

<!-- This phase prepares the disk(s) with the necessary partitions for ZFS and booting. -->
<!-- The script uses a function to apply partitioning to each disk specified in the $DISK variable. -->

### 2.1 Define Partitioning Function
<!-- This shell function partitions a single disk.
     It creates a GPT label and several partitions:
       - EFI: For UEFI boot files.
       - bpool: Small partition for ZFS boot pool (kernel, initrd).
       - rpool: Main partition for ZFS root pool (OS, home, etc.).
       - BIOS: BIOS boot partition for GRUB on BIOS systems (legacy boot compatibility).
-->
```bash
partition_disk () {
  local disk="${1}"
  echo "Partitioning disk: ${disk}..."

  # Securely wipe the disk by discarding all blocks. Use with extreme caution.
  # The '|| true' ensures the script continues even if blkdiscard fails (e.g., on non-SSD or virtual disks).
  sudo blkdiscard -f "${disk}" || true

  # Create partitions using parted.
  sudo parted --script --align=optimal "${disk}" -- \
    mklabel gpt \
    mkpart EFI 2MiB 1GiB        `# Partition 1: EFI System Partition (ESP)` \
    mkpart bpool 1GiB 5GiB      `# Partition 2: ZFS boot pool (bpool)` \
    mkpart rpool 5GiB -1s       `# Partition 3: ZFS root pool (rpool), -1s means use the rest of the disk` \
    mkpart BIOS 1MiB 2MiB       `# Partition 4: BIOS boot partition (for GRUB on BIOS systems)` \
    set 1 esp on                `# Set ESP flag on partition 1` \
    set 4 bios_grub on          `# Set bios_grub flag on partition 4 (for GRUB)` \
    set 4 legacy_boot on        `# Also set legacy_boot flag for broader compatibility (optional for pure UEFI)`

  # Inform the OS of partition table changes.
  sudo partprobe "${disk}"
  # Wait for udev to process partition changes.
  sudo udevadm settle
  echo "Disk ${disk} partitioned."
}
```

### 2.2 Apply Partitioning to Target Disk(s)
```bash
# This loop iterates through each disk defined in the $DISK variable (space-separated if multiple).
for i in ${DISK}; do
   partition_disk "${i}"
done
```
<!-- After this step, your disk(s) will have a new GPT partition table. Verify with `lsblk` or `fdisk -l`. -->

## Phase 3: ZFS Pool and Dataset Configuration

<!-- ZFS pools (zpools) are created from the partitions, and then ZFS datasets (filesystems) are created within these pools. -->
<!-- Note: `ashift=12` is generally recommended for modern drives with 4K sectors. -->
<!-- `autotrim=on` is beneficial for SSDs. -->

### 3.1 Create ZFS Boot Pool (`bpool`)
<!-- The boot pool (`bpool`) stores essential boot files. It's kept separate and typically smaller. -->
<!-- `compatibility=grub2` ensures GRUB can read the pool. -->
```bash
# The shellcheck disable is for SC2046, which warns against unquoted command substitution.
# Here, it's used intentionally to expand the disk part arguments.
# Constructing the list of partitions for bpool (e.g., /dev/disk/by-id/disk1-id-part2 /dev/disk/by-id/disk2-id-part2)
bpool_devices=""
for i in ${DISK}; do
  bpool_devices+="${i}-part2 " # Append partition 2 of each disk
done

echo "Creating bpool with devices: ${bpool_devices}"
sudo zpool create \
    -o compatibility=grub2 \
    -o ashift=12 \
    -o autotrim=on \
    -O acltype=posixacl \
    -O canmount=off \
    -O compression=lz4 \
    -O devices=off \
    -O normalization=formD \
    -O relatime=on \
    -O xattr=sa \
    -O mountpoint=/boot \
    -R "${MNT}" \
    bpool \
    ${bpool_devices} # Use the constructed list of devices
```

### 3.2 Create ZFS Root Pool (`rpool`)
<!-- The root pool (`rpool`) contains the main operating system and user data. -->
```bash
rpool_devices=""
for i in ${DISK}; do
  rpool_devices+="${i}-part3 " # Append partition 3 of each disk
done

echo "Creating rpool with devices: ${rpool_devices}"
sudo zpool create \
    -o ashift=12 \
    -o autotrim=on \
    -R "${MNT}" \
    -O acltype=posixacl \
    -O canmount=off \
    -O compression=lz4 \
    -O dnodesize=auto \
    -O normalization=formD \
    -O relatime=on \
    -O xattr=sa \
    -O mountpoint=/ \
    rpool \
    ${rpool_devices} # Use the constructed list of devices
```

### 3.3 Create Root System Container Dataset
<!-- This dataset acts as a container for other NixOS system datasets. `canmount=off` means it won't be mounted directly. -->
```bash
sudo zfs create \
    -o canmount=off \
    -o mountpoint=none \
    rpool/nixos
```

### 3.4 Create System Datasets
<!-- These datasets are for specific parts of the filesystem.
     `mountpoint=legacy` means ZFS won't automatically mount them; NixOS will handle mounting based on its configuration.
     The temporary mounts created here are for `nixos-install` to place files correctly.
-->
```bash
echo "Creating ZFS system datasets and mounting them under ${MNT}..."
# Root filesystem dataset
sudo zfs create -o mountpoint=legacy rpool/nixos/root
sudo mount -t zfs rpool/nixos/root "${MNT}/"

# Home directory dataset
sudo zfs create -o mountpoint=legacy rpool/nixos/home
sudo mkdir -p "${MNT}/home"
sudo mount -t zfs rpool/nixos/home "${MNT}/home"

# Var parent dataset (not mounted directly)
sudo zfs create -o mountpoint=none rpool/nixos/var

# Specific /var datasets
sudo zfs create -o mountpoint=legacy rpool/nixos/var/lib
sudo mkdir -p "${MNT}/var/lib"
sudo mount -t zfs rpool/nixos/var/lib "${MNT}/var/lib"

sudo zfs create -o mountpoint=legacy rpool/nixos/var/log
sudo mkdir -p "${MNT}/var/log"
sudo mount -t zfs rpool/nixos/var/log "${MNT}/var/log"

# Boot pool datasets
sudo zfs create -o mountpoint=none bpool/nixos
sudo zfs create -o mountpoint=legacy bpool/nixos/root
sudo mkdir -p "${MNT}/boot" # Ensure /boot exists before mounting bpool root on it
sudo mount -t zfs bpool/nixos/root "${MNT}/boot"
```
<!-- Note: Some /var subdirectories like /var/empty or /var/run might be needed by NixOS later, but NixOS usually creates them.
     The original script had some redundant mkdir and mount commands for /var/lib and /var/log. Streamlined above. -->

## Phase 4: EFI System Partition (ESP) Setup

<!-- This phase formats and mounts the EFI System Partition(s). -->
<!-- ESPs are crucial for UEFI booting. -->
```bash
echo "Formatting and mounting EFI System Partition(s)..."
# This loop iterates through each disk defined in the $DISK variable.
# It assumes partition 1 of each disk is the ESP.
for i in ${DISK}; do
  esp_partition="${i}-part1"
  # Extract a short, unique name for the mount point, e.g., from 'ata-QEMU_HARDDISK_QM00001' to 'QM00001'
  # This attempts to create a unique subdirectory for each ESP under /boot/efis/
  # It takes the part after the last '/' (basename) and then the part after the last '-' if any, or uses the full basename.
  disk_id_short_name=$(basename "${i}")
  disk_id_short_name=${disk_id_short_name##*-}

  esp_mount_point="${MNT}/boot/efis/${disk_id_short_name}-part1"

  echo "Formatting ESP: ${esp_partition}"
  sudo mkfs.vfat -n EFI "${esp_partition}"

  echo "Mounting ESP: ${esp_partition} to ${esp_mount_point}"
  sudo mkdir -p "${esp_mount_point}"
  # Using iso8859-1 for iocharset is a common practice for VFAT to avoid issues with certain characters,
  # though utf8 might also work on modern systems.
  sudo mount -t vfat -o iocharset=iso8859-1 "${esp_partition}" "${esp_mount_point}"
done
```

## Phase 5: NixOS Configuration

<!-- This phase involves fetching a base NixOS Flake configuration, initializing it as a Git repository,
     and customizing it for the current hardware. -->

### 5.1 Clone Template Flake Configuration
<!-- This example clones a specific ZFS-ready NixOS Flake configuration from GitHub.
     You might want to use your own or a different template. -->
```bash
echo "Cloning NixOS Flake configuration template..."
sudo mkdir -p "${MNT}/etc"
# The original guide uses a specific branch `openzfs-guide` from `ne9z/dotfiles-flake`.
# Replace this with your preferred NixOS Flake configuration template if desired.
sudo git clone --depth 1 --branch openzfs-guide \
  https://github.com/ne9z/dotfiles-flake.git "${MNT}/etc/nixos"
```

### 5.2 Initialize Git Repository for System Configuration
<!-- The system's configuration in /etc/nixos will be tracked by Git. -->
<!-- The original cloned repo's .git directory is removed, and a new one is initialized. -->
```bash
echo "Initializing Git repository for system configuration in ${MNT}/etc/nixos..."
sudo rm -rf "${MNT}/etc/nixos/.git"
sudo git -C "${MNT}/etc/nixos/" init -b main # Initialize with 'main' as the default branch
sudo git -C "${MNT}/etc/nixos/" add . # Add all files in /etc/nixos
# Set your Git username and email. These are for local commits to your system configuration.
# Replace with your actual details.
sudo git -C "${MNT}/etc/nixos" config user.email "you@example.com"
sudo git -C "${MNT}/etc/nixos" config user.name "Your Name"
sudo git -C "${MNT}/etc/nixos" commit -asm 'Initial commit of NixOS configuration'
```
<!-- Note: `GENTOOBER` was used in the original; replaced with a more generic "Your Name". -->

### 5.3 Customize Configuration for Hardware
<!-- These `sed` commands adapt the cloned configuration to the specific hardware of this machine.
     This part is highly dependent on the structure of the cloned Flake. -->

#### 5.3.1 Update Disk Path in Configuration
<!-- Modifies a file (default.nix) to correctly reference the disk path.
     This assumes the template uses a placeholder for the disk path. -->
```bash
# This loop takes the first disk from the DISK variable and uses its directory path.
# This might need adjustment if you have multiple disks and the config expects something different.
# The original script breaks after the first disk, assuming all disks share the same by-id parent directory.
first_disk_path=""
for i in ${DISK}; do
  first_disk_path="${i%/*}/" # Gets the directory part, e.g., /dev/disk/by-id/
  break
done

if [ -n "${first_disk_path}" ]; then
  echo "Updating disk path in configuration to: ${first_disk_path}"
  sudo sed -i \
    "s|/dev/disk/by-id/|${first_disk_path}|" \
    "${MNT}/etc/nixos/hosts/exampleHost/default.nix"
else
  echo "Warning: Could not determine disk path for configuration update."
fi
```

#### 5.3.2 Update Boot Device Names
<!-- Updates a placeholder with the actual disk names for boot configuration. -->
```bash
diskNames=""
for i in ${DISK}; do
  diskNames="${diskNames} \"${i##*/}\"" # Extracts the basename of each disk ID
done
diskNames=$(echo "${diskNames}" | sed 's/^ *//;s/ *$//') # Trim leading/trailing whitespace

if [ -n "${diskNames}" ]; then
  echo "Updating boot device names in configuration to: ${diskNames}"
  sudo sed -i "s|\"bootDevices_placeholder\"|${diskNames}|g" \
    "${MNT}/etc/nixos/hosts/exampleHost/default.nix"
else
  echo "Warning: Could not determine disk names for boot devices configuration."
fi
```

#### 5.3.3 Generate a Unique System Identifier (bootloader ID)
<!-- Replaces a placeholder with a random 4-byte hex string for bootloader ID. -->
```bash
random_id=$(head -c4 /dev/urandom | od -A none -t x4 | sed 's/ //g' || true)
echo "Setting unique system identifier (bootloader ID) to: ${random_id}"
sudo sed -i "s|\"abcd1234\"|\"${random_id}\"|g" \
  "${MNT}/etc/nixos/hosts/exampleHost/default.nix"

```

#### 5.3.4 Set System Architecture
<!-- Updates the Flake configuration with the system's architecture (e.g., x86_64-linux). -->
```bash
system_arch=$(uname -m || true)-linux
echo "Setting system architecture in Flake to: ${system_arch}"
sudo sed -i "s|\"x86_64-linux\"|\"${system_arch}\"|g" \
  "${MNT}/etc/nixos/flake.nix"
```

### 5.4 Detect Kernel Modules for Boot (initrd)
<!-- This section attempts to automatically detect necessary kernel modules for the initrd.
     It copies `nixos-generate-config`, modifies it to print modules, and then extracts them.
     This is a clever but complex way to get this information. -->
```bash
echo "Detecting kernel modules for initrd..."
# Ensure nixos-generate-config is executable and in PATH from the live environment
nixos_generate_config_path=$(command -v nixos-generate-config || true)

if [ -z "${nixos_generate_config_path}" ]; then
    echo "Error: nixos-generate-config not found. Cannot detect kernel modules."
else
    sudo cp "${nixos_generate_config_path}" ./nixos-generate-config.tmp
    sudo chmod a+rw ./nixos-generate-config.tmp

    # Modify the copy to print the $initrdAvailableKernelModules variable from Perl script
    # This is a highly specific modification depending on nixos-generate-config internal structure.
    # It assumes $initrdAvailableKernelModules is a Perl variable that gets interpolated.
    # shellcheck disable=SC2016 # We want literal $initrdAvailableKernelModules here
    sudo sed -i -e '/my @warnings;/a print STDOUT $initrdAvailableKernelModules . "\\n"; exit 0;' ./nixos-generate-config.tmp
    # The original script appended 'print STDOUT $initrdAvailableKernelModules'. This modification tries to make it exit after printing.
    # A safer way to get modules might be to parse the hardware-configuration.nix generated by the tool.
    # However, proceeding with the guide's method:

    # Run the modified script to get the modules.
    # The `--no-filesystems` option is used as we are setting up ZFS manually.
    kernelModules=$(sudo ./nixos-generate-config.tmp --show-hardware-config --no-filesystems | tail -n1 || true)
    # The original `tail -n1` might be fragile. If the script prints other things, this could fail.
    # Assuming $kernelModules variable now holds a space-separated string of module names.

    # Clean up the temporary script
    sudo rm ./nixos-generate-config.tmp

    if [ -n "${kernelModules}" ]; then
        echo "Detected kernel modules: ${kernelModules}"
        # Format for Nix list: "module1" "module2"
        formattedKernelModules=$(echo "${kernelModules}" | sed 's/ /" "/g;s/^/"/;s/$/"/')
        sudo sed -i "s|\"kernelModules_placeholder\"|${formattedKernelModules}|g" \
            "${MNT}/etc/nixos/hosts/exampleHost/default.nix"
    else
        echo "Warning: Could not detect kernel modules. You may need to add them manually."
    fi
fi
```
<!-- Note: The method to extract kernel modules is quite advanced and specific to the `nixos-generate-config` script's internals. -->
<!-- A more standard approach might be to run `nixos-generate-config --root ${MNT}` to generate a hardware config,
     then manually copy relevant modules from the generated `/mnt/etc/nixos/hardware-configuration.nix` if needed. -->

### 5.5 Set Root Password
<!-- Generates a hashed root password and adds it to the NixOS configuration. -->
<!-- `mkpasswd` (from `whois` package usually) is used here. Ensure it's available or use `nix-shell -p whois --run "mkpasswd..."` -->
```bash
echo "Setting root password (hashed)..."
# Ensure mkpasswd is available
if ! command -v mkpasswd &> /dev/null; then
  echo "mkpasswd command not found, attempting to install via nix-shell..."
  rootPwd=$(nix-shell -p whois --run "mkpasswd -m SHA-512" || echo "password_hash_placeholder")
else
  rootPwd=$(mkpasswd -m SHA-512)
fi

if [[ "${rootPwd}" == "password_hash_placeholder" || -z "${rootPwd}" ]]; then
    echo "Error: Failed to generate root password hash. Please set it manually in configuration.nix."
else
    echo "Root password hash generated."
    # Replace placeholder in the main configuration.nix (path might vary based on Flake structure)
    # This assumes configuration.nix is at the root of the Flake.
    sudo sed -i \
        "s|rootHash_placeholder|${rootPwd}|" \
        "${MNT}/etc/nixos/configuration.nix" # Adjust path if your Flake's main config is elsewhere
fi
```
<!-- The user will be prompted to enter the password for `mkpasswd`. -->

## Phase 6: NixOS Installation

<!-- At this point, the configuration should be ready. -->
<!-- The original guide mentions enabling NetworkManager or GNOME in configuration.nix as a manual step for the user. -->
<!-- This is where you would customize `${MNT}/etc/nixos/configuration.nix` or related files in the Flake structure. -->
```
# Reminder: YOU CAN ENABLE NETWORKMANAGER FOR WIRELESS NETWORKS AND GNOME DESKTOP ENVIRONMENT
# IN ${MNT}/etc/nixos/configuration.nix (or other files within your Flake structure) BEFORE PROCEEDING.
# Example:
#   services.networkmanager.enable = true;
#   services.xserver.enable = true;
#   services.xserver.displayManager.gdm.enable = true;
#   services.xserver.desktopManager.gnome.enable = true;
```

### 6.1 Commit Configuration Changes
<!-- Commit any manual changes made to the configuration. -->
```bash
echo "Committing final configuration changes before installation..."
sudo git -C "${MNT}/etc/nixos" add .
sudo git -C "${MNT}/etc/nixos" commit -am 'Finalize configuration before install' || echo "No changes to commit or initial commit was the same."
```

### 6.2 Update Flake Lock File
<!-- This locks the specific versions of dependencies (inputs) for the Flake, ensuring reproducibility. -->
```bash
echo "Updating Flake lock file..."
# The Flake path needs to be accessible by the nix command.
# If running as root, ensure root can access this path or adjust permissions.
sudo nix flake update --commit-lock-file "git+file://${MNT}/etc/nixos"
```

### 6.3 Install NixOS
<!-- This command installs NixOS to the target system defined by `${MNT}` using the specified Flake. -->
<!-- `--no-root-passwd` is used because the password hash is already set in `configuration.nix`. -->
```bash
echo "Starting NixOS installation..."
sudo nixos-install \
    --root "${MNT}" \
    --no-root-passwd \
    --flake "git+file://${MNT}/etc/nixos#exampleHost"
#   `exampleHost` should match a NixOS configuration defined in your flake.nix.
```

## Phase 7: Post-Installation

### 7.1 Unmount Filesystems
<!-- Cleanly unmount all filesystems and export ZFS pools before rebooting. -->
```bash
echo "Unmounting filesystems..."
sudo umount -Rl "${MNT}" # Recursive and lazy unmount
echo "Exporting ZFS pools..."
sudo zpool export -a
```

### 7.2 Reboot
<!-- Reboot into your new NixOS on ZFS system. -->
```bash
echo "Installation complete. System will now reboot."
echo "Reboot and be happy : )"
sudo reboot
```

