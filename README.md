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



สแกนหา AP เป้าหมายก่อน
รัน sudo airodump-ng wlan1mon --band a (หรือเปล่าเปล่าเข้ามาก่อนเปิด monitor mode) เพื่อส่องหาเป้าหมาย จจำ BSSID, channel (CH), และ ESSID ของเป้าหมายที่จะเจาะไว้ใช้ต่อ

เปิด monitor mode
รัน sudo airmon-ng check kill (ปิด process ที่จะมาแซง monitor mode) แล้ว sudo airmon-ng start wlan1 เสร็จแล้วเช็คด้วย iwconfig ควรเห็น interface ชื่อ wlan1mon โหมดขึ้นเป็น Mode:Monitor แล้ว (ไม่ใช่ wlan0/wlan0mon)

ล็อกเป้าด้วย airodump-ng
รัน sudo airodump-ng -c <channel จากข้อ 1> --bssid <BSSID เป้าหมาย> -w capture wlan1mon แล้วปล่อยค้างไว้ (อย่าปิด) เพื่อดักฟังและบันทึกแพ็กเก็ตที่เกี่ยวข้องกับ AP นี้ลงไฟล์ capture-01.cap เป้าหมายจะหลุดออกทันทีหลังที่เจอ

ยิง deauth บังคับ reconnect
เปิดหน้าต่างใหม่ รัน sudo aireplay-ng -0 2 -a <BSSID> -c <client MAC จากตารัง STATION ในหน้าต่างก่อน> wlan1mon เพื่อปลอม deauth เตะ client ออกจาก AP ชั่วครู่ ให้ client reconnect เองอัตโนมัติ

เช็คยืนยันว่าจับ handshake ได้จรง
กลับไปดูหน้าต่าง airodump-ng ควรเห็นข้อความ 'WPA handshake: <BSSID>' ขึ้นที่มุมขวาบน และตาราง STATION มีคอละมน์ EAPOL ขึ้นเป็นหลักฐานว่าจับ handshake โดยสมบูรณ์จำเป็นก่อนจะกด Ctrl+C หยุด capture

Crack ด้วย aircrack-ng
รัน aircrack-ng capture-01.cap -w /usr/share/wordlists/rockyou.txt ไล้ทดเดารหัสจากลิสต์เทียบ MIC จนกว่าจะเจอคำที่ตรง (ถ้าเจอ error ว่า ESSID is required เพราะไม่มี beacon frame ติดมา ให้เตีม -e <ชื่อ WiFi> ท้ายคำสั่ง)


## Photo
หา BSSID: คอลัมน์นี้คือ MAC Address ของ Router (Access Point) แต่ละตัว เช่น 66:20:E1:18:2F:0E
CH (Channel): คอลัมน์นี้คือ ช่องสัญญาณ ที่ Router ตัวนั้นกำลังทำงานอยู่ เช่น ช่อง 7, 5, 3
ESSID: คือชื่อของ Wi-Fi (Network Name) ที่เราคุ้นเคย เช่น 5817_2.4G
ENC / AUTH: บอกถึงประเภทการเข้ารหัสความปลอดภัย ซึ่งในภาพส่วนใหญ่เป็น WPA2 PSK
<img width="1477" height="1108" alt="21440_0" src="https://github.com/user-attachments/assets/5f24f2ad-1590-47f6-b1f3-b23626330452" />

<img width="1477" height="1108" alt="21438_0" src="https://github.com/user-attachments/assets/9f0cecc0-1279-4b2b-bdfe-8259ade9f63f" />

<img width="1280" height="720" alt="21439_0" src="https://github.com/user-attachments/assets/2b4e7310-171c-4acc-8aa7-56322eaa0b5b" />


