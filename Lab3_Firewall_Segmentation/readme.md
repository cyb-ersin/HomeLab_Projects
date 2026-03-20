# Lab 03 — Firewall & Segmentation

> **Series:** homelab-foundation  
> **Status:** Complete  
> **Date:** 2026-03-20  
> **Platform:** Physical hardware + VirtualBox (Bridged)

---

## Objective

Design and validate a host-based firewall policy on a Linux system. Simulate real-world attack patterns from an adversarial machine and analyze firewall logs to identify and respond to threats.

---

## Lab Environment

```
iPhone Hotspot (172.20.10.0/28)
    ├── lab-defender   172.20.10.2   Ubuntu VM (VirtualBox/MacBook Pro)  ← defender
    ├── MacBook Pro    172.20.10.3   Management station / documentation
    └── Kali Linux     172.20.10.4   ThinkPad X250 (baremetal)           ← attacker
```

### Zone Design

| Zone | Subnet | Trust Level | Policy |
|---|---|---|---|
| UNTRUSTED | 172.20.10.0/28 | Low | Whitelist — port 22 only (Kali) |
| TRUSTED | 10.10.10.0/24 | High | Allow all |

---

## Attack Scenarios

| # | Scenario | Tool | Target |
|---|---|---|---|
| 1 | Port Scan | nmap -sS | lab-defender:1-1000 |
| 2 | SSH Brute Force | hydra + rockyou.txt | lab-defender:22 |

---

## Key Findings

- `deny (incoming)` default policy rendered the host invisible to nmap — all ports returned `filtered`
- After `allow 22/tcp`: port 22 became visible, confirming firewall controls surface exposure
- IP-restricted rule (`allow from 172.20.10.4`) blocked MacBook access while Kali connected successfully
- Brute force signature identified in `journalctl`: same source IP, same target port, rotating source ports, 4 parallel attempts per second
- `ufw limit` rate limiting automatically blocked repeated SSH attempts after threshold

---

## Structure

```
lab-03-firewall-segmentation/
├── 01-architecture/
│   ├── lab-topology.md       — network diagram and zone design
│   └── zone-design.md        — trust model and policy rationale
├── 02-assets/
│   ├── lab-defender/         — defender machine config and firewall rules
│   └── kali-attacker/        — attack tools and commands used
├── 03-attack-scenario/
│   └── README.md             — port scan and brute force walkthrough
├── 04-evidences/
│   ├── port-scan-logs/       — nmap output and ufw BLOCK logs
│   └── brute-force-logs/     — hydra run and journalctl Failed password
└── 05-lessons-learned/
    └── README.md             — takeaways, SOC relevance, next steps
```

---

## Tools Used

| Tool | Purpose |
|---|---|
| ufw | Host-based firewall management |
| iptables | Underlying rule inspection |
| nmap | Port scanning and service detection |
| hydra | SSH brute force simulation |
| ss | Local socket and port inspection |
| journalctl | SSH authentication log analysis |
| tail -f | Real-time log monitoring |

---

## Lessons Learned

- Open port ≠ accessible port — firewall controls the gap between the two
- Default DROP makes a host invisible to external scanners (filtered state)
- ufw operates as an abstraction over iptables — same rules, different syntax
- Brute force is identifiable without IDS: same SRC, same DPT, rotating SPT, mechanical timing
- Rate limiting (`ufw limit`) is a lightweight first-response to brute force — not a replacement for fail2ban or IDS

---

*Next: Lab 04 — IDS & SIEM (Snort + Graylog)*
