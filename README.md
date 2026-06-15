<div align="center">

# 🍯 Azure Honeynet - Live Attack Detection with NSG, SQL & Microsoft Sentinel

![Domain](https://img.shields.io/badge/SIEM-Microsoft%20Sentinel-blue?style=for-the-badge)
![Infrastructure](https://img.shields.io/badge/Infrastructure-Azure%20Honeynet-red?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-Microsoft%20Azure-0078D4?style=for-the-badge)

<img src="https://img.shields.io/badge/Microsoft%20Sentinel-SIEM-blue?style=flat-square" />
<img src="https://img.shields.io/badge/Azure%20NSG-Network%20Security-0078D4?style=flat-square" />
<img src="https://img.shields.io/badge/SQL%20Server-Audit%20Logging-CC2927?style=flat-square" />
<img src="https://img.shields.io/badge/OS-Windows%20%7C%20Linux-red?style=flat-square" />
<img src="https://img.shields.io/badge/KQL-Threat%20Detection-green?style=flat-square" />
<img src="https://img.shields.io/badge/Defender%20for%20Cloud-Enabled-blue?style=flat-square" />

---

**Prepared by:** Bikash Raya  
**Project Type:** Honeynet · Live Attack Simulation · SOC Monitoring

</div>

---

## 📁 Repository Structure

| File | Description |
| --- | --- |
| [HoneyNet_Project.docx](./HoneyNet_Project.docx) | Complete project documentation with screenshots |
| README.md | Project overview |

---

## 📋 Overview

This repository documents the design and implementation of a live Azure Honeynet — an intentionally vulnerable cloud environment built to attract real-world cyber attackers, capture their activity, and detect threats using Microsoft Sentinel as the SIEM.

The lab simulates a realistic attack scenario by:

* 🪟 **Windows 11 VM** — Honeypot exposed to RDP brute-force and SQL attacks
* 🐧 **Linux VM** — Honeypot exposed to SSH brute-force attacks
* 💥 **Attacker VM** — Separate Azure VM (Japan East) simulating an external threat actor
* 🔓 **Open NSGs** — All inbound traffic allowed to maximise attack surface
* 🗄️ **SQL Server 2019** — Installed with audit logging to capture failed/successful logins
* 📊 **Microsoft Sentinel** — Central SIEM for log ingestion, enrichment, and detection
* 🌍 **GeoIP Watchlist** — Geographic enrichment of attacker IP addresses

---

## 🛠️ Technologies Used

* Microsoft Azure
* Microsoft Sentinel
* Azure Network Security Groups (NSG)
* NSG Flow Logs
* Log Analytics Workspace (LAW)
* Data Collection Rules (DCR)
* Azure Monitor Agent (AMA)
* Microsoft Defender for Cloud
* SQL Server 2019 + SSMS 20.1
* Azure Key Vault
* Azure Storage Account
* Microsoft Entra ID
* KQL (Kusto Query Language)
* Windows Event Viewer
* Linux auth.log / Syslog

---

## 🧪 Lab Components

| Resource | Role | Location |
| --- | --- | --- |
| Windows-VM | Windows honeypot — RDP & SQL attack target | Australia East |
| Linux-VM | Linux honeypot — SSH attack target | Australia East |
| Attacker-VM | External threat actor simulation | Japan East (separate VNet) |
| LAW-RG-soc-honeynet-Lab | Log Analytics Workspace | Australia East |
| Microsoft Sentinel | SIEM platform | Australia East |
| Windows-VM-nsg / Linux-VM-nsg | Deliberately open NSGs | Australia East |
| labstorageac123 | Storage account for NSG Flow Logs | Australia East |
| testkeyv aultbikash | Azure Key Vault | Australia East |
| RG-SOC-Honeynet-Lab | Main resource group | Australia East |
| RG-Attacker | Attacker resource group | Japan East |

---

## 🌐 Solution Architecture

```
[Internet / Real-World Attackers]
            ↓
[Open NSGs — Any/Any inbound rules]
            ↓
[Windows-VM]          [Linux-VM]
  RDP Port 3389         SSH Port 22
  SQL Server 1433       auth.log
  Windows Event Viewer
            ↓
[NSG Flow Logs → Storage Account]
            ↓
[Data Collection Rule (DCR) + AMA]
            ↓
[Log Analytics Workspace (LAW)]
            ↓
[Microsoft Sentinel]
            ↓
[GeoIP Watchlist Enrichment]
            ↓
[KQL Detection Queries → Alerts → Incidents]
```

---

## 🔓 Phase 1 — Infrastructure & Deliberate Exposure

* Created Resource Group: **RG-SOC-Honeynet-Lab** (Australia East)
* Deployed **Windows-VM** and **Linux-VM** inside **lab-vnet**
* Edited both NSGs to allow **Any source / Any destination / Any port / Any protocol** inbound
* Disabled **Windows Defender Firewall** on Windows-VM across Domain, Private, and Public profiles
* This transforms both VMs into live honeypots visible to the entire internet

---

## 🗄️ Phase 2 — SQL Server Installation & Audit Logging

* Installed **SQL Server 2019 Evaluation Edition** on Windows-VM with Mixed Mode authentication
* Installed **SQL Server Management Studio (SSMS) 20.1**
* Configured SQL audit logging to the **Windows Security log** via:
  * Registry permissions on `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\EventLog\Security`
  * `auditpol` command to enable Application Generated events:

```cmd
auditpol /set /subcategory:"application generated" /success:enable /failure:enable
```

* Set SSMS Server Properties → Login auditing → **Both failed and successful logins**
* Verified SQL login failures appear in Windows Event Viewer as **EventID 18456**

---

## 💥 Phase 3 — Attack Simulation

A separate **Attacker-VM** was deployed in **Japan East** on its own isolated VNet to simulate an external threat actor.

### Attacks Simulated

| Attack Type | Target | Method |
| --- | --- | --- |
| RDP Brute-Force | Windows-VM | Remote Desktop with wrong credentials |
| SQL Brute-Force | Windows-VM SQL Server | SSMS connection with wrong credentials |
| SSH Brute-Force | Linux-VM | PowerShell SSH with invalid username |

### Logs Confirmed

* **Windows Event Viewer → Security log** — EventID 4625 (failed RDP logon), workstation: Attacker-VM
* **Windows Event Viewer → Application log** — EventID 18456 (SQL login failure), client IP: Attacker-VM
* **Linux auth.log** — Hundreds of real-world brute-force SSH attempts from internet IPs confirmed via:

```bash
cat /var/log/auth.log | grep password
```

---

## 📊 Phase 4 — SIEM: Log Analytics & Microsoft Sentinel

* Deployed **Log Analytics Workspace**: MicrosoftLogAnalyticsOMS
* Added **Microsoft Sentinel** to the workspace
* Uploaded **GeoIP CSV Watchlist** (named: network) — maps IP ranges to country/city for alert enrichment
* Enabled **Microsoft Defender for Cloud** — Servers plan enabled, data collection set to **All Events**
* Configured **Continuous Export** to stream Defender alerts to Log Analytics
* Created **Storage Account**: labstorageac123 for NSG Flow Log storage
* Deployed **NSG Flow Logs** for both Windows-VM-nsg and Linux-VM-nsg
* Deployed **Data Collection Rule (DCR)** with two data sources:
  * Windows Event Logs
  * Linux Syslog
* Installed Sentinel **Content Hub** connectors: Windows Security Events via AMA, Microsoft Entra ID, Azure Activity
* Configured **Storage Account diagnostic logs** forwarded to LAW
* Created **Azure Key Vault**: testkeyv aultbikash
* Added test user to **Entra ID Global Administrator** role to generate audit log events

---

## 🔍 Phase 5 — KQL Threat Detection

### Heartbeat — Confirm Both VMs Reporting

```kql
Heartbeat
| summarize LastSeen=max(TimeGenerated) by Computer
| order by LastSeen desc
```

✅ Both Linux-VM and Windows-VM confirmed reporting to LAW

### Linux Failed SSH Logins

```kql
Syslog
| where SyslogMessage contains "Failed password"
| project TimeGenerated, Computer, ProcessName, SyslogMessage
| order by TimeGenerated desc
```

✅ Real-world SSH brute-force attempts confirmed streaming into Sentinel

### Windows Defender Malware Detection

```kql
Microsoft-Windows-Windows Defender/Operational!*[System[(EventID=1116 or EventID=1117)]]
```

### Windows Firewall Tampering Detection

```kql
Microsoft-Windows-Windows Firewall With Advanced Security/Firewall!*[System[(EventID=2003)]]
```

---

## 🎯 Skills Demonstrated

* Azure infrastructure provisioning (VMs, VNets, NSGs, Storage, Key Vault)
* Network Security Group rule management and flow log configuration
* SQL Server installation and security audit log configuration
* Linux administration and log analysis (auth.log, grep, Syslog)
* Honeynet design — deliberate attack surface exposure
* Attack simulation across RDP, SSH, and SQL attack vectors
* Microsoft Sentinel deployment and connector configuration
* Data Collection Rule (DCR) setup with Azure Monitor Agent (AMA)
* KQL (Kusto Query Language) for threat hunting and log analysis
* Microsoft Defender for Cloud configuration and continuous export
* GeoIP enrichment via Sentinel Watchlists
* Microsoft Entra ID audit log collection
* Azure Key Vault deployment and monitoring

---

## 🎯 Key Takeaway

> This project demonstrates practical experience in building and exposing a live Azure Honeynet to attract real-world attackers, configuring SQL Server audit logging, simulating multi-vector attacks (RDP, SSH, SQL brute-force) from a separate threat actor VM, and centralising all attack telemetry into Microsoft Sentinel for detection and analysis using KQL — going beyond standard SIEM configuration to show end-to-end offensive and defensive security skills.

---

## 🔗 Connect With Me

<div align="center">

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/bikash-raya/)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black?style=for-the-badge&logo=github)](https://github.com/Bikash-Raya)

</div>

---

<div align="center">

⭐ If you find this project useful, feel free to star the repository ⭐

</div>
