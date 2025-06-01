# NixOS & Nix Cheatsheet

<!--
This cheatsheet provides useful commands and configuration tips for NixOS and the Nix package manager.
NixOS is a Linux distribution built on the Nix package manager, featuring:
- Declarative Configuration: The entire system configuration (packages, services, settings) is defined in a central file (usually /etc/nixos/configuration.nix).
- Reproducible Builds: Nix builds packages in isolated environments, ensuring that a given Nix expression always produces the same result.
- Atomic Upgrades & Rollbacks: System updates are atomic; if an update fails, the system can easily roll back to a previous working state (generation).
- The Nix Store: All packages and their dependencies are stored in `/nix/store` in unique, immutable paths based on their hash.

The Nix package manager can also be used on other Linux distributions and macOS.
-->

## Enabling Experimental Nix Features (User-Specific)

<!--
The Nix package manager has experimental features that can enhance its functionality.
`nix-command` refers to the newer, more consistent CLI (e.g., `nix build`, `nix develop`).
`flakes` are a newer way to manage Nix expressions, providing better reproducibility and dependency management.
These features are becoming standard for modern Nix workflows.

The following commands enable these features for the current user by modifying their local Nix configuration file.
This does not affect the system-wide NixOS configuration if you are on NixOS.
For system-wide enabling on NixOS, add to `/etc/nixos/configuration.nix`:
  nix.extraOptions = ''
    experimental-features = nix-command flakes
  '';
Or, for NixOS 22.11 and later:
  nix.settings.experimental-features = [ "nix-command" "flakes" ];
-->

### Steps to Enable `nix-command` and `flakes` for Current User:

#### 1. Create the Nix Configuration Directory (if it doesn't exist)
<!-- This directory stores user-specific Nix settings. -->
```bash
mkdir -p ~/.config/nix
```

#### 2. Add Experimental Features to `nix.conf`
<!-- This command appends the line to enable the features. If the file or options already exist, manage them carefully. -->
```bash
# Ensure the line is added only once, or manage this file with a text editor for more control.
# Check if the line already exists
if ! grep -q "experimental-features = nix-command flakes" ~/.config/nix/nix.conf 2>/dev/null; then
  echo "experimental-features = nix-command flakes" >> ~/.config/nix/nix.conf
  echo "Experimental features (nix-command, flakes) enabled for user. Restart your terminal or Nix daemon."
else
  echo "Experimental features (nix-command, flakes) already seem to be enabled in ~/.config/nix/nix.conf."
fi
```
<!-- After these changes, you might need to restart any Nix daemons or open a new shell session for the settings to take full effect. -->

## NixOS System Configuration (`configuration.nix`)

<!-- The entire state of a NixOS system is declaratively defined in `/etc/nixos/configuration.nix`.
     Modifying this file and then running `nixos-rebuild` is the primary way to manage the system. -->

### Basic Structure of `configuration.nix`
```nix
# /etc/nixos/configuration.nix
{ config, pkgs, ... }: # Function arguments: current config, nixpkgs set, and other modules

{
  imports = [ # List of other .nix files to include
    ./hardware-configuration.nix # Hardware-specific settings detected during installation
    # ./my-custom-module.nix
  ];

  # Bootloader configuration (example for GRUB on UEFI)
  boot.loader.grub.enable = true;
  boot.loader.grub.version = 2;
  boot.loader.grub.device = "nodev"; # For UEFI
  boot.loader.grub.efiSupport = true;
  # boot.loader.grub.useOSProber = true; # If you need to dual boot other OSes

  # Networking
  networking.hostName = "nixos-pc"; # Set your hostname
  networking.networkmanager.enable = true; # Use NetworkManager
  # networking.firewall.enable = true; # Enable the firewall
  # networking.firewall.allowedTCPPorts = [ 80 443 ]; # Allow HTTP and HTTPS

  # System packages available globally
  environment.systemPackages = with pkgs; [
    vim # Text editor
    git # Version control
    wget # File downloader
    firefox # Web browser
    # Add more packages here
  ];

  # Services to enable
  services.openssh.enable = true; # Enable SSH daemon
  # services.nginx.enable = true; # Example: enable Nginx web server

  # User accounts
  users.users.your_username = {
    isNormalUser = true;
    extraGroups = [ "wheel" "networkmanager" ]; # 'wheel' for sudo access
    # home = "/home/your_username"; # Default
    # shell = pkgs.zsh; # Example: set default shell to zsh
    # hashedGassword = "..."; # Set initial password (use `mkpasswd -m sha-512` to generate)
  };

  # Allow unfree packages (if needed, e.g. for some drivers or specific software)
  # nixpkgs.config.allowUnfree = true;

  # Systemd journal settings (optional)
  # services.journald.extraConfig = ''
  #   SystemMaxUse=2G
  # '';

  # This value determines the NixOS release from which the default
  # settings for stateful data, like file locations and database versions
  # on your system were taken. It‘s perfectly fine and recommended to leave
  # this value at the release version of the first install of this system.
  # Before changing this value read the documentation for this option
  # (e.g. man configuration.nix or on https://nixos.org/manual/nixos/stable/).
  system.stateVersion = "23.11"; # Or whatever version you installed with.
}
```

