<div align="center">

# 🔐 Lab 3 — Splunk SIEM & Log Analysis

![Splunk](https://img.shields.io/badge/Splunk-Enterprise-FF6600?style=for-the-badge&logo=splunk&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-VM-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Windows](https://img.shields.io/badge/Windows_Server-AD-0078D4?style=for-the-badge&logo=windows&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Cost](https://img.shields.io/badge/Cost-$0%20Free-28a745?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

**I deployed a fully functional SIEM using Splunk Free, ingested real Windows Active Directory logs, wrote SPL detection queries, built security dashboards, and automated brute-force alerting — all at zero cost on Azure.**

[Overview](#-overview) • [Architecture](#-architecture) • [Prerequisites](#-prerequisites) • [Deployment](#-deployment) • [SPL Queries](#-spl-detection-queries) • [Dashboard](#-dashboard) • [Alerting](#-automated-alerting) • [Verification](#-verification) • [Portfolio Evidence](#-portfolio-evidence)

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

### Windows Server VM — Lab 1 AD Domain Controller

The Windows Server DC02 VM was provisioned as the Active Directory log source that feeds this SIEM lab.

**DC02 VM creation — configuration:**

![DC02 VM creation](screenshots/2026-05-16_16-16.png)

**DC02 deployment in progress:**

![DC02 deployment in progress](screenshots/2026-05-16_16-26.png)

**DC02 deployment complete:**

![DC02 deployment complete](screenshots/2026-05-16_16-27.png)

**DC02 — Native RDP connection configured (Port 3389):**

![DC02 RDP connect page](screenshots/2026-05-16_16-29.png)

**RDP session establishing into DC02:**

![DC02 RDP session loading](screenshots/2026-05-16_16-36.png)

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

**VM configuration — image, size, and authentication:**

![Splunk VM configuration](screenshots/2026-05-16_15-22.png)

**VM configuration — subscription and resource group:**

![Splunk VM subscription and resource group](screenshots/2026-05-16_15-28.png)

**Ubuntu 24.04 LTS VM deployment in progress:**

![Splunk VM deployment in progress](screenshots/2026-05-16_16-03.png)

**Deployment complete — splunk-server live:**

![Splunk VM deployment complete](screenshots/2026-05-16_16-04.png)

**splunk-server VM properties — OS, size, and network confirmed:**

![Splunk VM properties](screenshots/2026-05-16_16-05.png)

---

#### NSG Rules Configured

Port 8000 and port 9997 inbound rules were added to the Network Security Group after deployment.

**Adding inbound rule — Port 8000 (Splunk Web UI):**

![NSG rule port 8000](screenshots/2026-05-16_17-48.png)

**Adding inbound rule — Port 9997 (Forwarder input):**

![NSG rule port 9997](screenshots/2026-05-16_17-49.png)

**All NSG inbound rules confirmed — RDP :3389, Web UI :8000, Forwarder :9997:**

![NSG all rules confirmed](screenshots/2026-05-16_17-50.png)

---

#### VNet Connectivity Verified Between VMs

Before installing Splunk, connectivity between the Windows Server VM and the Ubuntu VM was verified across the Azure VNet. A ping to the Splunk VM private IP (10.0.0.4) returned 0% packet loss.

**Ping test from DC02 to Splunk VM — 0% packet loss:**

![VNet connectivity ping test](screenshots/2026-05-16_16-43.png)

**OpenSSH confirmed installed on Windows Server — SSH client ready:**

![OpenSSH installed](screenshots/2026-05-16_16-45.png)

---

#### Splunk Enterprise Installed via SSH

SSH was established from the Windows Server VM into the Ubuntu Splunk VM using the private IP over the VNet.

**SSH session established — azureuser@splunk-server (Ubuntu 24.04.4 LTS):**

![SSH into Splunk VM](screenshots/2026-05-16_16-47.png)

**System packages updated before Splunk installation:**

![apt update running](screenshots/2026-05-16_16-48.png)

```bash
# Connected to the Ubuntu 24.04 LTS VM over SSH, then ran the following:

# Download Splunk Enterprise — Linux .deb package
wget -O splunk-10.2.2-linux-amd64.deb \
  "https://download.splunk.com/products/splunk/releases/10.2.2/linux/splunk-10.2.2-80b90d638de6-linux-amd64.deb"

# Install the package
sudo dpkg -i splunk-10.2.2-linux-amd64.deb

# Start Splunk and accept the licence
sudo /opt/splunk/bin/splunk start --accept-license

# Enable Splunk to auto-start on VM reboot
sudo /opt/splunk/bin/splunk enable boot-start
```

**Splunk installation complete — web server at :8000 confirmed running:**

![Splunk install complete](screenshots/2026-05-16_17-36_1.png)

---

#### Splunk Web UI Accessed

**Splunk Enterprise login page — accessed via browser at 10.0.0.4:8000:**

![Splunk login page](screenshots/2026-05-16_17-53.png)

**Splunk Enterprise home page — logged in as Administrator:**

![Splunk home page](screenshots/2026-05-16_17-53_1.png)

---

### Step 2 — Data Inputs Configured

#### Sysmon Installed on Windows Server

Sysmon (System Monitor) was deployed on the Windows Server VM to enrich Windows Event Logs with process creation, network connection, and file change events — providing deeper visibility into host activity beyond standard Windows security events.

**Sysmon v15.2 downloaded from Microsoft Sysinternals:**

![Sysmon download page](screenshots/2026-05-16_17-55.png)

**Sysmon files extracted to C:\Tools\Sysmon:**

![Sysmon files extracted](screenshots/2026-05-16_18-02.png)

**Sysmon configuration file (sysmonconfig-export.xml) saved:**

![Sysmon config saved](screenshots/2026-05-16_18-06.png)

**Sysmon64 installed and started — SplunkForwarder service confirmed Running:**

![Sysmon installed and forwarder running](screenshots/2026-05-16_18-33.png)

---

#### Universal Forwarder Installed on Windows Server

The Splunk Universal Forwarder was installed on the Windows Server VM and pointed at the Splunk indexer over the private VNet IP.

- **Deployment Server:** `10.0.0.4:8089`
- **Receiving Indexer:** `10.0.0.4:9997`

**Universal Forwarder setup wizard — on-premises Splunk Enterprise instance selected:**

![Universal Forwarder setup wizard](screenshots/2026-05-16_18-37.png)

**Universal Forwarder installation in progress:**

![Universal Forwarder installing](screenshots/2026-05-16_20-09.png)

#### inputs.conf Configured

`inputs.conf` was created at `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf` on the Windows Server VM to define which event logs to collect:

```ini
[WinEventLog://Security]
disabled = 0
start_from = oldest
current_only = 0
evt_resolve_ad_obj = 1

[WinEventLog://System]
disabled = 0

[WinEventLog://Application]
disabled = 0
```

---

## 🔎 SPL Detection Queries

All searches were run inside **Search & Reporting** in the Splunk web UI.

### Data Flow Confirmed

**Initial search — first events arriving from DC02 (WinEventLog:System):**

![Initial data flowing](screenshots/2026-05-16_20-25.png)

**Security events search — 40,642 events indexed from DC02:**

![40642 security events](screenshots/2026-05-16_20-28.png)

### Successful Logins (Event ID 4624)

**EventCode=4624 search — 15 successful login events returned:**

![EventCode 4624 results](screenshots/2026-05-16_20-28_1.png)

```spl
index=* sourcetype=WinEventLog:Security EventCode=4624
| stats count by Account_Name, Logon_Type
| sort -count
```

| Logon_Type | Meaning |
|---|---|
| `2` | Interactive (user at keyboard) |
| `3` | Network (file share / network resource) |
| `5` | Service account (automated — expected) |
| `10` | Remote Interactive (RDP session) |

### Failed Login Attempts (Event ID 4625)

```spl
index=* sourcetype=WinEventLog:Security EventCode=4625
| stats count by Account_Name, Workstation_Name
| sort -count
```

> 🔴 **Detection logic:** 5+ failures for one account in a short window indicates a possible brute force attempt.

### Account Lockout Events (Event ID 4740)

```spl
index=* sourcetype=WinEventLog:Security EventCode=4740
| table _time, Account_Name, Caller_Computer_Name
| sort -_time
```

> 🔴 **Detection logic:** Multiple lockouts from one source machine indicate a brute force or password spray attack.

### Top 10 Failed Login Usernames — Threat Hunting

```spl
index=* sourcetype=WinEventLog:Security EventCode=4625 earliest=-24h
| stats count as failures by Account_Name
| sort -failures
| head 10
```

### After-Hours Login Detection

```spl
index=* sourcetype=WinEventLog:Security EventCode=4624
| eval hour=strftime(_time, "%H")
| where hour < 7 OR hour > 19
| table _time, Account_Name, Workstation_Name, Logon_Type
| sort -_time
```

---

## 📊 Dashboard

A **Windows Security Monitoring Dashboard** was built in Splunk to provide a persistent, at-a-glance view of security posture.

**Create New Dashboard dialog — Classic dashboard selected:**

![Create new dashboard](screenshots/2026-05-16_20-33.png)

**Adding Login Activity Over Time panel (Line Chart — EventCode=4624 timechart):**

![Adding line chart panel](screenshots/2026-05-16_20-40.png)

**Adding After-Hours Logins panel (Events list — after-hours detection query):**

![Adding after-hours panel](screenshots/2026-05-16_20-41.png)

**Dashboard — Account Activity Last 24h and Top Processes panels:**

![Dashboard account activity and top processes](screenshots/2026-05-16_20-59.png)

**Dashboard — Login Activity Over Time and After-Hours Logins panels:**

![Dashboard login activity and after-hours](screenshots/2026-05-16_20-59_1.png)

| Panel Title | SPL Basis | Visualisation |
|---|---|---|
| Account Activity — Last 24h | `EventCode=4624` + `stats count by Account_Name` | Bar chart |
| Top Processes — Last 24h | `EventCode=4688` + `stats count by Creator_Process_Name` | Events list |
| Login Activity Over Time | `EventCode=4624` + `timechart count` | Line chart |
| After-Hours Logins | After-hours query + `eval hour` filter | Events list |

---

## 🚨 Automated Alerting

A scheduled alert was configured to automatically detect high privileged logon counts — eliminating the need for manual dashboard checks.

**Save As Alert dialog — High Privileged Logon Count, scheduled every 15 minutes:**

![Save as alert configuration](screenshots/2026-05-16_20-46.png)

**Detection query results — privileged logons by account and computer:**

![Alert SPL query results](screenshots/2026-05-16_20-50.png)

```spl
index=* sourcetype=WinEventLog:Security EventCode=4672
| stats count as privilege_logons by Account_Name, ComputerName
| where privilege_logons > 0
```

**Alert confirmed active — "High Privileged Logon Count" status: Enabled ✅:**

![Alert enabled confirmed](screenshots/2026-05-16_20-51.png)

| Setting | Value |
|---|---|
| **Name** | `High Privileged Logon Count` |
| **Alert type** | Scheduled |
| **Cron expression** | `*/15 * * * *` (every 15 minutes) |
| **Time range** | Last 24 hours |
| **Trigger condition** | Number of results greater than `0` |
| **Trigger** | For each result |

---

## ✔️ Verification

**splunk-server VM — Status: Running, Standard D2s v3, Ubuntu 24.04:**

![Splunk server VM running](screenshots/2026-05-16_20-53.png)

**Network settings — all NSG rules confirmed (final state):**

![NSG rules final state](screenshots/2026-05-16_20-53_1.png)

**kingsvm_group resource group — complete lab infrastructure (DC02 + splunk-server + VNet + NSG):**

![Resource group all resources](screenshots/2026-05-16_20-55.png)

**PowerShell verification — Test-NetConnection :8000 = True, SplunkForwarder Running, Sysmon 71,742 events logged:**

![PowerShell verification](screenshots/2026-05-16_21-01.png)

| Check | Result |
|---|---|
| **Data flowing into Splunk** | 40,642 Security events indexed from DC02 ✅ |
| **EventCode=4624 search** | 15 successful login events returned ✅ |
| **Dashboard populated** | All panels rendering with live data ✅ |
| **Alert active** | "High Privileged Logon Count" — Status: Enabled ✅ |
| **Port 8000 reachable** | `Test-NetConnection 10.0.0.4 -Port 8000` → TcpTestSucceeded: True ✅ |
| **SplunkForwarder service** | Status: Running ✅ |
| **Sysmon operational** | 71,742 events in Microsoft-Windows-Sysmon/Operational log ✅ |

---

## 📁 Portfolio Evidence

### SPL Search History

All detection queries executed during the lab — confirming hands-on SPL experience across authentication, privilege escalation, process activity, and after-hours analysis:

![Splunk search history](screenshots/2026-05-16_20-58.png)

### Security Dashboard — Live Data

**Windows Security Monitoring Dashboard — Account Activity and Top Processes panels with real DC02 data:**

![Dashboard panel 1 — account activity and top processes](screenshots/2026-05-16_20-59.png)

**Windows Security Monitoring Dashboard — Login Activity Over Time and After-Hours Logins:**

![Dashboard panel 2 — login activity and after-hours](screenshots/2026-05-16_20-59_1.png)

### What Each Panel Demonstrates

**Account Activity — Last 24h:** Surfaces which accounts are generating the most authentication events on DC02. In a real SOC environment this panel would immediately highlight any account behaving anomalously — for example a service account suddenly generating interactive logons, or a user account with a volume of events far above baseline. The bar chart format makes it easy to spot outliers at a glance without running a manual search.

**Top Processes — Last 24h:** Powered by Sysmon EventCode=4688 (process creation), this panel reveals which executables are being spawned on the domain controller. Legitimate DCs run a predictable set of processes — any unfamiliar binary appearing in this list warrants immediate investigation. This is the kind of visibility that separates a well-instrumented environment from one flying blind.

**Login Activity Over Time:** The line chart maps authentication volume against time, making it straightforward to identify spikes that fall outside normal business hours. A sudden surge in logins at 2 AM is a pattern no static table would surface as clearly. This panel turns raw event counts into a story.

**After-Hours Logins:** Directly queries for interactive and remote logons occurring outside 7 AM–7 PM. Service account logins at night are expected and filtered out contextually — what this panel is looking for are human logins at unusual times, which in a real environment would trigger an immediate Tier 1 investigation.

---

## 🧠 Skills Demonstrated

| Skill | Real-World Application |
|---|---|
| **Splunk deployment & data input configuration** | Universal Forwarder is the standard enterprise method for feeding logs into Splunk |
| **Sysmon deployment and configuration** | Industry-standard host telemetry tool used across enterprise SOC environments |
| **SPL query writing** | The core analytical skill for SOC investigations and threat hunting |
| **Security dashboard construction** | Persistent visibility into login failures, lockouts, and anomalies |
| **Privileged logon detection** | Monitoring Event ID 4672 for privilege escalation patterns |
| **Automated detection rule engineering** | Foundation of all SIEM-based SOC operations |
| **After-hours anomaly detection** | Surfaces suspicious access outside business hours |

### Career Relevance

| Role | Application |
|---|---|
| **SOC Analyst Tier 1** | Monitoring dashboards, searching logs for suspicious activity, escalating findings |
| **SOC Analyst Tier 2–3** | Building detection rules, correlating events across sources, threat hunting with SPL |
| **Cloud Security Engineer** | Microsoft Sentinel and AWS Security Hub operate on the same SIEM concepts |
| **Incident Responder** | Log timeline reconstruction, scoping the extent of compromise during active incidents |

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
- [Sysmon — Microsoft Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [CIS Benchmark — Windows Server](https://www.cisecurity.org/benchmark/microsoft_windows_server)

---

<div align="center">

*Part of a hands-on cloud security lab series · Azure · Splunk Free · $0 cost*

</div>
