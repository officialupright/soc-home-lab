# 📝 Incident Report Template (Tier 1 SOC)

> Copy this file for each investigation: `reports/YYYY-MM-DD-short-title.md`.
> Rule of thumb: **another analyst should be able to pick up your ticket and continue without asking you a single question.**

---

## 1. Summary

| Field | Value |
|---|---|
| **Incident ID** | INC-YYYY-#### |
| **Date / Time detected (UTC)** | |
| **Analyst** | |
| **Detection source** | e.g. Wazuh rule 100101 / EDR / user report |
| **Severity** | Low · Medium · High · Critical |
| **Verdict** | True Positive · False Positive · Benign True Positive |
| **Status** | Open · Escalated to Tier 2 · Contained · Closed |

**One-sentence summary:**
> *Example: Unknown process `rundll32.exe` accessed LSASS memory on WIN-LAB-01 under user lab\jdoe, consistent with credential dumping via comsvcs.dll MiniDump (T1003.001).*

---

## 2. Affected Assets

| Hostname | IP | User(s) | Business role |
|---|---|---|---|
| | | | |

---

## 3. Timeline (UTC)

| Time | Source | Event |
|---|---|---|
| 10:02:11 | Sysmon EID 1 | `powershell.exe -enc ...` launched by `explorer.exe` |
| 10:02:15 | Sysmon EID 10 | `rundll32.exe` opened `lsass.exe` (GrantedAccess 0x1FFFFF) |
| 10:02:16 | Sysmon EID 11 | `C:\Windows\Temp\lsass.dmp` created |
| | | |

---

## 4. Indicators of Compromise (defanged)

| Type | Value | Reputation / Notes |
|---|---|---|
| File hash (SHA-256) | | VirusTotal: x/70 |
| Domain | `example[.]com` | |
| IP | `203[.]0[.]113[.]5` | AbuseIPDB score |
| File path | | |

---

## 5. MITRE ATT&CK Mapping

| Tactic | Technique | Evidence |
|---|---|---|
| Execution | T1059.001 PowerShell | Encoded command at 10:02:11 |
| Credential Access | T1003.001 LSASS Memory | Sysmon EID 10 at 10:02:15 |

---

## 6. Analysis

- **What happened?**
- **How did it start?** (initial access, if known)
- **Why is this malicious / benign?** Link your evidence.
- **Scope:** were other hosts or users affected? (Show the search you ran.)

---

## 7. Actions Taken

- [ ] Host isolated (EDR network containment)
- [ ] Affected credentials reset
- [ ] Malicious files quarantined / hashes blocked
- [ ] IOCs blocked at firewall / proxy / email gateway
- [ ] Escalated to: ______

---

## 8. Recommendations & Lessons Learned

- Detection gaps found:
- Tuning needed (false positives?):
- Preventive controls (e.g. enable LSA Protection / Credential Guard, restrict PowerShell with Constrained Language Mode):

---

*Template follows the incident handling phases of [NIST SP 800-61](https://csrc.nist.gov/pubs/sp/800/61/r3/final): Detection & Analysis → Containment → Eradication & Recovery → Post-Incident Activity.*