### `nixos-rebuild`: Applying Configuration Changes
<!-- `nixos-rebuild` is the command to evaluate `configuration.nix`, build the new system generation, and activate it.
     All commands require `sudo`. -->
```bash
# Build the configuration, activate it, and make it the default boot entry.
sudo nixos-rebuild switch

# Build the configuration, activate it, and set it as the default for the NEXT boot.
# If the current build fails to boot properly, the previous working generation will still be the default.
sudo nixos-rebuild boot

# Build and activate the configuration for the current session only.
# Does NOT add an entry to the bootloader or change the default.
# Useful for quick tests. If it breaks, reboot to revert to the previous configuration.
sudo nixos-rebuild test

# Build the configuration but do not activate it or add to boot menu.
# The resulting system path will be printed (e.g., /nix/store/...-nixos-system-...).
sudo nixos-rebuild build

# Build and activate, but without creating a bootloader entry.
# Similar to `test` but might be used in specific scripting scenarios.
sudo nixos-rebuild dry-activate

# Show available system generations that you can roll back to.
sudo nixos-rebuild list-generations

# Roll back to a specific generation (e.g., generation 42).
# sudo nixos-rebuild switch --profile /nix/var/nix/profiles/system-42-link
```

## Nix Channels

<!-- Channels are pointers to a set of Nix expressions, typically a version of `nixpkgs` (the Nix Packages collection).
     They provide a way to get updates for packages and NixOS itself. Flakes offer a more modern way to manage dependencies. -->

#### Listing Channels
```bash
# List channels for the current user
nix-channel --list
# List channels for root (system channels on NixOS)
sudo nix-channel --list
```

#### Adding a Channel
```bash
# Add the unstable NixOS channel for the current user
nix-channel --add https://nixos.org/channels/nixos-unstable nixos-unstable-user
# Add for root (system channel)
sudo nix-channel --add https://nixos.org/channels/nixos-23.11 nixos-23.11-stable
```

#### Updating Channels
```bash
# Update all channels for the current user
nix-channel --update
# Update all channels for root (system channels)
sudo nix-channel --update

# Update a specific channel
sudo nix-channel --update nixos # Updates the channel named 'nixos'
```
<!-- After updating system channels (e.g., `sudo nix-channel --update nixos`),
     you typically run `sudo nixos-rebuild switch` to apply the updates to your system. -->

## Nix Package Management (User & System)

### Declarative Package Management (NixOS - Recommended)
<!-- On NixOS, the preferred way to manage system-wide packages is declaratively
     by adding them to `environment.systemPackages` in `/etc/nixos/configuration.nix`. -->
```nix
# In /etc/nixos/configuration.nix:
environment.systemPackages = with pkgs; [
  git
  htop
  ripgrep
  # ... other packages
];
```
<!-- Then run `sudo nixos-rebuild switch`. -->

### Imperative Package Management (`nix-env`)
<!-- `nix-env` allows users to install packages into their personal profile (`~/.nix-profile`).
     This is more traditional but less reproducible than declarative management.
     Useful for temporary tools or when not on NixOS. -->

