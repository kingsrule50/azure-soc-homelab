<div align="center">

# 🔐 Lab 3 — Splunk SIEM & Log Analysis

![Splunk](https://img.shields.io/badge/Splunk-Enterprise-FF6600?style=for-the-badge&logo=splunk&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-VM-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Windows](https://img.shields.io/badge/Windows_Server-AD-0078D4?style=for-the-badge&logo=windows&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Cost](https://img.shields.io/badge/Cost-$0%20Free-28a745?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

**I deployed a fully functional SIEM using Splunk Free, ingested real Windows Active Directory logs, wrote SPL detection queries, built security dashboards, and automated brute-force alerting — all at zero cost on Azure.**

[Overview](#-overview) • [Architecture](#-architecture) • [Prerequisites](#-prerequisites) • [Deployment](#-deployment) • [SPL Queries](#-spl-detection-queries) • [Dashboard](#-dashboard) • [Alerting](#-automated-alerting) • [Verification](#-verification) • [Skills Gained](#-skills-gained)

</div>

---

## 📋 Lab Summary

| Field | Detail |
|---|---|
| **Certification Alignment** | CompTIA Security+ · CySA+ · Splunk Core Certified User |
| **Estimated Time** | 4–6 hours (across multiple sessions) |
| **Estimated Cost** | **$0** — Splunk Free licence (500 MB/day) |
| **Difficulty** | Intermediate |
| **Azure Resources** | 2× VMs (Ubuntu 24.04 LTS + Windows Server from Lab 1) |
| **Career Relevance** | SOC Analyst T1–T3 · Security Engineer · Incident Responder |

---

## 🎯 Overview

### The Problem This Lab Addresses

A medium-sized organisation generates **millions of log events daily** — Windows Event Logs, Active Directory authentication events, firewall logs, and cloud resource logs. Without a SIEM, those logs sit isolated across separate systems with no way to search across them, correlate events, or identify attack patterns spanning multiple sources.

This lab replicates how **enterprise SOC environments** operate:

- A **Windows Server VM** (Active Directory environment from Lab 1) generates real authentication events
- A **Splunk Universal Forwarder** ships those logs over the internal Azure network
- A **Splunk SIEM instance** indexes, searches, and visualises the data
- **SPL detection rules** surface brute force attempts, after-hours logins, and account lockouts
- **Automated alerts** fire when threat thresholds are crossed — no manual checking required

> Splunk is the most widely deployed commercial SIEM. Hands-on experience with it appears on job descriptions for nearly every security operations role.

---

## 🏗 Architecture

### Data Flow Diagram

![Splunk SIEM Architecture](architecture.svg)

### Trust Boundaries & Security Design Decisions

| Boundary | Rule Applied | Rationale |
|---|---|---|
| **Port 8000 (Web UI)** | Restricted to **my IP only** | Prevents unauthorised access to the Splunk interface |
| **Port 9997 (Forwarder)** | Restricted to **VNet range only** (10.0.0.0/16) | Log ingestion should never be exposed to the public internet |
| **Port 22 (SSH)** | Restricted to **my IP only** | Limits admin access to the Ubuntu VM |
| **Port 3389 (RDP)** | Restricted to **my IP only** (Windows VM) | Follows least-privilege access to the domain controller |

---

## ✅ Prerequisites

- Active Directory Windows Server VM deployed in Azure (Lab 1)
- Azure subscription (free tier sufficient)
- SSH client — Terminal on macOS/Linux or PuTTY on Windows
- Splunk free account registered at [splunk.com](https://splunk.com)

---

## 🚀 Deployment

### Step 1 — Splunk Ubuntu 24.04 LTS VM Provisioned in Azure

#### Azure VM Configuration

| Setting | Value |
|---|---|
| **OS** | Ubuntu 24.04 LTS |
| **Size** | Standard_D2s_v3 (2 vCPU, 8 GB RAM) |
| **OS Disk** | 30 GB |
| **Inbound NSG Rule 1** | Port **8000** → TCP → Source: My IP |
| **Inbound NSG Rule 2** | Port **9997** → TCP → Source: VNet (10.0.0.0/16) |
| **Inbound NSG Rule 3** | Port **22** → TCP → Source: My IP |

> Port 9997 was scoped to the VNet address range only — never opened to `0.0.0.0/0`. This ensures raw log data cannot be received from the public internet.

---

#### Splunk Enterprise Installed via SSH

```bash
# Connected to the Ubuntu 24.04 LTS VM over SSH, then ran the following:

# Download Splunk Enterprise — Linux .deb package (v10.2.2, April 2026)
wget -O splunk-10.2.2-linux-amd64.deb \
  "https://download.splunk.com/products/splunk/releases/10.2.2/linux/splunk-10.2.2-80b90d638de6-linux-amd64.deb"

# Install the package
sudo dpkg -i splunk-10.2.2-linux-amd64.deb

# Start Splunk and accept the licence — admin credentials set at first run
sudo /opt/splunk/bin/splunk start --accept-license

# Configured Splunk to auto-start on VM reboot
sudo /opt/splunk/bin/splunk enable boot-start

# Web UI accessible at: http://<VM_PUBLIC_IP>:8000
```

---

### Step 2 — Data Inputs Configured

#### Part A — Receiving Port Enabled in Splunk

Inside the Splunk web UI:

- Navigated to **Settings → Forwarding and Receiving**
- Configured a new **Receiving Port** on `9997`
- Created a new **Index** named `windows_logs` to store all incoming Windows events

---

#### Part B — Universal Forwarder Installed on Windows Server

The Splunk Universal Forwarder was installed on the Windows Server VM from Lab 1 and pointed at the Splunk instance:

- **Deployment Server:** `<Splunk VM Private IP>:8089`
- **Receiving Indexer:** `<Splunk VM Private IP>:9997`

> The private IP was used for all forwarder-to-indexer communication — keeping log traffic contained within the Azure VNet.

---

#### Part C — inputs.conf Configured to Define Log Sources

`inputs.conf` was created at the following path on the Windows Server VM:

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

```ini
# inputs.conf — defines which Windows Event Logs the forwarder collects

[WinEventLog://Security]
# Security log: all authentication events — logins, failures, lockouts
disabled = 0
start_from = oldest
current_only = 0
evt_resolve_ad_obj = 1  # Resolves AD object names so usernames appear instead of SIDs

[WinEventLog://System]
# System log: OS-level events — service starts/stops, driver failures
disabled = 0

[WinEventLog://Application]
# Application log: events from installed applications
disabled = 0
```

After saving the file, the forwarder service was restarted to apply the configuration:

```powershell
# Ran in PowerShell as Administrator on the Windows Server VM
Restart-Service SplunkForwarder
```

---

## 🔎 SPL Detection Queries

All searches were run inside **Search & Reporting** in the Splunk web UI.

### Confirming Data Flow

```spl
index=windows_logs | head 100
```
> ✅ Returned events — confirming the forwarder was shipping logs successfully.

---

### Failed Login Attempts (Event ID 4625)

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4625
| stats count by Account_Name, Workstation_Name
| sort -count
```

| SPL Component | Purpose |
|---|---|
| `EventCode=4625` | Filters to failed logon events only |
| `stats count by ...` | Counts events grouped by username and source machine |
| `sort -count` | Sorts highest count first (descending) |

> 🔴 **Detection logic:** 5+ failures for one account in a short window indicates a possible brute force attempt.

---

### Successful Logins by Type (Event ID 4624)

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4624
| stats count by Account_Name, Logon_Type
| sort -count
```

| Logon_Type | Meaning |
|---|---|
| `2` | Interactive (user at keyboard) |
| `3` | Network (file share / network resource) |
| `5` | Service account (automated — expected) |
| `10` | Remote Interactive (RDP session) |

---

### Account Lockout Events (Event ID 4740)

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4740
| table _time, Account_Name, Caller_Computer_Name
| sort -_time
```

> 🔴 **Detection logic:** Multiple lockouts for the same account originating from one machine indicate a brute force or password spray attack in progress.

---

### Top 10 Failed Login Usernames — Threat Hunting

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4625 earliest=-24h
| stats count as failures by Account_Name
| sort -failures
| head 10
```

> 🔴 Accounts with 20+ failures in 24 hours warrant investigation  
> 🔴 Usernames absent from Active Directory indicate an account enumeration attack

---

### After-Hours Login Detection

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4624
| eval hour=strftime(_time, "%H")
| where hour < 7 OR hour > 19
| table _time, Account_Name, Workstation_Name, Logon_Type
| sort -_time
```

| SPL Component | Purpose |
|---|---|
| `eval hour=strftime(_time, "%H")` | Extracts the hour from the event timestamp |
| `where hour < 7 OR hour > 19` | Retains only events outside the 7 AM–7 PM window |

> ✅ After-hours **service account** logins (Type 5) are expected and normal  
> 🔴 After-hours **interactive** logins (Type 2 or 10) from regular users require review

---

## 📊 Dashboard

A **Windows Security Overview** dashboard was built in Splunk to provide a persistent, at-a-glance view of the security posture without running searches manually each time.

| Panel Title | SPL Basis | Visualisation |
|---|---|---|
| Failed Logins — Last 24h | `EventCode=4625` + `stats count by Account_Name` | Bar chart |
| Account Lockouts — Last 7d | `EventCode=4740` + `table` output | Events list |
| Login Activity Over Time | `EventCode=4624` + `timechart count` | Line chart |
| Top Source Machines — After Hours | After-hours query + `stats count by Workstation_Name` | Column chart |

---

## 🚨 Automated Alerting

A scheduled alert was configured to automate brute force detection — eliminating the need for manual dashboard checks.

#### Detection Query

```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4625
| stats count as failures by Account_Name
| where failures > 10
```

#### Alert Configuration

| Setting | Value |
|---|---|
| **Name** | `Potential Brute Force — High Failure Count` |
| **Alert type** | Scheduled |
| **Run frequency** | Every 15 minutes |
| **Trigger condition** | Number of results greater than `0` |
| **Action** | Add to Triggered Alerts |

> **On threshold tuning:** The threshold of 10 failures is a baseline starting point. In a production environment this would be tuned over time based on observed false positive rates. Alert quality — the balance between coverage and noise — is one of the core competencies of detection engineering.

---

## ✔️ Verification

| Check | Result |
|---|---|
| **Data flowing into Splunk** | `index=windows_logs \| head 10` returned recent events ✅ |
| **Failed login search functional** | EventCode=4625 query returned results after generating test failures ✅ |
| **Dashboard populated** | All four panels in Windows Security Overview rendered with live data ✅ |
| **Alert active** | Alert visible under Settings → Searches, Reports, and Alerts with status: Enabled ✅ |

---

## 🧠 Skills Demonstrated

| Skill | Real-World Application |
|---|---|
| **Splunk deployment & data input configuration** | Universal Forwarder is the standard enterprise method for feeding logs into Splunk |
| **SPL query writing** | The core analytical skill for SOC investigations and threat hunting |
| **Security dashboard construction** | Persistent visibility into login failures, lockouts, and anomalies |
| **Brute force pattern identification** | One of the most common Tier 1 SOC investigation types |
| **Automated detection rule engineering** | Foundation of all SIEM-based SOC operations |
| **Account lockout analysis** | Identifies password spray attacks in progress |
| **After-hours anomaly detection** | Surfaces suspicious access outside business hours |

### Career Relevance

| Role | Application |
|---|---|
| **SOC Analyst Tier 1** | Monitoring dashboards, searching logs for suspicious activity, escalating findings |
| **SOC Analyst Tier 2–3** | Building detection rules, correlating events across sources, threat hunting with SPL |
| **Cloud Security Engineer** | Microsoft Sentinel and AWS Security Hub operate on the same SIEM concepts |
| **Incident Responder** | Log timeline reconstruction, scoping the extent of compromise during active incidents |

---

## 📁 Portfolio Evidence

- [ ] Screenshots of the Windows Security Overview dashboard with populated panels
- [ ] Screenshot of the alert configuration page showing the rule as enabled
- [ ] Saved SPL searches exported as Splunk reports
- [ ] Written summary of each dashboard panel — what it surfaces and why it matters operationally

---

## 🔗 Related Labs

| Lab | Description |
|---|---|
| **Lab 1** | Active Directory + Windows Server in Azure — the log source feeding this SIEM |
| **Lab 2** | *(link your Lab 2 here)* |
| **Lab 4** | *(link your next lab here)* |

---

## 📚 References

- [Splunk Enterprise Documentation](https://docs.splunk.com/Documentation/Splunk)
- [Splunk SPL Quick Reference](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference/ListOfSearchCommands)
- [Windows Security Event IDs — Microsoft Docs](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/appendix-l--events-to-monitor)
- [Splunk Universal Forwarder Download](https://www.splunk.com/en_us/download/universal-forwarder.html)
- [CIS Benchmark — Windows Server](https://www.cisecurity.org/benchmark/microsoft_windows_server)

---

<div align="center">

*Part of a hands-on cloud security lab series · Azure · Splunk Free · $0 cost*

</div>
