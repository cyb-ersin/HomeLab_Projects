# 🗺️ Lab Topology

![Network](https://img.shields.io/badge/Network-172.20.10.0%2F28-blue)
![AP](https://img.shields.io/badge/AP-iPhone%20Hotspot-lightgrey)
![Zones](https://img.shields.io/badge/Zones-Trusted%20%7C%20Untrusted-orange)

> **Lab:** 03 — Firewall & Segmentation | **Date:** 2026-03-20

---

## 🌐 Network Layout

| Role | Device | IP Address | OS | Interface |
|---|---|---|---|---|
| 🛡️ **DEFENDER** | `lab-defender` | `172.20.10.2` | Ubuntu VM (VirtualBox) | enp0s3 · Bridged |
| 💻 **MANAGEMENT** | `MacBook Pro` | `172.20.10.3` | macOS | en0 · WLAN |
| ⚔️ **ATTACKER** | `Kali Linux` | `172.20.10.4` | Kali Linux · ThinkPad X250 | wlan0 |
| 📱 **GATEWAY** | `iPhone Hotspot` | `172.20.10.1` | — | — |

---

## 🔥 Firewall Zone Design

### 🔴 UNTRUSTED ZONE — `172.20.10.0/28`

| Source | Destination | Port | Action |
|---|---|---|---|
| `172.20.10.4` (Kali) | `172.20.10.2` (lab-defender) | `22/tcp` | ✅ ALLOW |
| `172.20.10.3` (MacBook) | `172.20.10.2` (lab-defender) | `22/tcp` | ❌ BLOCK |
| `172.20.10.0/28` (any) | `172.20.10.2` (lab-defender) | all others | ❌ BLOCK (default deny) |

### 🟢 TRUSTED ZONE — `10.10.10.0/24`

| Source | Destination | Port | Action |
|---|---|---|---|
| `10.10.10.0/24` (any) | `172.20.10.2` (lab-defender) | all | ✅ ALLOW |

> Trusted zone simulated via `dummy0` interface — no additional hardware required.

---

## ⚙️ ufw Rule Set

```
Status: active
Default: deny (incoming) · allow (outgoing)

[ 1] 22/tcp    ALLOW IN    172.20.10.4      ← Kali only
[ 2] Anywhere  ALLOW IN    10.10.10.0/24   ← Trusted zone
```

---

## 🔌 VirtualBox — Bridged Mode

| Mode | VM IP | Visible to Kali? | Used |
|---|---|---|---|
| NAT | `10.0.2.x` (hidden) | ❌ No | — |
| **Bridged** | `172.20.10.2` (hotspot) | ✅ Yes | ✅ This lab |

> Bridged Adapter (`en0: WLAN`) places the VM directly on the hotspot network.  
> The VM gets its own DHCP lease — fully reachable by Kali for real firewall testing.

---

## 📡 Network Parameters

| Parameter | Value |
|---|---|
| Subnet | `172.20.10.0/28` |
| Mask | `255.255.255.240` |
| Usable Hosts | 14 |
| Broadcast | `172.20.10.15` |
| Gateway | `172.20.10.1` |

> **/28** — iPhone hotspot default. 14 usable addresses, sufficient for a 3-device lab.
