# Gentoo Linux Cheatsheet

<!--
This cheatsheet provides a collection of common commands and configurations specific to Gentoo Linux.
Gentoo is a source-based Linux distribution, meaning software is typically compiled from source code
according to user-specified USE flags and other configurations. Portage is its powerful package management system.

CAUTION: Many commands require root privileges. Always understand a command before running it.
Incorrect configurations, especially in `/etc/portage/`, can lead to system instability or breakage.
This guide assumes you have a working Gentoo installation and basic familiarity with Linux command line.
-->

## Portage: Package Management

<!-- Portage is the heart of Gentoo, handling software installation, updates, and system configuration.
     Its behavior is primarily controlled by files in `/etc/portage/`. -->

### Core Configuration Files & Directories

#### `/etc/portage/make.conf`
<!-- This is the main configuration file for Portage. It defines global settings for compilation and package management. -->
```bash
# Example /etc/portage/make.conf entries:

# Compiler flags for C and C++ (optimize for native CPU, common -O2 -pipe settings)
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}" # Fortran flags
FFLAGS="${COMMON_FLAGS}"  # Fortran flags

# Number of parallel jobs for make (typically number of CPU cores + 1, or just number of cores)
# MAKEOPTS="-j$(nproc)"
MAKEOPTS="-j17" # Example: 16 cores + 1

# Global USE flags (features to enable/disable across packages)
# Example: enable X, ALSA, Wayland; disable GNOME, KDE
USE="X alsa wayland -gnome -kde pulseaudio dbus elogind systemd -consolekit"

# Accept keywords for package stability (e.g., ~amd64 for testing branch on amd64)
# ACCEPT_KEYWORDS="~amd64" # For a testing system
ACCEPT_KEYWORDS="amd64"  # For a stable system

# License acceptance (use @FREE for only free licenses, or list specific licenses)
ACCEPT_LICENSE="*" # Accepts all licenses, review carefully.
# ACCEPT_LICENSE="-* @FREE @BINARY-REDISTRIBUTABLE" # More restrictive example

# Video card drivers (used by packages like Mesa, Xorg)
VIDEO_CARDS="amdgpu radeonsi" # Example for AMD GPUs
# VIDEO_CARDS="nvidia" # Example for NVIDIA (proprietary driver)
# VIDEO_CARDS="intel i965" # Example for Intel integrated graphics

# Input devices (for Xorg)
INPUT_DEVICES="libinput synaptics" # Example: libinput for general input, synaptics for touchpad

# Grub bootloader platforms (if using grub)
GRUB_PLATFORMS="efi-64" # For UEFI systems
# GRUB_PLATFORMS="pc" # For BIOS systems

# Localization settings
L10N="en-US" # Targetted localizations for packages that support it
LINGUAS="en_US en" # For gettext messages, space separated

# Default location for downloaded source tarballs
# DISTDIR="/var/cache/distfiles" # Default, ensure this is on persistent storage

# Location for binary packages (if you build and use them)
# PKGDIR="/var/cache/binpkgs"

# CPU_FLAGS_X86: For CPU-specific optimizations, run `emerge -v app-portage/cpuid2cpuflags`
# Then copy the output (e.g. CPU_FLAGS_X86="aes avx ...") into make.conf
# CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt sse sse2 sse3 sse4_1 sse4_2 ssse3"

# Features for Portage
FEATURES="candy parallel-fetch" # 'candy' for fun progress indicator, 'parallel-fetch' for parallel source downloads
# FEATURES="usersync userfetch" # Allow non-root user to sync and fetch (less common, security implications)
# FEATURES="binpkg-multi-instance" # Allows multiple versions of the same binary package
```
<!-- After modifying make.conf, it's often a good idea to run `emerge -uDN @world --keep-going`
     or `emerge -e @world` (full rebuild) to apply changes, depending on what was altered. -->

#### `/etc/portage/package.use/`
<!-- Directory to manage USE flags for specific packages, overriding global settings in `make.conf`. -->
<!-- Files in this directory (e.g., `package.use/myflags`) contain lines like:
     <category>/<package_name> <USE_flag_1> <USE_flag_2> -<disabled_USE_flag>
