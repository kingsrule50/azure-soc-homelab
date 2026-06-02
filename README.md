<div align="center">

# 🔐 Lab 3 — Splunk Enterprise SIEM Deployment and Security Monitoring on Microsoft Azure

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

---

**DC02 VM creation — configuration:**

The VM was created in Azure under the `kingsvm_group` resource group. Windows Server 2025 Datacenter (Azure Edition) was selected as the image, the size was set to Standard_D2s_v3 (2 vCPUs, 8 GiB memory), and the region was set to East US. Naming the VM `DC02` follows the domain controller naming convention used in enterprise environments.

![DC02 VM creation](screenshots/2026-05-16_16-16.png)

---

**DC02 deployment in progress:**

After clicking **Review + create** and then **Create**, Azure began provisioning the VM. The deployment screen confirms the network interface and public IP address resources are being created alongside the VM itself.

![DC02 deployment in progress](screenshots/2026-05-16_16-26.png)

---

**DC02 deployment complete:**

The deployment completed successfully in under 2 minutes. The green **Deployment succeeded** notification confirms all resources — VM, NIC, and public IP — were provisioned without errors.

![DC02 deployment complete](screenshots/2026-05-16_16-27.png)

---

**DC02 — Native RDP connection configured (Port 3389):**

From the DC02 VM blade, navigated to **Connect > Native RDP**. The public IP (20.120.38.222) and port 3389 were confirmed. The source IP restriction for RDP was set to my IP only in the NSG before connecting, following least-privilege access principles.

![DC02 RDP connect page](screenshots/2026-05-16_16-29.png)

---

**RDP session establishing into DC02:**

After downloading and opening the RDP file, the session connected as `Admin01`. The "Please wait for the User" screen confirms the session handshake completed and the Windows desktop is loading — the VM is responsive and accessible.

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

---

**VM configuration — image, size, and authentication:**

In the Azure portal, navigated to **Virtual Machines > Create**. Ubuntu Server 24.04 LTS (x64 Gen2) was selected as the image and Standard_D2s_v3 (2 vCPUs, 8 GiB) as the size — the minimum spec Splunk recommends for a lab environment. Authentication was set to SSH public key, which is more secure than password authentication for a Linux server.

![Splunk VM configuration](screenshots/2026-05-16_15-22.png)

---

**VM configuration — subscription and resource group:**

The VM was added to the existing `kingsvm_group` resource group so all lab resources are managed together and can be deleted as a single unit at the end of the lab. The region was set to East US to match DC02, keeping both VMs in the same Azure VNet for low-latency internal communication.

![Splunk VM subscription and resource group](screenshots/2026-05-16_15-28.png)

---

**Ubuntu 24.04 LTS VM deployment in progress:**

After validation passed, the deployment was confirmed. The network interface (`splunk-server31`) and public IP (`splunk-server-ip`) resources are being provisioned first, followed by the VM itself.

![Splunk VM deployment in progress](screenshots/2026-05-16_16-03.png)

---

**Deployment complete — splunk-server live:**

The **Deployment succeeded** notification confirms all resources were created without errors. Clicked **Go to resource** to open the VM blade and confirm the public IP address before proceeding with SSH.

![Splunk VM deployment complete](screenshots/2026-05-16_16-04.png)

---

**splunk-server VM properties — OS, size, and network confirmed:**

From the VM **Properties** tab, confirmed: OS is Linux (Ubuntu 24.04), size is Standard D2s v3 (2 vCPUs, 8 GiB RAM), private IP is 10.0.0.4, and the VM is connected to the `kingsvm-vnet/default` subnet — the same VNet as DC02. This confirms both VMs share an internal network and can communicate without traversing the public internet.

![Splunk VM properties](screenshots/2026-05-16_16-05.png)

---

#### NSG Rules Configured

After deployment, three inbound rules were added to the `kingsvm-nsg` Network Security Group to control access to the Splunk VM.

---

**Adding inbound rule — Port 8000 (Splunk Web UI):**

Navigated to the Splunk VM > **Networking > Network settings > Create port rule > Inbound port rule**. Set destination port to `8000`, protocol to TCP, action to Allow, and named the rule `allow-splunk-web`. The source was restricted to my IP address only — port 8000 must never be open to the public internet as it provides direct access to the Splunk admin interface.

![NSG rule port 8000](screenshots/2026-05-16_17-48.png)

---

**Adding inbound rule — Port 9997 (Forwarder input):**

