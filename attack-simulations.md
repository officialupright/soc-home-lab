# ⚔️ Attack Simulations: Attack → Detect → Triage

Each exercise follows the same loop a real SOC runs every day:

| Phase | What you do |
|---|---|
| 🔴 **Attack** | Run a safe simulation of a real attacker technique on the Windows VM |
| 🔵 **Detect** | Find the alert in Wazuh and inspect the raw event |
| 🧠 **Triage** | Answer the analyst questions and decide: true positive, false positive, or benign |

> ⚠️ Run these **only** inside the isolated lab. Take a VM snapshot first so you can roll back.

### One-time prep: install Atomic Red Team (Windows VM, admin PowerShell)

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```

> 💡 Windows Defender will block some tests. That is a detection too! For lab learning you can temporarily add an exclusion for `C:\AtomicRedTeam`.

---

## Exercise 1: Encoded PowerShell (T1059.001 / T1027)

**🔴 Attack**: run a harmless command, base64-encoded the way attackers do it:

```powershell
powershell.exe -NoProfile -EncodedCommand VwByAGkAdABlAC0ATwB1AHQAcAB1AHQAIAAnAFMATwBDACAAbABhAGIAIAB0AGUAcwB0ACAALQAgAFQAMQAwADUAOQAuADAAMAAxACcA
```

**🔵 Detect**: in Wazuh, go to **Threat Hunting** and filter `rule.id: 100100`.

**🧠 Triage questions**
- Decode the payload in [CyberChef](https://gchq.github.io/CyberChef/) (*From Base64* → *Decode text: UTF-16LE*). What does it do?
- What is the **parent process** (`win.eventdata.parentImage`)? `explorer.exe` suggests a user ran it; `winword.exe` suggests a malicious macro.
- Did the same host make a network connection right afterwards (Sysmon Event ID 3)?

---

## Exercise 2: Credential Dumping from LSASS (T1003.001)

**🔴 Attack**
```powershell
Invoke-AtomicTest T1003.001 -ShowDetailsBrief   # list available tests
Invoke-AtomicTest T1003.001 -TestNumbers 2       # comsvcs.dll MiniDump technique
```

**🔵 Detect**: filter `rule.id: 100101` (Sysmon Event ID 10, ProcessAccess).

**🧠 Triage questions**
- Which **source process** touched `lsass.exe`? What were the **GrantedAccess** rights (`0x1010` and `0x1FFFFF` are classic dumping values)?
- Is there a new `.dmp` file on disk (Sysmon Event ID 11, FileCreate)?
- 🚨 In a real environment this is a **critical** alert: isolate the host and assume every credential cached on it is compromised.

**🧹 Cleanup**: `Invoke-AtomicTest T1003.001 -TestNumbers 2 -Cleanup`

---

## Exercise 3: Persistence via Scheduled Task (T1053.005)

**🔴 Attack**
```cmd
schtasks /create /tn "WindowsUpdateCheck" /tr "C:\Windows\System32\calc.exe" /sc onlogon /ru SYSTEM
```

**🔵 Detect**: filter `rule.id: 100102`.

**🧠 Triage questions**
- The task name *sounds* legitimate. Does the **action** (`/tr`) match? Real Windows Update tasks don't launch from user folders.
- Which account created it, and is that normal for this host?

**🧹 Cleanup**: `schtasks /delete /tn "WindowsUpdateCheck" /f`

---

## Exercise 4: Backdoor Local Account (T1136.001)

**🔴 Attack**
```cmd
net user labbackdoor P@ssw0rd!Lab123 /add
net localgroup administrators labbackdoor /add
```

**🔵 Detect**: filter `rule.id: 100103`. Also look for Windows Security Event **4720** (account created) and **4732** (added to a security group).

**🧠 Triage questions**
- Was the new account added to **Administrators** within minutes? That combination is a strong signal of malicious activity.
- Is there a matching change request / IT ticket?

**🧹 Cleanup**: `net user labbackdoor /delete`

---

## Exercise 5: Living off the Land with Certutil (T1105)

**🔴 Attack**
```cmd
certutil.exe -urlcache -split -f https://raw.githubusercontent.com/redcanaryco/atomic-red-team/master/LICENSE.txt C:\Users\Public\test.txt
```

**🔵 Detect**: filter `rule.id: 100104`.

**🧠 Triage questions**
- What URL was contacted? Check the domain's reputation in **VirusTotal** and **URLScan**.
- Where was the file written? `C:\Users\Public`, `%TEMP%` and `AppData` are common staging folders.

**🧹 Cleanup**: `del C:\Users\Public\test.txt`

---

## Exercise 6: Covering Tracks by Clearing Logs (T1070.001)

**🔴 Attack**
```cmd
wevtutil cl Application
```

**🔵 Detect**: filter `rule.id: 100105`, and look for Windows Event **104** (System log) or **1102** (Security log cleared).

**🧠 Triage questions**
- Because Wazuh already shipped the events off the host, **the evidence survives**. Rebuild the timeline of what happened *before* the logs were cleared.
- Who cleared the logs, and why would they need to?

---

## ✅ After Each Exercise

Write it up using [reports/incident-report-template.md](reports/incident-report-template.md). Practising short, clear write-ups is one of the fastest ways to sound like an experienced analyst in interviews.

## 🧩 Challenge Yourself

1. Tune rule **100102** to ignore tasks created by your patch-management tool (reduce false positives).
2. Write a new rule for **T1547.001** (Registry Run Keys) using Sysmon Event ID 13.
3. Chain exercises 1 → 3 → 6 as one "attack story" and investigate it as a single incident.