-->
```
# Example content for a file in /etc/portage/package.use/ (e.g., /etc/portage/package.use/custom):
# Enable 'bluetooth' and 'pulseaudio' for pipewire, disable 'jack'
media-video/pipewire bluetooth pulseaudio -jack

# Enable 'foo' for app-misc/bar package
app-misc/bar foo

# Disable 'qt5' for media-gfx/krita if you prefer a different toolkit version (hypothetical)
# media-gfx/krita -qt5
```

#### `/etc/portage/package.accept_keywords/`
<!-- Directory to accept specific keywords (like `~amd64` for testing) for individual packages,
     without changing the global `ACCEPT_KEYWORDS` in `make.conf`. -->
<!-- Files contain lines like:
     <category>/<package_name> ~amd64  # To accept the testing version
     <category>/<package_name> **      # To accept any masked version (use with extreme caution)
-->
```
# Example content for /etc/portage/package.accept_keywords/custom:
# Accept the testing version of Firefox
www-client/firefox ~amd64

# Accept a specific live/VCS ebuild (e.g., git master)
# dev-util/my-tool **
```

#### `/etc/portage/package.license/`
<!-- Directory to specify accepted licenses for individual packages if they are not covered by `ACCEPT_LICENSE` in `make.conf`. -->
<!-- Files contain lines like:
     <category>/<package_name> <license_name_1> <license_name_2>
     <category>/<package_name> @BINARY-REDISTRIBUTABLE # If it's a binary package
-->
```
# Example content for /etc/portage/package.license/custom:
# Accept the 'example-commercial-license' for 'app-editors/example-editor'
app-editors/example-editor example-commercial-license

# Accept binary firmware for sys-kernel/linux-firmware
sys-kernel/linux-firmware @BINARY-REDISTRIBUTABLE
```

#### `/etc/portage/package.mask/` and `/etc/portage/package.unmask/`
<!-- `package.mask`: Prevent specific packages or versions from being installed.
     `package.unmask`: Override masks, allowing installation of packages that are normally masked by the profile.
     Use with caution, especially unmasking. -->
```
# Example /etc/portage/package.mask/custom:
# Mask a specific problematic version of a package
# >net-libs/package-1.2.3
# Mask all versions of a package
# net-libs/unwanted-package

# Example /etc/portage/package.unmask/custom (use less frequently):
# Unmask a package that is masked by the system profile (understand why it's masked first!)
# =dev-util/dangerous-tool-0.1
```

### Common `emerge` Commands

#### Synchronizing the Portage Tree (ebuild repository)
```bash
# Fetches the latest package information (ebuilds) from the configured repositories.
sudo emerge --sync
# This may also show news items (read with `eselect news read`).

# Modern alternative using emaint (part of portage-utils):
sudo emaint sync --auto # Combines sync for all repos defined in repos.conf
```

#### Updating Installed Packages
```bash
# Update all installed packages (@world set) and their dependencies.
# -u or --update: Update packages to the best version available.
# -D or --deep: Consider the entire dependency tree, not just direct dependencies.
# -N or --newuse: Include packages whose USE flags have changed.
# --keep-going: Continue with other packages even if one fails to merge.
# --with-bdeps=y: Pull in build dependencies as well (useful for developers or specific setups).
sudo emerge -uDN @world --keep-going

# To see what would be updated (pretend mode):
sudo emerge -uDNp @world
```

#### Searching for Packages
```bash
# Search by package name or description.
# -s or --search: Search package names and descriptions using a regex.
emerge -s <keyword_regex>

# -S or --searchdesc: Search only descriptions using a regex (more comprehensive).
emerge -S <keyword_regex>

# For installed packages, `equery` or `qlist` are often better.
# qlist -Iv <package_name_fragment> # List installed packages matching fragment
```

