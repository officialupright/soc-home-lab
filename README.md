<h1 align="center">🛡️ SOC Home Lab: Detect Real Attacks with Wazuh + Sysmon</h1>

<p align="center">
  <img src="https://img.shields.io/badge/SIEM-Wazuh-005571?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Telemetry-Sysmon-0078D6?style=for-the-badge&logo=windows&logoColor=white" />
  <img src="https://img.shields.io/badge/Framework-MITRE%20ATT%26CK-C8102E?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Cost-%240-2E8B57?style=for-the-badge" />
</p>

<p align="center"><b>A free, hands-on lab that teaches you how a SOC analyst actually works:<br/>generate attacker behavior → collect the telemetry → write the detection → triage the alert.</b></p>

---

## 📌 Why this lab exists

Most entry-level SOC candidates can *define* a SIEM. Far fewer can show that they have **built one, attacked it, and caught the attack**. This repo is a step-by-step guide to doing exactly that on a single laptop, with free tools.

By the end you will be able to:

- ✅ Deploy a working SIEM (Wazuh) and onboard Windows and Linux endpoints
- ✅ Collect high-value Windows telemetry with **Sysmon**
- ✅ Safely simulate attacker techniques with **Atomic Red Team**
- ✅ Write **custom detection rules** and map them to **MITRE ATT&CK**
- ✅ Triage alerts and write a clear incident report, the way a Tier 1 analyst would

---

## 🏗️ Lab Architecture

```
                    ┌──────────────────────────────┐
                    │     Wazuh Manager + Indexer   │
                    │     + Dashboard (Ubuntu VM)   │
                    │     192.168.56.10             │
                    └──────────────▲───────────────┘
                                   │  agent traffic (1514/1515 TCP)
             ┌─────────────────────┼─────────────────────┐
             │                     │                     │
   ┌─────────┴─────────┐ ┌─────────┴─────────┐ ┌─────────┴─────────┐
   │ Windows 10/11 VM  │ │  Ubuntu Server VM │ │   Kali Linux VM   │
   │ Wazuh agent       │ │  Wazuh agent      │ │   "Attacker"      │
   │ Sysmon            │ │  auditd           │ │   (no agent)      │
   │ 192.168.56.20     │ │  192.168.56.30    │ │   192.168.56.50   │
   └───────────────────┘ └───────────────────┘ └───────────────────┘
               Host-only network: 192.168.56.0/24 (isolated)
```

| Component | Role | Minimum resources |
|---|---|---|
| Ubuntu 22.04 (Wazuh all-in-one) | SIEM, indexer, dashboard | 2 vCPU · 4–8 GB RAM · 50 GB |
| Windows 10/11 | Victim endpoint + Sysmon | 2 vCPU · 4 GB RAM |
| Ubuntu Server | Linux victim endpoint | 1 vCPU · 1 GB RAM |
| Kali Linux | Attacker | 2 vCPU · 2 GB RAM |

> 💡 **Low on RAM?** Run only Wazuh + Windows. That pair covers about 80% of the exercises.

---

## 🚀 Setup Guide

### Step 1: Create an isolated network
In VirtualBox: **File → Tools → Network Manager → Host-only Networks → Create** (`192.168.56.0/24`). Attach every VM to it. Keep the lab **off your home network**, because you will be running attack tooling.

### Step 2: Install Wazuh (all-in-one)
On the Ubuntu VM:

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
# Save the admin password printed at the end!
```
Browse to `https://192.168.56.10` and log in as `admin`.

### Step 3: Install Sysmon on Windows
Sysmon is where most of the detection value comes from: process creation, network connections, LSASS access, and more.

```powershell
# Run as Administrator
Invoke-WebRequest https://download.sysinternals.com/files/Sysmon.zip -OutFile Sysmon.zip
Expand-Archive Sysmon.zip -DestinationPath C:\Tools\Sysmon
Invoke-WebRequest https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile C:\Tools\Sysmon\sysmonconfig.xml
C:\Tools\Sysmon\Sysmon64.exe -accepteula -i C:\Tools\Sysmon\sysmonconfig.xml
```

