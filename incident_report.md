# SOC Investigation Lab: Incident Report

## Executive Summary

On June 16, 2026, a malicious executable (update.exe) was executed on workstation DESKTOP-9BB4HVN under user admin. The payload established a reverse shell connection to attacker infrastructure at 192.168.1.18:4444, enabling command execution and system enumeration. Attempted persistence via backdoor account creation was blocked by Windows UAC controls. The incident was detected by endpoint protection identifying the payload as Trojan:Win32/Zusy.NCDIMTB.

## Incident Timeline

| Time (UTC) | Event | Details |
|---|---|---|

|2026-06-16|Process Creation (T1059)| update.exe executed from C:\Users\admin\Downloads\ by explorer.exe (user double-click). MD5: 0B831CBACBE56220C4746F478AE7DE4B, SHA256: C414F683EDA8D7650B706A0644CA44293DB27664324564438DBDF6BE6D42DD16 |

| 2026-06-16 22:28:03.133 | Network Connection (T1071) | Reverse TCP callback: 192.168.1.21:49919 → 192.168.1.18:4444. Meterpreter session established. |

| ~2026-06-16 22:28-22:30 | System Enumeration (T1033/T1087) | Attacker ran whoami /all, net user. Discovered admin account in Administrators group at Medium integrity (UAC non-elevated). |

| 2026-06-16 22:30 (approx) | Failed Persistence (T1136.001) | Attempted net user hacker Pass123! /add — blocked with "Access is denied" due to UAC integrity restriction. |

| 2026-06-16 19:37 | Malware Detection | Windows Defender flagged payload as Trojan:Win32/Zusy.NCDIMTB. |

## Indicators of Compromise (IOCs)

**File:**
- Filename: update.exe
- Path: C:\Users\admin\Downloads\update.exe
- MD5: 0B831CBACBE56220C4746F478AE7DE4B
- SHA256: C414F683EDA8D7650B706A0644CA44293DB27664324564438DBDF6BE6D42DD16
- Detection: Trojan:Win32/Zusy.NCDIMTB

**Network:**
- Source: 192.168.1.21 (victim, Win10 VM)
- Destination: 192.168.1.18:4444 (attacker, Kali Linux)
- Protocol: TCP
- Direction: Outbound (reverse shell callback)

## MITRE ATT&CK Mapping

| Technique | ID | Evidence |
|---|---|---|

| Phishing: Malicious File | T1566.001 | update.exe delivered as suspicious download |

| User Execution: Malicious File | T1204.002 | File executed by user double-clicking from Downloads |

| Command and Scripting Interpreter | T1059.003 | Meterpreter reverse shell enabling cmd.exe execution |

| System Information Discovery | T1033 | whoami /all, systeminfo commands executed |

| Account Discovery | T1087.001 | net user enumeration of local accounts |

| Create Account | T1136.001 | Attempted backdoor user creation (net user hacker...), blocked by UAC |

## Detection Opportunities

**What Sysmon Captured :**
- Process creation events (EventID 1) showing exact command line, parent process, and file hash
- Network connection events (EventID 3) showing source/destination IP:port and direction
- Both events timestamped to the second, enabling precise correlation

**What Could Have Alerted Sooner:**
1. File reputation / hash lookup: The file hash should be checked against threat intelligence before execution (pre-execution blocking)
2. Outbound connection filtering: TCP port 4444 to non-internal IPs should be flagged as suspicious (egress filtering)
3. UAC event logging: Enable and monitor UAC bypass attempts (Event ID 4673 in Security log)
4. Behavioral detection: Newly-launched process immediately making external network connections (common malware pattern)

## Remediation

**Immediate:**
- Isolate DESKTOP-9BB4HVN from the network
- Terminate the Meterpreter session / kill process 3156
- Quarantine update.exe (already done by Defender)
- Review user's email/recent downloads for additional malicious files

**Short-term:**
- Audit admin account's recent activity (logon times, executed commands via command history)
- Check for lateral movement attempts from this workstation to other systems
- Force password reset for admin account

**Long-term:**
- Implement application whitelisting to prevent execution from Downloads folder
- Enable Windows Defender Application Guard for untrusted files
- Deploy EDR (Endpoint Detection & Response) for behavioral detection of suspicious processes
- Enforce strict UAC policies and monitor bypass attempts

## Lab Infrastructure

**Objective:** Simulate a complete attack-to-detection workflow with Sysmon logging and Splunk ingestion.

**Architecture:**
- Attacker (Kali Linux): 192.168.1.18, running Metasploit for payload generation and listener
- Target (Windows 10): 192.168.1.21, instrumented with Sysmon + SwiftOnSecurity config, Universal Forwarder
- SIEM (Splunk): Huawei laptop, ingesting Sysmon and Windows Security logs via TCP:9997

**Tools Used:**
- msfvenom: reverse shell payload generation
- Metasploit Framework: exploit handler and post-exploitation
- Sysmon: process/network/file monitoring
- Splunk: log aggregation and analysis

**Key Findings:**
- Field extraction quirk: Sysmon XML events require custom regex for EventID filtering (<EventID> as element text, not attribute)
- Firewall rules: Both ICMP and TCP port 9997 required explicit allow rules (default-deny stance)
- UAC boundaries: Medium-integrity admin user cannot create accounts, requiring token impersonation/privilege escalation (all attempts failed on patched Win10 build)