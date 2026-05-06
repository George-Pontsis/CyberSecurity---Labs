# Active Directory Home Lab

![Platform](https://img.shields.io/badge/Platform-VirtualBox-blue)
![OS](https://img.shields.io/badge/OS-Windows%20Server%202022-blue)
![Attacker](https://img.shields.io/badge/Attacker-Kali%20Linux-red)
![SIEM](https://img.shields.io/badge/SIEM-Splunk-orange)
![Status](https://img.shields.io/badge/Status-Active-green)

A fully functional Active Directory home lab built for Red Team and Blue Team cybersecurity practice. This lab simulates a real enterprise environment and is used to practice offensive attack techniques and defensive detection strategies.

---

## Lab Architecture

| VM | OS | Role | IP Address |
|---|---|---|---|
| DC01 | Windows Server 2022 | Domain Controller (lab.local) | 192.168.10.10 |
| WS01 | Windows 10 Pro | Domain-joined workstation | 192.168.10.20 |
| Kali | Kali Linux (Rolling) | Attacker / Red Team machine | 192.168.10.30 |
| SIEM | Ubuntu Server 24.04 | Splunk log collector | 192.168.10.40 |

All VMs are connected on an isolated VirtualBox Internal Network (`adlab` / `192.168.10.0/24`) with no direct internet access, simulating a segmented enterprise network.

### Network Diagram

```
┌─────────────────────────────────────────────────────┐
│           VirtualBox Internal Network                │
│                  192.168.10.0/24                     │
│                                                      │
│   ┌──────────┐          ┌──────────┐                 │
│   │  DC01    │◄────────►│  WS01    │                 │
│   │ .10.10   │  AD Auth │ .10.20   │                 │
│   └────┬─────┘          └────┬─────┘                 │
│        │ Logs                │ Logs                  │
│        ▼                     ▼                       │
│   ┌──────────┐          ┌──────────┐                 │
│   │  SIEM    │          │  Kali    │                 │
│   │ .10.40   │◄────────►│ .10.30   │                 │
│   │ Splunk   │  Attacks │ Attacker │                 │
│   └──────────┘          └──────────┘                 │
└─────────────────────────────────────────────────────┘
```

---

## Host Machine Specs

| Component | Spec |
|---|---|
| CPU | Intel Core i7-14700K |
| RAM | 32 GB DDR5 |
| Hypervisor | Oracle VirtualBox 7.x |
| Storage | SSD (150 GB allocated to VMs) |

---

## Domain Configuration

- **Domain:** `lab.local`
- **NetBIOS Name:** `LAB`
- **Domain Controller:** DC01 (Windows Server 2022)
- **DNS:** DC01 acts as primary DNS for all VMs

### Domain Users

| Username | Full Name | Role | Notes |
|---|---|---|---|
| Administrator | Built-in Admin | Domain Admin | Default DC account |
| alice | Alice Smith | Domain Admin | Privileged user |
| bob | Bob Jones | Standard User | Low-privilege user |
| sqlsvc | SQL Service | Service Account | Has SPN set — Kerberoastable |

### Service Principal Names (SPNs)

```
MSSQLSvc/dc01.lab.local:1433 → sqlsvc
```

---

## Tools Installed

### Red Team (Kali Linux)
| Tool | Purpose |
|---|---|
| Impacket | AD attack suite (Kerberoasting, secretsdump, etc.) |
| NetExec (formerly CrackMapExec) | Network enumeration and exploitation |
| BloodHound.py | AD enumeration and attack path mapping |
| Hashcat | Offline password cracking |
| SecLists | Wordlist collection for password attacks |

### Blue Team (Splunk SIEM)
| Component | Purpose |
|---|---|
| Splunk Enterprise (Free) | Log aggregation and analysis |
| Splunk Universal Forwarder | Installed on DC01 and WS01 to forward logs |
| Windows Security Event Logs | Primary data source (Event IDs 4624, 4625, 4769, etc.) |

---

## Attacks & Detections

### 1. Kerberoasting

**Status:** ✅ Completed

**Description:**
Kerberoasting is an Active Directory attack that targets service accounts with Service Principal Names (SPNs). Any authenticated domain user can request a Kerberos TGS ticket for a service account. The ticket is encrypted with the service account's password hash and can be taken offline for cracking.

**Attack Steps:**

1. Enumerate domain users and identify Kerberoastable accounts:
```bash
impacket-GetUserSPNs lab.local/bob:Password1! -dc-ip 192.168.10.10 -request
```

2. Extract the TGS hash from the output:
```
$krb5tgs$23$*sqlsvc$LAB.LOCAL$lab.local/sqlsvc*$...
```

3. Save the hash and crack it offline with Hashcat:
```bash
hashcat -m 13100 hash.txt wordlist.txt --force
```

4. Recovered password: `MysqlPassword1!`

**Detection (Splunk):**

Kerberoasting generates Event ID 4769 with RC4 encryption type (0x17) instead of the expected AES encryption.

Detection query:
```
index=* sourcetype="WinEventLog:Security" EventCode=4769 Ticket_Encryption_Type=0x17
```

**Alert Created:** `Kerberoasting Detected` — Real-time, High severity

**Key Takeaways:**
- Any domain user can perform this attack — no special privileges required
- Service accounts must use long, complex, randomly generated passwords
- Enforce AES encryption for Kerberos to prevent RC4 downgrade attacks
- Regularly audit accounts with SPNs using: `Get-ADUser -Filter {ServicePrincipalName -ne "$null"}`

---

### Kali Linux Network Setup

**Kali Linux desktop — first boot, network disconnected:**

![Kali First Boot](screenshots/kali_first_boot.png)

**Configuring static IP on eth0 — editing /etc/network/interfaces:**

![Kali Network Config](screenshots/kali_network_interfaces.png)

**Verifying IP assignment (192.168.10.30) and restarting networking service:**

![Kali IP Assigned](screenshots/kali_ip_assigned.png)

**Successful ping to DC01 (192.168.10.10) — confirming full network connectivity:**

![Kali Ping DC01](screenshots/kali_ping_dc01.png)

---

### SIEM Setup (Ubuntu Server + Splunk)

**SIEM VM first boot — Ubuntu Server running, successful ping to DC01 (192.168.10.10) confirming network connectivity:**

![SIEM Ping DC01](screenshots/siem_ping_dc01.png)

**Downloading Splunk via curl (520MB) and installing with dpkg — then starting Splunk for the first time and creating admin account:**

![Splunk Download and Install](screenshots/splunk_download_install.png)

**SIEM pinging DC01 and beginning Splunk download via wget — NAT adapter providing internet access:**

![SIEM Network and Download](screenshots/siem_network_download.png)

---

**Joining WS01 to lab.local domain — Computer Name/Domain Changes dialog showing successful domain join, restart required:**

![Domain Join](screenshots/ws01_domain_join.png)

**WS01 login screen after domain join — LAB\bob available as domain user, confirming successful AD integration:**

![WS01 Domain Login](screenshots/ws01_domain_login.png)

**Adding Windows Firewall rule to allow ICMP — enabling ping from other VMs for connectivity testing:**

![WS01 Firewall Rule](screenshots/ws01_firewall_icmp.png)

**Downloading Splunk Universal Forwarder on WS01 via PowerShell — dual NIC visible (192.168.10.20 internal + 10.0.3.15 NAT for internet access):**

![WS01 Splunk Download](screenshots/ws01_splunk_download.png)

---

**Splunk login page accessed from Kali's browser at http://192.168.10.40:8000:**

![Splunk Login](screenshots/splunk_login.png)

**Splunk dashboard — navigating settings to configure log receiving:**

![Splunk Dashboard](screenshots/splunk_dashboard.png)

**Configuring Splunk to listen on port 9997 for incoming forwarder connections:**

![Splunk Port 9997](screenshots/splunk_port_9997.png)

---

### Blue Team — Detections in Splunk

**Event ID 4625 (Failed Logon) detected in Splunk — sourced from DC01.lab.local:**

![Failed Login Detection](screenshots/failed_login_splunk.png)

**Event ID 4769 (Kerberos Service Ticket Request) detected — 3 events from DC01.lab.local matching our Kerberoast attack:**

![Event 4769 Detected](screenshots/event_4769_detected.png)

**Filtering for RC4 encryption (Ticket_Encryption_Type=0x17) — the key Kerberoasting indicator. Normal Kerberos uses AES (0x11/0x12). 1 suspicious event confirmed:**

![RC4 Detection](screenshots/rc4_encryption_detection.png)

**Creating the "Kerberoasting Detected" real-time alert in Splunk:**

![Alert Creation](screenshots/alert_creation.png)

**Final alert configuration — Title: "Kerberoasting Detected", Real-time, High severity, Never expires:**

![Alert Saved](screenshots/alert_saved.png)

---

### Attack — Kerberoasting (Full Walkthrough)

**Step 1 — Enumerate domain users and identify Kerberoastable accounts using impacket-GetUserSPNs as low-privilege user bob. sqlsvc with SPN MSSQLSvc/dc01.lab.local:1433 is identified:**

![AD Enumeration](screenshots/kerberoast_enumeration.png)

**Step 2 — Request the Kerberos TGS ticket for sqlsvc — the full hash is returned:**

![Kerberoast Hash](screenshots/hash_of_the_ticket.png)

**Step 3 — Save the hash to /home/kali/hash.txt for offline cracking:**

![Saving Hash](screenshots/saving_hash.png)

**Step 4 — First attempt: rockyou.txt wordlist (14 million passwords) — Status: Exhausted. rockyou.txt needed to be extracted first with gunzip:**

![Extract Rockyou](screenshots/extract_rockyou.png)

**Attempt 1 result — rockyou.txt exhausted, hash not cracked:**

![First Exhaustion](screenshots/first_exhaustion.png)

**Step 5 — Installing SecLists for more comprehensive wordlists (3.11 GB cloned from GitHub):**

![SecLists Install](screenshots/seclists_install.png)

**Browsing available SecLists wordlists — selecting Pwdb_top-10000000.txt (10 million passwords):**

![SecLists Wordlists](screenshots/seclists_wordlists.png)

**Attempt 2 — Pwdb_top-10000000.txt (10 million passwords) — Status: Exhausted:**

![Second Exhaustion](screenshots/second_exhaustion.png)

**Attempt 3 — 100k-most-used-passwords-NCSC.txt with best66.rule mutations — listing available hashcat rules first:**

![Hashcat Rules](screenshots/hashcat_rules.png)

**Attempt 3 result — best66.rule exhausted, hash not cracked:**

![Third Exhaustion](screenshots/third_exhaustion.png)

**Attempt 4 — Targeted wordlist (Mysql/mysql/MYSQL) with dive.rule (98,670 rules) — starting the attack:**

![Fourth Attempt](screenshots/fourth_attempt_dive.png)

**Attempt 4 result — dive.rule exhausted with targeted wordlist, hash not cracked:**

![Fourth Exhaustion](screenshots/fourth_exhaustion.png)

**Attempt 5 — Pwdb_top-10000000.txt combined with dive.rule — estimated time: 6 days, 19 hours! Attack aborted:**

![Fifth Attempt](screenshots/fifth_attempt_dive_pwdb.png)

**Attempt 5 aborted — confirmed that CPU-only cracking of complex passwords is impractical without a GPU:**

![Fifth Aborted](screenshots/fifth_aborted.png)

**Attempt 6 — Creating a custom wordlist with the actual password and running hashcat:**

![Sixth Attempt](screenshots/sixth_attempt_custom.png)

**Attempt 6 result — Status: CRACKED — MysqlPassword1! recovered successfully:**

![Kerberoast Cracked](screenshots/kerberoast_cracked.png)

---

### Troubleshooting & Lessons Learned

**Splunk log installation log being read as PowerShell commands — caused by incorrect use of Get-Content on a log file:**

![Troubleshooting 1](screenshots/troubleshoot_log_as_commands1.png)

**Continuation of the same error — msiexec help output being interpreted as PowerShell syntax:**

![Troubleshooting 2](screenshots/troubleshoot_log_as_commands2.png)

**Splunk Universal Forwarder splunkd.log confirming successful connection to indexer at 192.168.10.40:9997 — resolving the log forwarding issue:**

![Splunk Forwarder Connected](screenshots/splunk_forwarder_connected.png)

**SIEM showing Splunk running (PID 969) but forward-server list showing None — helped identify missing inputs.conf as root cause:**

![SIEM Splunk Status](screenshots/siem_splunk_status.png)

> **Note:** These troubleshooting steps are documented intentionally. Real-world lab work always involves errors — learning to read logs, identify root causes, and fix issues is a core skill in both Red and Blue Team work.

---

### Splunk Configuration

- **Receiving port:** 9997
- **Universal Forwarder installed on:** DC01, WS01
- **Indexes:** main

### Key Event IDs Monitored

| Event ID | Description | Why It Matters |
|---|---|---|
| 4624 | Successful logon | Baseline normal activity |
| 4625 | Failed logon | Brute force / password spray detection |
| 4769 | Kerberos service ticket requested | Kerberoasting detection |
| 4768 | Kerberos TGT requested | AS-REP Roasting detection |
| 4776 | NTLM authentication attempt | Pass-the-Hash detection |
| 4732 | User added to privileged group | Privilege escalation detection |

---

## Screenshots

> All screenshots are located in the `screenshots/` folder.

### Domain Controller Setup (DC01)

**Creating domain users (Alice, Bob) and service account (sqlsvc) with SPN:**

![Creating AD Users](screenshots/creating_ad_users.png)

**Full domain user and SPN setup complete — sqlsvc configured for Kerberoasting practice:**

![AD Users and SPN Setup](screenshots/ad_users_spn_setup.png)

---

### Splunk Universal Forwarder Setup (DC01)

**Splunk Universal Forwarder successfully installed on DC01:**

![Splunk Installation](screenshots/splunk_installation_on_windows_server.png)

**SplunkForwarder service confirmed running:**

![Splunk Service Running](screenshots/splunk_service_running.png)

**Creating inputs.conf to configure Windows Event Log collection (Security, System, Application):**

![Creating inputs.conf](screenshots/creating_inputs_conf.png)

---

### Blue Team — Attack Detection

**Simulating failed login attempts to generate Event ID 4625 — confirms Splunk forwarder is connected and sending logs:**

![Failed Login Attack Simulation](screenshots/attack_related_login_failure.png)

**Searching for Event ID 4769 (Kerberos Service Ticket Requests) — 3 events detected from DC01.lab.local matching the Kerberoast attack timeframe:**

![Event 4769 Detected](screenshots/event_4769_detected.png)

**Narrowing the search with Ticket_Encryption_Type=0x17 (RC4) — the key Kerberoasting indicator. Normal Kerberos uses AES (0x11/0x12). 1 suspicious event confirmed from DC01:**

![RC4 Encryption Detection](screenshots/rc4_encryption_detection.png)

**Creating the real-time Splunk alert — Real-time type, triggers for each result greater than 0:**

![Saving Alert](screenshots/saving_alert.png)

**Full alert configuration — title "Kerberoasting Detected", description, Real-time type, trigger conditions and severity all configured:**

![Full Alert Config](screenshots/full_alert_config.png)

---

## Upcoming Labs

- [ ] Password Spraying + Detection
- [ ] AS-REP Roasting
- [ ] BloodHound AD Enumeration
- [ ] Pass the Hash
- [ ] DCSync Attack
- [ ] LLMNR/NBT-NS Poisoning with Responder

---

## References

- [MITRE ATT&CK — Kerberoasting (T1558.003)](https://attack.mitre.org/techniques/T1558/003/)
- [Impacket GitHub](https://github.com/fortra/impacket)
- [Splunk Security Essentials](https://splunkbase.splunk.com/app/3435)
- [BloodHound](https://github.com/BloodHoundAD/BloodHound)

---

*Lab built for educational purposes only. All attacks performed in an isolated environment.*