### Step 4: Enroll the Windows agent and forward Sysmon logs
In the Wazuh dashboard: **Agents → Deploy new agent → Windows**, then run the generated command. Next, add this to `C:\Program Files (x86)\ossec-agent\ossec.conf`:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```
Restart the agent: `Restart-Service WazuhSvc`

### Step 5: Load the custom detection rules
Copy [`wazuh/local_rules.xml`](wazuh/local_rules.xml) into `/var/ossec/etc/rules/local_rules.xml` on the manager, then:

```bash
sudo /var/ossec/bin/wazuh-logtest     # optional: test a log line
sudo systemctl restart wazuh-manager
```

### Step 6: Attack, detect, triage
Follow [`attack-simulations.md`](attack-simulations.md). Each exercise gives you the command to run, what to look for in the SIEM, and the questions a Tier 1 analyst should answer.

---

## 🎯 Detection Coverage

| Rule ID | Detection | MITRE ATT&CK | Severity |
|---|---|---|---|
| 100100 | Encoded / obfuscated PowerShell | [T1059.001](https://attack.mitre.org/techniques/T1059/001/) · [T1027](https://attack.mitre.org/techniques/T1027/) | 🔴 High |
| 100101 | Process accessing LSASS memory (credential dumping) | [T1003.001](https://attack.mitre.org/techniques/T1003/001/) | 🔴 Critical |
| 100102 | Scheduled task created from the command line | [T1053.005](https://attack.mitre.org/techniques/T1053/005/) | 🟠 Medium |
| 100103 | Local user account created with `net user /add` | [T1136.001](https://attack.mitre.org/techniques/T1136/001/) | 🟠 Medium |
| 100104 | Certutil used to download a file (LOLBin) | [T1105](https://attack.mitre.org/techniques/T1105/) | 🔴 High |
| 100105 | Windows event logs cleared | [T1070.001](https://attack.mitre.org/techniques/T1070/001/) | 🔴 High |

---

## 🧭 How to Triage an Alert (Tier 1 workflow)

When one of these rules fires, work through the same five questions every time:

1. **What happened?** Read the full event: process, command line, parent process, user, host.
2. **Is it expected?** Check whether this user or host normally does this (admin script? IT tool?).
3. **What came before and after?** Pivot on host and time (±15 min). Look at parent/child processes and network connections.
4. **What's the scope?** Search the same hash, command line or user across *all* agents.
5. **Decide and document.** Mark it true positive, false positive or benign, record your evidence, and escalate if needed.

> 📝 A ready-to-use template lives in [`reports/incident-report-template.md`](reports/incident-report-template.md).

---

## 📂 Repository Structure

```
soc-home-lab/
├── README.md                          ← you are here
├── attack-simulations.md              ← exercises: attack → detect → triage
├── wazuh/
│   └── local_rules.xml                ← custom detection rules (ATT&CK-mapped)
└── reports/
    └── incident-report-template.md    ← Tier 1 incident write-up template
```

---

## 📚 Resources I Learned From

- [Wazuh documentation](https://documentation.wazuh.com/)
- [Sysmon (Microsoft Sysinternals)](https://learn.microsoft.com/sysinternals/downloads/sysmon) and [SwiftOnSecurity sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Atomic Red Team (Red Canary)](https://github.com/redcanaryco/atomic-red-team)
- [MITRE ATT&CK](https://attack.mitre.org/)

---

## ⚠️ Disclaimer

This lab is for **education and defensive research only**. Run attack simulations only on systems you own, inside an isolated network.

<p align="center">Built by <a href="https://github.com/officialupright">Ifeanyi Solomon Uneze</a> · SOC Analyst · <a href="https://www.linkedin.com/in/ifeanyi-solomon-uneze-6536b8450/">LinkedIn</a><br/>⭐ If this helped you, star the repo and share it with someone starting out in blue team!</p>
