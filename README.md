# SOC + Active Directory Home Lab - Full Report

> Precision is everything. This repo documents the complete build of my cybersecurity home lab - from setup, through errors, to final SOC Cockpit display. Every twist, mistake, and fix became part of the learning process.

---

## 🎯 Lab Goals
- Deploy a working **Active Directory domain** (Windows Server 2022 DC02 + Windows 10 client).
- Join client machines to the domain (`lab.local`).
- Install and configure **Sysmon** for endpoint telemetry.
- Install and configure **Splunk Universal Forwarder** to send logs to **Splunk Enterprise**.
- Build a **SOC dashboard (“cockpit”)** with:
  - Failed logons (4625)
  - Successful logons (4624)
  - Top Security Event Codes (pie/bar)
- Display the cockpit in real-time on a **Raspberry Pi5**.

---

## 🖥️ Architecture

**Host (ThinkPad)**  
- VMware Workstation Pro 17  
- Splunk Enterprise (Trial)  
- Runs DC02 + CLIENT01 VMs  

**VMs (on Laptop)**  
- **DC02**: Windows Server 2022, Domain Controller, `192.168.100.X2` (VMnet1 Host-Only + Bridged NIC)  
- **CLIENT01**: Windows 10/11, Domain joined, `192.168.100.X3` (VMnet1 Host-Only + Bridged NIC)  
- **Kali Linux** (optional, adversary simulation — Nmap, Hydra, etc.)  

**Raspberry Pi5**  
- Chromium browser, fullscreen dashboard view (Pi5 “SOC Cockpit”)  

---

## ⚙️ Installation & Setup

### A. Sysmon
```powershell
# Move sysmon64.exe + config to C:\Tools\Sysmon
Set-Location C:\Tools\Sysmon

# Example config (saved as sysmonconfig-export.xml)
@'
<Sysmon schemaversion="4.82">
  <HashAlgorithms>sha256</HashAlgorithms>
  <EventFiltering>
    <ProcessCreate onmatch="include"/>
    <NetworkConnect onmatch="include"/>
    <ImageLoad onmatch="include"/>
    <ProcessTerminate onmatch="include"/>
    <DriverLoad onmatch="include"/>
    <CreateRemoteThread onmatch="include"/>
    <RawAccessRead onmatch="include"/>
    <ProcessAccess onmatch="include"/>
    <FileCreate onmatch="include"/>
    <FileCreateTime onmatch="include"/>
    <RegistryEvent onmatch="include"/>
    <FileCreateStreamHash onmatch="include"/>
    <PipeEvent onmatch="include"/>
    <WmiEvent onmatch="include"/>
    <DnsQuery onmatch="include"/>
  </EventFiltering>
</Sysmon>
'@ | Out-File "C:\Tools\Sysmon\sysmonconfig-export.xml" -Encoding ASCII

# Install Sysmon with config
.\sysmon64.exe -accepteula -i .\sysmonconfig-export.xml

# Verify
Get-Service sysmon64
Get-WinEvent -ListLog Microsoft-Windows-Sysmon/Operational -MaxEvents 3
```

## B. Splunk Universal Forwarder (on DC02 + CLIENT01)
```
inputs.conf

[WinEventLog://Security]
disabled = 0
[WinEventLog://System]
disabled = 0
[WinEventLog://Application]
disabled = 0
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0


outputs.conf

[tcpout]
defaultGroup = default-autolb-group
[tcpout:default-autolb-group]
server = <Laptop_IP>:9997


Restart + verify

& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" restart
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" list forward-server
```
## C. VMware Networking (dual-NIC pattern)

Adapter 1: VMnet1 (Host-Only) → AD traffic only

Adapter 2: VMnet0 (Bridged) → connects VM traffic to ThinkPad (Splunk)
.\sysmon64.exe -accepteula -i .\sysmonconfig-export.xml

# Verify
``Get-Service sysmon64
Get-WinEvent -ListLog Microsoft-Windows-Sysmon/Operational -MaxEvents 3``

##🔎 Splunk Queries Used

Check ingestion

``index=* host=* | stats count by host sourcetype``


Failed logons (4625)

``index=* (host=DC02 OR host=CLIENT01) sourcetype=WinEventLog:Security EventCode=4625
| timechart span=1m count by host``


Successful logons (4624)

``index=* (host=DC02 OR host=CLIENT01) sourcetype=WinEventLog:Security EventCode=4624
| timechart span=1m count by host``


Top Security Event Codes

``index=* (host=DC02 OR host=CLIENT01) sourcetype=WinEventLog:Security
| stats count by EventCode
| sort - count | head 10``


Sysmon Process Creates (if sourcetype available)

``index=* (host=DC02 OR host=CLIENT01) (sourcetype=*Sysmon* OR source="*Microsoft-Windows-Sysmon/Operational*")
| eval Code = coalesce(EventCode, EventID)
| where Code=1
| stats count by host, Image, User
| sort - count | head 20``

## 📊 SOC Cockpit Dashboard

Panel 1: Failed Logons (4625) - line chart

Panel 2: Successful Logons (4624) - line chart

Panel 3: Security Events - Top Event Codes (pie or bar)

### Display:

Laptop - build + manage dashboard

Pi5 - fullscreen display (http://<Laptop_IP>:8000)

## 🛠️ Troubleshooting Lessons

VM not reaching Splunk → Needed dual-NIC setup (VMnet1 + Bridged).

Splunk UF no events → inputs.conf was saved as .txt → must save with no extension.

Sysmon subscription error (errorCode=5) → fixed by running SplunkForwarder under LocalSystem + resetting channel ACL.

Host showed as WIN_XXXX in Splunk → set [default] host = CLIENT01 in inputs.conf or rename computer.

Firewall issues → Allowed splunkd on port 9997 through Windows Firewall.

## 📸 Screenshots & Media

01_ingestion_table.png → Splunk table showing DC02 + CLIENT01 hosts

02_dashboard_thinkpad.png → SOC Cockpit on ThinkPad

03_dashboard_pi5.png → SOC Cockpit on Pi5 (Matrix-style)

04_sysmon_started.png → Sysmon service running

05_forwarder_active.png → Forwarder status active

media/01_cockpit_spike.mp4 → Demo clip: failed + successful logins lighting up dashboard

## 🎶 Reflections

Music production taught me: if the signal flow is wrong, nothing works.

Cybersecurity taught me: if routing, DNS, or event channels are wrong, detection fails.

In both, precision builds clarity.

## 🚀 Next Steps

Phase 3: Add Kali attacks (Hydra brute-force, Nmap scans).

Phase 4: Enrich Splunk with detection rules (Sigma → Splunk).

Phase 5: Explore Dockerized Splunk on Pi5 for full ARM-native cockpit.

## 📖 Author

Antwoine Collins - Documenting my pivot into cybersecurity through labs, projects, and persistence.

LinkedIn: https://www.linkedin.com/in/antwoinecollins/

