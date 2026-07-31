# WiFi Security Auditing Project (WPA2 Handshake Cracking)

> ⚠️ **Disclaimer**: This project was conducted exclusively on my own personal WiFi network and hardware, in a controlled home lab environment, for educational purposes. No third-party networks were accessed. Unauthorized access to networks you do not own is illegal — always obtain explicit written permission before testing any network you do not personally control.

## Overview

**July 2026**

A hands-on penetration testing project to evaluate the security of a personal WPA2-protected wireless network by capturing a WPA handshake and attempting to recover the password via dictionary attack.

- Enabled monitor mode on a wireless adapter and surveyed nearby access points
- Captured a WPA2 4-way handshake from my own access point
- Performed a targeted deauthentication to force a handshake re-capture
- Cracked the captured handshake offline using `aircrack-ng` with the `rockyou.txt` wordlist
- Documented findings and password hygiene recommendations

## Methodology

1. **Reconnaissance** — Put the wireless card into monitor mode (`airmon-ng`) and scanned for nearby networks with `airodump-ng` to identify the target BSSID and channel
2. **Handshake Capture** — Locked `airodump-ng` onto the target channel/BSSID and captured a WPA handshake
3. **Deauthentication** — Sent deauth packets (`aireplay-ng`) to a connected client to force a reconnect and capture the handshake
4. **Offline Cracking** — Ran `aircrack-ng` against the captured `.cap` file using the `rockyou.txt` wordlist to attempt password recovery
5. **Remediation** — Reviewed results and switched the network to a stronger, non-dictionary passphrase (WPA2/WPA3)

## Tools & Technologies

- Kali Linux
- Aircrack-ng suite (`airmon-ng`, `airodump-ng`, `aireplay-ng`, `aircrack-ng`)
- Wireless network adapter with monitor mode / packet injection support
- `rockyou.txt` wordlist

## Key Skills

- Wireless network reconnaissance
- WPA/WPA2 handshake capture
- Deauthentication attacks
- Dictionary-based password cracking
- Wireless security hardening recommendations

## Disclosure

This project is intended for learning wireless security concepts. Always test only on networks and devices you own or have explicit authorization to assess.
