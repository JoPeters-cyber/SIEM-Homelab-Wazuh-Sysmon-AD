# ⚔️ Attack Simulation & Detection Lab

This directory turns the homelab into a **detection engineering loop**: safely
simulate an adversary technique, capture the telemetry in Wazuh/Sysmon, write a
detection rule, then tune it. Every simulation is mapped to
[MITRE ATT&CK](https://attack.mitre.org/) so the lab doubles as a portfolio of
detections.

> ⚠️ **Lab use only.** Run every technique below **inside the isolated lab
> network** on VMs you own. Snapshot before each run. Never run these against
> systems you are not explicitly authorized to test.

---

## 🔁 The detection loop

1. **Simulate** – run one technique from the table below.
2. **Observe** – confirm the event reached Wazuh (Sysmon / Windows Security /
   Linux auth logs).
3. **Detect** – add or tune a rule in
   [`../detections/rules/local_rules.xml`](../detections/rules/local_rules.xml).
4. **Validate** – re-run the technique and confirm the alert fires with the
   right level and ATT&CK mapping.
5. **Write up** – record the attack, the evidence, the rule, and the tuning.

---

## 🧰 Recommended tooling

* **[Atomic Red Team](https://github.com/redcanaryco/atomic-red-team)** –
  small, scripted tests for individual ATT&CK techniques (Windows/Linux).
* **[Invoke-AtomicRedTeam](https://github.com/redcanaryco/invoke-atomicredteam)** –
  PowerShell runner for Atomic tests.
* **Kali Linux VM** – for network-based techniques (brute force, Kerberoasting).
* Built-in OS tooling (`net`, `schtasks`, `reg`, PowerShell) for the simplest tests.

Install the Atomic runner on a Windows lab VM:

```powershell
Install-Module -Name invoke-atomicredteam,powershell-yaml -Scope CurrentUser -Force
Import-Module invoke-atomicredteam
Install-AtomicRedTeam -getAtomics
```

---

## 🎯 Simulation catalogue

Each scenario lists the ATT&CK technique, a safe way to simulate it, the
expected telemetry, and the Wazuh rule that should catch it.

### 1. Local admin account creation — Persistence / Priv-Esc

* **ATT&CK:** [T1136.001 – Create Account: Local Account](https://attack.mitre.org/techniques/T1136/001/)
* **Telemetry:** Windows Security EventID 4720 (user created), 4732 (added to
  admin group); Sysmon process create (EventID 1) for `net.exe`.
* **Detection:** rule `100100` / `100101`.
* **Cleanup:** `net user lab_backdoor /delete`

Simulate (Windows, lab only):

```powershell
net user lab_backdoor P@ssw0rd123! /add
net localgroup Administrators lab_backdoor /add
```

### 2. Malicious scheduled task — Persistence

* **ATT&CK:** [T1053.005 – Scheduled Task](https://attack.mitre.org/techniques/T1053/005/)
* **Telemetry:** Sysmon EventID 1 (`schtasks.exe`), Security EventID 4698
  (scheduled task created).
* **Detection:** rule `100102`.
* **Cleanup:** `schtasks /delete /tn "LabPersist" /f`

Simulate:

```powershell
schtasks /create /tn "LabPersist" /tr "C:\Windows\System32\calc.exe" /sc onlogon /ru System
```

### 3. Suspicious binary in a staging directory — Execution / Defense Evasion

* **ATT&CK:** [T1204 – User Execution](https://attack.mitre.org/techniques/T1204/),
  [T1036 – Masquerading](https://attack.mitre.org/techniques/T1036/)
* **Simulate:** copy a benign EXE (e.g. `calc.exe`) to
  `C:\Users\Public\svchost.exe` and run it.
* **Telemetry:** Sysmon EventID 1 with image path under `\Users\Public\` or
  `\Temp\`, and/or a name that masquerades as a system process.
* **Detection:** rule `100103`.

### 4. Credential dumping behaviour — Credential Access

* **ATT&CK:** [T1003.001 – LSASS Memory](https://attack.mitre.org/techniques/T1003/001/)
* **Simulate (safe):** Atomic test `T1003.001-1` uses `procdump` against lsass.
  Prefer the Atomic procdump test so nothing malicious runs.
* **Telemetry:** Sysmon EventID 10 (ProcessAccess) targeting `lsass.exe` with a
  suspicious `GrantedAccess` mask (e.g. `0x1010`, `0x1410`).
* **Detection:** rule `100104` (requires Sysmon ProcessAccess logging enabled).

### 5. Kerberoasting — Credential Access

* **ATT&CK:** [T1558.003 – Kerberoasting](https://attack.mitre.org/techniques/T1558/003/)
* **Telemetry:** Security EventID 4769 (Kerberos service ticket) with
  encryption type `0x17` (RC4) — a strong Kerberoasting signal.
* **Detection:** rule `100105`.

Simulate (from a domain-joined host, lab only):

```powershell
setspn -T cyberlab.local -Q */*
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "MSSQLSvc/db.cyberlab.local:1433"
```

### 6. SSH brute force against the Linux client — Credential Access

* **ATT&CK:** [T1110 – Brute Force](https://attack.mitre.org/techniques/T1110/)
* **Telemetry:** repeated `Failed password` entries in `/var/log/auth.log`
  forwarded by the Wazuh agent.
* **Detection:** Wazuh ships this out of the box (rule group
  `authentication_failures`, e.g. rule `5712`). Confirm it fires and tune the
  threshold if needed.

Simulate (from the Kali VM, lab only):

```bash
hydra -l testuser -P /usr/share/wordlists/rockyou.txt ssh://<linux-client-ip>
```

---

## 🗺️ ATT&CK coverage summary

| # | Technique | ATT&CK ID | Tactic | Wazuh rule |
|---|-----------|-----------|--------|------------|
| 1 | Create local admin account | T1136.001 / T1098 | Persistence / Priv-Esc | 100100, 100101 |
| 2 | Scheduled task | T1053.005 | Persistence | 100102 |
| 3 | Masquerading binary in staging dir | T1036 / T1204 | Defense Evasion / Execution | 100103 |
| 4 | LSASS access | T1003.001 | Credential Access | 100104 |
| 5 | Kerberoasting | T1558.003 | Credential Access | 100105 |
| 6 | SSH brute force | T1110 | Credential Access | 5712 (built-in) |

---

## 📓 Write-up template

Copy this for each detection you build (one Markdown file per technique makes a
great portfolio):

```markdown
## <Technique name> (ATT&CK <ID>)
**Hypothesis:** what the adversary does and why it is noisy.
**Simulation:** exact commands run (lab only).
**Evidence:** screenshot of the Wazuh alert + the raw event fields.
**Detection logic:** the rule and which fields it keys on.
**Tuning:** false positives observed and how the rule was narrowed.
**Response:** what a SOC analyst should do when this fires.
```
