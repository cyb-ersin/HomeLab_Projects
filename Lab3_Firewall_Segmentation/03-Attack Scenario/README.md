# 💥 Attack Scenario

![Phase1](https://img.shields.io/badge/Phase%201-Port%20Scan-orange)
![Phase2](https://img.shields.io/badge/Phase%202-SSH%20Brute%20Force-red)
![Attacker](https://img.shields.io/badge/Attacker-172.20.10.4-red)
![Defender](https://img.shields.io/badge/Defender-172.20.10.2-green)

> **Attacker:** Kali Linux — ThinkPad X250 (`172.20.10.4`)  
> **Defender:** lab-defender — Ubuntu VM (`172.20.10.2`)  
> **Date:** 2026-03-20

---

## Overview

Two sequential attack phases were conducted against the defender machine to validate firewall policy and practice log-based threat detection.

| Phase | Type | Tool | Target |
|---|---|---|---|
| 1 | Reconnaissance | `nmap -sS` | `172.20.10.2:1-1000` |
| 2 | Credential Access | `hydra` + `rockyou.txt` | `172.20.10.2:22` |

---

## Phase 1 — Port Scan

### 1.1 Pre-Rule Scan

```bash
nmap -sS -p 1-1000 172.20.10.2
```

**Result:**
```
All 1000 scanned ports are in filtered state
```

Default DROP policy made the host completely invisible. ufw silently discarded every packet — nmap received no response and timed out on all ports.

> **Key Insight:** `filtered` ≠ `closed`. Filtered means no response — the host appears to not exist. Closed means the host responded with RST. This distinction matters in real threat hunting.

### 1.2 ufw Log — Port Scan Signature

```
[UFW BLOCK] SRC=172.20.10.4 DPT=135 PROTO=TCP SYN
[UFW BLOCK] SRC=172.20.10.4 DPT=995 PROTO=TCP SYN
[UFW BLOCK] SRC=172.20.10.4 DPT=23  PROTO=TCP SYN
[UFW BLOCK] SRC=172.20.10.4 DPT=554 PROTO=TCP SYN
[UFW BLOCK] SRC=172.20.10.4 DPT=199 PROTO=TCP SYN
```

**Port scan signature:** same `SRC`, rapidly changing `DPT` — classic nmap sweep pattern.

### 1.3 Allow Rule Added

```bash
# Defender
sudo ufw allow proto tcp from 172.20.10.4 to any port 22
```

### 1.4 Post-Rule Scan

```bash
nmap -sV --open -Pn 172.20.10.2
```

**Result:**
```
PORT   STATE  SERVICE  VERSION
22/tcp open   ssh      OpenSSH 9.x
```

Port 22 immediately visible — firewall directly controls the attack surface.

### 1.5 Access Control Test

| Source | Command | Result |
|---|---|---|
| Kali `172.20.10.4` | `ssh vboxuser@172.20.10.2` | ✅ Connected |
| MacBook `172.20.10.3` | `ssh vboxuser@172.20.10.2` | ❌ Blocked |

Whitelist confirmed — only the authorized IP reaches port 22.

---

## Phase 2 — SSH Brute Force

### 2.1 Setup

SSH service confirmed running on lab-defender:

```bash
ss -tlnp | grep 22
# 0.0.0.0:22   LISTEN   (sshd)
```

### 2.2 Brute Force Execution

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://172.20.10.2 -t 4
```

### 2.3 Detection — ufw Log

```bash
sudo grep "DPT=22" /var/log/ufw.log | grep "SRC=172.20.10.4"
```

```
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58470 SYN
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58468 SYN
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58472 SYN
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58466 SYN
```

**Brute force signature:**

| Indicator | Value | Meaning |
|---|---|---|
| `SRC` | `172.20.10.4` (fixed) | Single attacker |
| `DPT` | `22` (fixed) | Single target port |
| `SPT` | rotating | Each attempt = new connection |
| Frequency | 4 per second | Matches `-t 4` thread count |

### 2.4 Detection — SSH Auth Log

```bash
sudo journalctl _SYSTEMD_UNIT=ssh.service | grep "Failed" | tail -20
```

```
Mar 20 15:01:21 lab-defender sshd[5510]: Failed password for root from 172.20.10.4 port 44046 ssh2
Mar 20 15:01:21 lab-defender sshd[5508]: Failed password for root from 172.20.10.4 port 44026 ssh2
Mar 20 15:01:21 lab-defender sshd[5507]: Failed password for root from 172.20.10.4 port 44010 ssh2
Mar 20 15:01:21 lab-defender sshd[5509]: Failed password for root from 172.20.10.4 port 44042 ssh2
Mar 20 15:01:25 lab-defender sshd[5518]: Failed password for root from 172.20.10.4 port 40828 ssh2
```

4 failed attempts at `15:01:21`, 4 more at `15:01:25` — mechanical, non-human timing. Automated attack confirmed.

### 2.5 Response — Rate Limiting

```bash
sudo ufw limit proto tcp from any to any port 22
```

**Result:**
```
[UFW BLOCK] SRC=172.20.10.4 DPT=22 SPT=52904
[UFW BLOCK] SRC=172.20.10.4 DPT=22 SPT=52904
[UFW BLOCK] SRC=172.20.10.4 DPT=22 SPT=52904
```

SPT stopped rotating — new sessions blocked before authentication. Hydra throughput dropped significantly.

---

## Attack Timeline

| Time | Event |
|---|---|
| `14:49:24` | nmap SYN scan begins — all ports filtered |
| `14:49:xx` | `allow 22/tcp` rule added |
| `14:49:xx` | Rescan — port 22 visible, OpenSSH detected |
| `14:59:37` | hydra brute force begins — 4 threads |
| `15:01:21` | journalctl: 4x `Failed password` per second |
| `15:21:xx` | `ufw limit` applied |
| `15:21:xx` | UFW BLOCK on DPT=22 — hydra rate-limited |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID | Tool |
|---|---|---|---|
| Reconnaissance | Network Service Discovery | T1046 | nmap -sS |
| Credential Access | Brute Force: Password Spraying | T1110.003 | hydra + rockyou.txt |
| Credential Access | Valid Accounts (attempted) | T1078 | root account targeted |
