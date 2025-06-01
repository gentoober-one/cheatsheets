# Miscellaneous Command Cheatsheet

**General Note:** This document is a compilation of various command-line utilities and snippets primarily for Linux-based systems. It serves as a quick reference. Many commands require `root` privileges to run. Always understand a command and its potential impact before executing it, especially those involving data modification, system configuration, or network interaction. This cheatsheet is a continuous work in progress; some sections may be incomplete or contain examples that require adaptation to your specific system or needs.

## System Administration & Information

### USB Image Creation (Bootable Drives)

<!-- This command writes an ISO image (or any raw disk image) to a USB drive. -->
<!-- CAUTION: THIS IS A BLOCK-LEVEL OPERATION. -->
<!-- Ensure 'of=/dev/sdX' points to your correct USB drive. -->
<!-- Writing to the WRONG DEVICE WILL CAUSE IRREVERSIBLE DATA LOSS on that device (e.g., your main hard drive). -->
<!-- Use 'lsblk' or 'fdisk -l' to identify your USB device path carefully. -->
```bash
# Example: Write an ISO image to /dev/sda (replace /dev/sda with your actual USB device!)
#
# dd: Disk Duplicator - a utility for block-level copying.
#   bs=1M: Set block size to 1 Megabyte. This can affect performance. Larger bs can be faster but may use more RAM.
#   conv=fsync: Sync data and metadata to the destination drive before dd reports completion. Ensures data is physically written.
#   oflag=direct: Use direct I/O, bypassing the system's page cache. Can be faster for large sequential writes and reduces cache pressure.
#   status=progress: Show real-time progress of the data transfer (bytes copied, speed).
#   if=/path/to/your/image.iso: Input File - the path to the ISO or disk image you want to write.
#   of=/dev/sdX: Output File - the target USB device (e.g., /dev/sdb, /dev/sdc). DOUBLE CHECK THIS!
dd bs=1M conv=fsync oflag=direct status=progress if=/path/to/your/image.iso of=/dev/sdX
```

### File & Directory Search (`find`)

<!-- The 'find' command is a powerful tool for locating files and directories based on various criteria. -->

#### General Usage: Find by Name and Type
<!-- Find files with a specific name and extension, up to a certain directory depth, excluding hidden files/directories. -->
```bash
# .: Search starting from the current directory.
# -maxdepth 4: Limit the search to 4 levels deep from the starting point.
# -type f: Search only for regular files (not directories, links, etc.).
# -name "*.txt": Find files whose names end with the .txt extension. Case-sensitive. Use -iname for case-insensitive.
# -not -path '*/.*': Exclude any file or directory whose path contains a component starting with a dot (e.g., .git, .config).
find . -maxdepth 4 -type f -name "*.txt" -not -path '*/.*'
```

#### Excluding Specific Paths from Search
<!-- Find files by name while preventing 'find' from descending into specified directories. -->
```bash
# /: Start search from the root directory.
# -type d -path '/path/to/exclude' -prune:
#   If a directory matching '/path/to/exclude' is found, do not descend into it (prune it from the search tree).
# -o: OR operator. This combines the pruning action with the search action.
# -type f -iname 'filename_pattern': Find files (case-insensitive due to -iname) matching 'filename_pattern'.
# -print: Print the paths of the found files. (Often default, but explicit here).
find / -type d -path '/path/to/exclude' -prune -o -type f -iname 'filename_pattern' -print
```

#### Searching by File Size
<!-- Find files smaller or larger than a specific size. -->
```bash
# <path_to_search>: Directory to start the search (e.g., . for current, / for root).
# -type f: Search for files only.
# -size -10M: Find files strictly smaller than 10 Megabytes.
#   Use +10M for files strictly larger than 10M.
#   Use 10M for files with a size of exactly 10M (rounded up by 512-byte blocks unless a 'c' suffix is used for bytes).
#   Suffixes: c (bytes), k (Kilobytes), M (Megabytes), G (Gigabytes).
# <file_pattern>: Optional: further filter by filename or type (e.g., -name "*.log").
find <path_to_search> -type f -size -10M -name "*.log"
```

#### Finding and Executing Commands (e.g., Deleting Files)
<!-- Find files based on criteria and execute a command on each found file. -->
<!-- CAUTION: Using -exec rm {} \; is very powerful. Incorrect usage can lead to MASSIVE DATA LOSS. -->
<!-- Always test with -print or -ls first to ensure your 'find' criteria are correct before using -exec rm. -->
```bash
# .: Search in the current directory and its subdirectories.
# -type f: Search for files only.
# -name "*.rar": Find files with the .rar extension.
# -exec rm {} \;: For each file found:
#   rm: The command to execute (remove file).
#   {}: This placeholder is replaced by the path of the current file found by 'find'.
#   \;: Terminates the -exec command. The semicolon needs to be escaped (or quoted) to prevent shell interpretation.
# Alternative using -delete (safer if only deleting and supported by your 'find' version):
# find . -type f -name "*.rar" -delete
find . -type f -name "*.rar" -exec rm {} \;
```

### Security Checks with `find`

<!-- Using 'find' to identify files with potentially sensitive permissions or recent modifications. -->

#### Check File Permissions (SUID/SGID)
<!-- Find files with SUID (Set User ID) or SGID (Set Group ID) permissions. -->
<!-- These permissions allow users to run executables with the privileges of the file owner (SUID) or group (SGID). -->
<!-- While necessary for some system operations, they can be a security risk if misconfigured or on unintended files. -->
<!-- The 'grep' part attempts to exclude some common false positives on modern systems; this may need adjustment. -->
```bash
# /: Search from the root directory.
# \( -perm -02000 -o -perm -04000 \): Search for files with:
#   -perm -02000: SGID bit set (leading '2' in octal, e.g., 2755). The '-' means at least these bits are set.
#   -o: OR operator.
#   -perm -04000: SUID bit set (leading '4' in octal, e.g., 4755).
# -ls: Display found files in 'ls -li' format (detailed information).
# grep -vE '\.var|\.usr|proc|run|dev|sys|snap|flatpak':
#   Exclude lines containing common system paths like /var, /usr directories, and pseudo-filesystems
#   to reduce noise. This exclusion list might need to be tailored to your system.
find / \( -perm -02000 -o -perm -04000 \) -ls | grep -vE '\.var|\.usr|proc|run|dev|sys|snap|flatpak'
```

#### Find Recently Modified Files (Compared to a Reference File)
<!-- Find files modified (content) or changed (metadata) more recently than a reference file (e.g., /tmp). -->
<!-- This can be useful for tracking recent system changes or identifying potential unauthorized activity. -->
<!-- /tmp is often used as a reference because its modification time might reset on reboot or be fairly recent. -->
```bash
# /: Search from the root directory.
# \( -newer /tmp -o -cnewer /tmp \):
#   -newer /tmp: File's data (content) was modified more recently than /tmp was modified.
#   -o: OR operator.
#   -cnewer /tmp: File's status (metadata like permissions, ownership) was changed more recently than /tmp was modified.
# -ls: Display found files in 'ls -li' format.
find / \( -newer /tmp -o -cnewer /tmp \) -ls
```

### User Information

#### List All Usernames
<!-- Fetch all usernames from the /etc/passwd file. -->
<!-- The /etc/passwd file stores user account information. Each line represents a user, with fields separated by colons. -->
```bash
# cut: Command to remove sections from each line of files.
#   -f1: Select only the first field.
#   -d:: Use ':' as the delimiter between fields.
#   /etc/passwd: The input file.
cut -f1 -d: /etc/passwd
```

### Process and Service Monitoring

#### Check Running Services/Processes (Forest Format)
<!-- Display running processes in a hierarchical (tree/forest) view, showing parent-child relationships. -->
<!-- This helps visualize process lineage. -->
```bash
# ps: Report a snapshot of the current processes.
# 'auxf' and 'aef --forest' are common combinations.
#
# ps auxf:
#   a: Show processes for all users (same as -e but BSD-style).
#   u: Display user-oriented format (shows username, CPU/memory usage, etc.).
#   x: Show processes not attached to a terminal (daemons).
#   f: Forest format (ASCII art process hierarchy). Often implies sorting by hierarchy.
ps auxf

# OR
# ps aef:
#   a: Show processes for all users.
#   e: Show all processes (POSIX standard, equivalent to -A).
#   f: Full-format listing. Provides more details than default. Does NOT imply forest view by itself.
ps aef

# OR (explicitly requesting forest format with 'aef')
ps aef --forest 
#   --forest: Explicitly requests the ASCII art process tree.

# OR (explicitly requesting forest format with 'aux')
ps aux --forest
```

### User Login Information

#### Display User Login/Logout History
<!-- Show a detailed history of user logins, logouts, system boots, reboots, etc., by reading from /var/log/wtmp and /var/log/btmp. -->
```bash
# last: Shows a listing of last logged in users.
#   -a: Display hostname in the last column, even if it's a local login.
#   -i: Display IP addresses instead of hostnames (useful if DNS resolution is slow or unavailable).
#   -F: Print full login and logout times and dates (includes year).
last -aiF
```

## File and Disk Management

### Batch File Renaming

#### Removing `.old` Suffix from Files
<!-- This script iterates through all files ending with '.old' in the current directory
     and renames them by removing the '.old' suffix. -->
<!-- Useful for restoring backups of kernel modules or other files where '.old' was appended as a backup strategy. -->
```bash
# Example: If you have 'vmlinuz.old', it becomes 'vmlinuz'.
#          If you have 'config.txt.old', it becomes 'config.txt'.

# Loop through all files matching '*.old' in the current directory.
for file in *.old; do
  # Check if the file actually exists and is a regular file.
  # This avoids errors if no *.old files are present or if a match is a directory.
  if [ -f "$file" ]; then
    # mv: Move (rename) files.
    # "$file": The original filename (e.g., "something.old"). Quoted to handle spaces.
    # "${file%.old}": Parameter expansion. Removes the shortest suffix matching ".old" from $file.
    #                (e.g., "something.old" becomes "something"). Quoted.
    mv -v "$file" "${file%.old}" # -v for verbose, shows what is being done.
  fi
done
```

### Secure Disk Erase (`hdparm`)
<!-- CAUTION: These commands are for securely erasing data on ATA hard drives (HDDs) and some SSDs. -->
<!-- They can lead to COMPLETE AND IRREVERSIBLE DATA LOSS on the specified device. -->
<!-- VERIFY THE TARGET DEVICE (e.g., /dev/sda, /dev/sdb) MULTIPLE TIMES BEFORE EXECUTION. -->
<!-- These commands typically require root privileges. -->
<!-- The password 'your_password' is an example; use a password you will remember or use 'NULL' if supported by your drive. -->
<!-- This process leverages built-in ATA Security Erase commands in the drive's firmware. -->
```bash
# 1. Set a user password for security features on the drive.
#    This password enables access to the security erase commands.
#    '--user-master u': Specifies the user master (usually 'u' for user).
#    '--security-set-pass your_password': Sets the password to 'your_password'.
#                                         Replace 'your_password' with your chosen password or NULL.
#    '/dev/sdX': Target drive. Replace with your actual device (e.g., /dev/sdb).
sudo hdparm --user-master u --security-set-pass your_password /dev/sdX

# 2. Issue the secure erase command.
#    '--security-erase your_password': Erases the drive using the previously set password.
#    This process can take a significant amount of time depending on the drive size and type.
#    The drive itself performs the erase operation.
sudo hdparm --user-master u --security-erase your_password /dev/sdX

# Note on Enhanced Secure Erase:
# Some drives, especially SSDs, support an "enhanced security erase" which is often more thorough.
# sudo hdparm --user-master u --security-erase-enhanced your_password /dev/sdX

# Important Considerations:
# - Consult your drive's manual and hdparm documentation before using these commands.
# - Data recovery after these commands is generally considered impossible by software means.
# - For SSDs, ATA Secure Erase is one of the recommended methods for wiping data, as it triggers
#   internal SSD mechanisms to clear all blocks, including over-provisioned areas.
# - Ensure the drive is not mounted and not in use.
# - If the drive is frozen (security features locked), you might need to power cycle it (disconnect/reconnect power cable)
#   while the system is running, then re-issue hdparm commands. This is tricky and varies by system.
```

### Text Manipulation

#### Uppercasing Text using `tr`
<!-- Convert a string or input stream from lowercase to uppercase. -->
```bash
# echo "<your_message>": Prints the message to standard output.
# |: Pipe symbol, sends the output of 'echo' to the input of 'tr'.
# tr '[:lower:]' '[:upper:]': Translate (tr) characters.
#   '[:lower:]': Represents all lowercase characters according to the current locale.
#   '[:upper:]': Represents all uppercase characters according to the current locale.
echo "your message here" | tr '[:lower:]' '[:upper:]'
# Example output: YOUR MESSAGE HERE
```

### Duplicate File Detection

<!-- These commands help identify duplicate files based on their content checksums (MD5 in this example). -->
<!-- While MD5 is fast, for critical applications where collision resistance is paramount, consider 'sha256sum'. -->
<!-- A hash collision means different files produce the same hash, which is extremely rare for MD5 with typical files but theoretically possible. -->

#### Step 1: Generate Checksums, Sort, and Count Duplicates
<!-- This command calculates MD5 checksums for all files (or specified types like *.jpg),
     extracts just the hash, sorts them, counts unique hashes, and then sorts by count. -->
```bash
# md5sum *: Calculate MD5 checksums for all files in the current directory.
#             Replace '*' with your target file types (e.g., '*.jpg', '*.txt') or paths.
#             Be careful with '*' in directories with many files or subdirectories if not intended.
# cut -c1-32: Extract only the first 32 characters (the MD5 hash itself) from the 'md5sum' output.
#             'md5sum' output format is typically: <hash>  <filename>
# sort: Sort the hashes alphabetically. Duplicate hashes will now be adjacent.
# uniq -c: Count occurrences of each unique hash. Lines with identical hashes are grouped,
#          and 'uniq -c' prepends the count to each line.
# sort -nr: Sort numerically (-n) in reverse order (-r) based on the count (highest count first).
md5sum * | cut -c1-32 | sort | uniq -c | sort -nr
```
<!--
Example Output:
  3 f6464ed766daca87ba407aede21c8fcc  <- This hash (and thus file content) appears 3 times.
  2 c7978522c58425f6af3f095ef1de1cd5  <- This hash appears 2 times.
  1 d8ad913044a51408ec1ed8a204ea9502  <- This hash appears once (unique file content).
If all counts are '1', there are no duplicate files (based on content) among those checked.
-->

#### Step 2: Identify Filenames Associated with a Specific Checksum
<!-- After finding a hash that appears multiple times (from Step 1), use this command to find the actual filenames. -->
```bash
# md5sum *: Recalculate MD5s for the same set of files as in Step 1.
#             (Alternatively, if you saved the full output of 'md5sum *' from Step 1, you can grep that).
# grep <hash_value>: Search for lines containing the specific hash value you identified as a duplicate.
# Replace '<hash_value>' with the actual hash (e.g., f6464ed766daca87ba407aede21c8fcc).
md5sum * | grep your_checksum_here
```
<!--
Example Output (if 'your_checksum_here' was 'f6464ed766daca87ba407aede21c8fcc'):
f6464ed766daca87ba407aede21c8fcc  image001.jpg
f6464ed766daca87ba407aede21c8fcc  image003.jpg
f6464ed766daca87ba407aede21c8fcc  another_image_copy.jpg
These are the files that share the same content.
-->

<!-- Recommendation: For better security against hash collisions, especially for critical data verification, use sha256sum:
# Step 1 with sha256sum:
sha256sum * | cut -d' ' -f1 | sort | uniq -c | sort -nr
# (cut -d' ' -f1 extracts the first field using space as delimiter, as sha256sum output is similar to md5sum)

# Step 2 with sha256sum:
sha256sum * | grep your_sha256_checksum_here
-->

## Gentoo Portage Management (`emerge`)

<!-- Commands related to Gentoo's package manager, Portage. -->
<!-- These commands typically require root privileges (e.g., run with `sudo`). -->
<!-- Gentoo users should be familiar with Portage concepts like USE flags, /etc/portage/, and the `emerge` command. -->

### Excluding Packages from Updates/Recompilations
<!-- Useful for temporarily preventing specific packages (e.g., large ones like compilers, or potentially problematic ones)
     from being updated or recompiled during a world update (`emerge @world`). -->
```bash
# emerge -eq @world --exclude=<category/package_name_1> --exclude=<category/package_name_2>:
#   emerge: The command-line interface to Portage.
#   -e or --emptytree: Rebuild all packages in @world, not just those with recorded version changes.
#                      Often used for major system updates or after USE flag changes.
#   -q or --quiet: Reduce output verbosity.
#   @world: A system set that includes all packages explicitly installed by the user and their dependencies.
#   --exclude="<category/package_name>": Prevent the specified package(s) from being included in this operation.
#                                        Can be specified multiple times.
# Example: Exclude gcc (compiler) and llvm (another compiler infrastructure) from an update/rebuild.
sudo emerge -eq --exclude="sys-devel/gcc" --exclude="sys-devel/llvm" @world
```

### Installing a Specific Package Version
<!-- Allows you to install an exact version of a package, rather than the latest stable or testing version masked by Portage. -->
<!-- This is useful for downgrading a problematic package or installing an older version for compatibility. -->
```bash
# emerge -av =<category/package_name>-<version>:
#   -a or --ask: Display what changes would be made and ask for confirmation before proceeding. Highly recommended.
#   -v or --verbose: Increase output verbosity.
#   =<category/package_name>-<version>: The equals sign '=' specifies an exact package version.
#                                      The version string must match an available version in the Portage tree.
# Example: Install vanilla-kernel version 5.10.189.
sudo emerge -av =sys-kernel/vanilla-kernel-5.10.189
```

### Custom Environment Flags for Specific Packages (e.g., GCC Graphite)
<!-- Portage allows setting custom compilation flags (CFLAGS, CXXFLAGS, LDFLAGS) for individual packages. -->
<!-- This is achieved by creating environment files in /etc/portage/env/ and then associating them with packages
     via /etc/portage/package.env. -->
<!-- This example shows how to enable GCC's Graphite loop optimization flags for a package, like GCC itself. -->
<!-- Enabling such flags can potentially improve performance for CPU-bound tasks but may also increase compile times
     or, in rare cases, lead to instability if not well-tested for the specific package. -->

#### 1. Create a Custom Environment Configuration File
<!-- Example: /etc/portage/env/custom-gcc-graphite-flags.conf (filename can be descriptive) -->
<!-- This file defines environment variables that will be set during the compilation of packages associated with it. -->
```bash
# Content for /etc/portage/env/custom-gcc-graphite-flags.conf:

# GRAPHITE_FLAGS: Shell variable to hold Graphite-specific GCC flags.
# These flags enable advanced loop optimizations.
#   -floop-interchange: Performs loop interchange transformations.
#   -ftree-loop-distribution: Enables loop distribution.
#   -floop-strip-mine: Performs loop strip mining.
#   -floop-block: Enables loop blocking or tiling.
# Refer to GCC documentation for details on these flags.
GRAPHITE_FLAGS="-floop-interchange -ftree-loop-distribution -floop-strip-mine -floop-block"

# CFLAGS: C compiler flags.
#   -O2: Optimization level 2 (a common default, good balance).
#   -pipe: Use pipes between compilation stages (can speed up compilation on some systems).
#   -march=native: Generate code optimized for the specific architecture of the host CPU.
#   -flto=jobserver: Enable Link Time Optimization (LTO), using a job server for parallel linking if available.
#                    LTO can improve runtime performance but increases link times.
#   ${GRAPHITE_FLAGS}: Include the Graphite flags defined above.
#   -ftree-vectorize: Enable loop vectorization, attempting to use SIMD instructions.
CFLAGS="-O2 -pipe -march=native -flto=jobserver ${GRAPHITE_FLAGS} -ftree-vectorize"

# CXXFLAGS: C++ compiler flags. Here, set to the same as CFLAGS.
CXXFLAGS="${CFLAGS}"

# LDFLAGS: Linker flags.
#   ${CFLAGS}: Pass CFLAGS to the linker (some flags like -flto are relevant to both compiler and linker).
#   -fuse-linker-plugin: Enables the LTO linker plugin in GCC, necessary for LTO to work with some linkers (like GNU ld).
LDFLAGS="${CFLAGS} -fuse-linker-plugin"

# Note: The specific CFLAGS like -O2, -pipe, -march=native should be adjusted based on your system,
# preferences, and the package being compiled. These are advanced flags; test thoroughly.
# The original example had CFLAGS/CXXFLAGS/LDFLAGS="${CFLAGS}", which is unusual for LDFLAGS.
# LDFLAGS typically include paths (-L) or library links (-l), but can also share optimization flags like -flto.
# Consult GCC documentation: https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html
```

