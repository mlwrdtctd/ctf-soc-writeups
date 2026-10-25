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
The attacker subsequently performed credential harvesting, domain discovery, and established persistence via a malicious Windows Service and a Registry Run Key. 
Finally, the attacker moved laterally to a development server (`THM-SREYES`) and exfiltrated the source code of the unreleased game "NeuralNexus".

---

## 🔍 Kill Chain (Attack Timeline)

### 1. Initial Access (Phishing)
*   **Timestamp:** `2026-10-23 12:02:44`
*   **Event:** Compromised user `m.smith` opened a macro-enabled document from the `Downloads` folder.
*   **Process Chain:** `explorer.exe` -> `WINWORD.EXE` -> `powershell.exe`
*   **IOC:** `C:\Users\m.smith\Downloads\Q3_PressKit_Feedback.docm`

### 2. Execution & Defense Evasion
*   **Timestamp:** `2026-10-23 12:02:44`
*   **Event:** Word spawned a hidden PowerShell process (`ProcessId: 8572`). The command line used Base64 encoding (`-Enc`) and hidden flags (`-NoP -W Hidden`) to evade detection.
*   **Decoded Payload:** The Base64 string revealed a download-and-execute stager pulling `svchost_update.ps1` from the malicious domain `paylink-cdn.thm`.
*   **IOC:** `powershell.exe -NoP -W Hidden -Enc [Base64 String]`

### 3. Command and Control (C2)
*   **Timestamp:** `2026-10-23 12:02:47`
*   **Event:** PowerShell established outbound connections to the C2 server - IP `45.147.230.131`. This IP is marked as malicious in VirusTotal.
*   **IOC:** IP `45.147.230.131`, Port `8067`, Domain `paylink-cdn.thm`

### 4. Credential Access
*   **Timestamp:** `2026-10-23 12:02:48`
*   **Event:** The attacker used the legitimate Windows utility `VaultCmd.exe` to enumerate stored credentials from the Windows Credential Manager.
*   This was done to obtain credentials for lateral movement.
*   **IOC:** `"C:\Windows\Sysnative\vaultcmd.exe" "/listcreds:Windows Credentials"`

### 5. Discovery (Network Enumeration)
*   **Timestamp:** `2026-10-23 12:07:48`
*   **Event:** The attacker executed multiple `net.exe` commands to map the Active Directory environment:
    *   `net.exe user /domain` (Enumerate domain users)
    *   `net.exe group "Domain Admins" /domain` (Identify high-value targets)
    *   `net.exe use \\THM-SREYES\C$` (Attempt lateral movement to an admin share)
*   **IOC:** `net.exe` commands targeting `THM-SREYES`.

### 6. Persistence (Service Creation & Registry Run Key)
*   **Timestamp:** `2026-10-23 12:10:18` (Service) & `2026-09-10 12:37:20` (Registry)
*   **Event:** The attacker established persistence using two methods:
    1.  **On `THM-MSMITH`:** Created a Windows service named `SvcUpdate` running as `LocalSystem`. This service was configured to execute the malicious script `svchost_update.ps1` from `C:\Windows\Temp\` on startup.
    2.  **On `THM-SREYES`:** Created a Registry Run key named `NeuralNexusReview` under `HKU\...\Software\Microsoft\Windows\CurrentVersion\Run\`. This key points to a copied version of the malicious script (`C:\Users\s.reyes\AppData\Local\NeuralNexus\review-helper.ps1`) and ensures it executes every time the user logs in.
*   **IOC:** 
    *   Service Name: `SvcUpdate`
    *   Service File: `C:\Windows\Temp\svchost_update.ps1`
    *   Registry Key: `HKU\...\Run\NeuralNexusReview`
    *   Dropped File: `C:\Users\s.reyes\AppData\Local\NeuralNexus\review-helper.ps1`

### 7. Data Staging & Exfiltration
*   **Timestamp:** `2026-10-23 12:15:37 - 12:25:20`
*   **Event:** On the compromised host `THM-SREYES`, PowerShell (`PID 14264`) used the legitimate `7z.exe` utility to compress the game's source code (`C:\Dev\NeuralNexus\Build\*`) into an archive named `nx_build_beta.7z`.
*   The archive was then exfiltrated to the C2 server over multiple connections.
*   **IOC:** Archive `nx_build_beta.7z`, IP `45.147.230.131`.

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
| Persistence | Boot or Logon Autostart Execution: Registry Run Keys | T1547.001 |
| Collection | Archive Collected Data: Archive via Utility | T1560.001 |
| Exfiltration | Exfiltration Over C2 Channel | T1041 |

---

## 🛡️ Recommendations

### Containment
*   Immediately isolate the affected hosts (`THM-MSMITH` and `THM-SREYES`) from the network via EDR.

### Eradication
*   Delete the malicious script `svchost_update.ps1`.
*   Remove the `SvcUpdate` Windows Service.
*   Remove the `NeuralNexusReview` Registry Run key.
*   Block the malicious IP `45.147.230.131` and domain `paylink-cdn.thm` at the perimeter firewall and proxy.

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

# 4. Detect Registry Run Key Persistence
index=* EventCode=13 TargetObject="*CurrentVersion\\Run*"

# 5. Hunt for Lateral Movement via Net.exe
index=* EventCode=1 Image="*net.exe" CommandLine="*use*\\\\THM-SREYES*"
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
*   **C2 / Exfiltration:** Custom C2 server (`45.147.230.131`), `7z.exe` (archiving), `curl.exe` (uploading).
*   **Discovery:** `net.exe`, `net1.exe`.
*   **Credential Access:** `VaultCmd.exe` (Living off the Land).
*   **Persistence:** `sc.exe` (implicitly, via service creation).

### Frameworks
*   MITRE ATT&CK Framework (for mapping TTPs).