#### Installing a New Package
```bash
# Install a package (e.g., app-editors/vim).
# -a or --ask: Show changes and ask for confirmation. Highly recommended.
# -v or --verbose: Show more detailed output.
sudo emerge -av <category/package_name>
# Example:
sudo emerge -av app-editors/neovim
# If a specific version is needed:
# sudo emerge -av =app-editors/neovim-0.9.5
```

#### Uninstalling a Package
```bash
# Uninstall a package. This usually only removes the specified package, not its now-unneeded dependencies.
# -c or --unmerge: Uninstalls the package.
sudo emerge -c <category/package_name>
# Example:
sudo emerge -c app-editors/leafpad

# To also remove now-unneeded dependencies (orphaned packages):
sudo emerge --depclean -av
# Always run `emerge --depclean -pv` (pretend) first to check what will be removed!
```

#### Cleaning Unneeded Dependencies (`--depclean`)
```bash
# Removes packages that were installed as dependencies but are no longer needed by any explicitly installed package.
# CAUTION: Review the list of packages to be removed carefully.
# -p or --pretend: Show what would be removed without actually doing it. ALWAYS USE THIS FIRST.
sudo emerge -cpv @world # Pretend: check what would be removed
sudo emerge -c @world   # Actual removal after checking

# A common sequence:
sudo emerge -uDN @world      # Update everything
sudo emerge --depclean -pv   # Check for orphans
sudo emerge --depclean -v    # Remove orphans
sudo revdep-rebuild          # Check for broken reverse dependencies (see below)
```

