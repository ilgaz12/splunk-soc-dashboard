# Splunk SOC Dashboard

A **Comprehensive SOC Monitoring Dashboard** built in **Splunk** that combines **Windows Security Event Log analysis**, **logon activity tracking**, and **system health metrics** (CPU & Memory) in one place.  
This lab uses **Ubuntu Linux** for Splunk Enterprise and a **Windows 10 VM** with the Universal Forwarder.

---

## 🧩 Architecture / Lab Topology

- **Splunk Enterprise (Indexer + Search Head)** on **Ubuntu Linux (VM)**
  - Splunk Web: `http://<ubuntu-ip>:8000`
  - Receiver: **listens on TCP 9997** for forwarders

- **Windows 10 VM (Endpoint)** with **Splunk Universal Forwarder**
  - Forwards **WinEventLog** (Security/System/Application)
  - Forwards **Perfmon** metrics (CPU & Memory) to Ubuntu

```
Windows 10 VM (UF)  ───►  Ubuntu Linux (Splunk Enterprise)
   - WinEventLog                   - Receiver :9997
   - Perfmon (CPU / Memory)        - Splunk Web :8000
```

**Tested With:** Ubuntu 22.04 LTS, Windows 10 Pro 22H2, Splunk Enterprise 9.x, Splunk UF 9.x

---

## 📊 What the Dashboard Shows

- **Event Type Breakdown** (Successful Logon, Failed Logon, Logoff, Privileged Logon)
- **Privileged Logons** (single-value KPI)
- **Top Users (Failed Logons)** and **Top Source IPs**
- **Event Trend Over Time** (timechart by event category)
- **Windows Event Logs Overview** (table by sourcetype / EventCode)
- **CPU Usage Over Time** (Perfmon:CPU)
- **Memory Usage Over Time** (Perfmon:Memory)

> Dashboard XML filename suggestion: `splunk-soc-dashboard.xml`  
> Import it in Splunk: **Dashboards → Create New → Classic → Import from XML**

---

## ⚙️ Configuration (Windows 10 UF)

Place this in `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`:

```ini
[default]
host=Win10-Endpoint

[monitor://C:\Windows\System32\winevt\Logs\Security.evtx]
disabled=false

[monitor://C:\Windows\System32\winevt\Logs\System.evtx]
disabled=false

[monitor://C:\Windows\System32\winevt\Logs\Application.evtx]
disabled=false

[perfmon://CPU]
disabled=0
object=Processor
counters=% Processor Time
instances=*
interval=10
index=perfmon

[perfmon://Memory]
disabled=0
object=Memory
counters=Available MBytes
instances=*
interval=10
index=perfmon
```

Then point the UF to the Splunk server (replace IP/password):
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" add forward-server <UBUNTU_IP>:9997 -auth admin:<password>
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" enable boot-start
Restart-Service splunkforwarder
```

Verify:
```powershell
# Forwarding target should show Status: active
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" list forward-server

# Inputs should show Perfmon CPU/Memory as Active
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" list inputstatus
```

---

## 🖥️ Configuration (Ubuntu Splunk Enterprise)

Enable receiving and start Splunk:
```bash
sudo /opt/splunk/bin/splunk enable listen 9997
sudo /opt/splunk/bin/splunk start
# Splunk Web: http://<ubuntu-ip>:8000
```

Create an **Events** index for Perfmon (recommended): `Settings → Indexes → New Index → Name: perfmon`

---

## 🔎 Useful Searches

```spl
# Confirm perfmon data
index=perfmon sourcetype="Perfmon:CPU" OR sourcetype="Perfmon:Memory"
| timechart avg(Value) by sourcetype

# Security log activity by event code
index=* sourcetype=WinEventLog:Security
| stats count by EventCode
| sort - count
```

> **Note:** If CPU/Memory timelines show gaps, the Windows VM or UF may have been powered off. Ensure the service is set to auto-start:
```cmd
sc config SplunkForwarder start= auto
```

---

## 📁 Repository Structure

```
splunk-soc-dashboard/
├── README.md                    
├── splunk-soc-dashboard.xml     
└── screenshots/                 
```

---

## 📄 License
This project is for educational and demonstration purposes.