#### 2. Associate the Configuration File with a Package
<!-- Edit or create /etc/portage/package.env. -->
<!-- This file maps package atoms (or categories) to the custom environment files created in /etc/portage/env/. -->
```
# Content for /etc/portage/package.env:
# Format: <category/package_name_or_atom> <path_to_your_env_file_relative_to_/etc/portage/env/>
# Or: <category/package_name_or_atom> /etc/portage/env/<your_env_file_name>

# Example: Apply these custom flags to the 'sys-devel/gcc' package.
sys-devel/gcc /etc/portage/env/custom-gcc-graphite-flags.conf

# Example: Apply the same flags to another package.
# another/package /etc/portage/env/custom-gcc-graphite-flags.conf
```

#### 3. (Optional) Change Default Linker (e.g., to `gold` or `lld`)
<!-- The original note suggests changing the linker to 'gold'. This might be beneficial for LTO or faster linking on some systems. -->
<!-- This is a system-wide change affecting how all packages are linked by default, unless overridden. -->
<!-- Gentoo's `binutils-config` tool manages the active linker. -->
```bash
# binutils-config --linker ld.gold: Set 'ld.gold' as the default system linker.
# Alternatives might include 'ld.bfd' (default GNU linker on many systems) or 'ld.lld' (from LLVM, often very fast).
# Check available linkers with: binutils-config --list-profiles
sudo binutils-config --linker ld.gold
```
<!-- After making these changes in /etc/portage/, the specified packages will use these flags when they are next recompiled. -->
<!-- You can force a recompile of a specific package with: sudo emerge -1 <category/package_name> -->
<!-- Recompiling GCC with these flags will take a significant amount of time. -->

## Networking

### SSH Tunnel Persistence (Keep Alive)
<!-- Keep an SSH session alive by sending server alive messages periodically. -->
<!-- Useful if your connection tends to drop due to inactivity, common with NAT routers or firewalls. -->
```bash
# ssh -o ServerAliveInterval=15 user@hostname_or_ip:
#   -o ServerAliveInterval=15: Instructs the SSH client to send a null packet (server alive message)
#                              to the server every 15 seconds to keep the connection active.
#   user@hostname_or_ip: Your username and the server's address.
ssh -o ServerAliveInterval=15 user@hostname_or_ip

# You can also configure this globally in your ~/.ssh/config file:
# Host *
#   ServerAliveInterval 60
# This would apply to all SSH connections.
```

### Downloading Files with `wget`

#### Mirroring a Site for Specific File Types (e.g., PDFs)
<!-- Download all files of a certain type (e.g., PDF) from a website or a specific directory on it. -->
<!-- This can be used to download any file type, not just PDFs, by changing the --accept pattern. -->
```bash
# wget --mirror --no-parent --accept "*.pdf" http://example.com/somedirectory:
#   wget: Non-interactive network downloader.
#   --mirror: Turn on options suitable for mirroring. This includes:
#             -r (recursive), -N (timestamping), -l inf (infinite recursion depth), --no-remove-listing.
#   --no-parent: Do not ascend to the parent directory when retrieving recursively. Keeps downloads within the specified path.
#   --accept "*.pdf": Specify a comma-separated list of accepted filename patterns (globbing). Only download PDFs.
#                     Use e.g., --accept "*.pdf,*.doc" for multiple types.
#   http://example.com/somedirectory: The starting URL. wget will recursively explore from here.
wget --mirror --no-parent --accept "*.pdf" http://example.com/somedirectory/
# Note: Be mindful of website's robots.txt and terms of service. Aggressive mirroring can overload servers.
# Consider adding --wait=seconds and --random-wait options for politeness.
```

#### Downloading from a List of URLs in a File
<!-- Download multiple files listed in a text file, one URL per line. -->
```bash
# wget -c -N -i links.txt:
#   -c or --continue: Continue getting a partially-downloaded file. Useful if a previous download was interrupted.
#   -N or --timestamping: Don't re-retrieve files unless the remote file is newer than the local one.
#   -i links.txt: Read URLs from the specified input file (here, "links.txt"). Each URL should be on a new line.
wget -c -N -i links.txt
```

## Multimedia

### Video Remuxing/Re-encoding for Messaging Apps (e.g., WhatsApp/Telegram)
<!-- Re-encode a video to make it more suitable for sending via messaging apps. -->
<!-- This typically involves reducing resolution, bitrate, and ensuring compatible codecs (H.264/AAC). -->
```bash
# ffmpeg -i "Input Video.mp4" -vf "scale=640:-1" -b:v 1M -c:v libx264 -c:a aac -strict experimental "Output Video_Optimized.mp4":
#   ffmpeg: The command-line tool for converting multimedia between formats.
#   -i "Input Video.mp4": Specifies the input video file. Quote filename if it contains spaces.
#   -vf "scale=640:-1": Video Filtergraph.
#     scale=640:-1: Resizes the video.
#       width=640: Set width to 640 pixels.
#       height=-1: Automatically calculate height to maintain the original aspect ratio.
#                  Use -2 to ensure height is divisible by 2 (often required by codecs). E.g., scale=640:-2.
#   -b:v 1M: Set the target video bitrate to 1 Mbps (Megabit per second). Adjust for desired quality/size balance.
#            Lower bitrate = smaller file, lower quality.
#   -c:v libx264: Video Codec. Use libx264 for H.264 video encoding (widely compatible).
#                 Consider using presets for libx264, e.g., -preset medium (default) or -preset fast for faster encoding.
#   -c:a aac: Audio Codec. Use AAC (Advanced Audio Coding), also widely compatible.
#   -strict experimental: This flag was often needed for older versions of ffmpeg with AAC.
#                         Modern ffmpeg versions might not require it, or '-strict -2' might be used. Test without it first.
#   "Output Video_Optimized.mp4": Specifies the output video file.
#
# Note: The original filename "Buda - Aprenda a Silenciar a Mente.mp4" contained spaces and special characters.
# It's good practice to quote filenames in shell commands.
ffmpeg -i "Input Video File.mp4" -vf "scale=640:-2" -b:v 1M -c:v libx264 -c:a aac "Output Video File_Optimized.mp4"
```

## File Compression and Archiving (`tar` with various compressors)

<!-- These examples show how to use 'tar' (Tape Archiver) to create archives
     and pipe the output to various compression programs. -->
<!-- Replace 'your_directory/' with the actual directory you want to compress
     and 'archive_name' with your desired archive filename base. -->
<!-- `tar` options used:
       c: Create a new archive.
       v: Verbosely list files processed.
       f <filename>: Use archive file or device ARCHIVE. If '-', writes to standard output or reads from standard input.
       P: (Sometimes used, GNU tar) Don't strip leading slashes from filenames. Use with caution.
     General structure for compression: tar cvf - your_directory/ | compressor_command > archive_name.tar.compressor_ext
     General structure for decompression: decompressor_command -d < archive_name.tar.compressor_ext | tar xvf -
-->

### Using `lrzip` (Long Range ZIP)
<!-- lrzip is particularly effective for large files and can achieve good compression ratios.
     It uses a long-range redundancy reduction step before compression.
     Options: -l (LZO, fast), -z (ZPAQ, high compression, very slow), -g (GZIP), -b (BZIP2), -n (no backend compression). -->
```bash
# Compression:
# tar cvf archive_name.tar your_directory/ --one-file-system --remove-files --use-compress-program="lrzip -l -L1 -p$(nproc) -N0"
#   --one-file-system: Stay in the same file system (do not cross mount points).
#   --remove-files: Remove original files after adding them to the archive (USE WITH EXTREME CAUTION! DATA LOSS RISK!).
#   --use-compress-program="lrzip ...": Pipe the tar output through lrzip. This is a GNU tar specific option.
#     lrzip options for speed (LZO):
#       -l: Lempel-Ziv-Oberhumer (LZO) compression (very fast).
#       -L1: Compression level 1 (fastest for LZO). Levels go up to 9.
#       -p$(nproc): Set processor count to the number of available cores ($(nproc) gets this value).
#       -N0: Set nice value (CPU priority) to 0.
# The original example created `archive_name.tar` then lrzip would make `archive_name.tar.lrz`.
# A more common piping approach:
tar cv your_directory/ --one-file-system | lrzip -l -L1 -p$(nproc) -N0 -o archive_name.tar.lrz
# Consider lrzip options: -U for unlimited window size (good for very large files with distant redundancy), -z for ZPAQ (much better compression, but extremely slow).

# Decompression:
# 1. Decompress the .lrz file to a .tar file:
lrzip -d archive_name.tar.lrz
# This creates 'archive_name.tar'.
# 2. Extract the .tar file:
tar xvf archive_name.tar
# Or in one pipe:
lrzip -d -c archive_name.tar.lrz | tar xvf -
```

### Using `lz4` (Extremely Fast Compression)
<!-- lz4 is known for its very high speed for both compression and decompression, at the cost of a lower compression ratio compared to zstd or xz. -->
```bash
# Compression (Method 1: GNU tar's --use-compress-program):
# tar cvf archive_name.tar.lz4 your_directory/ --use-compress-program="lz4 --fast=12"
#   --use-compress-program="lz4 --fast=12": Pipe output through lz4.
#     --fast=12: Use a fast compression level. Higher numbers are faster with less compression. Max is usually around 12 for --fast.
#                For better compression with lz4, omit --fast or use -1 to -9 (e.g. -9 for hc mode). lz4 default is fast.
tar cvf archive_name.tar.lz4 your_directory/ --use-compress-program="lz4 -1" # Example for default compression
# Compression (Method 2: Piping)
tar cv your_directory/ | lz4 -1 - > archive_name.tar.lz4 # -1 is default compression, use e.g. -9 for HC

# Decompression (Method 1: If created with --use-compress-program):
# tar xvf archive_name.tar.lz4 --use-compress-program="lz4 -d"
# Decompression (Method 2: Piping, more general)
lz4 -d -c archive_name.tar.lz4 | tar xvf -
# Or first decompress, then extract:
# lz4 -d archive_name.tar.lz4 -f decompressed_archive.tar && tar xvf decompressed_archive.tar
```

#### Decompressing Multiple `.tar.lz4` Files (Original Snippet Clarified)
<!-- This loop processes files ending in '.tar.lz4'. It first decompresses the lz4 part,
     then extracts the resulting .tar archive. -->
```bash
# Loop through each file ending in .tar.lz4 in the current directory:
for i in *.tar.lz4; do
    # Check if the file exists to prevent errors if no matching files are found.
    if [ -f "$i" ]; then
        base_name="${i%.tar.lz4}" # Remove the .tar.lz4 suffix to get the base name.
        echo "Processing $i -> ${base_name}.tar ..."
        # 1. Decompress it using lz4. Output is named based on original without .lz4.
        #    lz4 -d "$i" -f "${base_name}.tar":
        #      -d: Decompress.
        #      "$i": Input file.
        #      -f: Force overwrite of output file if it exists.
        #      "${base_name}.tar": Explicit output filename for the decompressed tarball.
        if lz4 -d "$i" -f "${base_name}.tar"; then
            # 2. Extract the resulting .tar file.
            echo "Extracting ${base_name}.tar ..."
            tar xvf "${base_name}.tar"
            # Optional: remove the intermediate .tar file
            # rm "${base_name}.tar"
        else
            echo "Failed to decompress $i"
        fi
    fi
done

# If you have files that are just .lz4 (not .tar.lz4, meaning they were single files compressed with lz4):
# for i in *.lz4; do
#   if [ -f "$i" ]; then
#     lz4 -d "$i" # Decompresses to filename without .lz4
#   fi
# done
```

### Using `zstd` (Zstandard - Balanced Speed and Ratio)
<!-- zstd offers a good balance between compression speed and ratio, often better than gzip/bzip2 and competitive with lrzip's LZO mode but with better ratios. -->
```bash
# Compression:
# tar -cvf - your_source_directory/ | zstd -vz --ultra -o archive_name.tar.zst:
#   tar -cvf - your_source_directory/: Create a tar archive and output it to stdout (-).
#   zstd -vz --ultra -o archive_name.tar.zst: Compress the input from stdin.
#     -v: Verbose output (shows compression ratio, etc.).
#     -z: Compress (default action, can be omitted).
#     --ultra: Enable higher compression levels (typically levels 20-22). Can be very slow but yields better ratios.
#              For general use, a numbered level like -10 to -19 is often a good balance. Default is 3.
#     -T0: Use all available CPU cores/threads for compression.
#     -o archive_name.tar.zst: Output to specified file.
# Example using a specific compression level (e.g., 19) and multi-threading:
tar -cv your_source_directory/ | zstd -v -19 -T0 -o archive_name.tar.zst

# Decompression:
# Method 1: Decompress to .tar, then extract .tar
# zstd -d archive_name.tar.zst -o archive_name.tar
# tar -xvf archive_name.tar
# Method 2: One pipe (recommended)
zstd -d -c archive_name.tar.zst | tar xvf -
#   -d: Decompress.
#   -c: Write to standard output (for piping).
```

### Using `lzo` / `lzop` (Lempel-Ziv-Oberhumer - Fast)
<!-- lzop is another fast compression utility, similar in speed to lz4 but potentially different compression ratios. -->
```bash
# Compression:
# tar cf - your_directory/ | lzop -v -9 -o archive_name.tar.lzo:
#   tar cf - your_directory/: Create tar archive to stdout. (Omitting 'v' for less noise in pipe).
#   lzop -v -9 -o archive_name.tar.lzo: Compress stdin with lzop.
#     -v: Verbose.
#     -9: Compression level 9 (best compression for lzop, but still fast). Default is often level 3 or 5.
#     -o archive_name.tar.lzo: Output file.
tar cf - your_directory/ | lzop -v -9 -o archive_name.tar.lzo

# Decompression:
# Method 1: Decompress to .tar, then extract
# lzop -d archive_name.tar.lzo -o archive_name.tar
# tar xvf archive_name.tar
# Method 2: One pipe (recommended)
lzop -d -c archive_name.tar.lzo | tar xvf -
#   -d: Decompress.
#   -c: Write to standard output.
```

## System Configuration

### XDG Default Applications (`xdg-mime`, `xdg-settings`)
<!-- Manage default applications for opening file types (MIME types) and URL schemes (like http, mailto). -->
<!-- These tools interact with the XDG MIME Applications system, used by most desktop environments on Linux. -->

#### Setting Default Application for a MIME Type
```bash
# xdg-mime default <application.desktop> <MIME_type>:
#   <application.desktop>: The .desktop file of the application. These files are typically found in
#                          /usr/share/applications/ or ~/.local/share/applications/.
#                          You only need the filename, e.g., 'firefox.desktop'.
#   <MIME_type>: The MIME type you want to associate (e.g., inode/directory, image/jpeg, application/pdf, text/html).

# Example: Set pcmanfm (file manager) as default for opening directories.
xdg-mime default pcmanfm.desktop inode/directory
# Query the current default for directories to verify:
xdg-mime query default inode/directory

# Example: Set sxiv (image viewer) for various image types.
xdg-mime default sxiv.desktop image/jpeg
xdg-mime default sxiv.desktop image/png
xdg-mime default sxiv.desktop image/gif

# Example: Set mpv (media player) for all audio and video types.
# The ".*" is a wildcard for subtypes, but xdg-mime might not support it directly this way.
# It's usually better to list common specific MIME types or use a more robust method if available.
# For broad categories, you might need to list multiple specific types:
# xdg-mime default mpv.desktop audio/mpeg audio/ogg audio/wav video/mp4 video/webm video/x-matroska
# The original example `audio/.*` and `video/.*` syntax for xdg-mime is non-standard.
# Check `xdg-mime query filetype <example_file>` to find MIME type of a file.
# For instance, for audio files:
xdg-mime default mpv.desktop audio/mpeg audio/vorbis audio/flac audio/x-wav
# For video files:
xdg-mime default mpv.desktop video/mp4 video/webm video/x-matroska video/avi
```

#### Setting Default Web Browser
```bash
# xdg-settings set default-web-browser <browser.desktop>:
#   <browser.desktop>: The .desktop file of the web browser (e.g., firefox.desktop, brave-browser.desktop).

# Example: Set Brave Browser as default.
xdg-settings set default-web-browser brave-bin.desktop # Name might vary (e.g., brave-browser.desktop)
# Example: Set Firefox as default.
# xdg-settings set default-web-browser firefox.desktop

# You can find .desktop file names by listing files in /usr/share/applications/ or ~/.local/share/applications/.
# Use `xdg-settings get default-web-browser` to check the current setting.
```

### Sound Management (PulseAudio with `pactl`)

<!-- `pactl` is a command-line tool to control a running PulseAudio sound server. -->

#### List Audio Sources (Inputs)
<!-- Shows a list of available audio input sources (e.g., microphones, line-in). -->
```bash
# pactl list short sources:
#   list: Command to list objects.
#   short: Output in a more compact, parsable format.
#   sources: Specify that we want to list input sources.
pactl list short sources
# Example output might look like:
# 0 alsa_output.pci-0000_00_1f.3.analog-stereo.monitor module-alsa-card.c N/A RUNNING
# 1 alsa_input.pci-0000_00_1f.3.analog-stereo module-alsa-card.c s16le 2ch 44100Hz SUSPENDED
# The first field is the source ID, the second is its name.
```

#### Set Volume for a Specific Source (Input)
<!-- Adjust the volume of an audio input source (e.g., microphone). -->
```bash
# pactl set-source-volume <source_id_or_name> <volume_percentage>%:
#   set-source-volume: Command to set volume for an input source.
#   <source_id_or_name>: The ID number or symbolic name of the source (from 'pactl list short sources').
#   <volume_percentage>%: Desired volume level (e.g., 0%, 50%, 100%, 150% for amplification if supported).
#                        Can also use absolute values (e.g., 32768 for 50% if max is 65536) or decibels (e.g., -6dB).

# Example: Set volume of source '0' (replace with actual ID/name from the list command) to 25%.
pactl set-source-volume 0 25%
# Example using source name:
# pactl set-source-volume alsa_input.pci-0000_00_1f.3.analog-stereo 50%
```

### Basic Firewall Configuration (`iptables`)
<!-- CAUTION: Modifying iptables rules can affect your network connectivity, potentially locking you out of a remote server. -->
<!-- These are very basic examples. For comprehensive firewall setup, consider using `ufw` (Uncomplicated Firewall),
     `firewalld`, or more detailed iptables scripts with proper chain management and rule saving. -->
<!-- Ensure you have a way to reset rules if something goes wrong (e.g., physical access, `iptables-save`/`iptables-restore`,
     or a script to flush all rules: `sudo iptables -F && sudo iptables -X && sudo iptables -Z`). -->
<!-- These commands usually require root privileges (`sudo`). -->

#### Drop Incoming ICMP (Ping) Requests
<!-- This rule blocks incoming ping requests (ICMP echo-request messages). Outgoing pings from your server will still work. -->
```bash
# sudo iptables -A INPUT -p icmp --icmp-type echo-request -j DROP:
#   -A INPUT: Append this rule to the INPUT chain (handles incoming traffic to the server itself).
#   -p icmp: Match the ICMP protocol.
#   --icmp-type echo-request: Specifically match ICMP type 8 (echo-request). This is more precise than just '-p icmp'.
#   -j DROP: If the packet matches, drop it (silently discard, no response sent).
# && sudo iptables -L -v -n: After adding the rule, list all rules in all chains to verify.
#   -L: List rules.
#   -v: Verbose output (shows packet/byte counters, interface names).
#   -n: Numeric output (show IP addresses and port numbers, don't resolve names).
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j DROP && sudo iptables -L -v -n

# To remove this specific rule (if it's the only one matching this description):
# sudo iptables -D INPUT -p icmp --icmp-type echo-request -j DROP

# To save rules (distribution-dependent, e.g., using `iptables-persistent` package on Debian/Ubuntu):
# sudo apt-get install iptables-persistent
# sudo netfilter-persistent save
# On RHEL/CentOS, `firewalld` is common, or `iptables-services` for legacy iptables:
# sudo systemctl enable iptables --now
# sudo iptables-save > /etc/sysconfig/iptables
```

## File Synchronization (`rsync`)

<!-- rsync is a powerful and versatile tool for synchronizing files and directories locally or remotely.
     It's efficient because it uses a delta-transfer algorithm, copying only the differences between source and destination. -->