#### Installing a Package
```bash
# Install a package from the nixpkgs attribute path available in your channels.
# -iA or --install -A: Install by attribute path.
nix-env -iA nixpkgs.firefox  # Example for user profile
# For NixOS channel (if distinct, often same as nixpkgs for user):
# nix-env -iA nixos.firefox

# To install a specific version if available via multiple attributes (less common for general use):
# nix-env -iA nixpkgs.legacyPackages.gcc7
```

#### Uninstalling a Package
```bash
# -e or --uninstall: Uninstall a package by its name as listed in `nix-env -q`.
nix-env -e firefox
```

#### Listing User-Installed Packages
```bash
nix-env -q
```

#### Upgrading User-Installed Packages
```bash
# Upgrade all packages in the user's profile to the latest versions available in their channels.
nix-env -u '*' --always
# Use `nix-env -u -p` for a dry run.
```

### Searching for Packages
```bash
# Using the new Nix CLI (if nix-command and flakes are enabled):
nix search nixpkgs <keyword>
# Example: nix search nixpkgs ripgrep

# Using legacy nix-env:
# -qaP: Query all available packages, showing attribute Path.
nix-env -qaP '*<keyword>*' # Wildcards are often needed. Case-sensitive.
# Example: nix-env -qaP '*ripgrep*'

# Online search: https://search.nixos.org/packages
```

## Ad-hoc Developer/Shell Environments

### `nix-shell` (Legacy and Flakes)
<!-- `nix-shell` creates temporary environments with specified packages available, without installing them globally or in the user profile. -->
```bash
# Start a shell with specified packages available.
# -p: Packages to include.
nix-shell -p git nodejs ripgrep
# You are now in a new shell with git, nodejs, and ripgrep.
# Type `exit` or Ctrl+D to leave this temporary shell.

# Use a `shell.nix` or `default.nix` file for more complex environments:
# Create shell.nix:
# { pkgs ? import <nixpkgs> {} }:
# pkgs.mkShell {
#   buildInputs = [
#     pkgs.python3
#     pkgs.go
#   ];
#   shellHook = ''
#     echo "Entered Nix shell for Python and Go!"
#     export MY_VAR="Hello from Nix"
#   '';
# }
# Then run `nix-shell` in the same directory.
```

### `nix develop` (Flakes-based)
<!-- `nix develop` is the Flakes-based way to enter developer shells defined in `flake.nix`. -->
```bash
# Assuming a flake.nix in the current directory provides a devShell:
nix develop

# For a specific devShell output from a flake:
# nix develop <flake_url>#<devShellName>
# Example: nix develop nixpkgs#bashInteractive (if such an output existed)
```

## Nix Store & Garbage Collection

<!-- The Nix store (`/nix/store`) contains all packages, their dependencies, and build outputs.
     Paths in the store are immutable and identified by a hash of their build inputs.
     Garbage collection removes store paths that are no longer referenced by any system generation,
     user profile, or other roots (like active nix-shell environments or build results). -->

### Understanding Generations (NixOS)
<!-- Each successful `nixos-rebuild switch` or `nixos-rebuild boot` creates a new system generation.
     These are symlinks in `/nix/var/nix/profiles/system-XX-link` pointing to a Nix store path. -->
```bash
# List all system generations
sudo nixos-rebuild list-generations
# Example output:
# 100 /nix/store/xxx-nixos-system-myhost-23.11.20231201... 2023-12-01 10:00:00 (current)
#  99 /nix/store/yyy-nixos-system-myhost-23.11.20231128... 2023-11-28 15:30:00

# Roll back to a specific generation (e.g., generation 99)
# sudo nixos-rebuild switch --profile /nix/var/nix/profiles/system-99-link
```

### Garbage Collection Commands
```bash
# Dry run: Show what would be deleted (for unreferenced store paths).
# -n: Dry run.
# --delete or -d: Delete "dead" store paths (recommended for thorough cleaning).
nix-collect-garbage -n --delete # For user's unreferenced paths
sudo nix-collect-garbage -n --delete # For system's unreferenced paths

# Actual deletion of unreferenced store paths:
sudo nix-collect-garbage --delete

# Delete old generations of profiles (both system and user profiles).
# This removes symlinks to old generations, allowing their store paths to be garbage collected if no longer referenced.
# For user profiles (e.g., created by `nix-env`):
nix-collect-garbage --delete-old
# For system profiles (NixOS generations):
sudo nix-collect-garbage --delete-old
# You can specify how many old generations to keep, e.g., `sudo nixos-collect-garbage -d --delete-older-than 7d` (delete system generations older than 7 days).

# Clean up results of previous builds from `/nix/var/log/nix/drvs` and symlinks in current dir.
# This doesn't remove store paths but cleans up build artifacts.
nix-store --gc --print-result-roots
```