A second rule was created for port `9997` — the port Splunk uses to receive logs from the Universal Forwarder. Named `allow-splunk-forwarder`. The source was set to the VNet address range (10.0.0.0/16) rather than my IP, because DC02 (not my local machine) is the one sending logs to Splunk. Restricting to the VNet ensures this port is only reachable from other Azure VMs in the same network.

![NSG rule port 9997](screenshots/2026-05-16_17-49.png)

---

**All NSG inbound rules confirmed — RDP :3389, Web UI :8000, Forwarder :9997:**

The final NSG state shows six inbound rules. The three custom rules — RDP (300), allow-splunk-web (301), and allow-splunk-forwarder (302) — all show **Allow** status. The default Azure rules (AllowVnetInBound, AllowAzureLoadBalancer, DenyAllInBound) remain unchanged beneath them.

![NSG all rules confirmed](screenshots/2026-05-16_17-50.png)

---

#### VNet Connectivity Verified Between VMs

Before installing Splunk, connectivity from DC02 to the Splunk VM's private IP was tested to confirm both VMs could communicate over the Azure VNet.

---

**Ping test from DC02 to Splunk VM — 0% packet loss:**

From an Administrator PowerShell session on DC02, ran `ping 10.0.0.4` — the private IP of the Splunk VM. All 4 packets were received with 0% loss and sub-5ms response times, confirming the two VMs can reach each other over the internal VNet. This is essential — the Universal Forwarder will send logs to this private IP on port 9997.

![VNet connectivity ping test](screenshots/2026-05-16_16-43.png)

---

**OpenSSH confirmed installed on Windows Server — SSH client ready:**

Ran `Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Client*'` in PowerShell to confirm OpenSSH was available on DC02. The result showed `State: Installed`, meaning SSH commands could be run directly from the Windows Server terminal — no additional tools needed to connect to the Ubuntu Splunk VM.

![OpenSSH installed](screenshots/2026-05-16_16-45.png)

---

#### Splunk Enterprise Installed via SSH

SSH was established from DC02 into the Ubuntu Splunk VM using the private IP (10.0.0.4) over the VNet — keeping all traffic internal to Azure.

---

**SSH session established — azureuser@splunk-server (Ubuntu 24.04.4 LTS):**

From the DC02 PowerShell terminal, ran `ssh azureuser@10.0.0.4`. After accepting the host fingerprint and entering the password, the Ubuntu 24.04.4 LTS welcome screen confirmed a successful connection. The system information shows 28GB disk, 3% memory usage, and 134 running processes — confirming the VM is healthy and ready for Splunk installation.

![SSH into Splunk VM](screenshots/2026-05-16_16-47.png)

---

**System packages updated before Splunk installation:**

Before installing any software, ran `sudo apt update && sudo apt upgrade -y` to ensure all system packages were current. This is standard practice before installing a server application — outdated packages can cause dependency conflicts or leave known vulnerabilities in place. The output shows 24+ package repositories being updated from Azure's Ubuntu mirror.

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

---

**Splunk installation complete — web server at :8000 confirmed running:**

The installation output shows all preliminary checks passing, the RSA private key being generated for TLS, and the final line confirming the Splunk web server is available at `http://splunk-server:8000`. The `Done` status on the "Waiting for web server" line confirms Splunk started successfully and is listening for connections.

![Splunk install complete](screenshots/2026-05-16_17-36_1.png)

---

#### Splunk Web UI Accessed

With Splunk running, the web interface was accessed from the DC02 browser using the Splunk VM's private IP on port 8000.

---

**Splunk Enterprise login page — accessed via browser at 10.0.0.4:8000:**

Opened a browser on DC02 and navigated to `http://10.0.0.4:8000`. The Splunk Enterprise login page loaded successfully, confirming port 8000 is reachable from within the VNet and Splunk's web server process is running. Logged in with the admin credentials set during installation.

![Splunk login page](screenshots/2026-05-16_17-53.png)

---

**Splunk Enterprise home page — logged in as Administrator:**

The Splunk home page confirms a successful login as the Administrator account. The left sidebar shows the installed apps (Search & Reporting, Audit Trail, Splunk Secure Gateway) and the main area presents the onboarding options — Add data, Search your data, Visualise your data, and Manage alerts. This is the starting point for all subsequent configuration.

![Splunk home page](screenshots/2026-05-16_17-53_1.png)

---

### Step 2 — Data Inputs Configured

#### Sysmon Installed on Windows Server

