# 🛡️ SOC Analyst Portfolio (CTF Write-ups)

Welcome! This repository contains my hands-on incident response and threat hunting write-ups based on TryHackMe CTF simulations. I focus on log analysis (Splunk, Sysmon), attack chain reconstruction, and MITRE ATT&CK mapping.

## 🎯 About This Project
The goal of this portfolio is to demonstrate practical SOC Analyst skills, including:
*   **Incident Triage & Analysis:** Investigating alerts and determining True/False Positives.
*   **Log Analysis:** Pivoting through Windows Event Logs and Sysmon telemetry.
*   **Threat Hunting:** Reconstructing multi-stage attacks (Kill Chain) from raw logs.
*   **Detection Engineering:** Writing Splunk SPL queries to identify malicious activity.
*   **MITRE ATT&CK Mapping:** Aligning adversary behavior with known TTPs.

---

## 📂 Projects

### 1. [Repossessed: Phishing & Lateral Movement](01_Incident_Response_Reports/THM_Repossessed_Initial_Access.md)
*   **Scenario:** Initial Access via malicious `.docm` macro leading to C2 communication and Lateral Movement.
*   **Key Skills:** Base64 decoding, C2 traffic analysis, Windows Service Persistence, Scheduled Task analysis, Data Exfiltration (Game source code & Crash logs).
*   **Key Techniques:** T1566.001, T1059.001, T1543.003, T1053.005, T1041.

---

## 🛠️ Tools & Technologies Used
*   **SIEM:** Splunk (SPL queries for event correlation and pivoting).
*   **Logging:** Sysmon (Event IDs 1, 3, 11, 13, 22), Windows Security Event Logs (Event IDs 4697, 4624).
*   **Threat Intelligence:** VirusTotal (IP reputation check).
*   **Frameworks:** MITRE ATT&CK.
*   **Analysis:** Base64 decoding, PowerShell Script Block Logging (Event ID 4104).

---

## 📫 Contact
*   **GitHub:** [mlwrdtctd](https://github.com/mlwrdtctd)