### Optimizing the Nix Store
<!-- Deduplicates identical files in the Nix store by using hard links. This can save disk space. -->
```bash
sudo nix-store --optimise
```

## Nix Flakes (Modern Nix Feature)

<!-- Flakes provide a more reproducible and composable way to manage Nix expressions (code) and their dependencies.
     They use a `flake.nix` file at the root of a project and a `flake.lock` file to pin input versions.
     Ensure `flakes` and `nix-command` are enabled in your Nix configuration (see first section). -->

### Common Flake Commands

#### Initializing a new Flake
```bash
# Create a basic flake.nix in the current directory from a template
nix flake new -t templates#simple my-new-flake # Creates ./my-new-flake/flake.nix
# Common templates: templates#default, templates#minimal
```

#### Showing Flake Outputs
<!-- Inspect the outputs (packages, apps, devShells, etc.) provided by a flake. -->
```bash
nix flake show <flake_url_or_path>
# Example: nix flake show github:NixOS/nixpkgs # Show outputs of the main nixpkgs flake
# Example: nix flake show . # Show outputs of the flake in the current directory
```

#### Building a Package from a Flake
```bash
# nix build <flake_url>#<package_attribute_path>
# The <package_attribute_path> usually refers to `packages.<system>.<packageName>`.
# Example: Build the 'hello' package from the nixpkgs flake for your current system.
nix build nixpkgs#hello
# This creates a `./result` symlink to the built package in the Nix store.

# Build a specific package for a specific system:
# nix build github:NixOS/nixpkgs#legacyPackages.x86_64-linux.cowsay
```

#### Running an Application from a Flake
<!-- Runs an application (defined in `apps` output of a flake) without installing it. -->
```bash
# nix run <flake_url>#<app_name> -- [app_arguments]
# Example: Run 'cowsay' utility from the nixpkgs flake.
nix run nixpkgs#cowsay -- "Hello Flakes!"
```

#### Updating Flake Inputs (Lock File)
<!-- Updates all inputs (dependencies listed in `flake.nix`) and regenerates `flake.lock`. -->
```bash
nix flake update
# Update a specific input:
# nix flake lock --update-input <input_name>
```

#### Entering a Developer Shell from a Flake
<!-- Activates a development shell defined in the flake's `devShells` output. -->
```bash
# If flake.nix provides a default devShell for your system:
nix develop
# For a specific named devShell or from a remote flake:
# nix develop <flake_url>#<devShellName>
# Example: nix develop .#myDevShell
```

## Information & Troubleshooting

#### Show Derivation Information
<!-- Display the derivation (.drv file) information for a store path.
     The derivation describes how the store path was built. -->
```bash
nix show-derivation /nix/store/xxx-some-package-version
```

#### View Build Logs
<!-- Show the build log for a specific store path (if logs were preserved). -->
```bash
nix log /nix/store/xxx-some-package-version
```

#### Dependency Information
```bash
# Why does package1 depend on package2? (New Nix CLI)
# nix why-depends <store_path_or_flake_attr_of_package1> <store_path_or_flake_attr_of_package2>
# Example:
# nix why-depends nixpkgs#ripgrep nixpkgs#pcre2

# List runtime dependencies (references) of a store path:
nix-store -q --references /nix/store/xxx-some-package-version

# List store paths that depend on (refer to) a specific store path:
nix-store -q --referrers /nix/store/yyy-some-library-version

# List what is needed to build a derivation or instantiate a Nix expression (without building):
# nix-instantiate --eval some_expression.nix -A attribute.path --readonly-mode --check-trace
```

#### Nix Configuration
```bash
# Show active Nix configuration settings
nix show-config
```