#### Resolving Blockers and Conflicts
<!-- Portage will sometimes report blockers (one package preventing another's installation) or conflicts.
     Read the error messages carefully. They often suggest solutions like changing USE flags,
     unmasking a package, or waiting for updates. Manual intervention in /etc/portage/ is often needed. -->

#### Getting Package Information
```bash
# Show detailed information about a package (USE flags, dependencies, homepage, etc.)
emerge --info <category/package_name>
# Or, for a more focused view of USE flags and keywords:
emerge -vp <category/package_name>

# `equery` (from app-portage/gentoolkit) is very useful:
# equery uses <package_name>: Show USE flags for a package.
# equery depends <package_name>: Show dependencies of a package.
# equery files <package_name>: List files installed by a package.
# equery belongs <filename>: Find which package a file belongs to.
# equery hasuse <USE_flag>: List packages that have a specific USE flag.
# equery list <package_mask_or_name_with_wildcards>: List matching packages.
```

### Advanced Portage Operations

#### Full System Rebuild (`emerge -e @world`)
<!-- Rebuilds all packages that are part of the 'world' set. This is a comprehensive update and recompilation process.
     Useful after major system changes (e.g., compiler update, significant CFLAGS/USE flag changes).
     This can be VERY time-consuming. -->
```bash
# emerge -e @world:
#   -e or --emptytree: Re-emerge all specified packages. This means it will recompile them even if they are already at the latest version
#                      or if their USE flags haven't changed (unlike -N).
#   @world: A set containing all packages explicitly installed by the user, plus their dependencies.

# The `|| until ... done` part is a shell construct to automatically retry the command on failure,
# attempting to skip the problematic package.
# CAUTION: While this retry loop can help automate getting through a long rebuild with a few problematic packages,
# it's CRITICAL to note which packages were skipped. Repeatedly skipping packages can lead to an inconsistent or broken system
# or hide serious issues that need to be addressed manually (e.g., by filing bug reports, masking problematic versions, or fixing local configuration).
# After the process finishes, check the logs (e.g., /var/log/portage/elog/summary.log or build logs in /var/tmp/portage) to see what happened.
sudo emerge -e @world --keep-going || until sudo emerge --resume --skipfirst; do sudo emerge --resume --skipfirst; done
```
<!-- Consider running long emerge processes within `screen` or `tmux` to prevent interruption from network disconnects. -->

#### Preserving Downloaded Source Files and Temporary Build Files
<!-- `DISTDIR` in `/etc/portage/make.conf` (default: `/var/cache/distfiles`) stores downloaded source tarballs. Keep this on persistent storage.
     `/var/tmp/portage` is used for temporary build directories. If it's on tmpfs for speed, its contents are lost on reboot.
     Backing it up (as shown in the original cheatsheet) might preserve build artifacts if a build fails and you want to inspect it later,
     or if you want to avoid re-unpacking sources for very large packages if you reboot mid-compile.
     However, for typical use, Portage manages these temporary build dirs automatically. -->

##### Backing Up `/var/tmp/portage` (if on tmpfs and desired)
```bash
# Check disk usage
sudo du -sh /var/tmp/portage/

# Create a compressed backup (example using lz4)
# Ensure the backup path is on persistent storage.
BACKUP_PATH="/mnt/persistent_storage/backup_var_tmp_portage_$(date +%F).tar.lz4"
# This command tars the *contents* of /var/tmp/portage
sudo tar cpf - -C /var/tmp/portage . | lz4 -vz --best - > "${BACKUP_PATH}"
echo "Backup of /var/tmp/portage contents created at ${BACKUP_PATH}"
```

##### Restoring `/var/tmp/portage` (if on tmpfs and desired)
```bash
# Ensure /var/tmp/portage exists (it should if it's a tmpfs mount in /etc/fstab)
# This command restores the *contents* into /var/tmp/portage
RESTORE_PATH="/mnt/persistent_storage/backup_var_tmp_portage_YYYY-MM-DD.tar.lz4" # Adjust filename
if [ -f "${RESTORE_PATH}" ]; then
    sudo lz4 -d "${RESTORE_PATH}" -c | sudo tar -xpf - -C /var/tmp/portage/
    echo "Restored /var/tmp/portage contents from ${RESTORE_PATH}"
else
    echo "Backup file ${RESTORE_PATH} not found."
fi
```
<!-- Note: Restoring /var/tmp/portage is usually not necessary for normal operation. Portage will recreate build directories as needed. -->

### Cleaning Up

#### `eclean-dist`
<!-- Cleans old or unneeded source files from `DISTDIR` (typically `/var/cache/distfiles`). -->
```bash
# Show what would be cleaned (pretend mode)
sudo eclean-dist -p
# Clean files (interactive, will ask before deleting)
sudo eclean-dist -i
# Clean files non-interactively (be careful)
# sudo eclean-dist --deep # More aggressive, removes files not referenced by any ebuild in current tree
```

#### `eclean-pkg`
<!-- Cleans old or unneeded binary packages from `PKGDIR` (typically `/var/cache/binpkgs`). -->
```bash
# Show what would be cleaned (pretend mode)
sudo eclean-pkg -p
# Clean files (interactive)
sudo eclean-pkg -i
```

## `eselect`: System Configuration Utility

<!-- `eselect` is a Gentoo tool for managing system-wide configurations, often by manipulating symbolic links. -->

### Common `eselect` Modules

#### Kernel Management
```bash
# List available kernel versions (from /usr/src/linux-*)
sudo eselect kernel list
# Set the active kernel symlink (/usr/src/linux)
sudo eselect kernel set <number_or_name>
# Example: sudo eselect kernel set 1
```

#### OpenGL Implementation
```bash
# List available OpenGL implementations (e.g., nvidia, xorg-x11)
sudo eselect opengl list
# Set the active OpenGL implementation
sudo eselect opengl set <name>
# Example: sudo eselect opengl set nvidia
```

#### Java Virtual Machine
```bash
# List available Java VMs
sudo eselect java-vm list
# Set the system-wide default Java VM
sudo eselect java-vm set system <number_or_name>
# Set user-specific default Java VM
# eselect java-vm set user <number_or_name>
```

#### Python Interpreter
```bash
# List available Python versions for various targets (python2, python3, pypy etc.)
eselect python list
# Set the system default for a Python target (e.g., python3)
sudo eselect python set python3 <number_or_name>
# Example: sudo eselect python set python3 python3.11
```

#### News Items (from `emerge --sync`)
```bash
# List news items (often important system announcements or migration instructions)
eselect news list
# Read a specific news item
eselect news read <number_or_all_or_new>
# Mark items as read
# eselect news mark read <number_or_all>
```

#### System Profile
<!-- Profiles define default USE flags, package masks, and other settings for the system. Changing profiles can have significant impacts. -->
```bash
# List available system profiles
sudo eselect profile list
# Set a new system profile (read documentation before changing profiles!)
# sudo eselect profile set <number_or_name>
# Example: sudo eselect profile set default/linux/amd64/17.1/desktop/plasma
# After changing profile, a full system update or rebuild is often necessary:
# sudo emerge -uDN @world or sudo emerge -e @world
```

## Kernel Management

<!-- Managing the Linux kernel in Gentoo often involves manual configuration and compilation,
     though tools like `genkernel` can automate parts of it. -->

### Installing Kernel Sources
```bash
# Emerge the desired kernel sources (e.g., gentoo-sources for patched sources, vanilla-sources for mainline)
sudo emerge -av sys-kernel/gentoo-sources
# Other options: sys-kernel/vanilla-sources, sys-kernel/zen-kernel, etc.
```

### Configuring the Kernel
```bash
# 1. Set the /usr/src/linux symlink to your chosen sources (if not already done)
sudo eselect kernel list
sudo eselect kernel set <number_of_your_sources>

# 2. Navigate to the kernel source directory
cd /usr/src/linux

# 3. Configure the kernel:
#    Method A: Manual configuration (recommended for experienced users)
#      Copy existing config if available (e.g., from old kernel or live CD /proc/config.gz)
#      sudo make oldconfig # Updates current .config based on existing .config and new options
#      sudo make menuconfig # Text-based menu for configuration
#      sudo make nconfig # Alternative ncurses based configurator
#      sudo make xconfig # X11-based graphical configurator (needs Qt)
#      sudo make gconfig # X11-based graphical configurator (needs GTK)
#
#    Method B: Using `genkernel` (more automated, good for beginners or quick setups)
#      `genkernel` can also build and install the kernel and initramfs.
#      emerge sys-kernel/genkernel
#      Example: sudo genkernel --menuconfig all # Configure, then build kernel, modules, and initramfs
```

### Building and Installing the Kernel
```bash
# After configuration (if doing it manually):
cd /usr/src/linux
# Compile the kernel and modules (replace `$(nproc)` with your desired number of jobs)
sudo make -j$(nproc)

# Install modules
sudo make modules_install

# Install the kernel image itself
sudo make install
# This typically copies bzImage (or vmlinuz) to /boot/ and may update symlinks.

# (Optional) Create an initramfs (Initial RAM File System)
# Needed if your root filesystem driver or other critical drivers are modules.
# Using genkernel:
# sudo genkernel --install initramfs
# Or using dracut:
# sudo emerge sys-kernel/dracut
# sudo dracut --kver <kernel_version_string> # e.g., 5.15.80-gentoo
```

### Updating Bootloader
<!-- After installing a new kernel, you MUST update your bootloader configuration (e.g., GRUB, LILO, systemd-boot). -->
```bash
# For GRUB:
sudo grub-mkconfig -o /boot/grub/grub.cfg

# For systemd-boot:
# It often picks up kernels in /boot automatically if named correctly (e.g., /boot/vmlinuz-<version>).
# `bootctl update` might be needed, or simply ensuring files are in the right ESP location.
```

## OpenRC: Service and Runlevel Management

<!-- OpenRC is the default init system and service manager for Gentoo (unless systemd is chosen). -->

### Basic Service Commands
```bash
# Check status of a service
sudo rc-service <service_name> status
# Example: sudo rc-service sshd status

# Start a service
sudo rc-service <service_name> start

# Stop a service
sudo rc-service <service_name> stop

# Restart a service (stop then start)
sudo rc-service <service_name> restart
```

### Managing Services at Boot (Runlevels)
<!-- Runlevels define what services are started at boot. Common runlevels:
     `boot`: System initialization.
     `default`: Standard multi-user operational state.
     `nonetwork`: Multi-user, no network.
     `shutdown`: System shutdown.
-->
```bash
# Add a service to a runlevel (e.g., default runlevel)
sudo rc-update add <service_name> default
# Example: sudo rc-update add sshd default

# Remove a service from a runlevel
sudo rc-update del <service_name> default

# Show services in all runlevels and their status
rc-status --all

# Show services in a specific runlevel
# rc-status --runlevel default
```

## System Maintenance & Troubleshooting

### `dispatch-conf`: Managing Configuration File Updates
<!-- When Portage updates packages, it may install new versions of configuration files.
     `dispatch-conf` helps you merge your custom changes with the new defaults. -->
```bash
sudo dispatch-conf
# It will present diffs and options:
#   u: Use the new config file.
#   z: Zap (delete) the new config file (keep your old one).
#   n: Edit the new config file (then you can merge manually).
#   e: Edit your current config file.
#   m: Interactively merge.
#   q: Quit.
# It's often good to review `/etc/dispatch-conf.conf` for settings.
```

### `revdep-rebuild`: Rebuilding Reverse Dependencies
<!-- After library updates, some packages might be linked against older versions.
     `revdep-rebuild` (from app-portage/gentoolkit) scans for and rebuilds such packages. -->
```bash
sudo revdep-rebuild
# Can be time-consuming. Use `-p` or `--pretend` to see what it would do.
# sudo revdep-rebuild -p
# Consider running after major library updates (e.g., glibc, openssl, gcc).
```

### `glsa-check`: Gentoo Linux Security Advisory Check
<!-- Checks your installed packages against known Gentoo Linux Security Advisories. -->
```bash
# Install if needed: sudo emerge app-portage/glsa-check
# List all GLSAs affecting your system
sudo glsa-check --list all
# Test for specific GLSAs
# sudo glsa-check --test <GLSA-ID>
# Apply necessary updates based on output (usually `emerge -uDN @world` or specific package updates).
```

### Key Log File Locations
<!-- Check these logs for troubleshooting system or package issues. -->
```
/var/log/messages          # General system messages
/var/log/dmesg             # Kernel ring buffer (boot messages)
/var/log/boot.log          # System boot log
/var/log/Xorg.0.log        # X server log
/var/log/portage/elog/*    # Portage build/install logs (per package)
/var/log/emerge.log        # Summary of emerge operations
/var/log/genkernel.log     # If using genkernel
```

### Managing `/var/tmp/portage` (Original Cheatsheet Content)
<!-- This section was at the beginning of the original cheatsheet.
     Its primary use case is if `/var/tmp/portage` is on tmpfs to speed up compilations,
     and you want to preserve its contents (e.g., partially built packages, logs for a failed build) across reboots.
     Normally, Portage manages this directory and its contents are transient.
     If `DISTDIR` (for downloaded sources) is on tmpfs, that's a different problem; `DISTDIR` should be persistent. -->

#### Post-Restore Ebuild Operation (Example from Original)
<!-- This specific command directly invokes `ebuild` to merge a package.
     This is a low-level operation, usually not needed by end-users.
     It might be relevant in development or deep troubleshooting of a specific ebuild.
     Typically, `emerge <package_atom>` is the standard way to install or merge packages. -->
```bash
# sudo ebuild /var/db/repos/gentoo/dev-python/PyQt6/PyQt6-6.5.2.ebuild merge
# This command would attempt to perform the 'merge' phase for this specific ebuild file.
# This assumes you have the ebuild repository synced at /var/db/repos/gentoo/
# and any necessary source code or build dependencies are available.
#
# The note "same to: dev-qt/qtmultimedia-6.5.2" from original suggests another package
# that the author might have been working with in a similar fashion.
# Example:
# sudo ebuild /var/db/repos/gentoo/dev-qt/qtmultimedia/qtmultimedia-6.5.2.ebuild merge
```
<!-- For most users, direct `ebuild` command usage is unnecessary. `emerge` handles these steps. -->
