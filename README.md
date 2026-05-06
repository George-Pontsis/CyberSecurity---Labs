# 🛡️ Cybersecurity Labs

A personal collection of hands-on cybersecurity labs covering both offensive and defensive techniques. Each lab is built in an isolated virtual environment and documented with full writeups, commands, and screenshots.

---

## 👨‍💻 About

These labs are built for learning and practicing real-world cybersecurity skills including Active Directory attacks, network security, SIEM detection, and more. Every lab follows a Red Team → Blue Team approach — attack first, then detect and defend.

**Skills practiced:**
- Active Directory enumeration and exploitation
- Password cracking and hash analysis
- SIEM log analysis and alert creation
- Network traffic analysis
- Vulnerability assessment

---

## 🗂️ Labs

| Lab | Category | Status | Tools Used |
|---|---|---|---|
| [Active Directory Lab](Labs/Active-Directory-Lab/README.md) | Red & Blue Team | ✅ Completed | VirtualBox, Windows Server 2022, Kali Linux, Splunk, Impacket, Hashcat |

> More labs coming soon — see the roadmap below.

---

## 🔧 Lab Environment

All labs are built using:

- **Hypervisor:** Oracle VirtualBox 7.x
- **Host Machine:** Intel Core i7-14700K, 32 GB RAM
- **Network:** Isolated internal networks (no internet exposure)
- **Attacker OS:** Kali Linux (Rolling)
- **Defender/SIEM:** Splunk Enterprise (Free tier)

---

## 🗺️ Roadmap

### Active Directory Series
- [x] Lab Setup — Domain Controller, Windows 10 Client, Kali, Splunk SIEM
- [x] Kerberoasting Attack + Detection
- [ ] Password Spraying + Detection
- [ ] AS-REP Roasting + Detection
- [ ] BloodHound AD Enumeration
- [ ] Pass the Hash
- [ ] DCSync Attack
- [ ] LLMNR/NBT-NS Poisoning with Responder

### Network Security Series
- [ ] Wireshark Traffic Analysis
- [ ] Man-in-the-Middle Attack
- [ ] Network Scanning with Nmap

### Web Application Series
- [ ] OWASP Top 10 Practice
- [ ] SQL Injection
- [ ] Burp Suite Basics

---

## 📁 Repository Structure

```
CyberSecurity---Labs/
└── Labs/
    └── Active-Directory-Lab/
        ├── README.md
        └── screenshots/
```

---

## ⚠️ Disclaimer

All labs and techniques documented here are performed in isolated, self-contained virtual environments for **educational purposes only**. Nothing here should be used against systems without explicit permission.

---

## 📬 Connect

- **GitHub:** [George-Pontsis](https://github.com/George-Pontsis)

---

*Built from scratch — every command typed, every error fixed, every attack detected.* 🎯
