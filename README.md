# Home Lab Projects

## About 

Hands-on networking and cybersecurity lab projects.
Built alongside CompTIA Net+ studies to develop practical skills.

## Network Architecture :

```mermaid
graph TD
    Internet[Internet] --> FritzBox[Fritz!Box<br>192.168.178.1<br>Gateway]
    FritzBox --> MacBook[MacBook Pro<br>192.168.178.111<br>Management Station]
    FritzBox --> Repeater[TP-Link RE190<br>Repeater]
    Repeater --> iMac[iMac 12,1<br>Lab Server<br>32GB RAM · Ubuntu]
    Repeater --> iPad[iPad]
    Repeater --> Others[Others]
    FritzBox -.- Kali[Kali Laptop<br>Pentesting]
    FritzBox -.- Ubuntu[Ubuntu Laptop<br>Linux Practice]
```

## Lab Environment      
- MacBook Pro 16GB RAM, 500GB SSD (management station)
- iMac 12,1 (Mid 2011) 32GB RAM, 500GB HDD, Ubuntu (lab server)
- Fritz!Box Router (home network)

## Projects
| # | Project | Status | Topics Covered |
|---|---------|--------|----------------|
| 0 | Network Discovery | in progress | IP addressing, network topology, scanning |