### Common Local/Remote Sync Options
<!-- Base command structure: rsync [options] source/ destination/ -->
```bash
# Common rsync options:
#   -h, --human-readable: Output numbers in a human-readable format (e.g., 1K, 234M, 2G).
#   -a, --archive: Archive mode; equals -rlptgoD (no -H,-A,-X). Preserves:
#       -r, --recursive: Recurse into directories.
#       -l, --links: Copy symlinks as symlinks.
#       -p, --perms: Preserve permissions.
#       -t, --times: Preserve modification times.
#       -g, --group: Preserve group.
#       -o, --owner: Preserve owner (super-user only).
#       -D: Same as --devices --specials.
#   -A, --acls: Preserve ACLs (Access Control Lists), if supported.
#   -X, --xattrs: Preserve extended attributes, if supported.
#   -v, --verbose: Increase verbosity. Use -vv for more details.
#   -W, --whole-file: Copy files whole (no delta-transfer algorithm). Useful for local copies or when files are already compressed.
#   -z, --compress: Compress file data during transfer. Good for slow links, but adds CPU overhead. Not effective for already compressed files (jpg, zip, etc.).
#   --delete: Delete extraneous files from the destination directory (files that don't exist in the source). USE WITH CAUTION.
#   --progress: Show progress during transfer (per-file progress).
#   --exclude=<pattern>: Exclude files/directories matching the pattern. Can be specified multiple times.
#   -n, --dry-run: Perform a trial run with no changes made. Highly recommended for testing, especially with --delete.

# Example 1: Backup 'source_directory/' to a 'destination_directory/', excluding a subdirectory.
# Trailing slash on source ('source_directory/') means copy the *contents* of 'source_directory'.
# No trailing slash ('source_directory') would mean copy the directory 'source_directory' itself into the destination.
rsync -haAvWXz --delete --progress --exclude="some_subdirectory_to_exclude/" /path/to/source_directory/ /path/to/destination_directory/

# Example 2: Copy a single .tar file to a local destination.
# -W is often good for large single files like tarballs if copying locally.
rsync -havWz --progress source_archive.tar /path/to/local_destination/

# Example 3: Copy a single .tar file to a remote server using SSH.
# user@server_address: Your username and the remote server's hostname or IP.
# :/path/to/remote_destination/: Note the colon separating server from path. The path is relative to the remote user's home directory unless it starts with a slash.
rsync -havWz --progress source_archive.tar user@server_address:/path/to/remote_destination/
# For SSH on a non-standard port: rsync -e 'ssh -p <port_number>' ...
```

### Full System Backup (Clone Linux Distro)
<!-- CAUTION: This is a powerful command. Understand what it does before running.
     Creates a near-complete backup of the root filesystem to a specified destination (e.g., an external drive).
     Critical pseudo-filesystems like /dev, /proc, /sys must be excluded to avoid errors and unwanted behavior. -->
```bash
# sudo rsync -haAXvWz --progress --delete \
#     --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} \
#     / /mnt/backup_destination/
#
#   /: Source is the root directory.
#   /mnt/backup_destination/: Destination directory (ensure it's a different filesystem/drive, mounted, and preferably empty if using --delete).
#   -haAXvWz: Common options for a full backup.
#     -A, -X: Preserve ACLs and Extended Attributes, important for a full system restore.
#   --delete: Makes the destination an exact mirror. Files deleted from source will be deleted from destination.
#   --exclude={...}: List of directories to exclude. Brace expansion for multiple patterns.
#     "/dev/*", "/proc/*", "/sys/*": Essential pseudo-filesystems.
#     "/tmp/*", "/run/*": Runtime temporary files.
#     "/mnt/*", "/media/*": Mount points for other filesystems (don't back up other mounted drives into this backup).
#     "/lost+found": Filesystem recovery directory.
# Ensure the destination mount point (/mnt/backup_destination/) is empty or dedicated for this backup to avoid issues with --delete.
# This command should be run from a live environment or with the system as quiescent as possible for consistency.
sudo rsync -haAXvWz --progress --delete \
    --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} \
    / /mnt/backup_destination/
```

### Backup User's $HOME Directory
<!-- Similar to the system backup, but focused on a user's home directory. -->
```bash
# Example: Backup the current user's 'Documents' and 'Pictures' folders to an external drive,
# excluding a specific 'cache_folder' subdirectory within 'Documents'.
# Here, '~/' expands to the current user's home directory.
rsync -haAvWXz --delete --progress --exclude="Documents/cache_folder/" ~/Documents/ ~/Pictures/ /mnt/external_drive_backup_location/

# Original example: rsync -haAvWXz --delete --progress --exclude=gentoober-library/ books/ /mnt/pendrives/backups/books/
# This seems to be backing up a specific 'books' directory, not a full $HOME.

# To backup the entire current user's home directory (~/), excluding common cache and large library folders:
# This is a more typical home directory backup.
rsync -haAvWXz --delete --progress \
    --exclude=".cache/" \
    --exclude=".dbus/" \
    --exclude=".gvfs/" \
    --exclude=".local/share/Trash/" \
    --exclude="Downloads/" \
    --exclude="Videos/some_large_library/" \
    --exclude="*.tmp" \
    --exclude="*~" \
    ~/ /mnt/my_home_backup/
# Add more --exclude patterns as needed (e.g., virtual machine images, specific large directories).
# Test with -n (dry-run) first!
```

## Text Manipulation (`sed`)

<!-- sed (Stream EDitor) is used for performing basic text transformations on an input stream or files. -->

### In-place Search and Replace in a File
<!-- Modifies 'file.txt' directly (in-place), replacing all occurrences of 'old_text' with 'new_text'. -->
```bash
# sed -i 's/old_text/new_text/g' filename.txt:
#   -i: Edit file in-place.
#       GNU sed: `-i` (no argument) modifies the file directly. `-i.bak` creates a backup with .bak extension.
#       BSD sed (macOS): `-i ''` (empty string argument) modifies directly. `-i '.bak'` creates a backup.
#   's/old_text/new_text/g': Substitution command.
#     s: Substitute command.
#     old_text: The regular expression to search for. Special characters (/, &, etc.) need to be escaped with \ if used literally.
#     new_text: The replacement text. Special characters (& for entire match, \1, \2 for captures) can be used.
#     g: Global flag. Replace all occurrences on each line, not just the first.
#   Delimiters: '/' is traditional, but any character can be used (e.g., 's#old#new#g', 's|old|new|g')
#             This is useful if 'old_text' or 'new_text' contains many slashes.

# Example: Replace all occurrences of '/mnt/gentoo' with '/mnt/backup' in 'file.txt'.
# Slashes in the pattern are escaped with '\'.
sed -i 's/\/mnt\/source_path/\/mnt\/destination_path/g' file.txt
# Using a different delimiter to avoid escaping slashes:
# sed -i 's#/mnt/source_path#/mnt/destination_path#g' file.txt
```

## System Administration & Information (Continued)

### Sort Processes by Memory Usage

#### Using `vmstat` for a Memory Overview
<!-- `vmstat` provides a summary of system activity, including virtual memory, processes, CPU activity, etc. -->
<!-- While not directly sorting processes, it gives a quick snapshot of memory status. -->
```bash
# vmstat --unit M -as:
#   --unit M: Display memory sizes in Megabytes (M). Can also use k, K, g, G.
#   -a or --active: Display active and inactive memory pages.
#   -s or --stats: Display a table of various event counters and memory statistics (detailed list).
vmstat --unit M -as
```

#### Using `ps` and `awk` for Detailed Process Memory Usage
<!-- List processes sorted by Resident Set Size (RSS) or Virtual Size (VSZ). -->
<!-- RSS is the non-swapped physical memory a process has used. VSZ is the total virtual address space. -->
```bash
# ps -eo 'pid,user,group,gid,vsz,rss,comm' --sort=-rss | less:
#   -e: Select all processes.
#   -o '...': User-defined output format. Specify columns to display:
#     pid: Process ID.
#     user: Username.
#     group: Group name.
#     gid: Group ID.
#     vsz: Virtual memory size in KiB (Kibibytes).
#     rss: Resident Set Size (physical memory) in KiB.
#     comm: Command name (only executable name, not full path/args). Use 'args' for full command.
#   --sort=-rss: Sort by RSS in descending order (highest memory usage first). Use '%mem' for percentage of physical memory.
#   | less: Pipe the output to 'less' for scrollable viewing.
ps -eo 'pid,user,vsz,rss,comm' --sort=-rss | less

# Same as above, but convert VSZ (Virtual Size) and RSS (Resident Set Size) from KiB to MiB (Mebibytes) for readability.
# Also uses 'args' for full command path and arguments.
ps -eo 'pid,user,vsz,rss,args' --sort=-rss | \
  awk 'NR==1 {print; next} { $3 = $3 / 1024 " MiB"; $4 = $4 / 1024 " MiB"; print }' | \
  column -t | less
# awk script explained:
#   NR==1 {print; next}: Print the header line (NR==1 means first record/line) as is, then skip to next line.
#   { $3 = $3 / 1024 " MiB"; $4 = $4 / 1024 " MiB"; print }: For other lines:
#     $3 = $3 / 1024 " MiB": Divide the 3rd field (VSZ) by 1024 and append " MiB".
#     $4 = $4 / 1024 " MiB": Divide the 4th field (RSS) by 1024 and append " MiB".
#     print: Print the modified line.
# column -t: Format the output into columns.

# Same as above, but convert VSZ and RSS from KiB to GiB (Gibibytes).
# Note: This might result in very small numbers for most processes if they use less than 1 GiB.
# MiB is often more practical for RSS.
ps -eo 'pid,user,vsz,rss,args' --sort=-rss | \
  awk 'NR==1 {print; next} { $3 = $3 / (1024*1024) " GiB"; $4 = $4 / (1024*1024) " GiB"; print }' | \
  column -t | less
```

## Shell Tips and Tricks

### Real-time Output Redirection and Monitoring (Logging while Viewing)

<!-- These methods allow you to save the output of a command to a file while also seeing it on the terminal in real-time. -->

#### Method 1: Using `stdbuf` and `tee`
<!-- `stdbuf` modifies the standard I/O buffering operations of a command. -->
<!-- `-oL` sets the standard output of `your_command_here` to be line-buffered. -->
<!-- This ensures each line is passed to `tee` immediately, rather than waiting for a buffer to fill. -->
```bash
# stdbuf -oL your_command_here | tee output_log.txt
#   stdbuf -oL: Adjusts buffering for 'your_command_here' so its stdout is line-buffered.
#   your_command_here: Replace with the actual command whose output you want to capture.
#   | tee output_log.txt: Pipes the output of 'your_command_here' to 'tee'.
#     tee: Reads from standard input and writes to standard output AND to files.
#       output_log.txt: The file where the output will be saved.
#       tee will also print the output to the terminal.
stdbuf -oL your_command_here | tee output_log.txt
# To append to the log file instead of overwriting:
# stdbuf -oL your_command_here | tee -a output_log.txt
```

#### Method 2: Using `tee` (Standard Behavior)
<!-- `tee` by itself already writes to stdout and to a file. `stdbuf` (Method 1) or `unbuffer` (Method 3)
     are primarily needed if `your_command_here` heavily buffers its output, causing delays in seeing it live.
     For many commands, simple `tee` is sufficient. -->
```bash
# your_command_here | tee output_log.txt
# This is the most common and straightforward way.
# `tee` duplicates its input, sending one copy to `output_log.txt` and another to its stdout (the terminal).
your_command_here | tee output_log.txt
# To append:
# your_command_here | tee -a output_log.txt

# The original `tee >(cat)` is a bit unusual for simple file logging + stdout.
# `>(cat)` is process substitution. `tee >(cat)` would mean `tee` writes its output to a pipe
# that feeds `cat`, and `cat` then prints to stdout. If a filename was also given to `tee`,
# e.g., `your_command | tee log.txt >(cat)`, then `tee` writes to `log.txt` AND to the `cat` process.
# This is more complex than necessary for just viewing and logging.
# `your_command | tee log.txt` achieves the same primary goal more simply.
```

#### Method 3: Using `unbuffer`
<!-- `unbuffer` is part of the 'expect' package and disables output buffering for the command it runs. -->
<!-- This is similar to `stdbuf -o0` or `stdbuf -oL`. -->
<!-- May need to install 'expect': sudo apt-get install expect / sudo yum install expect -->
```bash
# unbuffer your_command_here | tee output_log.txt
#   unbuffer: Disables buffering for 'your_command_here'.
#   The rest works like the `stdbuf` example.
unbuffer your_command_here | tee output_log.txt
# To append:
# unbuffer your_command_here | tee -a output_log.txt
```

## Logical Volume Management (LVM)

<!-- LVM (Logical Volume Manager) allows for flexible disk space management. It provides a layer of abstraction
     between physical storage devices and filesystems, enabling features like dynamic resizing, snapshots, etc. -->
<!-- CAUTION: LVM operations can lead to data loss if not performed correctly.
     ALWAYS BACK UP IMPORTANT DATA before making changes to your LVM setup.
     Ensure you understand each command and its target device/volume before executing.
     These commands typically require root privileges (`sudo`). -->

### LVM Components Overview
<!--
  Physical Volumes (PV): Actual block devices like disk partitions (e.g., /dev/sda1) or whole disks.
                        These are the building blocks for LVM.
  Volume Groups (VG): A pool of storage created from one or more PVs. Think of it as a virtual disk.
  Logical Volumes (LV): "Virtual partitions" carved out from a VG. Filesystems (like ext4, XFS, BTRFS) are created on LVs.
-->

### Physical Volumes (PV)
<!-- PVs are the underlying physical storage devices (e.g., partitions, whole disks) that LVM uses. -->

#### Displaying Physical Volumes
```bash
sudo pvs # Brief summary of PVs
sudo pvdisplay # Detailed information about PVs
```

#### Creating a Physical Volume
<!-- Initializes a disk or partition for use by LVM. This writes LVM metadata to the device. -->
```bash
# pvcreate /dev/sdXN
# Replace /dev/sdXN with the actual device path (e.g., /dev/sdb1, /dev/sdc).
# Ensure the device is not in use and contains no important data (if using a whole disk or unpartitioned space).
sudo pvcreate /dev/sdXN
```

#### Removing a Physical Volume
<!-- Removes a PV from LVM management. The PV must not be part of any Volume Group,
     or if it is, all data (extents) must first be moved off it to other PVs in the VG (`pvmove`). -->
```bash
# First, if the PV is part of a VG, move data off it (if other PVs have space):
# sudo pvmove /dev/sdXN
# Then, remove it from the VG:
# sudo vgreduce your_vg_name /dev/sdXN
# Finally, remove LVM metadata from the PV:
sudo pvremove /dev/sdXN
```

### Volume Groups (VG)
<!-- VGs are pools of storage created from one or more PVs. LVs are then carved out of VGs. -->

#### Displaying Volume Groups
```bash
sudo vgs # Brief summary of VGs
sudo vgdisplay # Detailed information about VGs
```

#### Creating a Volume Group
<!-- Creates a new VG from one or more initialized PVs. -->
```bash
# vgcreate your_vg_name /dev/sdXN1 [/dev/sdYN1 ...]
#   your_vg_name: Your desired name for the Volume Group (e.g., vg_data, system_vg).
#   /dev/sdXN1 /dev/sdYN1: One or more PVs to include in this VG.
sudo vgcreate your_vg_name /dev/sdXN1 /dev/sdYN1
```

#### Adding a Physical Volume to (Extend) a Volume Group
<!-- Extends an existing VG by adding another initialized PV to it, increasing the VG's total capacity. -->
```bash
# vgextend your_vg_name /dev/sdZN1
#   your_vg_name: The name of the existing VG to extend.
#   /dev/sdZN1: The PV to add to the VG.
sudo vgextend your_vg_name /dev/sdZN1
```

#### Removing a Physical Volume from (Reduce) a Volume Group
<!-- Reduces a VG by removing a PV. Data must be moved off the PV first. -->
```bash
# 1. Move data from the PV to other PVs in the same VG:
#    sudo pvmove /dev/sdXN_to_remove
# 2. Remove the PV from the VG:
#    sudo vgreduce your_vg_name /dev/sdXN_to_remove
```

#### Removing an Entire Volume Group
<!-- Removes an empty VG. All LVs within the VG must be removed first.
     Then, the VG can be deactivated and removed. -->
```bash
# 1. Ensure all LVs in the VG are removed (see lvremove).
# 2. Deactivate the VG (optional, but good practice):
#    sudo vgchange -an your_vg_name
# 3. Remove the VG:
sudo vgremove your_vg_name
```

### Logical Volumes (LV)
<!-- LVs are like virtual partitions created within a VG. Filesystems are created on LVs. -->

#### Displaying Logical Volumes
```bash
sudo lvs # Brief summary of LVs
sudo lvdisplay # Detailed information about LVs
```

#### Creating a Logical Volume
<!-- Creates a new LV with a specific size and name from a VG. -->
```bash
# lvcreate -L DesiredSize -n your_lv_name your_vg_name
#   -L DesiredSize: Size of the LV (e.g., 10G for 10 Gigabytes, 500M for Megabytes, 2T for Terabytes).
#                   Can also use lowercase (e.g., 10g).
#   -n your_lv_name: Name for the new LV (e.g., lv_home, data_lv).
#   your_vg_name: The VG from which to allocate space for this LV.
# Example: Create a 10GB LV named 'data_lv' in 'my_vg'.
sudo lvcreate -L 10G -n data_lv your_vg_name

# To use a percentage of the VG's total space (or remaining free space):
# lvcreate -l <percentage>%VG -n your_lv_name your_vg_name (percentage of total VG size)
# lvcreate -l <percentage>%FREE -n your_lv_name your_vg_name (percentage of free space in VG)
# Example: Create an LV using 50% of the free space in 'your_vg_name'.
sudo lvcreate -l 50%FREE -n another_lv your_vg_name
```

#### Resizing a Logical Volume (Extending or Reducing)
<!-- CAUTION: Reducing an LV's size is DANGEROUS. You MUST shrink the filesystem on it *BEFORE* shrinking the LV
     to avoid cutting off part of the filesystem and causing severe data corruption/loss. Extending is generally safer. -->

##### Extending an LV:
```bash
# lvresize -L +AdditionalSize /dev/your_vg_name/your_lv_name
#   -L +AdditionalSize: Amount of space to add to the LV (e.g., +5G to add 5 Gigabytes).
#   Alternatively, set to a new total size: -L NewTotalSize (e.g., -L 20G to make it 20GB total).
#   /dev/your_vg_name/your_lv_name: Path to the LV. Also accessible via /dev/mapper/your_vg_name-your_lv_name.
# Example: Add 5GB to data_lv in my_vg.
sudo lvresize -L +5G /dev/your_vg_name/data_lv
# After extending the LV, you must resize the filesystem on it to use the new space (see Filesystem Resizing section).

# Extending LV to use all remaining free space in its VG:
# lvextend -l +100%FREE /dev/your_vg_name/your_lv_name  (lvextend is an alias for lvresize -L)
sudo lvextend -l +100%FREE /dev/your_vg_name/your_lv_name
# Then resize the filesystem.
```

##### Reducing an LV (DANGEROUS - Shrink filesystem on it FIRST!):
```bash
# 0. CRITICAL PRE-STEP: Unmount the filesystem and shrink it using filesystem-specific tools
#    (e.g., `resize2fs` for ext2/3/4, `btrfs filesystem resize` for BTRFS) to a size SMALLER
#    than the intended new LV size. This step is NOT optional and is CRITICAL to prevent data loss.
#    Example for ext4 (unmounted): sudo resize2fs /dev/your_vg_name/your_lv_name <NewSmallerFSSize>
#    Example for BTRFS (mounted): sudo btrfs filesystem resize <NewSmallerFSSize> /mount/point

# 1. Reduce the LV size:
#    lvresize -L -ReduceSize /dev/your_vg_name/your_lv_name
#      -L -ReduceSize: Amount to reduce by (e.g., -2G to reduce by 2 Gigabytes).
#      Or set to a new, smaller total size: -L NewSmallerTotalLVSize (e.g., -L 8G).
#    Example: Reduce data_lv by 2GB (AFTER the filesystem on it has been shrunk by at least 2GB).
#    sudo lvresize -L -2G /dev/your_vg_name/data_lv
#    Or set data_lv to be 8GB total (after filesystem is shrunk to, say, 7.8G):
#    sudo lvresize -L 8G /dev/your_vg_name/data_lv

# 2. (Optional but Recommended) Remount and verify filesystem.
#    If you reduced the filesystem more than the LV, you might be able to grow it back slightly to fill the new LV size.
```

#### Removing a Logical Volume
<!-- Removes an LV. Ensure any data is backed up or no longer needed. The LV should be unmounted first. -->
```bash
# 1. Unmount the filesystem on the LV:
#    sudo umount /dev/your_vg_name/your_lv_name (or /mount/point)
# 2. Deactivate the LV (optional, but good practice):
#    sudo lvchange -an /dev/your_vg_name/your_lv_name
# 3. Remove the LV:
#    lvremove /dev/your_vg_name/your_lv_name
#    Example: Remove data_lv from my_vg.
sudo lvremove /dev/your_vg_name/data_lv # Will prompt for confirmation. Use -f to force.
```

