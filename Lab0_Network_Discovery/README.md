# Lab 0: Network Discovery & Port Analysis

## Objective
Discover all devices on my home network, map the topology, scan for open ports, and perform a basic security assessment using command-line tools.

## Tools Used
- ifconfig (macOS network configuration)
- ip addr show (Linux network configuration)
- arp -a (ARP table inspection)
- ping / broadcast ping
- netstat (routing table)
- nmap -sn (host discovery)
- nmap -sV (service version detection)
- ufw status (firewall check)

## Date
March 12, 2026

## Network Overview
- **Network:** 192.168.178.0/24
- **Subnet Mask:** 255.255.255.0
- **Gateway:** 192.168.178.1 (Fritz!Box by AVM)
- **DHCP:** Enabled (assigned by Fritz!Box)

## Phase 1: Device Discovery

### Method 1 — ARP Table (arp -a)
Initial scan showed multiple devices sharing the same MAC address. This was suspicious and required further investigation.

### Method 2 — Broadcast Ping
Broadcast ping to 192.168.178.255 forced all devices to respond, revealing hostnames that were not visible in the initial ARP scan. This identified a Wi-Fi Repeater as the source of the shared MAC address.

### Method 3 — Nmap Host Discovery (nmap -sn)
Full subnet scan confirmed 7 active hosts.

**Discovered Devices:**

| Device | IP Address | MAC Address | Connection |
|--------|-----------|-------------|------------|
| Fritz!Box | 192.168.178.1 | 38:10:D5:xx:xx:xx (AVM) | Direct — Gateway |
| MacBook Pro | 192.168.178.111 | 7a:7d:0f:xx:xx:xx | Direct to Fritz!Box |
| Wi-Fi Repeater | 192.168.178.80 | 9A:03:8E:xx:xx:xx | Direct to Fritz!Box |
| iMac Lab Server | 192.168.178.78 | via Repeater | Through Repeater |
| Device-01 | 192.168.178.106 | via Repeater | Through Repeater |
| Device-02 | 192.168.178.112 | via Repeater | Through Repeater |
| Device-03 | 192.168.178.97 | via Repeater | Through Repeater |

## Network Topology
```mermaid
