# 🛡️ Active Directory SOC / IR Lab

## 🎯 Objective
Build a home lab that combines **Active Directory troubleshooting**, **incident detection with Sysmon**, and **SOC/IR workflows** — with Splunk running on a **Raspberry Pi 5** as the SIEM cockpit monitor.

This README documents the setup process step by step, with screenshots and notes.  
Goal: showcase hands-on skills for SOC Analyst / Incident Response roles.

---

## ⚙️ Lab Topology

- **DC01** → Windows Server 2022 Domain Controller (`lab.local`)
- **CLIENT1** → Windows 10/11 domain-joined workstation
- **KALI** → Kali Linux attacker machine
- **Splunk Enterprise** → Raspberry Pi 5 (ARM64 build) with raw monitor

**Network design:**  
- **VMnet1 (Host-only):** 192.168.100.0/24 → isolated AD traffic  
- **Bridged NIC:** 192.168.1.0/24 → home LAN, connects VMs to Pi5 Splunk  

**Specs (per VM):**
- DC01 → 2 vCPU, 4 GB RAM, 60 GB disk  
- CLIENT1 → 2 vCPU, 4 GB RAM, 60 GB disk  
- KALI → 2 vCPU, 4 GB RAM, 40 GB disk  
- SPLUNK (Pi5) → 4-core ARM, 8 GB RAM, 128 GB SD (external SSD recommended)

---

## 🚀 Phase 1 — Core Build

### 1. VMware Workstation Setup
✅ Installed VMware Workstation Pro 17  
✅ Configured **Host-only network (VMnet1)**  
✅ Installed VMware Tools on DC01 + CLIENT1  

📸 `screenshots/vmnet1-settings.png`

---

### 2. Windows Server (DC01)
- Installed Windows Server 2022 Eval ISO  
- Promoted to Domain Controller (`lab.local`)  
- Configured static IP: `192.168.100.10`

📸 `screenshots/dc01-ipconfig.png`  
📸 `screenshots/dc01-domain-setup.png`

---

### 3. Windows 10 Client (CLIENT1)
- Installed Windows 10 ISO  
- Configured static IP: `192.168.100.20`  
- Joined domain → `lab.local`  

📸 `screenshots/client1-domain-login.png`  
🎥 `video/domain-join.mp4`

---

### 4. AD User Management
- Created test user `jdoe`  
- Verified account enabled + successful login  

📸 `screenshots/get-aduser-jdoe.png`

---

### 5. Sysmon Deployment
- Deployed Sysmon (`sysmon64.exe`) + config (`sysmonconfig-export.xml`)  
- Verified events in Event Viewer  

📸 `screenshots/sysmon-install.png`  
📸 `screenshots/sysmon-events.png`

---

## 🚀 Phase 2 — Attack, Detect & Respond

### 1. Kali Attacker Setup
- Installed Kali Linux on VMnet1  
- Tools: `nmap`, `hydra`, `evil-winrm`, `mimikatz`

---

### 2. Adversary Simulation
- Recon with `nmap`  
- Credential spray with `hydra`  
- Attempted WinRM login  

📸 `screenshots/kali-nmap.png`  
📸 `screenshots/kali-cme.png`

---

### 3. Detection in Splunk (ThinkPad host)
- Installed Splunk Universal Forwarder on DC01 + CLIENT1  
- Forwarded Sysmon + Security logs  
- Built dashboards for failed logons (4625) & process creation (EID 1)  

📸 `screenshots/splunk-sysmon-events.png`  
📸 `screenshots/splunk-detection-timeline.png`

---

### 4. Response Actions
- Disabled compromised user  
- Pulled Sysmon history of attack  

📸 `screenshots/adaccount-disable.png`

---

## 🚀 Phase 3 — Splunk Pi5 Cockpit Upgrade

### 1. Pi5 Setup
- Installed Raspberry Pi OS 64-bit  
- Downloaded Splunk ARM64 tarball  
- Extracted to `/opt/splunk`  
- Started Splunk as `splunk` user → Web UI at `http://192.168.1.50:8000`

📸 `screenshots/pi5-splunk-start.png`

---

### 2. Dual-NIC Networking
- DC01 & CLIENT1 each configured with **two NICs**:
  - Host-only (`192.168.100.x`) for AD/DNS
  - Bridged (`192.168.1.x`) for Splunk connectivity  
- Default gateway only on Bridged NIC  

📸 `screenshots/vm-dual-nic.png`  
📸 `screenshots/ipconfig-dual-nic.png`

---

### 3. Forwarders to Pi5
On DC01 + CLIENT1:
```powershell
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" add forward-server 192.168.1.50:9997 -auth admin:YOURPASS
& "C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" add monitor "C:\Windows\System32\winevt\Logs\Microsoft-Windows-Sysmon%4Operational.evtx" -index sysmon -sourcetype=XmlWinEventLog:Microsoft-Windows-Sysmon/Operational -auth admin:YOURPASS