### Filesystem Resizing on LVM
<!-- After resizing an LV, the filesystem on it must also be resized to use the new space (if extended)
     or to fit within the new smaller LV (if shrunk, this must be done BEFORE shrinking LV). -->

#### Resizing an `ext2/ext3/ext4` Filesystem
<!-- Can usually be done online (while mounted) for extending. Shrinking requires unmounting. -->
```bash
# For EXTENDING (after lvextend/lvresize has increased LV size):
# resize2fs /dev/your_vg_name/your_lv_name
#   This command will grow the ext2/3/4 filesystem to fill the current size of the LV.
#   It can typically be run on a mounted filesystem for extending.
sudo resize2fs /dev/your_vg_name/your_lv_name

# For SHRINKING (MUST be done BEFORE lvreduce/lvresize to shrink LV):
# 1. Unmount the filesystem:
#    sudo umount /dev/your_vg_name/your_lv_name
# 2. Check the filesystem for errors (recommended):
#    sudo e2fsck -f /dev/your_vg_name/your_lv_name
# 3. Shrink the filesystem to a new, smaller size (e.g., 7G):
#    sudo resize2fs /dev/your_vg_name/your_lv_name 7G
#    (Ensure this new size is less than or equal to the target LV size).
# 4. Then, proceed to shrink the LV itself with `lvreduce` or `lvresize -L`.
```

#### Resizing a `BTRFS` Filesystem
<!-- BTRFS can be resized online (while mounted), whether growing or shrinking. -->
```bash
# For EXTENDING BTRFS to fill the (already extended) LV:
# btrfs filesystem resize max /mount/point/of/btrfs_lv
#   max: Tells BTRFS to expand to the maximum available size of the underlying LV.
#   /mount/point/of/btrfs_lv: The path where the BTRFS filesystem is mounted.
sudo btrfs filesystem resize max /mount/point/of/btrfs_lv

# For SHRINKING BTRFS (do this BEFORE shrinking the LV):
# btrfs filesystem resize <new_size> /mount/point/of/btrfs_lv
#   <new_size>: The new target size for the BTRFS filesystem (e.g., 100g, -10g to reduce by 10GB).
# Example: Shrink BTRFS filesystem at /mnt/my_btrfs by 10GB:
# sudo btrfs filesystem resize -10g /mnt/my_btrfs
# Example: Set BTRFS filesystem at /mnt/my_btrfs to 50GB:
# sudo btrfs filesystem resize 50g /mnt/my_btrfs
# After shrinking BTRFS, you can then shrink the underlying LV.

# Defragmenting BTRFS (example from original cheatsheet - not LVM specific but relevant to BTRFS)
# This is not directly an LVM command but often used with BTRFS volumes.
# -r: recursive through subvolumes and snapshots.
# -v: verbose.
# -clzo: compress with LZO during defragmentation (can also use zstd or zlib, or no compression).
# sudo btrfs filesystem defrag -r -v -clzo /mnt/my_btrfs_volume/
```

## Swap Management

<!-- Commands for creating and managing swap space. Swap can be a file on an existing filesystem or a dedicated partition. -->
<!-- These commands typically require root privileges (`sudo`). -->

### Standard Swap File (Unencrypted)
<!-- These steps create a regular swap file on an existing filesystem (e.g., ext4, XFS). -->

#### 1. Create an Empty File for Swap
<!-- `fallocate` is recommended for speed and ensuring space is allocated immediately (not sparse).
     `dd` can also be used but is slower as it writes zeros. -->
```bash
# Using fallocate (preferred):
# Replace '11G' with your desired swap file size (e.g., 4G, 8G, 16G).
# The file 'swapfile' will be created in the root directory ('/'). Choose path as needed.
sudo fallocate -l 11G /swapfile

# Or using dd (slower, writes zeros to preallocate):
# sudo dd if=/dev/zero of=/swapfile bs=1G count=11 status=progress
# (This creates an 11GiB file)
```

#### 2. Set Correct Permissions for the Swap File
<!-- Only root should be able to read/write the swap file for security. -->
```bash
sudo chmod 0600 /swapfile
# Verify permissions (should be -rw-------):
sudo ls -lh /swapfile
```

#### 3. (BTRFS Only) Disable Copy-on-Write (CoW) for Swap File
<!-- If placing the swap file on a BTRFS filesystem, CoW MUST be disabled for that specific file.
     Other filesystems (ext4, XFS) do not need this step. -->
```bash
# This must be done on an EMPTY file before data (like swap signature) is written to it.
# 1. Create an empty file:
#    sudo touch /path/to/btrfs_mountpoint/swapfile
# 2. Disable CoW (NoCOW attribute):
#    sudo chattr +C /path/to/btrfs_mountpoint/swapfile
# 3. Verify 'C' attribute is set:
#    sudo lsattr /path/to/btrfs_mountpoint/swapfile
# 4. Then allocate space using fallocate (as in Step 1 above):
#    sudo fallocate -l 11G /path/to/btrfs_mountpoint/swapfile

# The original commands `truncate -s 0 swapfile` then `chattr +C` then `fallocate` is a valid sequence for BTRFS.
# For non-BTRFS, `chattr +C` is not needed and may not be supported or have any effect.
```

#### 4. Format the File as Swap
<!-- This writes a swap signature to the file, preparing it for use as swap space. -->
```bash
sudo mkswap /swapfile
# (Or /path/to/btrfs_mountpoint/swapfile if on BTRFS)
```

#### 5. Activate the Swap File
<!-- Makes the swap file available to the system for use. -->
```bash
sudo swapon /swapfile
# Verify swap is active and see total swap space:
swapon --show
free -h # Shows memory and swap usage
cat /proc/swaps # Shows active swap areas
```

#### 6. (Optional) Add to `/etc/fstab` for Persistence Across Reboots
<!-- To make the swap file active automatically on boot, add an entry to /etc/fstab. -->
```
# Add this line to /etc/fstab:
# /swapfile    none    swap    sw     0    0

# If your swap file is on BTRFS at /path/to/btrfs_mountpoint/swapfile, the fstab entry would be:
# /path/to/btrfs_mountpoint/swapfile none swap sw,x-systemd.requires-mounts-for=/path/to/btrfs_mountpoint 0 0
# The x-systemd.requires-mounts-for is important for systemd to ensure the BTRFS volume is mounted before activating swap.
```
<!-- After editing fstab, you can test with `sudo swapon -a` (activates all swap in fstab) or reboot. -->

### Encrypted Swap File (using LUKS on a file)
<!-- This section outlines creating a swap file that is encrypted with LUKS (Linux Unified Key Setup). -->
<!-- This enhances security by encrypting data that gets swapped to disk. -->
<!-- This is more complex than standard swap and requires careful execution. -->
<!-- CAUTION: Incorrectly setting up encryption can lead to data loss or system instability. -->
<!-- The original section was marked as a "COMPLETE MESS". This is a revised, more logical approach. -->
<!-- For automated unlocking at boot, using a random keyfile (not a passphrase) is essential. -->

#### 1. Create the Swap File Container
<!-- This file will hold the LUKS encrypted container. Its contents will be encrypted. -->
```bash
# Using fallocate (recommended):
# Replace '11G' with desired size. The file is named 'encrypted_swapfile_container'.
# Store this container file somewhere appropriate, e.g., /var/encrypted_swap_container.
sudo fallocate -l 11G /encrypted_swapfile_container
sudo chmod 0600 /encrypted_swapfile_container # Secure permissions for the container
```

#### 2. Create a Random Keyfile (Recommended for Automated Swap)
<!-- Using a keyfile is more secure and practical for swap than a passphrase, as it needs to be unlocked at boot. -->
```bash
# Create a directory for crypto keys if it doesn't exist, and set permissions
sudo mkdir -p /etc/cryptsetup_keys
sudo chmod 700 /etc/cryptsetup_keys
# Create a 512-byte keyfile
sudo dd if=/dev/urandom of=/etc/cryptsetup_keys/swap.key bs=512 count=1
sudo chmod 0400 /etc/cryptsetup_keys/swap.key # Restrict read access to root
```

#### 3. Set up LUKS Encryption on the Container File using the Keyfile
<!-- This formats the container file as a LUKS encrypted volume. -->
```bash
# sudo cryptsetup luksFormat --type luks1 /encrypted_swapfile_container /etc/cryptsetup_keys/swap.key
#   --type luks1: Explicitly use LUKS1 format. LUKS2 is default on newer systems but LUKS1 is simpler and fine for this.
#                (For LUKS2, some options might differ slightly).
#   --cipher aes-xts-plain64: Cipher specification (AES with XTS mode, common for disk encryption).
#   --key-size 256: Key size in bits (e.g., 256 for AES).
#   --hash sha256: Hash algorithm for PBKDF.
#   --iter-time 2000: Time in milliseconds to spend with PBKDF (key derivation function).
#   --use-random: Use /dev/random for key material (generally good, though /dev/urandom is often sufficient).
#   /encrypted_swapfile_container: Path to the container file.
#   /etc/cryptsetup_keys/swap.key: Path to the keyfile.
sudo cryptsetup luksFormat --type luks1 --cipher aes-xts-plain64 --key-size 256 --hash sha256 --iter-time 2000 \
    /encrypted_swapfile_container /etc/cryptsetup_keys/swap.key
# This command will ask for confirmation as it's a destructive operation on the container file. Type YES.
```

#### 4. Open the LUKS Container (Map it to a Device)
<!-- This decrypts the container using the keyfile and makes it available as a mapped device (e.g., /dev/mapper/cryptswap). -->
```bash
# sudo cryptsetup luksOpen /encrypted_swapfile_container cryptswap_name --key-file /etc/cryptsetup_keys/swap.key
#   luksOpen (or just 'open'): Command to open (decrypt) the LUKS container.
#   /encrypted_swapfile_container: Path to the LUKS encrypted file.
#   cryptswap: The name for the mapped device (will appear as /dev/mapper/cryptswap).
#   --key-file /etc/cryptsetup_keys/swap.key: Specify the keyfile to unlock the container.
#   --allow-discards: (Optional) If your underlying storage for the container is an SSD and supports TRIM/discard.
sudo cryptsetup open /encrypted_swapfile_container cryptswap --key-file /etc/cryptsetup_keys/swap.key --allow-discards
```
<!-- The `losetup` command from the original cheatsheet is not strictly necessary if `cryptsetup` supports using the file directly, which modern versions do.
     If `losetup` were used, it would be: `sudo losetup --find --show /encrypted_swapfile_container` (to get e.g. /dev/loop0),
     then use /dev/loop0 with `cryptsetup luksFormat` and `luksOpen`. Using the file directly with cryptsetup is cleaner. -->

#### 5. Format the Mapped Device as Swap
<!-- Now treat the decrypted device /dev/mapper/cryptswap like any other swap device. -->
```bash
sudo mkswap /dev/mapper/cryptswap
```

#### 6. Activate the Encrypted Swap
```bash
sudo swapon /dev/mapper/cryptswap
# Verify:
swapon --show
free -h
cat /proc/swaps
```

#### 7. Configure for Automatic Activation at Boot
<!-- This is crucial for encrypted swap to work automatically. It involves:
     1. An entry in `/etc/crypttab` to open the LUKS device at boot using the keyfile.
     2. An entry in `/etc/fstab` for the decrypted mapper device. -->

##### `/etc/crypttab` entry:
```
# Add a line like this to /etc/crypttab:
# Format: <name_of_mapped_device> <path_to_LUKS_container> <path_to_keyfile> <options>
#
# cryptswap    /encrypted_swapfile_container    /etc/cryptsetup_keys/swap.key    luks,discard,swap,noearly,timeout=5
#
# Options:
#   luks: Specifies it's a LUKS volume.
#   discard: Propagate discard/TRIM requests (if using SSD and --allow-discards was used with luksOpen).
#   swap: Indicates this is a swap device. Allows optimizations like disabling read-ahead.
#   noearly: (systemd specific) Defers setup until later in boot, might be needed if keyfile is on a separate fs.
#   timeout=5: Wait up to 5 seconds for the device to become available.
# Consult your distribution's documentation for crypttab options (e.g., `man crypttab`).
```

##### `/etc/fstab` entry:
```
# Add a line like this to /etc/fstab for the decrypted swap device:
# /dev/mapper/cryptswap    none    swap    sw     0    0
#
# Some systems might prefer `sw,pri=<priority>` if you have multiple swap devices.
# The `x-systemd.requires=/etc/cryptsetup_keys/swap.key` and `x-systemd.requires=systemd-cryptsetup@cryptswap.service`
# dependencies are usually handled implicitly by systemd when processing crypttab and fstab.
```
<!-- After setting up crypttab and fstab, you'll need to update your initramfs for the changes to take effect at early boot,
     especially if the keyfile or container is on the root filesystem or requires modules not yet loaded.
     Command depends on distribution (e.g., `sudo update-initramfs -u -k all` on Debian/Ubuntu, `sudo mkinitcpio -P` on Arch).
     Then reboot to test. -->

## System Configuration (Continued)

### Kernel Module Management

#### Blacklisting the PC Speaker (`pcspkr`)
<!-- Prevents the `pcspkr` kernel module from loading, disabling the internal PC speaker beeps. -->
<!-- This is a common tweak to silence annoying system beeps on some motherboards. -->

##### 1. Create a Modprobe Configuration File
<!-- Kernel modules can be blacklisted by creating a .conf file in /etc/modprobe.d/. -->
```bash
# Create or edit a .conf file (e.g., /etc/modprobe.d/nopcspkr.conf or /etc/modprobe.d/blacklist.conf).
# Add the following line to the file:
# blacklist pcspkr

# Example of creating the file and adding the line directly:
echo "blacklist pcspkr" | sudo tee /etc/modprobe.d/disable-pcspkr.conf > /dev/null
```

##### 2. Apply Changes
<!-- Blacklist changes typically take effect after rebuilding the initramfs (if the module could be loaded early) and rebooting.
     Manually unloading the module (`rmmod pcspkr`) can make the change immediate if the module is already loaded and not in use. -->
```bash
# 1. Try to unload the module if currently loaded:
sudo rmmod pcspkr  # This might fail if the module is considered in use or not loaded.

# 2. Rebuild initramfs (distribution-specific, ensures blacklist is effective at early boot):
#    Debian/Ubuntu:
#    sudo update-initramfs -u
#    Arch Linux:
#    sudo mkinitcpio -P
#    Fedora/RHEL/CentOS:
#    sudo dracut -f -v
#    (Consult your distribution's documentation if unsure)

# 3. Reboot the system for the change to be fully effective.
# For many systems, simply creating the blacklist file and rebooting is sufficient if the module is not critical for boot.
# The original note about `sysctl -p` or `sysctl --system` is generally for applying sysctl settings (kernel parameters at /etc/sysctl.conf), not for modprobe blacklist changes.
```

### Temporary Filesystem (`tmpfs`) Management

<!-- `tmpfs` is a filesystem that keeps all files in virtual memory (RAM and possibly swap space).
     It's very fast but its contents are non-persistent (lost on reboot). -->

#### Resizing a User's `tmpfs` Mount (e.g., `/run/user/$(id -u)`)
<!-- `/run/user/<UID>` is often used for user-specific runtime files and is typically mounted as tmpfs by systemd. -->
<!-- This command remounts an existing tmpfs mount with a new size limit. -->
```bash
# mount -o remount,size=NewSize /path/to/tmpfs
#   -o remount,size=NewSize: Remount the existing tmpfs with a new size (e.g., 2.5G for 2.5 Gigabytes).
#                            The size can be specified in bytes, or with suffixes k, M, G.
#   /run/user/$(id -u): Dynamically gets the current user's runtime directory path.
#     $(id -u) command substitution gets the current user's ID (e.g., 1000).
# Example: Resize the current user's /run/user/<UID> tmpfs to 2.5 Gigabytes.
sudo mount -o remount,size=2.5G /run/user/$(id -u)
# Verify the change:
df -h /run/user/$(id -u)
```
<!-- This change is temporary and will revert on the next reboot unless configured persistently.
     Persistent configuration usually involves systemd unit overrides (e.g., for `user-runtime-dir@.service`)
     or modifying how the tmpfs is mounted initially (less common for /run/user/<UID>). -->

#### Setting XDG Base Directories (Redirect Cache to `tmpfs`)
<!-- The XDG Base Directory Specification defines standard locations for user-specific files (cache, config, data).
     This snippet redirects the XDG Cache Home (`XDG_CACHE_HOME`) to a directory in /tmp, which is often a tmpfs.
     This can speed up applications that heavily use cache and reduce wear on SSDs, at the cost of cache being non-persistent. -->

##### Create a Profile Script to Set Environment Variable
<!-- Create a shell script in /etc/profile.d/ to set the `XDG_CACHE_HOME` environment variable for all users upon login. -->
<!-- Example file: /etc/profile.d/xdg_cache_home_to_tmpfs.sh -->
```bash
# Content for /etc/profile.d/xdg_cache_home_to_tmpfs.sh:

# Check if USER is set (ensures it runs in a user login context)
if [ -n "${USER}" ]; then
  # Define the new cache home directory under /tmp
  USER_CACHE_DIR="/tmp/${USER}/.cache"
  export XDG_CACHE_HOME="${USER_CACHE_DIR}"

  # Optional: Create the directory if it doesn't exist.
  # Applications are generally expected to create their cache subdirectories within XDG_CACHE_HOME.
  # Creating the base user cache path itself can be helpful, but ensure proper permissions.
  # mkdir -p "${XDG_CACHE_HOME}"
  # chmod 0700 "${XDG_CACHE_HOME}" # Ensure only user can access
fi
```
<!-- After creating this file (e.g., `sudo nano /etc/profile.d/xdg_cache_home_to_tmpfs.sh`),
     users will need to log out and log back in for the change to take effect.
     Ensure /tmp is actually a tmpfs mount. Check with: `findmnt -T /tmp` or `mount | grep /tmp`.
     If /tmp is not tmpfs, this redirection won't use RAM primarily. -->

### Font Management

#### Installing New Fonts System-Wide
<!-- This example shows how to install new fonts (e.g., TrueType .ttf, OpenType .otf) for all users on the system. -->
```bash
# 1. Create a directory for your new custom fonts (if it doesn't exist).
#    Common locations: /usr/share/fonts/truetype/, /usr/share/fonts/opentype/,
#                      /usr/local/share/fonts/ (recommended for manually installed fonts to separate from package manager fonts).
#    The example uses '/usr/local/share/fonts/my_custom_fonts'.
sudo mkdir -p /usr/local/share/fonts/my_custom_fonts

# 2. Copy or Unzip your font files into this directory.
#    Replace 'font_archive.zip' with your font file or copy individual .ttf/.otf files.
#    Example: sudo unzip font_archive.zip -d /usr/local/share/fonts/my_custom_fonts/
#    Or copy individual font files:
#    sudo cp your_font_file.ttf /usr/local/share/fonts/my_custom_fonts/
#    sudo cp another_font.otf /usr/local/share/fonts/my_custom_fonts/

# 3. Set appropriate permissions for the font files (readable by all).
sudo chmod 644 /usr/local/share/fonts/my_custom_fonts/*

# 4. Rebuild the font cache.
#    This makes the system aware of the newly installed fonts.
#    fc-cache -f -v:
#      -f: Force rebuild (scan all font directories, not just new ones).
#      -v: Verbose output (shows what it's doing).
sudo fc-cache -f -v
```
<!-- Fonts should now be available to applications. You might need to restart some applications to see the new fonts. -->

### X11 Configuration (Touchpad - libinput)

#### Enabling Tap-to-Click for Touchpad (Xorg using libinput)
<!-- This example creates an Xorg configuration snippet to enable tap-to-click for touchpads handled by the libinput driver. -->
<!-- This method is generally applicable for modern Linux systems using Xorg server with libinput. -->

##### 1. Create Xorg Configuration Directory (if it doesn't exist)
<!-- Custom Xorg configuration snippets are typically placed in /etc/X11/xorg.conf.d/. -->
```bash
sudo mkdir -p /etc/X11/xorg.conf.d/
```