Sysmon (System Monitor) was deployed on DC02 to enrich Windows Event Logs with process creation, network connection, and file change events — providing deeper host visibility beyond standard Windows Security events.

---

**Sysmon v15.2 downloaded from Microsoft Sysinternals:**

On DC02, opened a browser and navigated to `learn.microsoft.com/en-us/sysinternals/downloads/sysmon`. Sysmon v15.2 (4.6 MB) was downloaded. Sysmon was chosen because it generates high-fidelity telemetry that standard Windows Event Logs do not capture — particularly process creation chains and network connections initiated by specific processes.

![Sysmon download page](screenshots/2026-05-16_17-55.png)

---

**Sysmon files extracted to C:\Tools\Sysmon:**

The downloaded Sysmon.zip was extracted to a dedicated folder at `C:\Tools\Sysmon`. The folder contains four files: Eula.txt, Sysmon.exe (32-bit), Sysmon64.exe (64-bit), and Sysmon64a.exe (ARM). Sysmon64.exe was used for this installation since DC02 runs a 64-bit Windows Server environment.

![Sysmon files extracted](screenshots/2026-05-16_18-02.png)

---

**Sysmon configuration file (sysmonconfig-export.xml) saved:**

A Sysmon configuration file was downloaded from a community-maintained repository (SwiftOnSecurity's sysmonconfig) and saved as `sysmonconfig-export.xml` inside `C:\Tools\Sysmon`. The configuration file controls which events Sysmon captures and which it ignores — using a well-maintained community config prevents excessive noise while ensuring critical events are logged.

![Sysmon config saved](screenshots/2026-05-16_18-06.png)

---

**Sysmon64 installed and started — SplunkForwarder service confirmed Running:**

From an Administrator PowerShell session, navigated to `C:\Tools\Sysmon` and ran `.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml`. The output confirms: configuration validated, Sysmon64 installed, SysmonDrv driver installed and started. Running `Get-Service *splunk*` afterwards confirmed the SplunkForwarder service is already running — both the telemetry collector and the log shipper are active simultaneously.

![Sysmon installed and forwarder running](screenshots/2026-05-16_18-33.png)

---

#### Universal Forwarder Installed on Windows Server

The Splunk Universal Forwarder was installed on DC02 to automatically ship Windows Event Logs to the Splunk indexer over the private VNet IP.

- **Deployment Server:** `10.0.0.4:8089`
- **Receiving Indexer:** `10.0.0.4:9997`

---

**Universal Forwarder setup wizard — on-premises Splunk Enterprise instance selected:**

On DC02, downloaded the Splunk Universal Forwarder Windows 64-bit installer from `splunk.com/en_us/download/universal-forwarder.html`. During installation, accepted the licence agreement and selected **An on-premises Splunk Enterprise instance** (not Splunk Cloud). The receiving indexer was set to `10.0.0.4:9997` — the private IP of the Splunk VM and the port configured to receive forwarder data.

![Universal Forwarder setup wizard](screenshots/2026-05-16_18-37.png)

---

**Universal Forwarder installation in progress:**

The installer is copying files to `C:\Program Files\SplunkUniversalForwarder`. The progress bar confirms the installation is proceeding. Once complete, the forwarder registers itself as a Windows service and starts automatically — no manual start required after the wizard finishes.

![Universal Forwarder installing](screenshots/2026-05-16_20-09.png)

---

#### inputs.conf Configured

`inputs.conf` was created at `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf` on DC02 to tell the forwarder exactly which Windows Event Logs to collect and forward to Splunk:

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

After saving the file, the forwarder service was restarted with `Restart-Service SplunkForwarder` to apply the configuration. The `start_from = oldest` setting ensures historical events already on the system were collected, not just new events going forward.

---

## 🔎 SPL Detection Queries

All searches were run inside **Search & Reporting** in the Splunk web UI. The search bar accepts SPL (Splunk Processing Language) queries and returns results from the indexed log data.

### Data Flow Confirmed

---

**Initial search — first events arriving from DC02 (WinEventLog:System):**

To verify logs were flowing, ran `index=* EventCode=1` in the Splunk search bar with the time range set to **Last 24 hours**. The search returned 9 events, all sourced from `DC02` with `sourcetype=WinEventLog:System`. This confirmed the Universal Forwarder on DC02 was successfully shipping logs to Splunk and they were being indexed correctly.

![Initial data flowing](screenshots/2026-05-16_20-25.png)

---

**Security events search — 40,642 events indexed from DC02:**

Ran `index=* sourcetype=WinEventLog:Security` to query the full Security log. The search returned **40,642 events** sourced from DC02 — confirming the Security Event Log, which contains all authentication events (logins, failures, lockouts), was being forwarded and indexed successfully. The Interesting Fields panel on the left shows `EventCode`, `Account_Name`, `Logon_Type`, and other fields automatically extracted by Splunk for Security event types.

![40642 security events](screenshots/2026-05-16_20-28.png)

---

### Successful Logins (Event ID 4624)

---

**EventCode=4624 search — 15 successful login events returned:**

Ran the query below in Search & Reporting with **Last 24 hours** selected. The search filtered Security events to EventCode 4624 (successful logon) only, then grouped results by account name and logon type. 15 events were returned from DC02 — all expected system and service account activity. No interactive user logons (Logon_Type 2 or 10) appeared outside of the admin sessions, confirming no unauthorised access.

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

---

### Failed Login Attempts (Event ID 4625)

The following search was run to identify failed authentication attempts. In a real investigation this would be one of the first queries run when an account lockout alert fires:

```spl
index=* sourcetype=WinEventLog:Security EventCode=4625
| stats count by Account_Name, Workstation_Name
| sort -count
```

> 🔴 **Detection logic:** 5+ failures for one account in a short window indicates a possible brute force attempt.

---

### Account Lockout Events (Event ID 4740)

```spl
index=* sourcetype=WinEventLog:Security EventCode=4740
| table _time, Account_Name, Caller_Computer_Name
| sort -_time
```

> 🔴 **Detection logic:** Multiple lockouts from one source machine indicate a brute force or password spray attack.

---

### Top 10 Failed Login Usernames — Threat Hunting

```spl
index=* sourcetype=WinEventLog:Security EventCode=4625 earliest=-24h
| stats count as failures by Account_Name
| sort -failures
| head 10
```

---

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

A **Windows Security Monitoring Dashboard** was built in Splunk to provide a persistent, at-a-glance view of security posture without running searches manually each time.

---

**Create New Dashboard dialog — Classic dashboard selected:**

Navigated to **Dashboards** in the top Splunk navigation bar, then clicked **Create New Dashboard**. The dashboard was named `Windows Security Monitoring Dashboard` with the description "Security Event monitoring, login tracking, and process activity analysis". **Classic Dashboards** was selected over Dashboard Studio because it uses the simpler XML-based panel editor — better suited for a lab environment where speed of setup matters over visual customisation.

![Create new dashboard](screenshots/2026-05-16_20-33.png)

---

**Adding Login Activity Over Time panel (Line Chart — EventCode=4624 timechart):**

Inside the dashboard editor, clicked **+ Add Panel > New > Line Chart**. The panel was titled "Login Activity Over Time" and the SPL search `index=* sourcetype=WinEventLog:Security EventCode=4624 | timechart count` was entered. A Line Chart was chosen because timechart output is a time-series — the line format makes login volume trends and anomalous spikes immediately visible at a glance.

![Adding line chart panel](screenshots/2026-05-16_20-40.png)

---

**Adding After-Hours Logins panel (Events list — after-hours detection query):**

A second panel was added using **New > Events**. Titled "After-Hours Logins", it uses the after-hours detection SPL with `eval hour` and `where hour < 7 OR hour > 19` to filter logins outside business hours. An Events list was chosen rather than a chart because the exact account name, timestamp, and workstation are the critical details — a chart would aggregate and hide that granularity.

![Adding after-hours panel](screenshots/2026-05-16_20-41.png)

---

**Dashboard — Account Activity Last 24h and Top Processes panels:**

The completed dashboard shows the first two panels populated with live data from DC02. The **Account Activity** bar chart shows DC02$, SYSTEM, and SplunkForwarder as the top authentication sources — all expected service accounts. The **Top Processes** events list shows recent Security events from DC02 with EventCode=4688 (process creation), driven by the Sysmon telemetry configured in Step 2.

![Dashboard account activity and top processes](screenshots/2026-05-16_20-59.png)

---

**Dashboard — Login Activity Over Time and After-Hours Logins panels:**

The bottom two panels show login volume over time and after-hours login events. The **Login Activity Over Time** line chart shows a spike in activity around midnight on May 16-17 — corresponding to the Splunk configuration and forwarder restart sessions. The **After-Hours Logins** panel captures those same sessions as EventCode=4624 events outside the 7AM–7PM window, demonstrating the detection logic is working correctly.

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

---

**Save As Alert dialog — High Privileged Logon Count, scheduled every 15 minutes:**

After running and validating the EventCode=4672 detection query in the search bar, clicked **Save As > Alert**. The alert was configured with: type set to **Scheduled**, cron expression `*/15 * * * *` to run every 15 minutes, time range of **Last 24 hours**, and trigger condition set to **Number of Results is greater than 0**. The trigger action was set to **For each result** so every individual high-privilege account triggers a separate alert entry rather than one combined notification.

![Save as alert configuration](screenshots/2026-05-16_20-46.png)

---

**Detection query results — privileged logons by account and computer:**

Before saving the alert, the underlying SPL query was validated in the search bar. The query returned 22 events showing two accounts — SYSTEM (21 events) and SplunkForwarder (1 event) — both on DC02. These are expected high-privilege service accounts. In a real environment, any unexpected account name appearing in this result would indicate a potential privilege escalation and would require immediate investigation.

![Alert SPL query results](screenshots/2026-05-16_20-50.png)

```spl
index=* sourcetype=WinEventLog:Security EventCode=4672
| stats count as privilege_logons by Account_Name, ComputerName
| where privilege_logons > 0
```

---

**Alert confirmed active — "High Privileged Logon Count" status: Enabled ✅:**

Navigated to **Settings > Searches, Reports, and Alerts** to verify the alert was saved and active. The alert appears in the list as type **Alert**, with next scheduled time of `2026-05-17 01:00:00 UTC` and status **Enabled**. This confirms Splunk will run the detection query every 15 minutes automatically without any manual intervention.

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

---

**splunk-server VM — Status: Running, Standard D2s v3, Ubuntu 24.04:**

Navigated to the `splunk-server` VM overview in the Azure portal to confirm the VM was in a Running state. The essentials panel confirms: OS is Linux (Ubuntu 24.04), size is Standard D2s v3, public IP is 20.102.65.90, private IP is 10.0.0.4, and the VM is connected to the `kingsvm-vnet/default` subnet. All values match the configuration applied during deployment.

![Splunk server VM running](screenshots/2026-05-16_20-53.png)

---

**Network settings — all NSG rules confirmed (final state):**

Navigated to `splunk-server > Networking > Network settings` to confirm the final NSG rule state. Six inbound rules are visible: RDP (3389), allow-splunk-web (8000), and allow-splunk-forwarder (9997) are the three custom rules, all showing **Allow**. The source for all three is correctly set — confirming no ports are accidentally open to the public internet beyond what was intended.

![NSG rules final state](screenshots/2026-05-16_20-53_1.png)

---

**kingsvm_group resource group — complete lab infrastructure (DC02 + splunk-server + VNet + NSG):**

Navigated to **Resource groups > kingsvm_group** to confirm all 10 lab resources were successfully deployed and visible in one place: DC02 (VM), DC02-ip (public IP), dc02363 (NIC), DC02 OS disk, kingsvm-nsg (NSG), kingsvm-vnet (VNet), splunk-server (VM), splunk-server-ip (public IP), splunk-server31 (NIC), and splunk-server disk. Organising all resources in a single resource group allows the entire lab to be deleted cleanly in one operation when no longer needed.

![Resource group all resources](screenshots/2026-05-16_20-55.png)

---

**PowerShell verification — Test-NetConnection :8000 = True, SplunkForwarder Running, Sysmon 71,742 events logged:**

Three final checks were run from DC02 PowerShell as Administrator:
1. `Test-NetConnection 10.0.0.4 -Port 8000` — returned `TcpTestSucceeded: True`, confirming DC02 can reach the Splunk web interface on port 8000 over the VNet
2. `Get-Service *splunk*` — returned `Status: Running` for the SplunkForwarder service, confirming logs are actively being shipped to Splunk
3. `Get-WinEvent -ListLog "Microsoft-Windows-Sysmon/Operational"` — showed a record count of **71,742 events**, confirming Sysmon is actively logging endpoint telemetry on DC02

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

---

**Splunk search history — all detection queries executed during the lab:**

Navigated to the Splunk home page > **Search history** tab to capture a complete record of all SPL queries run during the lab. The history shows 8 searches executed within the last 90 days, covering: after-hours login detection, process creation analysis (EventCode=4688), index enumeration, privileged logon detection (EventCode=4672), full security event queries, successful login analysis (EventCode=4624), Sysmon operational queries, and the initial data confirmation search (EventCode=1). This demonstrates hands-on SPL experience across multiple detection use cases.

![Splunk search history](screenshots/2026-05-16_20-58.png)

---

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
