# 🔍 Evidence — SSH Brute Force

![Type](https://img.shields.io/badge/Type-Credential%20Access-red)
![Tool](https://img.shields.io/badge/Tool-hydra-lightgrey)
![Source](https://img.shields.io/badge/Source-172.20.10.4-red)
![Target](https://img.shields.io/badge/Target-172.20.10.2%3A22-green)

> **Date:** 2026-03-20 · **Time:** ~14:59–15:21 UTC

---

## 📸 Screenshots

> Add screenshots here:
> - `hydra-running.png` — hydra terminal output during attack
> - `ufw-audit-ssh.png` — ufw.log showing AUDIT entries on DPT=22
> - `journalctl-failed.png` — Failed password entries from SSH auth log
> - `ufw-block-ratelimit.png` — UFW BLOCK entries after rate limiting applied

---

## 📋 ufw Log — Brute Force Signature

Filtered log output — only DPT=22 from attacker:

```bash
sudo grep "DPT=22" /var/log/ufw.log | grep "SRC=172.20.10.4"
```

```
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58470 SYN
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58468 SYN
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58472 SYN
[UFW AUDIT] SRC=172.20.10.4 DPT=22 SPT=58466 SYN
```

---

## 📋 SSH Auth Log — Failed Passwords

```bash
sudo journalctl _SYSTEMD_UNIT=ssh.service | grep "Failed" | tail -20
```

```
Mar 20 15:01:21 lab-defender sshd[5510]: Failed password for root from 172.20.10.4 port 44046 ssh2
Mar 20 15:01:21 lab-defender sshd[5508]: Failed password for root from 172.20.10.4 port 44026 ssh2
Mar 20 15:01:21 lab-defender sshd[5507]: Failed password for root from 172.20.10.4 port 44010 ssh2
Mar 20 15:01:21 lab-defender sshd[5509]: Failed password for root from 172.20.10.4 port 44042 ssh2
Mar 20 15:01:25 lab-defender sshd[5518]: Failed password for root from 172.20.10.4 port 40828 ssh2
Mar 20 15:01:25 lab-defender sshd[5516]: Failed password for root from 172.20.10.4 port 40800 ssh2
Mar 20 15:01:25 lab-defender sshd[5515]: Failed password for root from 172.20.10.4 port 40788 ssh2
Mar 20 15:01:25 lab-defender sshd[5517]: Failed password for root from 172.20.10.4 port 40814 ssh2
```

---

## 🔎 Log Field Analysis

| Field | Value | Meaning |
|---|---|---|
| `[UFW AUDIT]` | — | Kali is whitelisted — packet logged, not blocked |
| `SRC` | `172.20.10.4` (fixed) | Single attacker IP |
| `DPT` | `22` (fixed) | Single target — SSH only |
| `SPT` | rotating | Each attempt opens a new TCP session |
| Frequency | 4/second | Matches hydra `-t 4` thread count |

---

## 🚨 Detection Indicators

| Indicator | Description |
|---|---|
| Fixed `SRC` + fixed `DPT` | Targeted attack on specific service |
| Rotating `SPT` | Each password attempt = new connection |
| Mechanical timing | 4 attempts every ~4 seconds — not human |
| `Failed password for root` | High-privilege account targeted |
| `sshd` PID rotation | New process spawned per attempt |

**Verdict:** Automated SSH brute force — hydra signature confirmed via both ufw log and SSH auth log.

---

## 📋 Rate Limit Response Log

After `ufw limit` applied:

```
[UFW BLOCK] SRC=172.20.10.4 DPT=22 SPT=52904
[UFW BLOCK] SRC=172.20.10.4 DPT=22 SPT=52904
[UFW BLOCK] SRC=172.20.10.4 DPT=22 SPT=52904
```

**Observation:** SPT stopped rotating — hydra could no longer establish new sessions. Rate limit triggered after threshold exceeded (6 connections / 30 seconds).

---

## 🔎 Port Scan vs Brute Force — Log Comparison

| | Port Scan | Brute Force |
|---|---|---|
| `SRC` | Fixed | Fixed |
| `DPT` | **Rotating** | **Fixed (22)** |
| `SPT` | Fixed | **Rotating** |
| Log tag | `BLOCK` | `AUDIT` (Kali whitelisted) |
| Auth log | No entries | `Failed password` entries |
| Verdict | Reconnaissance | Credential access |