##### 2. Create Touchpad Configuration File
<!-- Example filename: /etc/X11/xorg.conf.d/30-touchpad.conf (number prefix influences loading order). -->
<!-- The content specifies matching criteria for the touchpad and enables the "Tapping" option. -->
```text
# Content for /etc/X11/xorg.conf.d/30-touchpad.conf:

Section "InputClass"
    Identifier "libinput_touchpad_enable_tap_to_click" # Descriptive identifier for this rule
    Driver "libinput"                                 # Specify that this applies to devices using the libinput Xorg driver
    MatchIsTouchpad "on"                              # Apply this class to devices recognized as touchpads

    # Optional: Match a specific touchpad if you have multiple pointing devices or specific needs.
    # Use `libinput list-devices` to find your touchpad's name or product ID.
    # Then uncomment and adapt one of the MatchProduct or MatchDevicePath lines.
    # MatchProduct "SynPS/2 Synaptics TouchPad" # Example product name string
    # MatchProduct "MSFT0001:00 04F3:31BE Touchpad" # Example Product ID from original
    # MatchDevicePath "/dev/input/event*" # Can also match by device path

    Option "Tapping" "on"                             # Enable tap-to-click functionality

    # Optional: Configure button mapping for tapping
    # "lrm" = 1-finger tap is Left click, 2-finger tap is Right click, 3-finger tap is Middle click.
    # Option "TappingButtonMap" "lrm"

    # Optional: Enable natural (reverse) scrolling
    # Option "NaturalScrolling" "true"

    # Optional: Disable touchpad while typing
    # Option "DisableWhileTyping" "true"
EndSection
```
<!-- You will need to restart your X server (usually by logging out and logging back in, or rebooting)
     for these changes to take effect. -->
<!-- The `MatchIsTouchpad "on"` is usually sufficient to apply to all touchpads.
     Use `MatchProduct` or other `Match*` directives if you need to target a specific device. -->

### SMTP Server Hardening (Postfix Example)

#### Disable VRFY Command in Postfix
<!-- The VRFY command in SMTP can be used to verify the existence of email addresses on the server,
     potentially leaking user information to spammers or attackers. Disabling it is a common hardening step. -->
```bash
# postconf -e <parameter>=<value>: Edits Postfix configuration files (main.cf by default).
#   -e: Edit main.cf.
#   disable_vrfy_command=yes: Sets the parameter to disable the VRFY command.
sudo postconf -e disable_vrfy_command=yes

# To make the change take effect, Postfix needs to reload its configuration:
sudo systemctl reload postfix
# Or, if systemctl is not used:
# sudo postfix reload
```

## Image Manipulation (`imagemagick`)

<!-- ImageMagick is a powerful suite of command-line tools for image manipulation.
     Ensure ImageMagick is installed: `sudo apt-get install imagemagick` or `sudo yum install ImageMagick`. -->

### Batch Convert JPG/JPEG to PNG and Remove Originals
<!-- This script finds all .jpg and .jpeg files in the current directory and its subdirectories,
     converts them to .png format using ImageMagick's `convert` command,
     and then (if conversion is successful) deletes the original .jpg/.jpeg files. -->
<!-- CAUTION: This script DELETES original files. Ensure you want this behavior or back them up first.
     Test thoroughly on a sample directory before running on important images. -->
```bash
# find . -type f \( -iname "*.jpg" -o -iname "*.jpeg" \) -exec bash -c '...' bash {} \;
#   find . -type f \( -iname "*.jpg" -o -iname "*.jpeg" \):
#     Finds regular files (-type f) with case-insensitive names (-iname) ending in ".jpg" OR ".jpeg",
#     starting from the current directory (.).
#   -exec bash -c '...script...' bash {} \;: Executes a short bash script for each found file.
#     '...' : The script itself, executed by a new bash instance.
#     bash: Explicitly invoke bash for the script.
#     {}: Placeholder for the found file path, passed as the first argument ($1) to the inline script.
#     \;: Terminates the -exec command for each file.

find . -type f \( -iname "*.jpg" -o -iname "*.jpeg" \) -exec bash -c '
    file="$1"  # Assign the filename passed by find to a variable.
    # Construct the output filename by replacing the extension with .png.
    # %.jpg removes .jpg suffix, %.jpeg removes .jpeg. This handles both.
    # Using parameter expansion: ${parameter%word}
    base_filename="${file%.*}" # Removes the last extension (e.g., .jpg or .jpeg)
    output_file="${base_filename}.png"

    echo "Converting \"$file\" to \"$output_file\"..."
    # convert: ImageMagick command for image conversion and processing.
    #   -verbose: Print more information about the conversion.
    #   -monitor: Show progress indicators for complex operations (may not be very visible for simple converts).
    #   "$file": Input image file.
    #   "$output_file": Output image file.
    if convert -verbose "$file" "$output_file"; then
        echo "Conversion successful, removing original: \"$file\""
        rm "$file" # Remove the original file only if convert was successful (exited with 0).
    else
        echo "ERROR: Conversion failed for \"$file\". Original not removed."
    fi
' bash {} \;

# Note: The original script had separate find and loop, then another find to delete.
# This combined version is more robust as it only deletes if `convert` succeeds for that specific file.
# For very large numbers of files, consider adding error handling or logging within the bash -c script.
```

### Listing Images with Specific Dimensions (e.g., 1920x1080)
<!-- This uses ImageMagick's `identify` command to get image dimensions and `awk` to filter for specific sizes. -->

#### 1. Identify and Filter Images by Dimensions
<!-- This command processes all PNG files in the current directory, printing the width, height, and filename
     for those that match the specified dimensions (e.g., 1920x1080). -->
```bash
# identify -format "%w %h %i\n" *.png:
#   identify: ImageMagick command to describe the format and characteristics of image(s).
#   -format "%w %h %i\n": Specifies the output format string.
#     %w: Width of the image.
#     %h: Height of the image.
#     %i: Input filename.
#     \n: Newline character.
#   *.png: Process all PNG files in the current directory. Change to other extensions as needed (e.g., *.jpg, *.* for all).
# awk '$1 == 1920 && $2 == 1080 {print $0}':
#   awk: Pattern scanning and processing language.
#   $1 == 1920 && $2 == 1080: Condition. Checks if the first field (width) is 1920 AND the second field (height) is 1080.
#   {print $0}: Action. If the condition is true, print the entire input line ($0).
identify -format "%w %h %i\n" *.png | awk '$1 == 1920 && $2 == 1080 {print $0}'

# Example Output:
# 1920 1080 image1.png
# 1920 1080 image5.png
```

#### 2. Creating an Alias for Convenience (Optional)
<!-- An alias can make it easier to run this command frequently. Add this to your shell's configuration file
     (e.g., ~/.bashrc for bash, ~/.zshrc for zsh). -->
```bash
# Example alias named 'img_is_hd':
# alias img_is_hd='identify -format "%w %h %i\n" *.png | awk '\''$1 == 1920 && $2 == 1080 {print $3}'\'''
# Note the escaping of single quotes within the alias definition: '\'' is used to embed a single quote inside a single-quoted string.
# This alias variant prints only the filename ($3) if it matches.
# After adding the alias, source your config file (e.g., `source ~/.bashrc`) or open a new terminal.
# You can then just run `img_is_hd` in a directory with PNG files.
```

#### 3. Using the Output (e.g., Moving Matching Files)
<!-- The original example had `&& mv-files` which is not a standard command.
     Here's how you could use the output to move the identified files to a subdirectory. -->
```bash
# Create a directory for the files:
mkdir -p ./1920x1080_images

# Run the command to get filenames of 1920x1080 PNGs and then move them using xargs:
identify -format "%w %h %i\n" *.png | awk '$1 == 1920 && $2 == 1080 {print $3}' | xargs -I {} mv -v "{}" ./1920x1080_images/
#   awk '{print $3}': Prints only the third field (the filename) from the identify output if dimensions match.
#   xargs -I {}: For each line of input (filename), execute the `mv` command.
#     -I {}: Replace occurrences of {} with the input line.
#     mv -v "{}" ./1920x1080_images/: Move the file (verbose) to the target directory. Quotes around "{}" handle filenames with spaces.
```

## Development & Compilation

### Compiling C Programs with Clang (Example Flags)
<!-- Example command for compiling a C source file (`your_source_file.c`) into an executable (`your_output_binary`)
     using Clang with various optimization and linking flags. -->
<!-- These flags are often specific to the target architecture, desired optimizations, and available libraries. -->
```bash
# clang -O2 -march=znver2 -pipe -flto=thin -fuse-ld=lld -rtlib=compiler-rt -unwindlib=libunwind \
#   -Wl,--as-needed -Wl,-plugin-opt=O3 -Wl,--lto-O3 \
#   your_source_file.c -o your_output_binary
#
# Explanation of flags (some are advanced/specific):
#   clang: The C/C++/Objective-C compiler (LLVM based).
#   -O2: Optimization level 2. A good balance between optimization effort and compile time/code size.
#        Other levels include -O0 (no optimization), -O1, -O3, -Os (optimize for size), -Ofast (aggressive, may break standards).
#   -march=znver2: Target architecture. 'znver2' is AMD Zen 2.
#                  Replace with your specific target CPU architecture (e.g., skylake, armv8-a)
#                  or use '-march=native' to optimize for the host CPU where compilation occurs.
#   -pipe: Use pipes instead of temporary files between compilation stages (can speed up compilation on some systems).
#   -flto=thin: Enable ThinLTO (Link Time Optimization). ThinLTO offers many benefits of LTO (better cross-module optimization)
#               with significantly lower build times compared to full LTO. `-flto` enables full LTO.
#   -fuse-ld=lld: Use LLD (LLVM's linker) instead of the system default linker (often GNU ld).
#                 LLD is generally faster, especially for LTO.
#   -rtlib=compiler-rt: Use compiler-rt as the runtime library (provides low-level functions like those for sanitizers, profiling, builtins).
#                       Alternatives: libgcc (for GCC compatibility).
#   -unwindlib=libunwind: Use libunwind for stack unwinding (e.t., for exceptions, stack traces).
#                         Alternatives: libgcc (on systems using GCC's unwinder).
#   -Wl,...: Pass comma-separated options to the linker.
#     --as-needed: Link against shared libraries only if they are actually used by the object files being linked.
#                  Can reduce dependencies and executable size.
#     -plugin-opt=O3: Linker plugin optimization level (for LTO, specific to some linkers like gold or lld).
#     --lto-O3: LTO optimization level for the linker itself (again, specific to linker and LTO type).
#
#   your_source_file.c: Your C source code file.
#   -o your_output_binary: Specifies the name of the output executable file.

# General Example (more portable, relying on Clang's defaults for linker and runtime libs):
clang -O2 -march=native -pipe -flto=thin \
    your_source_file.c -o your_output_binary \
    -Wl,--as-needed # Example linker flag
```

## Shell Tips and Tricks (Continued)

### Using a List of Files for `mv` or other commands

<!-- How to use a list of filenames (e.g., from a text file) as arguments to a command like `mv`. -->

#### Method 1: Command Substitution (Potentially Unsafe)
<!-- This method uses `$(cat files_to_move.txt)` to substitute the contents of the file directly onto the command line. -->
<!-- CAUTION: This is UNSAFE if filenames in 'files_to_move.txt' contain spaces, newlines, or shell glob characters (*, ?, etc.).
     Filenames will be split by whitespace, and glob characters might expand. -->
```bash
# Assuming 'files_to_move.txt' contains one filename per line.
# And 'destination_path/' is the target directory.
# mv $(cat files_to_move.txt) destination_path/  # <<<<<<< AVOID THIS FOR GENERAL USE
```

#### Method 2: Using `xargs` (Safer and Recommended)
<!-- `xargs` builds and executes command lines from standard input. It handles filenames with spaces and special characters
     more robustly, especially when combined with null-terminated input (e.g., from `find ... -print0`). -->
```bash
# If 'files_to_move.txt' has one filename per line:
# xargs -a files_to_move.txt -I {} mv -v "{}" destination_path/
# Explanation:
#   xargs -a files_to_move.txt: Read arguments from 'files_to_move.txt' instead of standard input.
#   -I {}: Replace occurrences of {} (or any chosen string) with the argument read from the file.
#          Each line from the file becomes a separate argument for a `mv` command.
#   mv -v "{}" destination_path/: The command to execute.
#     -v: Verbose, show what `mv` is doing.
#     "{}": Ensures the filename is treated as a single argument, even with spaces.
#     destination_path/: The target directory for the move.

# Example: Create a file list (e.g., all .tmp files in current dir) and then move them.
find . -maxdepth 1 -type f -name "*.tmp" > files_to_move.txt
# Check if the file list is not empty before running xargs:
if [ -s files_to_move.txt ]; then
  xargs -a files_to_move.txt -I {} mv -v "{}" ./archive_directory/
  # rm files_to_move.txt # Optional: remove the list file afterwards
else
  echo "No files to move."
  rm files_to_move.txt # Remove empty list file
fi

# Even safer with null-terminated filenames (e.g., from find -print0):
# find . -maxdepth 1 -type f -name "*.tmp" -print0 > files_to_move.0
# xargs -0 -a files_to_move.0 -I {} mv -v "{}" ./archive_directory/
#   -print0: `find` prints filenames separated by a null character.
#   -0: `xargs` reads null-terminated input. This is the most robust way for arbitrary filenames.
```

### `awk` / `sed` / `cut` Snippets (Illustrative Examples)

<!-- The original note "incomplete example, ignore it if you don't know what you doing" is important. -->
<!-- These are decontextualized examples. Their utility depends heavily on the specific format of the input data. -->
<!-- Always try to understand the structure of your input before constructing `awk`, `sed`, or `cut` commands. -->

#### `awk` Examples
<!-- `awk` is a powerful text processing tool that processes data line by line, field by field. -->
```bash
# Example 1: Extract text after a specific delimiter.
# Input: "Only in SFW: 121212.png"
# Goal: Extract "121212.png"
# -F ': ': Sets the field separator to ": " (a colon followed by a space).
# {print $2}: Prints the second field based on this delimiter.
echo "Only in SFW: 121212.png" | awk -F': ' '{print $2}'
# Output: 121212.png

# Example 2: Print a specific field if a line matches a pattern (default space/tab delimiter).
# Input: "User: gentoober ID: 1000 Home: /home/gentoober"
# Goal: Print the User ID (1000) if the line contains "gentoober".
# /gentoober/: This is a pattern. If the line contains "gentoober", the action is performed.
# {print $4}: Prints the fourth field (fields are space-separated by default).
echo "User: gentoober ID: 1000 Home: /home/gentoober" | awk '/gentoober/ {print $4}'
# Output: 1000
```

#### `sed` Example
<!-- `sed` (Stream EDitor) is primarily for search-and-replace operations on text streams. -->
```bash
# Example: Extract and reformat parts of a line using capturing groups.
# Input: "Image Details: filename=photo_01.jpg resolution=1920x1080"
# Goal: Extract "photo_01.jpg (1920x1080)"
# sed -n 's/.*filename=\([^ ]*\) .*resolution=\([^ ]*\).*/\1 (\2)/p'
#   -n: Suppress automatic printing of each line.
#   's/pattern/replacement/p': Substitute command.
#     .*filename=\([^ ]*\): Matches any characters up to "filename=", then captures non-space characters (\([^ ]*\)) into \1.
#     .*resolution=\([^ ]*\): Matches up to "resolution=", then captures non-space characters into \2.
#     .*: Matches any remaining characters on the line.
#     \1 (\2): Replacement string: the first capture, a space, an opening parenthesis, the second capture, a closing parenthesis.
#     p: Print the line if a substitution was made.
echo "Image Details: filename=photo_01.jpg resolution=1920x1080" | \
  sed -n 's/.*filename=\([^ ]*\) .*resolution=\([^ ]*\).*/\1 (\2)/p'
# Output: photo_01.jpg (1920x1080)
```
<!-- These examples are highly specific. For general use, understand your input data structure and use online tools
     or `man` pages to construct appropriate `awk`, `sed`, or `cut` commands. -->

## Backup Scripts and Snippets

<!-- This section contains more complete scripts for backups. -->
<!-- CAUTION: Always test backup scripts thoroughly. Ensure restores work as expected before relying on them.
     Adapt paths and exclusion lists to your specific needs. -->

### Home Directory Compressed Backup (using `tar` and various compressors)
<!-- The original section was marked "MESSED UP!". These are revised versions providing a more structured approach. -->
<!-- These scripts back up a user's home directory, excluding common cache/temporary files. -->

#### General Structure for Home Backup Script
```bash
#!/bin/bash

# --- Configuration ---
# Source directory to back up (e.g., /home/your_user, or use ${HOME} for current user)
BACKUP_SOURCE_DIR="${HOME}"
# Destination directory for the backup file (ensure this is on a different disk/partition)
BACKUP_DEST_DIR="/mnt/backups/home/" # Example: /mnt/external_drive/backups/
# Timestamp for unique backup filename
DATE_FORMAT=$(date "+%Y-%m-%d_%H-%M-%S")
# Current user (used in filename if BACKUP_SOURCE_DIR is generic like ${HOME})
CURRENT_USER=$(whoami)
# Base name for the archive
ARCHIVE_BASENAME="home_backup_${CURRENT_USER}_${DATE_FORMAT}"

# Exclude patterns for tar. Add more as needed.
# Each pattern should be a separate element in the array.
# Paths are relative to the source directory if not starting with '/'.
EXCLUDE_PATTERNS=(
    "--exclude=${BACKUP_SOURCE_DIR}/.cache/*"
    "--exclude=${BACKUP_SOURCE_DIR}/.dbus"
    "--exclude=${BACKUP_SOURCE_DIR}/.gvfs"
    "--exclude=${BACKUP_SOURCE_DIR}/.local/share/Trash/*"
    "--exclude=${BACKUP_SOURCE_DIR}/Downloads/*" # Example: exclude all Downloads
    "--exclude=*.tmp"                          # Example: exclude all .tmp files
    "--exclude=*~"                             # Example: exclude editor backup files
    "--exclude=node_modules"                   # Example: exclude project dependencies
    "--exclude=lost+found"
    # Add specific large directories you might want to exclude:
    # "--exclude=${BACKUP_SOURCE_DIR}/VirtualBox VMs"
    # "--exclude=${BACKUP_SOURCE_DIR}/Videos/LargeArchive"
)

# Ensure backup destination directory exists
mkdir -p "${BACKUP_DEST_DIR}"
if [ ! -d "${BACKUP_DEST_DIR}" ]; then
    echo "Error: Backup destination directory '${BACKUP_DEST_DIR}' could not be created."
    exit 1
fi

# --- Choose Compression Method ---
# Uncomment one of the following sections (zstd, xz, or lz4)

# Method 1: Using zstd (Good balance of speed and compression)
COMPRESSOR_CMD="zstd -v -T0 -10" # Level 10, all threads, verbose
ARCHIVE_EXT=".tar.zst"
# ---

# Method 2: Using xz (High compression, slower)
# COMPRESSOR_CMD="xz -T0 -9 -v" # Level 9, all threads, verbose
# ARCHIVE_EXT=".tar.xz"
# ---

# Method 3: Using lz4 (Very fast, lower compression)
# COMPRESSOR_CMD="lz4 -v --best" # Best lz4 compression (still fast), verbose
# ARCHIVE_EXT=".tar.lz4"
# ---

# Check if a compressor command is chosen
if [ -z "${COMPRESSOR_CMD}" ]; then
    echo "Error: No compression method selected. Please uncomment one in the script."
    exit 1
fi

FULL_ARCHIVE_PATH="${BACKUP_DEST_DIR}/${ARCHIVE_BASENAME}${ARCHIVE_EXT}"

echo "Starting backup of '${BACKUP_SOURCE_DIR}'"
echo "Destination: '${FULL_ARCHIVE_PATH}'"
echo "Excluding: ${EXCLUDE_PATTERNS[*]}"
echo "Press Ctrl+C to cancel within 5 seconds..."
sleep 5

# Enable globstar if your tar version or exclude patterns need it (e.g. for **).
# Not strictly needed for the example patterns above if paths are well-defined.
# shopt -s globstar

# tar options:
#   c: create a new archive.
#   v: verbosely list files processed (can make logs very large, consider removing for cron jobs).
#   p: preserve permissions (same as --preserve-permissions).
#   f -: use archive file or device ARCHIVE (here, '-' means stdout to pipe to compressor).
#   --xattrs: enable extended attributes support.
#   --acls: enable ACLs support (if your system uses them).
#   --one-file-system: Do not cross filesystem boundaries (e.g., if /home/user/external is a mount point).
#   "${EXCLUDE_PATTERNS[@]}": Pass exclude patterns array to tar.
#   "${BACKUP_SOURCE_DIR}": The directory to back up.
echo "Running: tar -cvpf - --xattrs --acls --one-file-system ${EXCLUDE_PATTERNS[*]} \"${BACKUP_SOURCE_DIR}\" | ${COMPRESSOR_CMD} - > \"${FULL_ARCHIVE_PATH}\""

tar -cpvf - \
    --xattrs \
    --acls \
    --one-file-system \
    "${EXCLUDE_PATTERNS[@]}" \
    "${BACKUP_SOURCE_DIR}" | ${COMPRESSOR_CMD} - > "${FULL_ARCHIVE_PATH}"

# Check exit status of tar and compressor (Bash specific for pipes)
tar_exit_status=${PIPESTATUS[0]}
compressor_exit_status=${PIPESTATUS[1]}

if [ ${tar_exit_status} -eq 0 ] && [ ${compressor_exit_status} -eq 0 ]; then
    echo "Backup completed successfully: ${FULL_ARCHIVE_PATH}"
    ls -lh "${FULL_ARCHIVE_PATH}"
else
    echo "Backup FAILED!"
    echo "  tar exit status: ${tar_exit_status}"
    echo "  compressor exit status: ${compressor_exit_status}"
    # Consider removing partial archive if failed
    # rm -f "${FULL_ARCHIVE_PATH}"
fi

# shopt -u globstar # Disable globstar if it was enabled
```

