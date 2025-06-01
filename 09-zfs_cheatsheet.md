# ZFS Cheatsheet

<!--
ZFS is an advanced filesystem and logical volume manager with a strong focus on data integrity, scalability, and ease of administration.
This cheatsheet provides common commands and concepts for managing ZFS storage pools and filesystems.
-->

## Core ZFS Concepts

*   **Pool (zpool):** The fundamental storage unit in ZFS. A pool is constructed from one or more virtual devices (vdevs). It's the highest-level container for ZFS data.
*   **Vdev (Virtual Device):** Represents the building blocks of a pool. A vdev can be a single disk (stripe/simple vdev), a mirror (RAID1-like), a RAID-Z/Z2/Z3 (RAID5/6-like), a hot spare, a log device (SLOG), or a cache device (L2ARC). The redundancy of a pool is determined by the redundancy of its vdevs.
*   **Dataset (ZFS Filesystem):** A ZFS filesystem that is part of a pool. Datasets can be nested, inherit properties from their parent dataset, and are the primary way users interact with stored data (like traditional filesystems). They can have quotas, reservations, compression, encryption, etc.
*   **Zvol (ZFS Volume):** A ZFS block device that resides in a pool. Zvols are often used for iSCSI targets, swap devices, or as virtual hard drives for Virtual Machines (VMs). They appear as devices in `/dev/zvol/<pool>/<zvol>`.
*   **Snapshot:** A read-only, point-in-time copy of a dataset or zvol. Snapshots are created almost instantaneously and are very space-efficient initially, as they only store the differences relative to when the snapshot was taken (due to ZFS's Copy-on-Write nature).
*   **Clone:** A writable copy of a snapshot. Initially, a clone shares space with its parent snapshot but diverges as changes are written to it. Clones depend on their originating snapshot.
*   **CoW (Copy-on-Write):** A core principle of ZFS. Instead of overwriting data in place, ZFS writes new data to a different block and then updates the metadata pointers to reference the new block. This is key for data integrity, snapshots, and clones.
*   **ARC (Adaptive Replacement Cache):** ZFS's primary in-RAM cache for frequently accessed data. It dynamically adjusts its size based on system memory usage and needs.
*   **L2ARC (Level 2 ARC):** An optional secondary caching layer, typically on fast SSDs, used to extend the ARC for read-heavy workloads when RAM is insufficient for the entire working set. L2ARC stores data evicted from ARC.
*   **SLOG (Separate Log Device):** An optional device (usually a fast, power-loss-protected SSD like an NVMe or Optane drive) used to store synchronous write intent logs (part of the ZIL - ZFS Intent Log). It improves performance specifically for synchronous write workloads (e.g., databases, NFS sync writes) by quickly acknowledging writes.
*   **Properties:** Configurable attributes for pools and datasets that control behavior like compression (`compression=lz4`), encryption (`encryption=aes-256-gcm`), quotas (`quota=100G`), reservations (`reservation=10G`), mount points (`mountpoint=/data`), case sensitivity (`casesensitivity=sensitive`), NFS/SMB sharing, etc. Properties can be inherited from parent datasets.
*   **Scrub:** A data integrity check that reads all data within a pool and verifies its checksum against the stored metadata. If data corruption is found and redundancy exists (mirror, RAID-Z), ZFS can repair it. Regular scrubs are highly recommended (e.g., monthly).
*   **Resilver:** The process of reconstructing data onto a replacement disk after a disk failure or replacement in a redundant vdev (mirror or RAID-Z). ZFS only resilvers actual data, not free space, which can be faster than traditional RAID rebuilds.

## Pool Management (`zpool` commands)

<!-- `zpool` is the command-line utility for creating, managing, and inspecting ZFS storage pools.
     Most `zpool` commands require root privileges (e.g., `sudo`). -->

### Identifying Disks for Pool Creation
<!-- Before creating a pool, identify the disks you want to use.
     Using stable device IDs (e.g., from /dev/disk/by-id/) is highly recommended over names like /dev/sda,
     as /dev/sdX names can change across reboots. -->
```bash
# List block devices, helps identify disk names and sizes. The -S option from original is for lsblk, not standard ls.
sudo lsblk -o NAME,SIZE,TYPE,VENDOR,MODEL,SERIAL,WWN

# List devices by their stable IDs (filter out unwanted entries if necessary)
ls -la /dev/disk/by-id/
# Example to find ATA disks, excluding partitions:
# find /dev/disk/by-id/ -name "ata-*" ! -name "*-part*"
```
<!-- For VMs, you might use virtual disk paths like /dev/vda, /dev/vdb.
The original `lsblk -S` is useful for showing SCSI devices; `lsblk -dpno KNAME,TYPE,SIZE,MODEL` is also good. -->

### Creating Pools (`zpool create`)

#### Basic Pool (Single Disk - No Redundancy)
<!-- Creates a simple pool named 'mypool' using a single disk.
     WARNING: NO REDUNDANCY. If this disk fails, the entire pool and all its data are LOST.
     Suitable for testing, scratch space, or data that is backed up elsewhere and can be easily restored. -->
```bash
# sudo zpool create mypool /dev/disk/by-id/ata-your_disk_id_for_single_pool
# Example using a loop device (for testing only, data is lost if loop device is removed or system reboots without recreating it):
truncate -s 20G /tmp/zfs_test_disk.img
sudo zpool create testpool /tmp/zfs_test_disk.img
# The command `zpool create myzfspool /dev/loop0` from the original cheatsheet is an example of this.
```

#### Mirrored Pool (RAID1-like)
<!-- Creates a pool named 'tank' with two (or more) disks mirrored. Provides redundancy: pool survives if one disk fails (for a 2-way mirror).
     Capacity of the mirror vdev is that of the smallest disk. -->
```bash
sudo zpool create tank mirror /dev/disk/by-id/ata-your_disk_id_1 /dev/disk/by-id/ata-your_disk_id_2
# Example from original: zpool create files mirror /dev/sda /dev/sdb
# (Note: using /dev/sdX names is discouraged for production due to potential name changes.)

# To create a pool with multiple mirror vdevs (striped mirrors, like RAID10):
# sudo zpool create tank mirror /dev/disk/by-id/disk1 /dev/disk/by-id/disk2 mirror /dev/disk/by-id/disk3 /dev/disk/by-id/disk4
```

#### Striped Pool (RAID0-like - No Redundancy)
<!-- Creates a pool by striping data across multiple disks. Increases performance and capacity but offers NO REDUNDANCY.
     If any disk fails, THE ENTIRE POOL IS LOST. Use with extreme caution. -->
```bash
# Example from original: zpool create -f files /dev/sdb /dev/sdc /dev/sdd
# (Using /dev/disk/by-id/ is preferred)
sudo zpool create -f bigvol /dev/disk/by-id/disk1 /dev/disk/by-id/disk2 /dev/disk/by-id/disk3
```
<!-- The original note "*used striped instead of mirror for RAID 5*" is incorrect.
     RAID0 is striping. RAID1 is mirroring. RAID5 is RAID-Z1 in ZFS terms. -->

#### RAID-Z1 Pool (RAID5-like, single parity)
<!-- Creates a pool named 'datapool' with multiple disks in a RAID-Z1 configuration.
     Allows one disk in the vdev to fail without data loss. Minimum 3 disks required for RAID-Z1.
     Capacity is roughly (N-1) * size_of_smallest_disk. -->
```bash
sudo zpool create datapool raidz1 /dev/disk/by-id/disk-a /dev/disk/by-id/disk-b /dev/disk/by-id/disk-c
```

#### Important Pool Creation Options
```bash
# -f: Force operation. For example, to use disks that ZFS thinks are already part of another pool (DANGEROUS, ensure disks are not in use).
# -o ashift=12: (Highly Recommended for most modern drives with 4K sectors, including virtually all SSDs and most HDDs >2TB).
#               Sets the physical block size alignment for vdevs. `ashift=12` means 2^12 = 4096 bytes.
#               This CANNOT be changed after pool creation for that vdev. Incorrect ashift can lead to poor performance.
#               Check drive sector size with `smartctl -i /dev/sdX | grep "Sector Size:"` or `sg_readcap --long /dev/sdX`. Use logical sector size if physical is 512 but logical is 4096 (512e).
# -O <property>=<value>: Set a ZFS filesystem property on the root dataset of the pool at creation time.
#                        e.g., -O compression=lz4 -O atime=off -O xattr=sa -O acltype=posixacl
# -m <mountpoint>: Set the mount point for the pool's root dataset (default: /<poolname>).
#                  Use `-m legacy` if you want to manage mounting via /etc/fstab or other means (less common for root dataset).
#                  Use `-m none` if the root dataset should not be mounted directly.
# -R <altroot>: Create the pool with an alternate root path. All mount points within the pool will be prefixed with <altroot>.
#               Useful for recovery scenarios or when installing an OS to a ZFS pool from a live environment.

# Example creating a mirrored pool with common options:
sudo zpool create \
    -o ashift=12 \
    -O compression=lz4 \
    -O atime=off \
    -O xattr=sa \
    -O acltype=posixacl \
    -m /data/mypool \
    mypool mirror /dev/disk/by-id/disk1 /dev/disk/by-id/disk2
```
<!-- Note on `autotrim=on`: This property can be set on the pool to enable automatic TRIM commands for SSDs within the pool.
     `sudo zpool set autotrim=on mypool` (after creation) or `-o autotrim=on` during creation. -->

### Listing Pools
```bash
sudo zpool list
# NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
# mypool  1.81T   700G  1.13T        -         -     8%    37%  1.00x    ONLINE  -
# Example from original: zpool list zfspool

# Show specific properties for all pools
sudo zpool list -o name,size,alloc,free,health,capacity,guid
```

### Pool Status and Health
```bash
# Display detailed status for all pools, including vdevs, disks, errors, and ongoing scrubs/resilvers.
sudo zpool status
# Example from original: zpool status zfspool

# Verbose status for a specific pool:
sudo zpool status -v mypool
# Example output showing healthy pool:
#   pool: mypool
#  state: ONLINE
#   scan: scrub repaired 0B in 00:30:00 with 0 errors on Sun Dec 25 00:30:00 2023
# config:
#         NAME        STATE     READ WRITE CKSUM
#         mypool      ONLINE       0     0     0
#           mirror-0  ONLINE       0     0     0
#             sda     ONLINE       0     0     0
#             sdb     ONLINE       0     0     0
# errors: No known data errors

# Show only pools that are not in a healthy (ONLINE) state
# sudo zpool status -x
```

### Destroying a Pool
<!-- DANGEROUS: Destroys the entire pool and all data within it. Use with extreme caution.
     Ensure the pool is exported or unmounted if possible. -->
```bash
# sudo zpool destroy mypoolname
# Example from original: zpool destroy $POOL
# Use -f to force if it's busy or contains mounted datasets.
# sudo zpool destroy -f mypoolname
```
<!-- The command `zpool remove zfspool /dev/mapper/gentoober-zfs` from the original is for removing a device from a pool,
     not destroying the pool. `zpool remove` is more complex and typically used for hot spares or detaching a disk from a mirror.
     Removing a top-level vdev (like a single disk, a raidz vdev, or a mirror vdev that is not the last one) is generally not supported,
     except for special "device removal" features for specific vdev types in very new ZFS versions. -->

### Clearing Pool Errors
<!-- If non-fatal errors are reported (e.g., checksum errors after a scrub that were repaired or couldn't be),
     this command clears the error counts from the `zpool status` output.
     It does NOT fix the underlying data if it couldn't be repaired by a scrub. -->
```bash
sudo zpool clear mypoolname
# Clear errors for a specific device within the pool:
# sudo zpool clear mypoolname /dev/disk/by-id/device_with_errors
```

### Importing and Exporting Pools
<!-- Pools need to be exported before moving disks to another system, or if you want to prevent the current system from using them.
     Pools are typically imported automatically on boot if ZFS services are configured correctly. -->
```bash
# Export a pool (makes it inactive on the current system, allows disks to be moved)
sudo zpool export mypool

# Import a previously exported pool (or a pool from another system)
# First, scan for available pools that are not currently imported:
sudo zpool import
# This will list pools by name and/or numeric ID.

# Import by name (if unique):
sudo zpool import mypool
# Import by numeric ID (if name conflicts or pool has no name):
# sudo zpool import <pool_id_number> new_pool_name_if_desired

# Common import options:
#   -f: Force import. May be needed if pool was not cleanly exported or seems active on another system (use with caution).
#   -R <altroot>: Import the pool with a different root mount point. All datasets will mount under <altroot>.
#   -d /dev/disk/by-id/ : Search for pool member devices only in this directory (recommended for stable paths).
#   -a: Import all available pools found by the scan.
# Example: Import all pools using /dev/disk/by-id paths, forcing if necessary.
# sudo zpool import -a -d /dev/disk/by-id/ -f
```

### Adding Devices to a Pool (Extending Capacity or Redundancy)
<!-- You can add new vdevs to an existing pool to increase its capacity.
     You CANNOT add a single disk to an existing mirror or RAID-Z vdev to expand that specific vdev's capacity or change its type.
     You add another complete vdev (e.g., another mirror to a pool of mirrors, or another disk for a new stripe). -->
```bash
# Add a new mirror vdev (disk-c and disk-d) to an existing pool 'mypool'
sudo zpool add mypool mirror /dev/disk/by-id/disk-c /dev/disk/by-id/disk-d

# Add a single disk as a new top-level vdev (stripe - adds capacity but no redundancy for this new disk).
# This means if this single disk vdev fails, the *entire pool* may be lost if other vdevs weren't also affected.
# Generally not recommended unless pool is already non-redundant or for specific use cases.
# sudo zpool add mypool /dev/disk/by-id/disk-e

# Add a hot spare to the pool (disk-s will be used automatically if another disk in a redundant vdev fails)
# sudo zpool add mypool spare /dev/disk/by-id/disk-s

# Add a log device (SLOG) using an SSD (e.g., nvme0n1p1 - a partition is fine)
# For redundancy, SLOG can also be mirrored.
# sudo zpool add mypool log /dev/disk/by-id/nvme-INTEL_SSDPEKxxxx
# sudo zpool add mypool log mirror /dev/disk/by-id/ssd_slog1 /dev/disk/by-id/ssd_slog2

# Add a cache device (L2ARC) using an SSD
# sudo zpool add mypool cache /dev/disk/by-id/ssd_cache_disk
```

### Replacing Devices
<!-- Replace a faulty or smaller disk in a redundant vdev. -->
```bash
# If a disk is FAULTY/UNAVAIL and pool is DEGRADED:
# Take the faulty disk offline (if it's not already and if possible)
# sudo zpool offline mypool /dev/disk/by-id/old_faulty_disk_id
# Physically replace the disk.
# Tell ZFS to replace it with the new disk (ZFS might detect new disk automatically in same slot for some hardware):
sudo zpool replace mypool /dev/disk/by-id/old_faulty_disk_id /dev/disk/by-id/new_disk_id
# If old_faulty_disk_id is just the device name (e.g. sda) and it's already been removed,
# and the new disk is in the same slot, you might just do:
# sudo zpool replace mypool /dev/sda # (if sda was the failed disk name)
# ZFS will start resilvering data to the new disk. Monitor with `zpool status mypool`.

# Proactive replacement (before disk fails, e.g., for size upgrade or suspected issues):
# Attach the new disk as a replacement for the old one in the vdev.
sudo zpool replace mypool /dev/disk/by-id/current_disk_id /dev/disk/by-id/new_larger_disk_id
# After resilvering is complete, the old disk is automatically detached if it was a 1-to-1 replacement.
# If replacing all disks in a vdev one by one to increase capacity of that vdev,
# ensure `autoexpand=on` for the pool for the new capacity to be available after all disks are replaced.
# sudo zpool set autoexpand=on mypool
```

### Detaching and Offlining Devices
```bash
# Detach a disk from a mirror (if it's a 2-way mirror, this makes the vdev non-redundant - a single disk stripe).
# If it's a 3-way mirror, detaching one disk makes it a 2-way mirror.
# sudo zpool detach mypool /dev/disk/by-id/disk_to_remove_from_mirror

# Take a disk offline (e.g., before physical removal if it's part of a redundant vdev and you intend to replace it soon)
sudo zpool offline mypool /dev/disk/by-id/disk_to_offline
# To bring it back online (if it was temporarily offlined and is healthy):
# sudo zpool online mypool /dev/disk/by-id/disk_to_online
```
<!-- Removing a top-level data vdev (like a raidz vdev or a single disk vdev that isn't a log/cache/spare) from a pool is generally not supported,
     except for special "device removal" features for specific vdev types (e.g. single-disk stripes) in very new ZFS versions, which can be complex.
     The command `zpool remove zfspool /dev/mapper/gentoober-zfs` from original was likely intended to remove a log/cache/spare or detach from mirror. -->

### Data Integrity (Scrubbing)
<!-- Scrubs read all data in the pool and verify checksums, repairing corrupted data if redundancy exists.
     Schedule regular scrubs (e.g., weekly or monthly) via cron or systemd timers. -->
```bash
# Start a scrub on 'mypool'
sudo zpool scrub mypool

# Stop/Cancel an ongoing scrub
# sudo zpool scrub -s mypool # or -p to pause

# Check scrub status (part of `zpool status`)
sudo zpool status mypool
```

## Dataset (Filesystem) & Zvol Management (`zfs` commands)

<!-- `zfs` is the command-line utility for managing ZFS datasets (filesystems) and zvols (volumes).
     Datasets are the ZFS equivalent of filesystems. Zvols are block devices. -->

### Listing Datasets and Zvols
```bash
# List all ZFS datasets and zvols with basic info
sudo zfs list
# NAME                 USED  AVAIL     REFER  MOUNTPOINT
# mypool              1.50G  16.5G       96K  /mypool
# mypool/data          512K  16.5G      128K  /mypool/data
# mypool/data/projects 384K  16.5G      192K  /mypool/data/projects
# mypool/vm_disk       1G    17.0G       50M  -

# List recursively, showing all datasets under 'mypool' (if not default)
# sudo zfs list -r mypool

# List only filesystems
sudo zfs list -t filesystem

# List only zvols
sudo zfs list -t volume

# List only snapshots
# sudo zfs list -t snapshot # Covered in Snapshots section

# Show specific properties, recursively for 'mypool'
sudo zfs list -r -o name,mountpoint,used,available,compression,compressratio,quota,reservation mypool
# The original had `zfs list -r small` - assuming 'small' was a pool or dataset name.
```

### Creating Datasets (Filesystems)
<!-- Datasets are created within a pool or another dataset (nested). -->
```bash
# Create a dataset 'data' inside 'mypool'. It will inherit properties from 'mypool'.
sudo zfs create mypool/data
# By default, it might mount at /mypool/data if mypool is at /mypool.

# Create a nested dataset 'projects' inside 'mypool/data'
sudo zfs create mypool/data/projects

# Create a dataset with specific properties set at creation time
# Example: Enable lz4 compression and set a specific mountpoint
sudo zfs create -o compression=lz4 -o mountpoint=/export/shared_docs mypool/shared_docs
# Example from original: zfs create -o compression=lz4 autotrim=on myzfspool/dataset1
# Note: 'autotrim' is a zpool property, not a zfs dataset property.
# For datasets on SSDs in a pool with autotrim=on, TRIM will generally work.
```

### Destroying Datasets or Zvols
<!-- DANGEROUS: Destroys the dataset/zvol and all data within it.
     If the dataset has snapshots, `zfs destroy` will fail unless `-r` (recursive) or `-R` (recursive + force unmount) is used.
     If it has clones, the snapshots cannot be destroyed until clones are dealt with. -->
```bash
# Destroy 'mypool/tempprojects' dataset. Fails if it has snapshots or mounted children.
# sudo zfs destroy mypool/tempprojects

# Destroy a dataset and all its snapshots and children (nested datasets and their snapshots)
# sudo zfs destroy -r mypool/oldstuff
# Use -f to force unmount if datasets are busy. Use with extreme caution.
# sudo zfs destroy -Rf mypool/oldstuff_and_children
```

### Mounting and Unmounting Datasets
<!-- Datasets are typically mounted automatically by ZFS if their `mountpoint` property is set to a path (not 'none' or 'legacy').
     The `canmount` property also affects this (`on` = can mount, `off` = cannot mount, `noauto` = don't mount automatically at import/boot). -->

#### Setting Mountpoints
```bash
# Set the mountpoint property for 'mypool/data' to '/data_archive'
sudo zfs set mountpoint=/data_archive mypool/data
# If the directory /data_archive doesn't exist, ZFS will try to create it.
# Example from original: zfs set mountpoint=/mnt/zfs-mount zpool-0/data-0

# If you want to manage mounting manually (e.g., via /etc/fstab or systemd mounts):
# sudo zfs set mountpoint=legacy mypool/data
# Then you can use: sudo mount -t zfs mypool/data /mnt/mymanualmount
# Example from original: mount -t zfs zpool-0/data-0 /mnt/zfs-mount
```

#### Mounting/Unmounting
```bash
# Mount a specific dataset (if its `mountpoint` property is set and `canmount=on`)
# sudo zfs mount mypool/data

# Mount all datasets that are eligible for auto-mounting
# sudo zfs mount -a

# Unmount a specific dataset
# sudo zfs unmount mypool/data
# Or by mountpoint: sudo zfs unmount /data_archive

# Unmount all datasets
# sudo zfs unmount -a
```
<!-- The original example `zfs get mountpoint rpool/docker` and its explanation about inheritance is a good illustration of how mountpoints work and will be integrated conceptually under properties. -->

### Creating Zvols (Block Devices)
<!-- Zvols act as virtual block devices within a ZFS pool. They are useful for providing storage to VMs,
     iSCSI targets, or any application that needs a raw block device. -->
```bash
# Create a 100GB zvol named 'vm_disk01' in 'mypool/volumes' dataset (assuming mypool/volumes is a dataset)
# -V or --volume-size: Specifies the size of the zvol.
# Zvols appear as /dev/zvol/<poolname>/<path_to_zvol>/<zvolname> or more simply /dev/zvol/<poolname>/<zvolname> if not nested.
# Example: /dev/zvol/mypool/volumes/vm_disk01
sudo zfs create -V 100G mypool/volumes/vm_disk01

# Zvols can have properties set on them like datasets:
sudo zfs set volblocksize=16K mypool/volumes/vm_disk01 # Default is often 8K or 16K for older ZFS, 64K or more for newer.
# For VMs, using a volblocksize that matches the guest OS's common block size (e.g., 4K for many Linux/Windows) can be beneficial if guest does small I/O.
# However, larger volblocksize (e.g., 64K, 128K) can be better for sequential I/O performance.
sudo zfs set compression=lz4 mypool/volumes/vm_disk01
# sudo zfs set primarycache=metadata mypool/volumes/vm_disk01 # Cache only metadata for this zvol if data is mostly random access and large
# sudo zfs set sync=always mypool/volumes/vm_disk01 # For data integrity critical zvols, at performance cost (SLOG helps here)
```

## ZFS Properties (Getting and Setting)

<!-- ZFS behavior for both pools and datasets/zvols is controlled by properties.
     Many dataset/zvol properties are inheritable from their parent. -->

### Getting Properties
```bash
# Get all properties for a dataset or zvol
sudo zfs get all mypool/data/projects

# Get a specific property (e.g., compression) for a dataset
sudo zfs get compression mypool/data/projects
# NAME                    PROPERTY     VALUE     SOURCE
# mypool/data/projects    compression  lz4       inherited from mypool
# Example from original: zfs get compression myzfspool/dataset1

# Get all properties for a pool
sudo zpool get all mypool

# Get a specific property for a pool (e.g., health, ashift, autotrim)
sudo zpool get health mypool
sudo zpool get ashift mypool
sudo zpool get autotrim mypool
```
<!-- The `SOURCE` column indicates if the property is `local` (set on this dataset), `inherited` (from a parent), `default`, or `temporary`. -->

### Setting Properties
```bash
# Set compression to zstd for 'mypool/data' (applies to new data written)
sudo zfs set compression=zstd mypool/data
# Example from original: zfs set compression=lz4 myzfspool/dataset1

# Set a quota of 100GB for 'mypool/users/john'
sudo zfs set quota=100G mypool/users/john

# Set a reservation of 10GB for 'mypool/important_logs' (guarantees this much space is available to it)
# sudo zfs set reservation=10G mypool/important_logs

# Change the mountpoint of a dataset (as seen in original examples)
# sudo zfs set mountpoint=/new/path mypool/oldpathdataset
# Example from original: zfs set mountpoint=/mnt/zfs-mount zpool-0/data-0

# Make a dataset read-only
# sudo zfs set readonly=on mypool/archive
```

### Common and Important Properties (for Datasets/Zvols)
<!-- This is not exhaustive. Refer to `man zfs` for all properties. -->
*   **`mountpoint`**: Filesystem mount location (or `none`, `legacy`). For zvols, this is not directly used for mounting like a filesystem.
*   **`canmount`**: (`on` | `off` | `noauto`) If `off`, cannot be mounted. `noauto` means not mounted automatically at boot/pool import.
*   **`compression`**: (`off` | `lz4` | `gzip` | `gzip-N` | `zstd` | `zstd-N` | `zle`) `lz4` is fast and often a good default. `zstd` offers better compression ratios with good performance. Applies to newly written data.
*   **`dedup`**: (`off` | `on` | `verify` | `sha256,verify` | ...) Deduplication. **WARNING:** Can be very RAM and I/O intensive. Requires careful planning, sufficient RAM (several GB per TB of deduped data), and thorough testing. Often not recommended unless a specific workload significantly benefits and resources are adequate.
*   **`encryption`**: (`off` | `on` | `aes-128-ccm` | `aes-256-ccm` | `aes-128-gcm` | `aes-256-gcm`) Enables dataset-level encryption. Requires setting `keyformat` (e.g., `passphrase`, `hex`, `raw`) and `keylocation` (e.g., `prompt`, `file:///path/to/keyfile`). Encrypted datasets need to be loaded with `zfs load-key`.
*   **`atime`**: (`on` | `off`) Controls whether access times are updated on files. `atime=off` can improve performance, especially for read-heavy workloads.
*   **`recordsize`** (datasets): Smallest block allocated for files. Default 128K. Can be tuned (e.g., smaller like 4K-16K for databases with random I/O, larger like 1M for sequential media storage). Must be a power of 2, from 512 bytes to 1M (or 16M with large_blocks feature).
*   **`volblocksize`** (zvols): Block size for zvols. Default often 8K or 16K. Similar tuning considerations as `recordsize`.
*   **`quota`**: Hard limit on the amount of space a dataset and its children (including snapshots and descendant datasets) can consume.
*   **`refquota`**: Hard limit on the amount of space a dataset itself can consume (excluding snapshots and children).
*   **`reservation`**: Guarantees a minimum amount of space for a dataset and its children. This space is "set aside" and not available to other datasets if the reservation isn't met.
*   **`refreservation`**: Guarantees space for the dataset itself (excluding snapshots and children).
*   **`sharenfs` / `sharesmb`**: Configure NFS/SMB sharing directly via ZFS properties (e.g., `sharenfs=on`, `sharesmb=on`, or with specific options).
*   **`xattr`**: (`on` | `sa` | `dir`) Extended attribute storage. `sa` (system attribute based) is often better for performance if extended attributes are heavily used and fit within inodes. `dir` for directory-based xattrs.
*   **`acltype`**: (`off` | `nfsv4` | `posix`) Type of ACLs to support. `posix` is common for Linux environments.
*   **`primarycache` / `secondarycache`**: (`all` | `none` | `metadata`) Control what data is cached in ARC (primary) and L2ARC (secondary). `metadata` can be useful for datasets with large files where data caching is not beneficial or for zvols with random I/O.
*   **`logbias`**: (`latency` | `throughput`) Hint for ZIL behavior for synchronous writes. `latency` (default) attempts to satisfy sync writes quickly (good with SLOG). `throughput` may delay sync writes slightly to group them for better overall throughput, potentially useful without a SLOG.
*   **`sync`** (zvols): (`standard` | `always` | `disabled`) For zvols, `always` ensures all writes are synchronous (data integrity critical, SLOG highly recommended). `disabled` makes all writes asynchronous (highest risk, highest performance). `standard` (default) respects application sync requests.

## Snapshot Management (`zfs snapshot`)

<!-- Snapshots are read-only, point-in-time copies of datasets or zvols. They are very space-efficient and quick to create due to ZFS's Copy-on-Write mechanism. -->

### Creating Snapshots
```bash
# Create a snapshot of 'mypool/data' named 'backup20231225'
sudo zfs snapshot mypool/data@backup20231225
# Original example: zfs snapshot rpool/test@demo

# Recursively snapshot a dataset and all its children datasets with the same snapshot name
sudo zfs snapshot -r mypool/data@recursive_snap_today
# This creates mypool/data@recursive_snap_today, mypool/data/projects@recursive_snap_today, etc.
```

### Listing Snapshots
```bash
# List all snapshots on the system
sudo zfs list -t snapshot
# Original example output:
# NAME                                                                                           USED  AVAIL  REFER  MOUNTPOINT
# rpool/test@demo                                                                                    0B      -    96K  -
# (USED is space occupied *by this snapshot itself* due to data blocks that were unique to it when it was active,
#  or blocks that have changed in the live filesystem since the snapshot was taken but are still referenced by this snapshot.
#  AVAIL is not applicable to snapshots. REFER is the amount of data referenced by this snapshot.)

# List snapshots for a specific dataset and its children, recursively
sudo zfs list -r -t snapshot mypool/data
```

### Rolling Back to a Snapshot
<!-- Reverts a dataset to the state it was in when the snapshot was taken.
     WARNING: All changes made to the live filesystem *since the snapshot was created* will be PERMANENTLY LOST.
     Any intermediate snapshots taken *after* the target snapshot will also be destroyed by default with the -r option. -->
```bash
# Roll back 'mypool/data' to the state of 'backup20231225'.
# This will fail if more recent snapshots exist, unless -r is used.
# sudo zfs rollback mypool/data@backup20231225
# Original example: zfs rollback rpool/test@demo (this would fail if afile.txt was created *after* another snapshot was taken post @demo)

# Use -r to destroy any snapshots taken *after* 'backup20231225' during the rollback.
# Use with extreme caution.
sudo zfs rollback -r mypool/data@backup20231225
```
<!-- Before rolling back, you might want to clone the current state or take a new snapshot if recent changes are valuable. -->

### Destroying Snapshots
```bash
# Destroy a single snapshot
sudo zfs destroy mypool/data@old_snapshot
# Original example: zfs destroy rpool/test@demo

# Destroy a range of snapshots (requires specific naming or a helper script).
# ZFS itself doesn't have a direct "range" delete by date.
# Example: Destroy all snapshots for mypool/data that start with "autosnap_" (use carefully!)
# for snap in $(zfs list -H -o name -t snapshot | grep '^mypool/data@autosnap_'); do sudo zfs destroy "$snap"; done

# Original example for deleting ALL snapshots on the system (VERY DANGEROUS, ensure you know what you're doing):
# for snapshot_to_delete in $(zfs list -H -o name -t snapshot); do
#     echo "Potentially destroying snapshot: $snapshot_to_delete"
#     # sudo zfs destroy "$snapshot_to_delete" # Uncomment with extreme caution to actually destroy
# done

# Destroy multiple snapshots between snapA (older) and snapB (newer), inclusive of snapB but not snapA:
# This means all snapshots from (but not including) snapA up to and including snapB are destroyed.
# sudo zfs destroy mypool/data@snapA%snapB
# To destroy snapA, snapB, and everything in between:
# Create a temporary bookmark for snapA: `sudo zfs bookmark mypool/data@snapA mypool/data#tempbm`
# Then destroy from the bookmark to snapB: `sudo zfs destroy mypool/data#tempbm%snapB`
# Then destroy snapA itself if needed: `sudo zfs destroy mypool/data@snapA` (if no longer bookmarked)
# And finally destroy the bookmark: `sudo zfs destroy mypool/data#tempbm`
# A simpler way for a known set: destroy them individually or by script.
```
<!-- Snapshots that have dependent clones cannot be destroyed until their clones are first destroyed or promoted. -->

### Accessing Snapshot Data
<!-- Snapshot data is accessible via a hidden `.zfs/snapshot/` directory within the root of the dataset's mountpoint.
     This allows you to browse and copy files from a snapshot without a full rollback. -->
```bash
# First, ensure the snapdir property is visible for the dataset (it's often 'hidden' by default)
sudo zfs get snapdir mypool/data
# If 'hidden', make it visible:
sudo zfs set snapdir=visible mypool/data

# Then, if mypool/data is mounted at /data:
# cd /data/.zfs/snapshot/backup20231225/
# ls
# cp /data/.zfs/snapshot/backup20231225/some_file /data/some_file_restored

# Set snapdir back to hidden if desired (for cleaner `ls` output in the dataset's root):
# sudo zfs set snapdir=hidden mypool/data
```

## Clone Management (`zfs clone`)

<!-- Clones are writable datasets created from a ZFS snapshot. Initially, a clone shares all its blocks
     with the snapshot, making creation very fast and space-efficient. As the clone is modified,
     new blocks are written (Copy-on-Write), and it starts diverging from the snapshot, consuming more space. -->

### Creating Clones
```bash
# Create a clone 'mypool/data_clone_from_demo' from 'mypool/data@demo_snapshot'
sudo zfs clone mypool/data@demo_snapshot mypool/data_clone_from_demo
# The new clone 'mypool/data_clone_from_demo' will be mounted according to its `mountpoint` property (often inherited or auto-generated).
# You can set properties on the clone at creation time:
# sudo zfs clone -o mountpoint=/mnt/cloned_data -o compression=zstd mypool/data@demo_snapshot mypool/data_clone_from_demo
# Original example: zfs clone rpool/test@demo rpool/demobackup
```
<!-- Important: A snapshot cannot be destroyed as long as it has dependent clones. You must destroy the clone(s) first, or promote a clone. -->

### Promoting Clones
<!-- If you want to make a clone independent of its original snapshot's filesystem (e.g., to delete the original filesystem
     while keeping the clone's data), you can "promote" the clone.
     Promoting a clone swaps the roles of the clone and its origin dataset. The clone becomes the parent dataset,
     and the original filesystem (if it still exists and was the origin of the snapshot) effectively becomes a clone of the promoted dataset.
     This allows the original snapshot (e.g., `mypool/data@demo_snapshot`) to be destroyed if no other clones depend on it,
     and potentially the original `mypool/data` dataset if it's now considered a clone of the promoted one. -->
```bash
# Promote 'mypool/data_clone_from_demo'.
sudo zfs promote mypool/data_clone_from_demo
# After promotion, `mypool/data_clone_from_demo` is no longer dependent on `mypool/data@demo_snapshot`
# in the same hierarchical way. The dependency is effectively reversed.
```

## ZFS Send and Receive (Replication/Backups)

<!-- `zfs send` creates a stream representation of a snapshot (or an increment between snapshots),
     which can be piped to `zfs recv` on another pool (local or remote via SSH) to replicate it.
     This is a very powerful feature for backups, data migration, and disaster recovery. -->

### Full Send/Receive (Initial Backup/Replication)
```bash
# Send snapshot 'mypool/data@snap1' to 'backuppool/data_backup_location'
# `backuppool` must exist on the destination. `data_backup_location` will be created by `zfs recv`.
sudo zfs send mypool/data@snap1 | sudo zfs recv backuppool/data_backup_location

# Sending to a remote machine via SSH:
# Ensure `backuppool` exists on `remote_host` and the user has ZFS permissions (often root or a user with `zfs allow`).
sudo zfs send mypool/data@snap1 | ssh user@remote_host 'sudo zfs recv remote_backuppool/data_backup_from_source'
# Original example: zfs send rpool/test@demo | ssh othermachine zfs recv backup/test
```

### Incremental Send/Receive (Subsequent Backups)
<!-- Send only the differences between two snapshots. This is much more efficient for regular backups.
     Use `-i` for incremental from a previous snapshot of the *same dataset*.
     Use `-I` for incremental from an older common *ancestor* snapshot (can span snapshot renames or dataset moves if lineage is maintained). -->
```bash
# 1. Create a new snapshot on the source (e.g., mypool/data@snap2, assuming mypool/data@snap1 already exists and was previously sent)
#    sudo zfs snapshot mypool/data@snap2

# 2. Send the increment from 'snap1' to 'snap2':
#    Using -i @snap1 (or the full snapshot name mypool/data@snap1 if ambiguous)
sudo zfs send -i mypool/data@snap1 mypool/data@snap2 | sudo zfs recv backuppool/data_backup_location
# On remote via SSH:
# sudo zfs send -i mypool/data@snap1 mypool/data@snap2 | ssh user@remote_host 'sudo zfs recv remote_backuppool/data_backup_from_source'

# Using -I @snap1 (or full name) sends all snapshots *between* the first and second snapshot specified,
# effectively sending a stream of incremental changes that would bring the destination up to date with snap2,
# assuming it already has snap1.
# sudo zfs send -I mypool/data@snap1 mypool/data@snap_newer | sudo zfs recv backuppool/data_backup_location
```

### Options for Send/Receive
```bash
# Common `zfs send` options:
#   -R or --replicate: Recursive send. Sends the specified snapshot AND all snapshots of its descendant datasets
#                      that have the same snapshot name (or are newer if using an incremental replication stream with -I).
#                      This is useful for backing up an entire dataset tree with consistent snapshots.
#   -p or --props: Send properties along with the snapshot data. Without this, received datasets get default/inherited properties.
#   -w or --raw: Send raw encrypted data (for encrypted datasets). This is faster if the destination doesn't need to decrypt
#                and re-encrypt, but requires the destination to handle the raw encrypted stream (e.g., by having the same encryption key loaded).
#   -c or --compressed: Send a compressed stream (useful if network is the bottleneck, but ZFS data might already be compressed on disk).
#   -L or --large-block: Send large blocks (can improve performance for some scenarios, especially over high-latency links).
#   -e or --embed: Send embedded data (e.g., for zvols with `embedded_data` property set).
#
# Common `zfs recv` options:
#   -F: Force a rollback of the target filesystem to its most recent snapshot before performing the receive.
#       DANGEROUS on the destination if it contains changes you want to keep that are not snapshotted.
#       Often used in replication scripts to ensure the target can receive the increment.
#   -d: Discard the source pool name from the received dataset path. E.g., if sending `pool1/data/fs1`,
#       `sudo zfs recv -d pool2` would create `pool2/data/fs1`. Without -d, it'd be `pool2/pool1/data/fs1`.
#   -e: Discard the source pool name AND the leading component(s) of the dataset path, up to the snapshot name.
#       E.g., if sending `pool1/data/fs1@snap`, `sudo zfs recv -e pool2/backups` would create `pool2/backups/fs1@snap`.
#   -u: Do not mount the received filesystem after receiving it.
#   -o <property>=<value>: Set ZFS properties on the received dataset (e.g., `-o mountpoint=/new/mount_path_on_backup`).
#   -x <property_to_exclude>: Exclude a specific property from being set on the received dataset.

# Example: Recursive send of all snapshots of mypool/appdata and its children, preserving properties.
# First, ensure consistent recursive snapshots exist, e.g., mypool/appdata@backup_job_XYZ and mypool/appdata/child@backup_job_XYZ
# sudo zfs snapshot -r mypool/appdata@backup_job_20231226
# sudo zfs send -R -p mypool/appdata@backup_job_20231226 | ssh backup_server 'sudo zfs recv -F -d -p backup_storage_pool'
```

## ZFS Caching (ARC, L2ARC) & SLOG (Write Log)

<!-- ZFS uses RAM for its primary cache (ARC). It can also use faster devices for secondary read cache (L2ARC)
     and for a synchronous write log accelerator (SLOG, part of ZIL - ZFS Intent Log). -->

### ARC (Adaptive Replacement Cache)
*   **What it is:** ZFS's primary disk cache, residing in the host computer's RAM. It's highly efficient and dynamically adjusts its size based on available system memory and workload. It caches both data and metadata.
*   **Monitoring:**
    ```bash
    # Using arc_summary (often a separate script/tool, part of zfs-utils or available from OpenZFS project)
    # sudo arc_summary # Provides a detailed summary of ARC usage and efficiency.

    # Direct kernel stats (less user-friendly, but always available)
    cat /proc/spl/kstat/zfs/arcstats
    # Look for parameters like `c` (target ARC size), `size` (current ARC size), `hits`, `misses`, `l2_hits`, `l2_misses`.
    ```
*   **Tuning:** Generally, ARC tunes itself well. Advanced tuning is possible via ZFS module parameters (e.g., in `/etc/modprobe.d/zfs.conf` or kernel command line), such as setting `zfs_arc_max` to limit its maximum size if needed (e.g., on systems with other memory-hungry applications). This is rarely necessary for most users. `zfs_arc_min` can also be set.

### L2ARC (Level 2 ARC - Read Cache)
*   **What it is:** An optional secondary layer of caching, typically using fast SSDs (NVMe or SATA). It stores blocks evicted from the ARC. When data is requested that's in L2ARC but not ARC, it's served from L2ARC and promoted back to ARC.
*   **Purpose:** Improves read performance for frequently accessed data ("hot" data) that doesn't fit entirely in RAM (ARC), especially for random read workloads. It effectively extends the read cache.
*   **Considerations:**
    *   L2ARC primarily stores data blocks, not metadata by default (this can be influenced by the `secondarycache` property of datasets).
    *   L2ARC devices are **not redundant** within ZFS; if an L2ARC device fails, performance degrades (as if it wasn't there), but **no data is lost** from the pool (data is still on the main pool vdevs). The L2ARC will be rebuilt on a new device if added.
    *   Adding L2ARC consumes some RAM for its own metadata (index of what's on the L2ARC device).
    *   L2ARC is populated by blocks evicted from ARC, so it takes time to "warm up" and become effective.
    *   Writes are not cached by L2ARC; it's a read cache.
*   **Adding L2ARC device(s) to a pool:**
    ```bash
    # sudo zpool add <poolname> cache <device_id_1> [<device_id_2> ...]
    # Example:
    sudo zpool add mypool cache /dev/disk/by-id/nvme-L2ARC_SSD_1
    # Multiple L2ARC devices will have their storage capacity combined (effectively striped for lookups).
    ```
*   **Removing L2ARC device(s):**
    ```bash
    # sudo zpool remove <poolname> <device_id_or_guid_of_cache_device>
    sudo zpool remove mypool /dev/disk/by-id/nvme-L2ARC_SSD_1
    # Data in L2ARC is lost, but this is fine as it's only a cache.
    ```

### SLOG (Separate Log Device - For ZIL Writes)
*   **What it is:** A device (or mirrored devices) used to store the ZFS Intent Log (ZIL) for **synchronous** writes.
*   **Purpose:** The ZIL logs synchronous write operations before they are written to the main pool storage (as part of a Transaction Group, which happens every few seconds). This allows ZFS to quickly acknowledge a synchronous write to the application. If the system crashes before the Transaction Group is committed to the main pool, the ZIL (from the SLOG if present, or from pool storage if not) is replayed on next import to prevent data loss for those acknowledged synchronous writes.
    Using a fast, low-latency SLOG device (like an NVMe SSD, ideally with power-loss protection, e.g., Intel Optane) significantly speeds up synchronous write performance.
*   **Considerations:**
    *   Only benefits **synchronous** writes. Large asynchronous writes (common for file copies, media streaming) will not see much, if any, benefit from a SLOG.
    *   Critical for NFS servers serving sync writes, or databases that rely heavily on `fsync()` or `O_SYNC` system calls.
    *   SLOG devices should ideally have power-loss protection (PLP) to ensure data written to them truly survives a power outage.
    *   SLOG should be mirrored for redundancy. If a non-redundant SLOG device is lost *and* the system crashes before ZIL data is committed to the main pool, acknowledged synchronous writes that were only in the ZIL could be lost.
    *   The size needed for a SLOG is typically small (a few GBs up to a few tens of GBs is often sufficient, depending on sync write throughput and pool transaction group frequency). Endurance (high DWPD - Drive Writes Per Day) is more important than raw capacity for SLOG devices.
*   **Adding SLOG device(s) to a pool:**
    ```bash
    # sudo zpool add <poolname> log <device_id_1>
    # Example (single SLOG device - not recommended for critical data without PLP):
    # sudo zpool add mypool log /dev/disk/by-id/nvme-SLOG_SSD_1

    # Example (mirrored SLOG devices for redundancy - highly recommended):
    sudo zpool add mypool log mirror /dev/disk/by-id/nvme-SLOG_SSD_A /dev/disk/by-id/nvme-SLOG_SSD_B
    ```
*   **Removing SLOG device(s):**
    ```bash
    # sudo zpool remove <poolname> <device_id_or_guid_of_slog_device_or_mirror_vdev>
    # Example removing a single SLOG device:
    # sudo zpool remove mypool /dev/disk/by-id/nvme-SLOG_SSD_A
    # Example removing a SLOG mirror (identify the mirror vdev, e.g., mirror-X, from `zpool status`):
    # sudo zpool remove mypool mirror-X
    # ZFS will wait for the ZIL to be committed from the SLOG to the pool before removing it.
    ```

## Health, Monitoring & Maintenance

### Pool I/O Statistics
```bash
# Show I/O statistics for all pools, updated every <interval> seconds (e.g., 2 seconds).
# Shows operations per second (ops), bandwidth (bw), and latency for reads/writes.
sudo zpool iostat -v 2
# Use Ctrl+C to stop.

# Show statistics for a specific pool, 5 second interval:
# sudo zpool iostat -v mypool 5
```

### Checking Pool Health
```bash
# Display detailed status for all pools. Look for "state: ONLINE" and "errors: No known data errors".
sudo zpool status

# Show only pools that are not in a healthy (ONLINE) state. If this command outputs nothing, all pools are healthy.
sudo zpool status -x
```

### System Logs
<!-- Check system logs for ZFS-specific messages (often prefixed with `ZFS:`) or disk-related error messages from the kernel.
     The location and command vary by operating system and logging daemon. -->
```bash
# For Linux systems using systemd's journal:
sudo journalctl -p err..emerg # Show critical system errors that might include ZFS issues
sudo journalctl -f | grep -i zfs # Follow logs and filter for ZFS messages

# For older Linux systems or specific log configurations:
# sudo dmesg | grep -i zfs
# sudo grep -i zfs /var/log/syslog
```

### Regular Scrubs (Data Integrity Check)
<!-- It's crucial to schedule periodic `zpool scrub <poolname>` to detect and potentially repair silent data corruption.
     This is already covered in the "Pool Management" section but reiterated here for maintenance context. -->
```bash
# Start a scrub on 'mypool'
sudo zpool scrub mypool

# Check progress:
sudo zpool status mypool
# A scrub can take many hours or even days for very large pools.
# Consider running monthly or weekly via cron jobs or systemd timers.
```

### Pool and Dataset History
<!-- View a history of ZFS commands run on a pool or dataset. Useful for auditing changes. -->
```bash
# Show command history for a specific pool
sudo zpool history mypool

# Show command history for a dataset (includes snapshots, property changes, etc.)
# sudo zfs history mypool/data

# Show full command history including internal ZFS operations (can be very verbose)
# sudo zpool history -i mypool
```

### Replacing a Failing Drive (Proactive Maintenance Summary)
<!-- If `smartctl -a /dev/sdx` (from smartmontools package) shows pre-failure warnings for a disk
     that is part of a redundant ZFS pool, replace it proactively. Detailed steps are in "Pool Management". -->
<!-- Key steps: `zpool offline`, physically replace, `zpool replace`, monitor `zpool status`. -->

### Auto-Expanding Pool After Disk Replacement
<!-- If you replace all disks in a vdev (e.g., all disks in a RAIDZ or all disks in all mirrors of a pool)
     with larger disks, the pool will not automatically use the new larger capacity unless the `autoexpand` pool property is `on`. -->
```bash
# Check current autoexpand setting for the pool
sudo zpool get autoexpand mypool

# Enable autoexpand if it's off (recommended if you plan to grow disks)
sudo zpool set autoexpand=on mypool
# Once all disks in a vdev are replaced with larger ones and resilvering is complete,
# ZFS will automatically expand that vdev to use the new available space if autoexpand=on.
```

---
<!-- This ZFS cheatsheet provides a starting point. ZFS has many more features and options.
     Always consult the official ZFS documentation (`man zfs`, `man zpool`) and resources like OpenZFS documentation for comprehensive details.
     Test complex operations in a non-production environment first. -->