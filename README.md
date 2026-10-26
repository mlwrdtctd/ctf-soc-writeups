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

### 2. [Quiet as the Grave: Compile After Delivery & Data Exfiltration](https://github.com/mlwrdtctd/ctf-soc-writeups/blob/main/02_Incident_Response_Reports/THM_QuietAstheGrave_CompileAfterDelivery%26_DataExfiltration.md)
*   **Scenario:** Defense Evasion via on-the-fly C# compilation, Registry Persistence, Lateral Movement, and Data Exfiltration.
*   **Key Skills:** Analyzing `csc.exe` compilation, Living off the Land (LotL) using `curl.exe`, PowerShell Remoting (`New-PSSession`), Data Staging (`Compress-Archive`), Registry Run Key analysis.
*   **Key Techniques:** T1059.001, T1027.004, T1547.001, T1021.006, T1560.001, T1041.
---

## 🛠️ Tools & Technologies Used

### Defensive Tools (Analyst Side)
*   **SIEM:** Splunk (SPL queries for event correlation, pivoting, and hunting).
*   **Logging & Telemetry:** Sysmon (Process Create, FileCreate, NetworkConnect, RegistrySet), Windows Security Event Logs (Service Creation, Logon Events), PowerShell Script Block Logging (Event ID 4104).
*   **Threat Intelligence:** VirusTotal (IP/Domain reputation check).
*   **Frameworks:** MITRE ATT&CK.

### Analysis & Investigation Skills
*   **Execution Analysis:** Base64 decoding, deobfuscation of malicious command lines.
*   **Defense Evasion Detection:** Identifying "Compile After Delivery" (analyzing `csc.exe` and `cvtres.exe`), Living off the Land (LotL) binaries.
*   **Persistence Analysis:** Investigating Windows Services (`SvcUpdate`) and Registry Run Keys (`HKU\...\Run\`).
*   **Lateral Movement Tracking:** Tracing PowerShell Remoting (`New-PSSession`) via Process IDs and Logon IDs.

### Offensive Tools (Adversary Side - Observed in Logs)
*   **Execution:** PowerShell (`powershell.exe`), `cmd.exe`, `WINWORD.EXE`.
*   **Defense Evasion:** `csc.exe` (C# Compiler), `cvtres.exe` (Resource Compiler).
*   **C2 / Exfiltration:** Custom C2 servers (`45.147.230.131`, `104.18.32.47`), `7z.exe` (Archiving), `curl.exe` (Uploading), `Invoke-RestMethod`.
*   **Discovery:** `net.exe`, `net1.exe`, `VaultCmd.exe` (Credential Access).

---

## 📫 Contact
*   **GitHub:** [mlwrdtctd](https://github.com/mlwrdtctd)