#### Decompressing Backups (General Examples)
```bash
# Adjust archive_name.tar.xxx and restore location as needed.
# -C /path/to/restore_location/: Change directory before extracting.

# For .tar.zst:
# zstd -d -c archive_name.tar.zst | tar -xpvf - -C /path/to/restore_location/
#   -x: extract
#   -p: preserve permissions
#   -v: verbose
#   -f -: read from stdin

# For .tar.xz:
# xz -d -c archive_name.tar.xz | tar -xpvf - -C /path/to/restore_location/

# For .tar.lz4:
# lz4 -d -c archive_name.tar.lz4 | tar -xpvf - -C /path/to/restore_location/

# Original example for lz4 decompression and tar extraction:
# lz4 -d rootfs_backup-*.tar.lz4 # This decompresses to rootfs_backup-*.tar
# tar --xattrs --xattrs-include='*.*' --numeric-owner -xpvf home_gentoober_backup-*.tar -C /
#   --numeric-owner: Important if restoring to a different system or as a different user to preserve original UIDs/GIDs.
#   --xattrs-include='*.*': Generally covered by just --xattrs in modern tar.
```

### Gentoo Stage 4 Backup Script (`mkstage4`)
<!-- This is a script for creating a "Stage 4" tarball in Gentoo, which is essentially a full system backup.
     It's designed to be run from a working Gentoo system to back itself up.
     The original script uses lz4 for compression. This version is enhanced for clarity and options. -->
```bash
#!/bin/bash

# Gentoo Stage 4 Backup Script (mkstage4.sh)
# Creates a compressed tarball of the root filesystem, excluding pseudo-filesystems.

# --- Configuration ---
BACKUP_TIMESTAMP=$(date +"%Y-%m-%d_%H%M%S")
# Destination for the backup file. Ensure this is on a SEPARATE, MOUNTED filesystem.
BACKUP_DESTINATION_DIR="/mnt/backups/gentoo/"
STAGE4_HOSTNAME=$(hostname -s) # Get short hostname
STAGE4_FILENAME="stage4-${STAGE4_HOSTNAME}-${BACKUP_TIMESTAMP}.tar.lz4" # Default to lz4
LOG_FILE="${BACKUP_DESTINATION_DIR}/mkstage4_log_${BACKUP_TIMESTAMP}.txt"

# Compression: Choose one by uncommenting. LZ4 is fast, ZSTD good balance, XZ high compression.
# COMPRESSION_CMD="lz4 -vz --fast" && STAGE4_FILENAME="stage4-${STAGE4_HOSTNAME}-${BACKUP_TIMESTAMP}.tar.lz4"
COMPRESSION_CMD="zstd -v -T0 -5" && STAGE4_FILENAME="stage4-${STAGE4_HOSTNAME}-${BACKUP_TIMESTAMP}.tar.zst" # Level 5
# COMPRESSION_CMD="xz -T0 -6 -v" && STAGE4_FILENAME="stage4-${STAGE4_HOSTNAME}-${BACKUP_TIMESTAMP}.tar.xz" # Level 6

# Exclude paths/patterns for the backup. These are relative to the backup source ('/').
# CRITICAL: Ensure /dev, /proc, /sys, /run are excluded. Also exclude the backup destination itself.
EXCLUDE_PATHS=(
    "--exclude=/dev/*"
    "--exclude=/proc/*"
    "--exclude=/sys/*"
    "--exclude=/tmp/*"
    "--exclude=/run/*"
    "--exclude=/mnt/*"          # Exclude common mount points for external media/other filesystems.
    "--exclude=/media/*"        # Common mount point for removable media.
    "--exclude=/home/*/.cache/*" # User-specific cache files.
    "--exclude=/var/cache/distfiles/*" # Gentoo distfiles cache (can be huge)
    "--exclude=/var/cache/binpkgs/*"   # Gentoo binary package cache
    "--exclude=/var/tmp/*"      # Temporary files in /var.
    "--exclude=/usr/src/linux*/.config*" # Kernel .config files (often symlinks, can be backed up separately)
    "--exclude=/usr/portage/*" # Can be synced separately, very large
    # CRUCIALLY, exclude the backup destination directory itself if it's under /mnt or similar:
    "--exclude=${BACKUP_DESTINATION_DIR}/*"
    "--exclude=/lost+found"
)

# Tar options
TAR_OPTIONS=(
    "--acls"            # Preserve ACLs
    "--xattrs"          # Preserve extended attributes
    "-cpvf"             # c: create, p: preserve permissions, v: verbose, f -: output to stdout
    "--one-file-system" # Do not cross filesystem boundaries from '/'
    # --numeric-owner  # Optional: use if restoring to a system with different UIDs/GIDs
)

# Color codes for output
GREEN="\033[1;32m"
YELLOW="\033[1;33m"
RED="\033[1;31m"
NC="\033[0m" # No Color

# --- Script Logic ---
# Check if running as root
if [[ "$(id -u)" -ne 0 ]]; then
    echo -e "${RED}This script must be run as root.${NC}" >&2
    exit 1
fi

# Check if backup destination exists and is writable
if [[ ! -d "${BACKUP_DESTINATION_DIR}" ]] || [[ ! -w "${BACKUP_DESTINATION_DIR}" ]]; then
    echo -e "${YELLOW}Backup destination '${BACKUP_DESTINATION_DIR}' does not exist or is not writable.${NC}" >&2
    echo -e "${YELLOW}Please create it and ensure it's writable, or adjust BACKUP_DESTINATION_DIR.${NC}" >&2
    # Example: sudo mkdir -p /mnt/backups/gentoo && sudo chown $(whoami) /mnt/backups/gentoo
    exit 1
fi

FULL_ARCHIVE_PATH="${BACKUP_DESTINATION_DIR}/${STAGE4_FILENAME}"

echo -e "${GREEN}Starting Stage 4 backup process...${NC}"
echo "Backup Source: / (root filesystem)"
echo "Backup Destination: ${FULL_ARCHIVE_PATH}"
echo "Excluded paths: ${EXCLUDE_PATHS[*]}"
echo "Tar options: ${TAR_OPTIONS[*]}"
echo "Compression command: ${COMPRESSION_CMD}"
echo "Log File: ${LOG_FILE}"
echo -e "${YELLOW}Press Ctrl+C to cancel within 10 seconds...${NC}"
sleep 10

# Start logging
exec > >(tee -a "${LOG_FILE}") 2>&1
echo "Backup started at $(date)"

# Create the tarball, compress it, and save.
# Using ionice and nice to reduce impact on system performance.
echo "Running: sudo ionice -c 3 nice -n 19 tar ${TAR_OPTIONS[*]} ${EXCLUDE_PATHS[*]} / | ${COMPRESSION_CMD} - > \"${FULL_ARCHIVE_PATH}\""

sudo ionice -c 3 nice -n 19 tar "${TAR_OPTIONS[@]}" "${EXCLUDE_PATHS[@]}" / | ${COMPRESSION_CMD} - > "${FULL_ARCHIVE_PATH}"

# Check tar and compressor exit codes (Bash specific for pipes)
tar_exit_status=${PIPESTATUS[0]}
compressor_exit_status=${PIPESTATUS[1]}

if [[ ${tar_exit_status} -eq 0 && ${compressor_exit_status} -eq 0 ]]; then
    echo -e "${GREEN}Stage 4 backup completed successfully!${NC}"
    echo -e "${GREEN}Archive created at: ${FULL_ARCHIVE_PATH}${NC}"
    ls -lh "${FULL_ARCHIVE_PATH}"
    echo "Backup ended at $(date)"
    exit 0
else
    echo -e "${RED}Backup process FAILED!${NC}" >&2
    echo -e "${RED}tar exit status: ${tar_exit_status}${NC}" >&2
    echo -e "${RED}compressor exit status: ${compressor_exit_status}${NC}" >&2
    # Consider removing the partial/failed archive
    # sudo rm -f "${FULL_ARCHIVE_PATH}"
    echo "Backup failed at $(date)"
    exit 1
fi
```

## Networking (Continued)

### Wi-Fi Network Scanning

#### Scan for Available Wi-Fi Networks
<!-- `iwlist` is an older tool; `nmcli` (NetworkManager) or `iw` are more modern alternatives on many systems. -->
```bash
# Using iwlist (older, may require root)
# Replace 'wlan0' with your wireless interface name (e.g., wlp2s0). Find with `ip link` or `iw dev`.
sudo iwlist wlan0 scanning

# Modern alternative with `iw` (may require root)
# sudo iw dev wlan0 scan | less

# Modern alternative with `nmcli` (NetworkManager, usually doesn't require root for scanning)
# nmcli dev wifi list
# nmcli dev wifi list ifname wlan0 # For a specific interface
```

### Network Path Tracing

#### Trace Route to a Host (Numeric Output)
<!-- Shows the path (sequence of routers) packets take to reach a network host. -->
```bash
# traceroute -n <hostname_or_IP>:
#   -n: Do not resolve IP addresses to hostnames (numeric output, avoids DNS lookups).
#   <hostname_or_IP>: The destination host.
traceroute -n example.com

# For ICMP based tracing (like Windows tracert), use:
# sudo traceroute -I example.com
# For TCP based tracing (can sometimes bypass firewalls that block UDP/ICMP):
# sudo traceroute -T -p 80 example.com # Trace using TCP SYN packets to port 80
```

### Routing Table Management
<!-- `route` is an older command; `ip route` is the modern replacement from the iproute2 suite. -->
<!-- These commands usually require root privileges (`sudo`). -->

#### Display Kernel IP Routing Table (Numeric)
```bash
# Using route (older):
sudo route -n
#   -n: Show numerical addresses instead of resolving hostnames.

# Modern alternative with `ip route`:
ip route show
# ip -n route show # For numeric output (though `ip route` often shows IPs by default)
```

#### Add a Default Gateway
<!-- Sets the default route for traffic destined for outside the local network (e.g., to the internet). -->
```bash
# Using route (older):
# route add default gw <Gateway_IP_Address> <Interface>:
#   <Gateway_IP_Address>: The IP address of your router/gateway.
#   <Interface>: (Optional but recommended) The network interface to use (e.g., eth0, wlan0).
sudo route add default gw 192.168.1.1 eth0

# Modern alternative with `ip route`:
sudo ip route add default via 192.168.1.1 dev eth0
```

#### Delete a Default Gateway
```bash
# Using route (older):
sudo route del default gw 192.168.1.1 dev eth0 # Specify interface if it was used during add

# Modern alternative with `ip route`:
sudo ip route del default via 192.168.1.1 dev eth0
```

### Network Traffic Analysis (`tcpdump`)
<!-- `tcpdump` is a powerful command-line packet analyzer. Requires root privileges (`sudo`). -->
<!-- CAUTION: Can capture sensitive data if traffic is unencrypted. Use responsibly. -->

#### Basic Packet Capture on an Interface
<!-- Capture all traffic on interface `wlan0`. Press Ctrl+C to stop. -->
```bash
# tcpdump -vv -i wlan0:
#   -i wlan0: Specify the network interface to listen on (e.g., eth0, any).
#   -vv: Very verbose output. Increase 'v' for more verbosity (e.g., -v, -vvv).
#   -n: Don't convert addresses (host addresses, port numbers, etc.) to names.
#   -X: Print hex and ASCII dump of packet data.
#   -s0: Snaplen 0 - capture full packets. Default is often truncated.
sudo tcpdump -i wlan0 -s0 -XAnn # Common starting point: full packet, ASCII/hex, no name resolution
```

#### Capture ICMP Packets and Write to File
<!-- Capture only ICMP (e.g., ping) packets on `wlan0` and save them to `icmp.pcap` file for later analysis. -->
```bash
# tcpdump -vv -i wlan0 icmp -w icmp.pcap:
#   icmp: Filter expression to capture only ICMP protocol packets.
#   -w icmp.pcap: Write the raw captured packets to a file named `icmp.pcap` (libpcap format).
sudo tcpdump -i wlan0 icmp -w icmp.pcap -s0
```

#### Read Captured Packets from File
<!-- Display packets from a previously saved .pcap file. -->
```bash
# tcpdump -r icmp.pcap:
#   -r icmp.pcap: Read packets from the specified file.
#   -n: Don't resolve names (useful for offline analysis).
#   -X: Show hex/ASCII.
tcpdump -r icmp.pcap -nX
```

