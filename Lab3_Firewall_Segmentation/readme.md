# Lab 03 — Firewall & Segmentation

![Status](https://img.shields.io/badge/Status-Completed-brightgreen)
![Type](https://img.shields.io/badge/Type-Defensive%20%2F%20Offensive-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Intermediate-orange)

---

## Overview

This lab implements and validates a **host-based firewall policy** on a Linux system using `ufw` and `iptables`. An adversarial machine (Kali Linux) simulates real-world attack patterns — port scanning and SSH brute force — while firewall rules, zone segmentation, and log analysis are used to detect and respond to threats.

> **Scope:** Isolated lab network (iPhone hotspot). All attack simulations were performed against a dedicated lab VM in a controlled environment. No third-party systems were targeted.

---

## Environment

| Component | Details |
|---|---|
| Defender Machine | Ubuntu VM (VirtualBox / MacBook Pro) — `lab-defender` |
| Attacker Machine | Lenovo ThinkPad X250 — Kali Linux (baremetal) |
| Management Station | MacBook Pro — documentation & monitoring |
| Network | iPhone Hotspot — 172.20.10.0/28 |
| VirtualBox Mode | Bridged Adapter (en0: WLAN) |

### Network Topology

```
iPhone Hotspot (172.20.10.0/28)
    ├── lab-defender   172.20.10.2   Ubuntu VM (VirtualBox)  ← defender
    ├── MacBook Pro    172.20.10.3   Management station
    └── Kali Linux     172.20.10.4   ThinkPad X250           ← attacker
```

### Zone Design

| Zone | Subnet | Trust Level | Policy |
|---|---|---|---|
| UNTRUSTED | 172.20.10.0/28 | Low | Whitelist — port 22 (Kali only) |
| TRUSTED | 10.10.10.0/24 | High | Allow all |

---

## Tools Used

| Tool | Purpose |
|---|---|
| `ufw` | Host-based firewall management |
| `iptables` | Underlying kernel-level rule inspection |
| `nmap` | Port scanning and service detection |
| `hydra` | SSH brute force simulation |
| `ss` | Local socket and port inspection |
| `journalctl` | SSH authentication log analysis |
| `tail -f` | Real-time log monitoring |
| `grep` + `awk` | Log filtering and pattern extraction |

---

## Phase 1: Firewall Baseline

### 1.1 ufw Activation

```bash
sudo ufw enable
sudo ufw status verbose
```

**Result:**
```
Status: active
Default: deny (incoming), allow (outgoing), disabled (routed)
```

Default deny policy established — all inbound traffic blocked unless explicitly allowed.

### 1.2 Port Inventory (Internal View)

```bash
ss -tlnp
```

**Listening services on lab-defender:**

| Port | Address | Service |
|---|---|---|
| 22 | 0.0.0.0 | SSH (OpenSSH) |
| 53 | 127.0.0.53 | DNS resolver (localhost only) |
| 631 | 127.0.0.1 | CUPS print service (localhost only) |

> **Key Insight:** SSH is listening on all interfaces internally — but ufw blocks external access until an explicit rule is added. Open port ≠ accessible port.

### 1.3 iptables Inspection

```bash
sudo iptables -L -n -v
```

ufw operates as an abstraction over iptables. Key chains confirmed:

| Chain | Default Policy | Description |
|---|---|---|
| INPUT | DROP | All inbound traffic dropped by default |
| FORWARD | DROP | Routing/forwarding disabled |
| OUTPUT | ACCEPT | All outbound traffic allowed |

ICMP (ping) allowed via `ufw-before-input` rules — explains why ping passes despite deny incoming policy.

---

## Phase 2: Port Scan

### 2.1 Initial Scan (No Rules)

```bash
# Attacker (Kali)
nmap -sS -p 1-1000 172.20.10.2
```

**Result:**
```
All 1000 scanned ports are in filtered state
```

Default DROP policy made the host completely invisible. nmap received no response and timed out on every port.

### 2.2 ufw Log — Port Scan Signature

```
[UFW BLOCK] SRC=172.20.10.4 DPT=135 PROTO=TCP SYN
[UFW BLOCK] SRC=172.20.10.4 DPT=995 PROTO=TCP SYN
[UFW BLOCK] SRC=172.20.10.4 DPT=23  PROTO=TCP SYN
[UFW BLOCK] SRC=172.20.10.4 DPT=554 PROTO=TCP SYN
```

Port scan signature: same `SRC`, sequential `DPT` values — classic nmap sweep pattern.

### 2.3 Allow Rule & Rescan

```bash
# Defender — IP-restricted rule
sudo ufw allow proto tcp from 172.20.10.4 to any port 22

# Attacker — rescan
nmap -sV --open -Pn 172.20.10.2
```

**Result after rule:**
```
PORT   STATE  SERVICE  VERSION
22/tcp open   ssh      OpenSSH 9.x
```

Port 22 immediately visible — firewall controls surface exposure directly.

### 2.4 Access Control Validation

| Source | SSH Attempt | Result |
|---|---|---|
| Kali (172.20.10.4) | `ssh vboxuser@172.20.10.2` | ✅ Connected |
| MacBook (172.20.10.3) | `ssh vboxuser@172.20.10.2` | ❌ Blocked |

Whitelist rule confirmed — only the authorized IP can reach port 22.

---

## Phase 3: Network Segmentation

### 3.1 Dummy Interface (Trusted Zone Simulation)

```bash
sudo ip link add dummy0 type dummy
sudo ip addr add 10.10.10.1/24 dev dummy0
sudo ip link set dummy0 up
```

Simulates a second network interface representing a trusted internal zone — no additional hardware required.

### 3.2 Zone-Based Firewall Policy

```bash
# TRUSTED zone — unrestricted access
sudo ufw allow from 10.10.10.0/24 to any
```

**Final rule set:**

```
[1] 22/tcp    ALLOW IN    172.20.10.4      ← UNTRUSTED: Kali only, SSH only
[2] Anywhere  ALLOW IN    10.10.10.0/24   ← TRUSTED: all ports allowed
```

---

## Phase 4: SSH Brute Force & Detection

### 4.1 Brute Force Simulation

```bash
# Attacker (Kali)
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://172.20.10.2 -t 4
```

| Flag | Meaning |
|---|---|
| `-l root` | Target username |
| `-P rockyou.txt` | Password wordlist (~14M entries) |
| `-t 4` | 4 parallel threads |

### 4.2 Detection — ufw Log

```bash
sudo grep "DPT=22" /var/log/ufw.log | grep "SRC=172.20.10.4"
```

```
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58470 SYN
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58468 SYN
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58472 SYN
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58466 SYN
```

Brute force signature: same `SRC`, same `DPT=22`, rotating `SPT` — each attempt opens a new source port.

### 4.3 Detection — SSH Auth Log

```bash
sudo journalctl _SYSTEMD_UNIT=ssh.service | grep "Failed" | tail -20
```

```
Mar 20 15:01:21 lab-defender sshd[5510]: Failed password for root from 172.20.10.4 port 44046 ssh2
Mar 20 15:01:21 lab-defender sshd[5508]: Failed password for root from 172.20.10.4 port 44026 ssh2
Mar 20 15:01:21 lab-defender sshd[5507]: Failed password for root from 172.20.10.4 port 44010 ssh2
Mar 20 15:01:21 lab-defender sshd[5509]: Failed password for root from 172.20.10.4 port 44042 ssh2
```

4 failed attempts per second — mechanical, non-human timing. `-t 4` thread count confirmed in log pattern.

### 4.4 Response — Rate Limiting

```bash
sudo ufw limit proto tcp from any to any port 22
```

`ufw limit` blocks connections from any IP exceeding 6 attempts in 30 seconds.

**Result:**
```
[UFW BLOCK] SRC=172.20.10.4 DPT=22 SPT=52904
[UFW BLOCK] SRC=172.20.10.4 DPT=22 SPT=52904
[UFW BLOCK] SRC=172.20.10.4 DPT=22 SPT=52904
```

Hydra connection rate dropped significantly — SPT stopped rotating, confirming new sessions were blocked before authentication.

---

## Findings

### Finding 1 — Default DROP Makes Host Invisible
**Severity:** Informational (expected behavior)
**Detail:** With no allow rules, all 1000 scanned ports returned `filtered`. The host was undetectable from outside — attackers receive no feedback, making reconnaissance significantly harder.

### Finding 2 — Port Exposure is Firewall-Controlled
**Severity:** High (if misconfigured)
**Detail:** Adding a single `allow 22/tcp` rule immediately exposed port 22 to nmap. Firewall rules directly control the visible attack surface — overly permissive rules expand exposure instantly.

### Finding 3 — Brute Force Detectable Without IDS
**Severity:** Medium
**Detail:** SSH brute force was identified through ufw and journalctl logs without any dedicated IDS. Key indicators: rotating SPT on fixed DPT, mechanical timing, repeated "Failed password" entries.

### Finding 4 — Rate Limiting Effective Against Hydra
**Severity:** Informational
**Detail:** `ufw limit` successfully degraded hydra's throughput. Not a full countermeasure — full protection requires fail2ban or IDS-based blocking.

### Finding 5 — ICMP Bypasses Default Deny
**Severity:** Low
**Detail:** Despite `deny (incoming)` policy, ping passed through. ufw's `before.rules` explicitly allows ICMP echo (type 8). A hardened system may want to block ICMP explicitly.

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID | Description |
|---|---|---|---|
| Reconnaissance | Network Service Discovery | T1046 | nmap SYN scan ports 1–1000 |
| Credential Access | Brute Force: Password Spraying | T1110.003 | hydra + rockyou.txt against SSH |
| Credential Access | Valid Accounts (attempted) | T1078 | root account targeted |

---

## Lessons Learned

1. **Open port ≠ accessible port** — a service can listen internally while the firewall blocks all external access. `ss` and `nmap` show two different realities.
2. **Default deny is the foundation** — DROP policy makes a host invisible; every exposed port is a deliberate decision.
3. **ufw is iptables** — learning ufw means learning iptables in disguise. The kernel rules are identical.
4. **Brute force is identifiable without IDS** — rotating SPT on a fixed DPT, combined with journalctl "Failed password" entries, is sufficient to confirm automated attack.
5. **Rate limiting buys time, not immunity** — `ufw limit` degrades brute force effectiveness but does not stop a patient attacker. fail2ban or Snort/Suricata is the next layer.
6. **Zone segmentation is a mindset** — trusted vs untrusted shapes how you think about every network decision in SOC work.

---

## Recommendations

| Priority | Action |
|---|---|
| 🔴 High | Deploy fail2ban alongside ufw for persistent IP banning |
| 🔴 High | Disable root SSH login — use key-based auth only |
| 🟡 Medium | Deploy IDS (Snort/Suricata) for real-time detection — next lab |
| 🟡 Medium | Change default SSH port to reduce automated scan noise |
| 🟢 Low | Enable ufw logging at `medium` permanently |
| 🟢 Low | Review firewall rules periodically — remove unused allow entries |

---

## References

- [ufw Manual](https://manpages.ubuntu.com/manpages/focal/man8/ufw.8.html)
- [iptables Documentation](https://netfilter.org/documentation/)
- [MITRE ATT&CK T1046](https://attack.mitre.org/techniques/T1046/)
- [MITRE ATT&CK T1110](https://attack.mitre.org/techniques/T1110/)
- [Hydra — THC](https://github.com/vanhauser-thc/thc-hydra)

---

*Lab completed: 2026-03-20 | Author: cyb-ersin | Platform: Ubuntu VM + Kali Linux*