### MAC Address Spoofing (`macchanger`)
<!-- Changes the MAC (Media Access Control) address of a network interface. -->
<!-- Requires root privileges. The interface should usually be down before changing the MAC and then brought back up. -->
<!-- Useful for privacy or bypassing MAC filtering (ensure you have authorization if on networks you don't own). -->
```bash
# Install macchanger if not present: sudo apt-get install macchanger / sudo yum install macchanger

# 1. Identify your network interface (e.g., eth0, wlan0, enp3s0)
#    ip link show

# 2. Bring the interface down (recommended before changing MAC).
sudo ip link set dev wlan0 down

# 3. Change the MAC address to a random one.
#    macchanger -r wlan0:
#      -r: Set a fully random vendor MAC address of the same kind (e.g., Ethernet, Wi-Fi).
#      wlan0: The target network interface.
sudo macchanger -r wlan0
#    Other options:
#      -e: Set random MAC but keep original vendor bytes (less likely to break things).
#      -m XX:XX:XX:XX:XX:XX: Set a specific MAC address.
#      -p: Reset to original, permanent hardware MAC address.

# 4. Bring the interface back up.
sudo ip link set dev wlan0 up

# 5. Verify the change (optional).
#    ip link show wlan0
#    macchanger -s wlan0
```

#### Restore Original (Permanent) MAC Address
```bash
# 1. Bring the interface down.
sudo ip link set dev wlan0 down
# 2. Restore the permanent MAC address using -p.
sudo macchanger -p wlan0
# 3. Bring the interface back up.
sudo ip link set dev wlan0 up
```

## Antivirus Scanning (`ClamAV`)

<!-- ClamAV is an open-source antivirus engine for detecting trojans, viruses, malware & other malicious threats. -->
<!-- Install ClamAV: `sudo apt-get install clamav clamav-daemon` or `sudo yum install clamav clamd`. -->

### Update Virus Definitions
<!-- Before running a scan, ensure your ClamAV virus definitions are up-to-date. -->
<!-- `freshclam` updates the signature databases. This typically requires root privileges. -->
```bash
sudo freshclam
# If running `clamd` (the daemon), it might auto-update, or you might need to restart/reload it.
```

### Basic Root Filesystem Scan (Excluding Pseudo-filesystems)
<!-- Scans the entire root filesystem ('/'), excluding common pseudo-filesystems, cache, and temporary directories
     to save time, reduce false positives, and avoid errors on special files. -->
```bash
# sudo clamscan -r / --exclude-dir="^/proc" --exclude-dir="^/sys" --exclude-dir="^/dev" \
#   --exclude-dir="^/boot" --exclude-dir="^/var/lib/docker" --exclude-dir="^/tmp" --exclude-dir="^/run" \
#   --log=/var/log/clamscan_root_$(date +%Y-%m-%d).log
#
# Explanation:
#   clamscan: The command-line scanner.
#   -r /: Recursively scan the root directory and its subdirectories.
#   --exclude-dir="^/path": Exclude specified directories. Using '^' anchors the path to the root
#                           to prevent excluding user directories like `/home/user/dev`.
#     Common exclusions:
#       /proc, /sys, /dev: Kernel pseudo-filesystems. Accessing them can cause issues.
#       /boot: Bootloader files (less common target, can be included if desired but usually static).
#       /var/lib/docker, /var/lib/containerd: Docker/container overlay filesystems, can be very large and slow to scan.
#       /tmp, /run: Temporary runtime directories.
#       /mnt, /media: Often mount points for external or network storage.
#   --log=/path/to/logfile.log: Write scan report to a log file.
#   -i: Only print infected files.
#   --infected: Alias for -i.
#   --remove[=yes/no]: Remove infected files (USE WITH EXTREME CAUTION!). Default is 'no'.
#   --move=/path/to/quarantine: Move infected files to a quarantine directory.
#
# Example (logging infected files only):
sudo clamscan -r / \
    --infected \
    --exclude-dir="^/proc" --exclude-dir="^/sys" --exclude-dir="^/dev" \
    --exclude-dir="^/boot" --exclude-dir="^/var/lib/docker" --exclude-dir="^/var/lib/containerd" \
    --exclude-dir="^/tmp" --exclude-dir="^/run" --exclude-dir="^/mnt" --exclude-dir="^/media" \
    --log=/var/log/clamscan_$(date +%Y-%m-%d)_root.log
```
<!-- Note: Excluding the entire /var might be too broad for some use cases, as it can contain web server content or mail spools.
     Tailor --exclude-dir patterns to your specific needs and risk assessment. -->

### Focused Scan on a Specific Directory (e.g., Home or Downloads)
<!-- Scans a user's directory (e.g., '~/Downloads') with more detailed options. -->
```bash
# Ensure quarantine directory exists and has restricted permissions:
# sudo mkdir -p /var/quarantine/clamav && sudo chmod 700 /var/quarantine/clamav

# sudo clamscan -vr --max-filesize=2000M --max-scantime=2700 --cross-fs=yes \
#   --follow-dir-symlinks=2 --follow-file-symlinks=2 \
#   --scan-elf=yes --scan-pdf=yes --detect-pua=yes --heuristic-alerts=yes \
#   --move=/var/quarantine/clamav/ --log=/var/log/clamscan_mydir_$(date +%Y-%m-%d).log /path/to/your_directory
#
# Explanation of some options:
#   -v: Verbose output (shows all files scanned). Omit for quieter operation.
#   -r: Recursive scan.
#   --max-filesize=2000M: Skip files larger than 2000 MB. (Original: 2000M). Adjust as needed.
#   --max-scantime=2700: Skip files that take longer than 2700 seconds (45 minutes) to scan. (Original: 2700).
#   --cross-fs=yes: Scan files on different filesystems (e.g., if /home is a separate partition). Set to 'no' to stay on one filesystem.
#   --follow-dir-symlinks=2: Follow directory symlinks up to depth 2. (0=never, 1=direct, 2=indirect). Be careful with symlink loops.
#   --follow-file-symlinks=2: Follow file symlinks up to depth 2.
#   --official-db-only=no: Use official AND unofficial signature databases (if available). Default is often 'yes'.
#                          Unofficial dbs can increase detection but also false positives.
#   --scan-elf=yes: Scan ELF (Linux executable) files.
#   --scan-pdf=yes: Scan PDF files.
#   --detect-pua=yes: Detect Potentially Unwanted Applications (PUA). These are not strictly malware but may be undesirable.
#   --heuristic-alerts=yes: Enable heuristic scanning (can find new/unknown threats but may cause false positives).
#   --move=/var/quarantine/clamav/: Move infected/suspicious files to this directory.
#   --log=/path/to/logfile.log: Log scan results.
#   /path/to/your_directory: The directory to scan.
#
# Example for a user's Downloads directory, logging only infected files and moving them:
sudo clamscan -r --infected \
    --max-filesize=2G --max-scantime=1800 --cross-fs=no \
    --scan-elf=yes --scan-pdf=yes --detect-pua=yes --heuristic-alerts=yes \
    --move=/var/quarantine/clamav/ \
    --log=/var/log/clamscan_$(date +%Y-%m-%d)_downloads.log ~/Downloads/
```

### Full System Scan with Detailed Options (More Intensive)
<!-- Similar to the focused scan but targets the entire root filesystem, with exclusions. -->
<!-- This can be very time-consuming and resource-intensive. Schedule during off-peak hours. -->
```bash
# Ensure quarantine directory exists:
# sudo mkdir -p /var/quarantine/clamav && sudo chmod 700 /var/quarantine/clamav

# sudo clamscan -vr --max-filesize=4000M --max-scantime=14400 --cross-fs=yes \
#   --follow-dir-symlinks=2 --follow-file-symlinks=2 --official-db-only=no \
#   --scan-elf=yes --detect-pua=yes --heuristic-alerts=yes \
#   --move=/var/quarantine/clamav/ \
#   --exclude-dir="^/proc" --exclude-dir="^/sys" --exclude-dir="^/dev" \
#   --exclude-dir="^/boot" --exclude-dir="^/var/lib/docker" --exclude-dir="^/var/lib/containerd" \
#   --exclude-dir="^/tmp" --exclude-dir="^/run" --exclude-dir="^/mnt" --exclude-dir="^/media" \
#   --log=/var/log/clamscan_full_$(date +%Y-%m-%d).log /
#
# Example (logging only infected files and moving them):
sudo clamscan -r --infected \
    --max-filesize=4G `# Max file size to scan ` \
    --max-scantime=10800 `# Max scan time per file (3 hours) ` \
    --cross-fs=yes `# Scan other filesystems mounted under / ` \
    --scan-elf=yes --detect-pua=yes --heuristic-alerts=yes \
    --move=/var/quarantine/clamav/ \
    --exclude-dir="^/proc" --exclude-dir="^/sys" --exclude-dir="^/dev" \
    --exclude-dir="^/boot" --exclude-dir="^/var/lib/docker" --exclude-dir="^/var/lib/containerd" \
    --exclude-dir="^/tmp" --exclude-dir="^/run" --exclude-dir="^/mnt" --exclude-dir="^/media" \
    --exclude-dir="^/lost+found" \
    --log=/var/log/clamscan_$(date +%Y-%m-%d)_system_full.log /
```
<!-- Remember to regularly update virus definitions using `sudo freshclam`. -->
<!-- Review log files and quarantined files carefully. Heuristic scanning can sometimes produce false positives. -->


## Shell Tips and Tricks (Continued)

### Bash Random Number Generation
<!-- Different methods to generate pseudo-random numbers in Bash. -->

#### Method 1: Using `$RANDOM` (Bash Built-in)
<!-- `$RANDOM` is a Bash internal variable that returns a pseudo-random integer between 0 and 32767.
     This is the simplest method for non-cryptographic random numbers. -->
```bash
# Generate a random number between 0 and 32767:
echo $RANDOM

# To get a number in a specific range (e.g., 1 to 100):
# Formula: $((RANDOM % (Max - Min + 1) + Min))
echo $((RANDOM % 100 + 1)) # Generates a random number between 1 and 100
```

#### Method 2: Using `/dev/urandom` with `od` and `awk` (More Complex)
<!-- Generates a random integer within a specific range using the kernel's random number generator.
     More complex than `$RANDOM` or `shuf`. -->
```bash
# head -c 4 /dev/urandom: Read 4 bytes from /dev/urandom (a source of cryptographic quality randomness).
# od -N4 -tu4 -An: Format the 4 bytes as an unsigned 4-byte integer.
#   -N4: Read at most 4 bytes.
#   -tu4: Output format is unsigned decimal, 4 bytes per integer.
#   -An (or -A n): Suppress the address/offset column.
# awk '{print $1 % 100000000 + 1}':
#   $1: Use the first field from `od` output (the integer).
#   % 100000000: Modulo operation to get a number in range 0-99,999,999.
#   + 1: Add 1 to make the range 1-100,000,000.
head -c 4 /dev/urandom | od -N4 -tu4 -An | awk '{print $1 % 100000000 + 1}'
# Note: The modulo bias applies here; distribution might not be perfectly uniform for arbitrary ranges.
```

#### Method 3: Using `/dev/urandom` and `od` for Hexadecimal Output
<!-- Generates a random sequence of hexadecimal characters. -->
```bash
# head -c4 /dev/urandom: Read 4 bytes from /dev/urandom.
# od -A n -t x1: Format the output as hexadecimal, 1 byte at a time, no offsets.
#   -A n: Suppress the address/offset column.
#   -t x1: Output format is hexadecimal, 1 byte at a time.
# tr -d ' ': Remove spaces from `od` output.
head -c4 /dev/urandom | od -A n -t x1 | tr -d ' '
# Example Output (will vary, 8 hex characters for 4 bytes):
# 1a2b3c4d
```

#### Method 4: Using `shuf` (Shuffle - GNU Coreutils)
<!-- `shuf` is a versatile command, part of GNU coreutils, often the easiest and most flexible way
     for generating random numbers or selecting random lines from a file. -->
```bash
# shuf -i MinRange-MaxRange -n Count:
#   -i MinRange-MaxRange: Specify the input range of integers (inclusive).
#   -n Count: Output 'Count' number of random numbers.
# Example: Generate one random number between 1 and 100,000,000.
shuf -i 1-100000000 -n 1

# Example: Generate 5 random numbers between 1 and 10.
shuf -i 1-10 -n 5
```

## System Administration & Information (Continued)

### Continuous Process Monitoring

#### Using `watch`
<!-- The `watch` command executes a program periodically, showing its output full screen.
     The output updates in place, allowing you to monitor changes over time. -->
```bash
# watch -n <seconds> "your_command_here":
#   -n <seconds>: Specify the interval in seconds between command executions. Default is 2 seconds.
#   "your_command_here": The command to execute and monitor. Must be quoted if it contains spaces or special characters.
# Example: Watch free memory every 30 seconds.
watch -n 30 "free -h"

# Example: Watch for changes in a directory listing every 10 seconds.
watch -n 10 "ls -l /path/to/directory"
```
<!-- Press Ctrl+C to exit `watch`. -->

#### Using a `while` Loop with `ps` and `grep` (Process Completion Monitor)
<!-- This script monitors if a specific process is running and prints a message when it's no longer found. -->
```bash
# PROCESS_IDENTIFIER="<PID_or_Name_or_CommandPattern>"
# while ps aux | grep -v grep | grep -qF "$PROCESS_IDENTIFIER"; do
#    echo "Process '$PROCESS_IDENTIFIER' is still running... ($(date))" # Optional: progress message
#    sleep 30 # Check every 30 seconds
# done && echo "Process '$PROCESS_IDENTIFIER' has completed or is no longer running. ($(date))"

# Explanation:
#   PROCESS_IDENTIFIER: Set this to the PID, process name, or a unique part of the command you are looking for.
#     Examples:
#       PROCESS_IDENTIFIER="12345" (for a specific PID)
#       PROCESS_IDENTIFIER="my_long_script.sh" (for a script name)
#       PROCESS_IDENTIFIER="backup_job --full" (for a command pattern)
#
#   while ps aux | grep "[P]ROCESS_NAME_PATTERN" -q; do ... done:
#     ps aux: List all running processes.
#     grep "[P]ROCESS_NAME_PATTERN": A common trick to prevent grep from matching itself.
#                                    The [P] ensures "grep [P]ROCESS_NAME_PATTERN" doesn't match "[P]ROCESS_NAME_PATTERN".
#                                    Replace PROCESS_NAME_PATTERN with the actual pattern.
#     grep -q: Quietly search. Success/failure is determined by exit status.
#              The loop continues as long as grep finds the process (exit status 0).
#   sleep 30: Wait for 30 seconds before checking again.
#   && echo "...": This part executes only when the `while` loop condition becomes false (grep no longer finds the process).
#                 The original message "A execução de $PROCESS foi concluída." means "The execution of $PROCESS has finished."

# Example: Monitor a process with PID 4242
PROCESS_PID="4242"
# Using `ps -p` is more direct and reliable for PID checking.
while ps -p "${PROCESS_PID}" > /dev/null; do
  echo "Process PID ${PROCESS_PID} is still running... ($(date))"
  sleep 15
done && echo "Process PID ${PROCESS_PID} has completed or is no longer running. ($(date))"
```
<!-- Note: Grepping process lists by name can sometimes be unreliable if names are too generic or change.
     Using PIDs (`ps -p <PID>`) or more specific/anchored patterns is safer.
     For robust job completion notification, consider application-specific logging or job control features if available. -->

## Encoding & Decoding Utilities

### `openssl` for Base64 Encoding/Decoding
<!-- OpenSSL can be used for various cryptographic operations, including Base64 encoding/decoding. -->

#### Encode a File to Base64
<!-- Reads 'input_file' and outputs its Base64 encoded version. -->
```bash
# To output to another file:
openssl base64 -in input_file -out output_file.b64
# To output to terminal:
# openssl base64 -in input_file
```

#### Decode Base64 to File
<!-- Reads 'input_file.b64' (Base64 encoded) and outputs the decoded binary data. -->
```bash
# To decode from a file to another file:
openssl base64 -d -in input_file.b64 -out output_file_decoded
# To decode from a pasted string (ensure no trailing newline in the pasted content if sensitive):
# echo "PASTED_BASE64_STRING" | openssl base64 -d -out output_file_decoded
```

### `xxd` for Hexadecimal Encoding/Decoding
<!-- `xxd` creates a hex dump of a given file or standard input. It can also convert a hex dump back to binary. -->

#### Encode a File to Hexadecimal (Plain Style)
<!-- Reads 'input_file' and outputs its hexadecimal representation without offsets or ASCII interpretation. -->
```bash
# To output to another file:
xxd -p input_file > output_hex.txt
# To output to terminal:
# xxd -p input_file
```

#### Decode Hexadecimal (Plain Style) to File
<!-- Reads 'input_hex.txt' (containing a continuous string of hex digits) and outputs the decoded binary. -->
```bash
# xxd -p -r input_hex.txt > output_file_decoded
#   -p: Plain hexdump style.
#   -r: Reverse operation: convert hex dump into binary.
xxd -p -r input_hex.txt > output_file_decoded
```

## Network Probing & Security Assessment

<!--
CAUTION: ETHICAL USE AND AUTHORIZATION REQUIRED.
The tools and commands listed in this section can be misused.
Always ensure you have explicit, WRITTEN PERMISSION from the system owner before scanning or attempting any form of penetration testing on any system or network that you DO NOT OWN.
Unauthorized scanning or testing can lead to legal consequences and is unethical.
Use these tools responsibly for learning, testing your own systems, or with proper authorization for security assessments.
Ignorance of the law is not an excuse.
-->

### Anonymous/Less-Traceable SSH Sessions (Host Key Handling)

#### Creating an SSH Session without Saving Host Key
<!-- Useful for connecting to a server once without adding its key to `~/.ssh/known_hosts` or failing if the key changes. -->
<!-- This does NOT make the connection itself anonymous on the network. It only affects local SSH client behavior. -->
```bash
# ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -T user@target_server "bash -i"
#   -o UserKnownHostsFile=/dev/null: Does not save the host key to any file and ignores existing known_hosts.
#   -o StrictHostKeyChecking=no: Automatically accept new or changed host keys.
#                                 WARNING: This disables protection against man-in-the-middle (MITM) attacks if the host key is unexpected.
#   -T: Disable pseudo-tty allocation. Useful for running a specific command.
#   "bash -i": Command to execute on the remote server (request an interactive bash shell).
ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -T user@target_server.org "bash -i"
# Use with caution, especially `StrictHostKeyChecking=no`, as it makes you vulnerable to MITM if the server's identity is compromised or spoofed.
```

### Network Discovery (Local Network)

#### ARP Scan with `nmap` (Local Network Host Discovery)
<!-- Discovers active hosts on the local network segment using ARP requests.
     Much faster and more reliable than ICMP pings on a LAN as ARP is fundamental to local network communication. -->
```bash
# nmap -sn -PR 192.168.0.1/24
#   sudo is usually required for ARP scans.
#   -sn: Ping scan - disables port scanning. (Older nmap versions used -sP).
#   -PR: ARP Ping. Nmap will send ARP packets to hosts on the local network.
#   192.168.0.1/24: Your local network range. Adjust as necessary (e.g., 192.168.1.0/24, 10.0.0.0/24).
#                  Use `ip addr` or `ifconfig` to find your network range.
#   -r: (Original had this) Scan ports in sequential order, not random. Not typically used or needed with -sn -PR for host discovery.
sudo nmap -sn -PR 192.168.0.0/24
```

#### ICMP Network Sweep (Less Reliable on LANs than ARP)
<!-- Attempts to find live hosts by sending ICMP echo requests (pings) to a range of IP addresses. -->
<!-- Many systems or firewalls block ICMP echo requests, so this is not always reliable, especially on external networks. -->
```bash
# TARGET_NET="192.168.0"  # Define your network prefix.
# seq 1 254 | xargs -P20 -I{} ping -n -c1 -W1 "${TARGET_NET}.{}" | grep 'bytes from' | awk '{print $4 $7}' | sed 's/://' | sort -uV -k1,1
#
# Explanation:
#   TARGET_NET="192.168.0": Sets the base network address (first three octets).
#   seq 1 254: Generates numbers from 1 to 254 (for the last octet of the IP).
#   xargs -P20 -I{}: Executes the ping command in parallel (up to 20 processes).
#     -P20: Max 20 parallel processes. Adjust based on your system resources and network.
#     -I{}: Replace {} with the input number.
#   ping -n -c1 -W1 "${TARGET_NET}.{}": Pings the constructed IP address.
#     -n: Numeric output (no DNS resolution).
#     -c1: Send only 1 packet.
#     -W1: Wait 1 second for a reply (timeout for each ping). Adjust if network is slow.
#   grep 'bytes from': Filters for successful ping replies (lines containing "bytes from").
#   awk '{print $4 $7}': Extracts the IP address (field 4) and latency (field 7, e.g., "time=1.23").
#   sed 's/://': Removes the colon from the IP address if present (e.g., from MAC address output if not careful).
#                This might be overly broad. Better to ensure awk captures only IP.
#   sort -uV -k1,1: Sorts unique IPs version-style.
#
# Safer awk and sed to extract just the IP:
TARGET_NET="192.168.0"
for i in $(seq 1 254); do
  ping -c 1 -W 0.2 "${TARGET_NET}.${i}" | grep 'bytes from' | cut -d' ' -f4 | cut -d':' -f1 &
done | sort -uV
# This version uses a loop with backgrounding for parallelism, and `cut` for more precise IP extraction.
# -W 0.2: Short timeout (200ms) for faster scanning on responsive LANs. Adjust as needed.
# `&` backgrounds each ping. Output is then piped to sort.
```

### Network Traffic Monitoring with `tcpdump` (Continued)

#### Monitor New TCP Connections (SYN packets)
<!-- Captures only the initial SYN packet of TCP connections. Useful for seeing new connection attempts to/from your machine. -->
```bash
# tcpdump -n 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack == 0'
#   -n: No DNS resolution for addresses or ports.
#   'tcp[tcpflags] & tcp-syn != 0': Checks if the SYN flag is set in the TCP flags field.
#   'and tcp[tcpflags] & tcp-ack == 0': Ensures ACK flag is NOT set (to filter out SYN-ACKs from established connections).
# A simpler way to express "only SYN packets" (initiating a connection) is often just `tcp[tcpflags] == tcp-syn`.
sudo tcpdump -n 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack == 0'
# Or more commonly and often sufficient for seeing new connection attempts:
# sudo tcpdump -n 'tcp[tcpflags] == tcp-syn'
```

#### Play a Sound for New SSH Connections
<!-- Monitors for incoming SSH connections (destination port 22) and plays a terminal bell for each new connection attempt. -->
```bash
# sudo tcpdump -nlq 'tcp[tcpflags] & tcp-syn != 0 and dst port 22' | while read x; do printf "%s\n\a" "${x}"; done
#   -n: No DNS resolution.
#   -l: Line buffer output (makes it available to `while read` sooner).
#   -q: Quiet output (less verbose from tcpdump itself).
#   'tcp[tcpflags] & tcp-syn != 0 and dst port 22': Filters for TCP SYN packets going to destination port 22 (SSH).
#   while read x; do ... done: Reads each line of tcpdump output (summary of the packet).
#     printf "%s\n\a" "${x}": Prints the captured packet summary line, then prints a newline, then sends the ASCII bell character (\a).
sudo tcpdump -nlq 'tcp[tcpflags] & tcp-syn != 0 and dst port 22' | while read x; do printf "%s\n\a" "${x}"; done
```

#### Capture Full Packet Data with ASCII Output
<!-- Captures more of the packet data (or full packets with -s0) and displays printable characters as ASCII, non-printable as dots. -->
```bash
# tcpdump -s0 -nAq 'tcp and (ip[2:2] > 60)'
#   -s0: Snaplen (snapshot length) 0 means capture the full packet. Use a specific value (e.g., -s 2048) to limit capture size.
#   -n: No DNS resolution.
#   -A: Print each packet (minus its link level header) in ASCII.
#   -q: Quiet output (less tcpdump chatter).
#   'tcp and (ip[2:2] > 60)': Filter for TCP packets where the IP total length (bytes at IP header offset 2, for 2 bytes) is greater than 60.
#                            This attempts to filter out small TCP control packets (e.g., pure ACKs without data) to focus on packets likely carrying data.
#                            The condition `ip[2:2]` reads the 2-byte total length field from the IP header.
sudo tcpdump -s0 -nAq 'tcp and (ip[2:2] > 60)'
```

### Tunneling & Port Forwarding

<!-- Various methods to tunnel traffic, forward ports, or create proxies. -->
<!-- Useful for bypassing firewalls (with permission), accessing internal services, or securing traffic. -->

#### Reverse HTTPS Tunnel with `ssh` and Public Services
<!-- Forwards a port from a remote public server back to your local machine, making a local service accessible via the remote server's public URL. -->
<!-- Useful for exposing a local web server (e.g., during development) temporarily without direct public IP or complex firewall rules. -->
<!-- These services provide a temporary public URL. Always check their terms of service. -->
```bash
# General structure:
# ssh -R<remote_port_on_service>:localhost:<local_port_of_your_app> -o StrictHostKeyChecking=accept-new -o ServerAliveInterval=60 <user>@<service_host>
#   -R<remote_port>:localhost:<local_port>: Remote port forwarding.
#     Traffic to <remote_port> on <service_host> is forwarded to localhost:<local_port> on the machine running this ssh command.
#   -o StrictHostKeyChecking=accept-new: Automatically accept new host keys (common for these dynamic services, but less secure than verifying).
#   -o ServerAliveInterval=60: Keep connection alive by sending a message every 60 seconds.
#   nokey@localhost.run / nokey@remote.moe: Example user/host for these services. They often use 'nokey' or similar for public access.

# Example with localhost.run (often provides HTTP/HTTPS on port 80/443 for the public URL):
# Assuming your local web service is running on port 8080.
ssh -R80:localhost:8080 -o StrictHostKeyChecking=accept-new -o ServerAliveInterval=60 nokey@localhost.run
# localhost.run will provide you with a public URL like http://<random_subdomain>.localhost.run

# Example with remote.moe (similar service):
# ssh -R80:localhost:8080 -o StrictHostKeyChecking=accept-new -o ServerAliveInterval=60 nokey@remote.moe

# Example with Cloudflare Tunnel (requires `cloudflared` CLI installed and configured/logged in):
# cloudflared tunnel --url http://localhost:8080 --no-autoupdate
#   This creates a tunnel from a unique Cloudflare edge URL directly to your local service on port 8080.
#   `--no-autoupdate` prevents the cloudflared client from auto-updating during operation.
cloudflared tunnel --url http://localhost:8080 --no-autoupdate
# Cloudflared will output a public HTTPS URL like https://<something>.trycloudflare.com
```

#### STDIN/STDOUT Pipe over HTTPS (`websocat`, `gost`)
<!-- Create a simple data pipe between two machines using WebSockets, potentially over HTTPS if a reverse proxy handles TLS. -->

##### Using `websocat`
<!-- `websocat` is a versatile tool for WebSocket connections. Install from https://github.com/vi/websocat -->
```bash
# On the server (listening side, e.g., public_server_ip):
# websocat --text -s 8080
#   -s 8080: Listen (server mode) on port 8080.
#   --text: Use text mode (lines terminated by newline). For binary, omit this.
#   This listens on ws://public_server_ip:8080. For wss:// (secure), run `websocat` behind a reverse proxy like Nginx that handles TLS termination.

# On the client (connecting side, e.g., your machine):
# websocat --text ws://<public_server_ip>:8080
#   Replace <public_server_ip> with the actual IP or hostname of the server.
#   If server is behind TLS reverse proxy: websocat --text wss://<public_domain_of_server>
# Now, data piped into websocat on one side will appear on stdout of the other.
# Example: Client sends "hello", Server receives "hello".
# Client: echo "hello from client" | websocat --text ws://<public_server_ip>:8080
# Server: websocat --text -s 8080  (will print "hello from client")
```

##### Using `gost` (more versatile tunneling tool)
<!-- `gost` is a powerful and flexible tunneling tool. Install from https://github.com/ginuerzh/gost -->
```bash
# On the server (listening for WebSocket - ws://):
# gost -L ws://:8080
#   -L ws://:8080: Listen on port 8080 for WebSocket connections.
#   For secure WebSocket (wss:// or mws:// for multiplexed): gost -L "mws://:8080?cert=server.crt&key=server.key"
#   (Requires generating cert.pem and key.pem, e.g., with openssl, or use a reverse proxy for TLS).

# On the workstation/client (connecting to server and forwarding local SSH for example):
# This example makes the server's port 2222 act as a gateway to the client's local SSH service on port 22.
# gost -L tcp://:2222/127.0.0.1:22 -F ws://<server_ip_or_hostname>:8080
#   -L tcp://:2222/127.0.0.1:22: This part is confusing in context of client.
#      If this runs on client, it means: listen on client's port 2222, forward to client's 127.0.0.1:22.
#      Then -F means this whole setup is tunneled via the gost server.
#      This effectively exposes client's SSH to itself via the server (not very useful).

# More practical: Expose client's local service (e.g. port 8000) to be accessible on server's port 9000
# On Client:
# gost -L relay+ws://<user>:<pass>@<server_ip>:8080/localhost:8000
# On Server:
# gost -L tcp://:9000 -F relay+ws://<user>:<pass>@:8080
# (Client connects to server's relay, server listens on 9000 and forwards to client's 8000 via relay)

# Simpler stdin/stdout pipe with gost (like websocat):
# Server: gost -L ws://:8080 - F stdio://
# Client: gost -L stdio:// -F ws://<server_ip>:8080
```

#### Port Forwarding and SOCKS Proxy with `gost`

##### Forward Remote Port to Local Service (Reverse Port Forwarding)
```bash
# Scenario: Make a service on your local machine (client_local_port) accessible on a remote server (server_public_port).
# On remote server (e.g., public_server_ip):
# gost -L tcp://:<server_public_port> -F "relay+ws://<user>:<password>@:8080"
#   Listens on server_public_port and relays traffic via a WebSocket listener on port 8080.

# On your local machine (client):
# gost -L "relay+ws://<user>:<password>@<public_server_ip>:8080/127.0.0.1:<client_local_port>"
#   Connects to the server's relay and forwards traffic to your client_local_port.
# Example: Make local web server on port 8000 accessible on remote_server:9000
# Remote Server: gost -L tcp://:9000 -F "relay+ws://myuser:mypass@:8080"
# Local Machine: gost -L "relay+ws://myuser:mypass@<remote_server_ip>:8080/127.0.0.1:8000"
```

##### Use `gost` as a SOCKS Proxy Server
```bash
# On a server (e.g., your remote server with public IP, call it `your_gost_server_ip`):
# gost -L socks5://:1080
#   Runs an unauthenticated SOCKS5 proxy on port 1080 of `your_gost_server_ip`.
# For a SOCKS proxy accessible via an encrypted tunnel (e.g., MWS - Multiplexed WebSocket Secure):
# gost -L "mws://your_user:your_password@:8443"
#   This runs a SOCKS5 proxy service that is only accessible by connecting to `your_gost_server_ip:8443`
#   via MWS protocol with `your_user:your_password`. The SOCKS5 service itself is encapsulated.
```

##### Use `gost` on Workstation to Connect to a Remote `gost` SOCKS Proxy (Client to Server SOCKS)
```bash
# Scenario: `your_gost_server_ip` is running a gost SOCKS proxy accessible via MWS on port 8443
#           (e.g., using the `gost -L "mws://user:pass@:8443"` command on the server).
# On your workstation (client):
# gost -L :1080 -F "mws://your_user:your_password@your_gost_server_ip:8443"
#   -L :1080: Listen locally on your workstation's port 1080, acting as a SOCKS5 proxy.
#   -F "mws://...": Forward all traffic from this local SOCKS proxy through the remote gost server
#                   (which itself provides SOCKS5 access to the internet or its local network).
# Now, configure your local applications (e.g., browser, curl) to use SOCKS5 proxy at localhost:1080.
# Traffic will go: App -> localhost:1080 (gost client) -> MWS tunnel -> your_gost_server_ip:8443 (gost server) -> Internet.
```

##### Test SOCKS Proxy
```bash
# After setting up the local SOCKS proxy (e.g., via gost client as above, or `ssh -D 1080 user@host`),
# test it with curl:
curl -x socks5h://localhost:1080 https://ipinfo.io/json
#   -x socks5h://localhost:1080: Use SOCKS5 proxy on localhost:1080.
#     'h' means DNS resolution also happens on the proxy side (remote DNS). Use `socks5://` for local DNS.
#   https://ipinfo.io/json: A service to check your public IP address and get details in JSON.
# This command should show the IP address of the server where the SOCKS proxy endpoint is effectively running (e.g., `your_gost_server_ip`).
```

### Using Tools over a SOCKS Proxy (`proxychains`)
<!-- `proxychains` or `proxychains-ng` forces TCP connections from specified applications through a proxy (SOCKS4, SOCKS5, HTTP). -->
<!-- Useful for running tools that don't natively support SOCKS proxies. -->

#### Setup SOCKS Proxy Relay (Example with SSH Dynamic Port Forwarding)
<!-- `ssh -D` is a common way to create a local SOCKS proxy that tunnels traffic through a remote SSH server. -->
```bash
# On your workstation, connect to an SSH server (`jumphost.example.com`):
# ssh -D 1080 -C -N user@jumphost.example.com
#   -D 1080: Creates a SOCKS proxy on localhost:1080. Traffic sent to this port is tunneled via jumphost.
#   -C: Enable compression (good for slow links).
#   -N: Do not execute a remote command (just set up the tunnel).
#   -f: Optional, run in background.
# This command creates a SOCKS proxy on your workstation at 127.0.0.1:1080.
# The `gs-netcat` examples from original were less common for this specific proxychains setup.
```

#### Configure `proxychains`
<!-- `proxychains` uses a configuration file (default: /etc/proxychains.conf, or specify with -f). -->
```bash
# Create a custom proxychains configuration file (e.g., pc.conf in the current directory).
# Replace 127.0.0.1 1080 with the IP and port of your SOCKS proxy.
# If you used `ssh -D 1080 ...`, then 127.0.0.1 1080 is correct.
# If using `gs-netcat -l -S` on a remote machine `target_proxy_ip` on port `target_proxy_port`, use that.
echo -e "[ProxyList]\nsocks5 127.0.0.1 1080" > ./my_proxy.conf
# For SOCKS4, use `socks4`. For HTTP proxy, use `http <ip> <port>`.
```

#### Use `proxychains` with various tools
```bash
# proxychains -f ./my_proxy.conf -q <command> [arguments]
#   -f ./my_proxy.conf: Use the specified configuration file.
#   -q: Quiet mode (optional, less output from proxychains itself).
#   <command> [arguments]: The command to run through the proxy.

# Example: Check public IP through the proxy (should show IP of jumphost.example.com).
proxychains -f ./my_proxy.conf -q curl https://ipinfo.io/ip

# Example: Scan a router on the target's internal network (e.g., 192.168.1.1, reachable from jumphost).
# Assumes 192.168.1.1 is reachable from the SSH server (jumphost.example.com).
proxychains -f ./my_proxy.conf -q nmap -n -Pn -sV -F --open 192.168.1.1
#   nmap options:
#     -n: No DNS resolution by nmap (proxychains might still do DNS based on its config).
#     -Pn: Skip host discovery (treat hosts as online). Necessary when ping is blocked or going through proxy.
#     -sV: Service Version detection.
#     -F: Fast scan (fewer common ports).
#     --open: Show only open ports.

# Example: Start 10 nmap scans in parallel for a /24 range through the proxy.
# This scans 192.168.1.1 through 192.168.1.254.
TARGET_SUBNET="192.168.1" # Ensure this network is accessible from the SOCKS proxy endpoint.
seq 1 254 | xargs -P10 -I{} proxychains -f ./my_proxy.conf -q nmap -n -Pn -sV -F --open "${TARGET_SUBNET}.{}"
```

### Brute-Force Examples (Ethical Use & Authorization REQUIRED)
<!--
WARNING: UNAUTHORIZED BRUTE-FORCE ATTACKS ARE ILLEGAL AND UNETHICAL.
These commands should ONLY be used on systems you own or have explicit, WRITTEN PERMISSION from the system owner to test for security vulnerabilities.
Always respect privacy, legal boundaries, and obtain proper authorization.
Placeholders:
  <target_IP>: Target IP address or hostname.
  <path_to_passwords.txt>: Path to a password list file (e.g., /usr/share/wordlists/rockyou.txt).
  <path_to_users.txt>: Path to a username list file.
-->

#### SSH Brute-Force
```bash
# Using nmap's ssh-brute script:
# sudo nmap -p 22 --script ssh-brute --script-args userdb=<path_to_users.txt>,passdb=<path_to_passwords.txt>,ssh-brute.timeout=4s <target_IP>

# Using ncrack:
# ncrack -P <path_to_passwords.txt> --user root "ssh://<target_IP>"

# Using hydra:
# hydra -L <path_to_users.txt> -P <path_to_passwords.txt> <target_IP> ssh -t 4
#   -t 4: Number of parallel tasks (threads).
```

#### Remote Desktop Protocol (RDP) Brute-Force (Port 3389)
```bash
# Using ncrack:
# ncrack -P <path_to_passwords.txt> --user Administrator "rdp://<target_IP>" # Common user is Administrator

# Using hydra:
# hydra -L <path_to_users.txt> -P <path_to_passwords.txt> rdp://<target_IP> -t 4
```

#### FTP Brute-Force (Port 21)
```bash
# Using hydra:
# hydra -L <path_to_users.txt> -P <path_to_passwords.txt> ftp://<target_IP> -t 4
```

#### POP3 Brute-Force (Port 110, 995 for SSL)
```bash
# Using nmap's pop3-brute script:
# sudo nmap -p 110,995 --script pop3-brute --script-args userdb=<path_to_users.txt>,passdb=<path_to_passwords.txt> <target_IP>
```

#### MySQL Brute-Force (Port 3306)
```bash
# Using nmap's mysql-brute script:
# sudo nmap -p 3306 --script mysql-brute --script-args userdb=<path_to_users.txt>,passdb=<path_to_passwords.txt> <target_IP>
# Or check for blank root password:
# sudo nmap -p 3306 --script mysql-empty-password <target_IP>
```

#### PostgreSQL Brute-Force (Port 5432)
```bash
# Using nmap's pgsql-brute script:
# sudo nmap -p 5432 --script pgsql-brute --script-args userdb=<path_to_users.txt>,passdb=<path_to_passwords.txt> <target_IP>
```

#### SMB (Windows Shares) Brute-Force (Port 139, 445)
```bash
# Using nmap's smb-brute script:
# sudo nmap -p 139,445 --script smb-brute --script-args userdb=<path_to_users.txt>,passdb=<path_to_passwords.txt> <target_IP>
# Or check for null sessions / anonymous access to shares:
# sudo nmap -p 139,445 --script smb-enum-shares --script-args smbusername=guest,smbpassword= <target_IP>
```

#### Telnet Brute-Force (Port 23)
```bash
# Using nmap's telnet-brute script:
# sudo nmap -p 23 --script telnet-brute --script-args userdb=<path_to_users.txt>,passdb=<path_to_passwords.txt>,telnet-brute.timeout=8s <target_IP>
```

#### VNC Brute-Force (Port 5900 series)
```bash
# Using nmap's vnc-brute script (often targets port 5900):
# sudo nmap -p 5900 --script vnc-brute --script-args passdb=<path_to_passwords.txt> <target_IP>
# (VNC often just uses a password, not username, but some servers might require username)

# Using ncrack:
# ncrack -P <path_to_passwords.txt> "vnc://<target_IP>"

# Using hydra:
# hydra -P <path_to_passwords.txt> vnc://<target_IP> -e ns
#   -e ns: Try null password and try login as password.

# Using Metasploit Framework (example from original, more involved):
# msfconsole
# > use auxiliary/scanner/vnc/vnc_login
# > set RHOSTS <target_IP>
# > set PASS_FILE /path/to/your/password_list.txt # e.g., /usr/share/wordlists/rockyou.txt
# > set USERNAME <username_if_known_or_blank_if_not_used> # Some VNC servers might use a username
# > run
```

#### HTTP Basic Authentication Brute-Force
```bash
# Using nmap's http-brute script:
# Create user.txt (e.g., echo "admin" > user.txt)
# Create pass.txt (e.g., echo -e "password\n12345\nadmin" > pass.txt)
# sudo nmap -p80 --script http-brute --script-args http-brute.hostname=<target_domain>,http-brute.path=/protected_path,userdb=user.txt,passdb=pass.txt <target_IP_or_domain>
#   http-brute.hostname: The hostname to use in HTTP requests (virtual host).
#   http-brute.path: The path that is protected by basic auth (e.g., /admin, /login).
#   The original example used POST, but Basic Auth is typically on GET requests. Adjust http-brute.method if needed.
```

### File Transfer using WebDAV with Cloudflare Tunnel
<!-- This setup allows you to expose a local directory via WebDAV over a public HTTPS URL provided by Cloudflare Tunnel. -->
<!-- Useful for temporary, secure file sharing from a local machine without configuring firewalls or port forwarding on your router. -->

#### 1. Start Cloudflare Tunnel (`cloudflared`)
<!-- Exposes your local port 8080 (where WebDAV will run) to a public Cloudflare URL. -->
```bash
# cloudflared tunnel --url http://localhost:8080 &
#   --url http://localhost:8080: Tunnels traffic to your local port 8080.
#   &: Runs in the background.
# You need to have `cloudflared` CLI installed and logged in (`cloudflared login`).
cloudflared tunnel --url http://localhost:8080 --no-autoupdate &
# Note the public URL provided by cloudflared output, e.g., https://example-foo-bar.trycloudflare.com
# This URL will be your WebDAV endpoint.
```

#### 2. Start a Local WebDAV Server (`wsgidav`)
<!-- `wsgidav` is a Python-based WebDAV server. Install with `pip install wsgidav cheroot`. -->
```bash
# wsgidav --port=8080 --root=/path/to/share --auth=anonymous:
#   --port=8080: Listen on port 8080 (same as specified in cloudflared).
#   --root=/path/to/share: Serve the contents of this local directory. Replace with your desired path. Using '.' serves the current directory.
#   --auth=anonymous: Allow anonymous access (no username/password).
#                     For security, consider digest authentication:
#                     `wsgidav --port=8080 --root=. --auth=digest --auth-user-file=users.txt`
#                     (You'd need to create users.txt in the format: user:realm:md5hash)
wsgidav --port=8080 --root=/tmp/webdav_share --auth=anonymous
# (Example: serving /tmp/webdav_share. Create this directory first: mkdir /tmp/webdav_share)
```
<!-- Now your WebDAV server is accessible via the Cloudflare URL from Step 1. -->

#### 3. Interacting with the WebDAV Server (Examples using `curl`)
<!-- Replace `https://your-cloudflare-url.trycloudflare.com` with the actual URL from `cloudflared`. -->
```bash
# Upload a file:
# curl -T local_file.dat https://your-cloudflare-url.trycloudflare.com/remote_filename.dat

# Create a directory:
# curl -X MKCOL https://your-cloudflare-url.trycloudflare.com/new_directory

# Create directory hierarchy (example from original, adapted):
# This would attempt to recreate the local directory structure of './local_src_dir'
# under '/sources_on_webdav/' on the WebDAV server.
# (cd ./local_src_dir && find . -type d) | while read dir; do \
#   curl -X MKCOL "https://your-cloudflare-url.trycloudflare.com/sources_on_webdav/${dir#./}"; \
# done
# Ensure 'sources_on_webdav' directory exists first on server or adapt path.

# Upload all *.c files from current local directory into '/sources_on_webdav/' (example from original, adapted):
# find . -maxdepth 1 -name '*.c' | xargs -P4 -I{} curl -T "{}" "https://your-cloudflare-url.trycloudflare.com/sources_on_webdav/{}"
#   -P4: Run up to 4 uploads in parallel.
```

## Internet-Wide Scanning & Reconnaissance

<!-- ETHICAL USE AND AUTHORIZATION REQUIRED. See previous warnings. -->
<!-- These tools query public databases or perform scans. Be mindful of rate limits and terms of service. -->

### Shodan InternetDB Quick Search
<!-- Shodan provides tools to explore internet-connected devices. InternetDB is one of their services for quick IP lookups. -->
<!-- This command queries Shodan's InternetDB for information about a specific IP address (e.g., open ports, vulnerabilities). -->
```bash
# TARGET_IP="your_target_ip" # Replace with the IP address you are authorized to investigate.
# curl https://internetdb.shodan.io/${TARGET_IP} | jq
#   curl .../${TARGET_IP}: Fetches data for the IP from InternetDB.
#   | jq: Pipes the JSON output to `jq` for pretty-printing and easier reading. (Install jq if needed: `sudo apt-get install jq`).
TARGET_IP="8.8.8.8" # Example IP (Google Public DNS)
curl -s https://internetdb.shodan.io/${TARGET_IP} | jq
```

### Nmap Scan with Vulners Script for Vulnerability Information
<!-- Uses nmap with the `vulners` NSE script to check for vulnerabilities based on discovered service versions. -->
<!-- Requires `nmap` and the `vulners` NSE script (often included or installable with nmap, or from https://github.com/vulnersCom/nmap-vulners). -->
<!-- Ensure you have permission to scan the target IP. -->
```bash
# sudo nmap -vv -A -F -Pn --min-rate 10000 --script vulners --script-timeout=5s <target_IP>
#   -vv: Very verbose output from nmap.
#   -A: Enable OS detection, version detection, script scanning, and traceroute (aggressive combination).
#   -F: Fast scan (fewer common ports than default full scan).
#   -Pn: Skip host discovery (treat host as online). Use if ICMP is blocked or target is known to be up.
#   --min-rate 10000: Send packets no slower than 10000 per second. EXTREMELY AGGRESSIVE.
#                     Can overwhelm networks/hosts or get you blocked. Use with extreme caution and authorization.
#                     A more reasonable rate: --min-rate 100 or remove for default rate.
#   --script vulners: Run the vulners script. (Original had .nse, often not needed to specify extension).
#   --script-timeout=5s: Limit script execution time. Original was 5s, might be too short for some scripts/hosts.
#                        Increase to e.g., '2m' for 2 minutes.
#   <target_IP>: The IP address to scan (must be authorized).
# Example with more moderate rate and timeout:
sudo nmap -vv -sV -Pn --min-rate 100 --script vulners --script-args mincvss=7.0 --script-timeout 2m <target_IP>
#   -sV: Service/version detection (needed for vulners to work effectively).
#   --script-args mincvss=7.0: Optional, tell vulners to only report vulnerabilities with CVSS score 7.0 or higher.
```

## Public IP and Geolocation Discovery

### Discover Your Public IP Address (Various Methods)
<!-- These commands query external services to determine your machine's public IP address as seen by the internet. -->
```bash
# Using wtfismyip.com (JSON output):
curl -s wtfismyip.com/json | jq '.YourFuckingIPAddress' # Extracts just the IP address string

# Using ifconfig.me:
curl ifconfig.me # Returns just the IP address as plain text

# Using OpenDNS resolvers with dig (requires dig utility):
dig +short myip.opendns.com @resolver1.opendns.com

# Using OpenDNS resolvers with host (requires host utility):
host myip.opendns.com resolver1.opendns.com | awk '{print $NF}' # $NF prints the last field (the IP)
```

### Discover Geolocation of an IP Address (Various Services)
<!-- These services provide geolocation information (country, city, ISP, etc.) for a given IP address. -->
```bash
# TARGET_IP_FOR_GEO="8.8.8.8" # Replace with the IP you want to geolocate.

# Using ipinfo.io (provides detailed JSON output):
curl -s https://ipinfo.io/${TARGET_IP_FOR_GEO} | jq

# Using ip-api.com (provides detailed JSON output):
curl -s http://ip-api.com/json/${TARGET_IP_FOR_GEO} | jq

# Using ipapi.co (another service with JSON output):
curl -s https://ipapi.co/${TARGET_IP_FOR_GEO}/json/ | jq

# Note: Geolocation accuracy varies, especially for city level. Country is usually accurate.
# These services may have rate limits for free use.
```
The key changes include:
*   Standardized headings and improved overall structure.
*   Added extensive comments in US English to clarify commands, options, and potential risks.
*   Ensured code blocks are correctly formatted.
*   Reviewed and significantly improved sections previously marked as "Messed Up" or "Incomplete," such as Encrypted Swap, Home Directory Backups, and various LVM/networking commands.
*   Provided context and safer alternatives for potentially dangerous commands (e.g., `find -exec rm`, `dd`, LVM shrinking).
*   Clarified the purpose and usage of complex tools like `rsync`, `tar` with various compressors, `gost`, `proxychains`, and `nmap` scripting.
*   Emphasized ethical considerations and the need for authorization for sections involving network probing, security assessment, and brute-force examples.
*   Made placeholders and examples more consistent and understandable.
*   Added explanations for different methods where applicable (e.g., random number generation, Wi-Fi scanning).

The file should now be much more readable, informative, and safer to use as a reference.

