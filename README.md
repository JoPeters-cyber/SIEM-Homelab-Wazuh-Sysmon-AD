# 🛡️ SIEM Homelab Wazuh · Sysmon · Active Directory

![Network Topology](networktopology.png)

This repository documents my **SIEM homelab**: a hands-on environment combining **Wazuh** for centralized detection, **Sysmon** for detailed Windows telemetry, and an **Active Directory** domain to emulate a realistic enterprise network.

The goal of this project is to build a **repeatable learning platform** for detection engineering, threat hunting, and incident response practice.

📖 Full walkthrough, screenshots, and step-by-step guide:
[Medium Article →](https://medium.com/@cyberxaman/building-a-siem-homelab-with-wazuh-sysmon-and-active-directory-ff90ff37fbcb)

---

## 🏗️ Project Contents

This repo captures **architecture, configuration snippets, and example artifacts**. It’s a showcase and launchpad — not a turnkey installer. The homelab includes:

* **Wazuh Manager** (Elasticsearch + Kibana + Wazuh components) collecting logs and surfacing alerts
* **Windows endpoints** instrumented with Sysmon for process, network, and persistence telemetry
* **Active Directory Domain Controller** providing centralized identity, DNS, and DHCP services
* **Linux client** joined to the domain to demonstrate cross-platform monitoring
* Sample detections and event mappings to **MITRE ATT&CK**

---

## ⚡ Quick Start (Overview)

1. Create VMs for each role on a **private lab network**.

   * Recommended resources: ≥16 GB RAM, ~200 GB disk total
2. Install **Wazuh** on Ubuntu VM using the official script.
3. Install **Active Directory Domain Services** on Windows Server and promote to DC.
4. Join Windows/Linux clients to the domain.
5. Install **Sysmon** on Windows with a hardened config.
6. Register Wazuh agents on all endpoints so logs flow into the SIEM.

**Example commands:**

```bash
# Ubuntu: update and install Wazuh
sudo apt update && sudo apt upgrade -y
curl -sO https://packages.wazuh.com/4.8/wazuh-install.sh
sudo bash ./wazuh-install.sh -a

# Ubuntu client: join AD domain
sudo apt install realmd sssd adcli krb5-user -y
sudo realm join cyberlab.local -U Administrator
```

```powershell
# Windows: install Sysmon with config
.\sysmon64.exe -i sysmonconfig.xml
```

---

## 🔍 Detection & Validation

Simulated adversary activity in the isolated lab produced telemetry and alerts, including:

* Execution of unsigned executables in critical directories
* Creation of unauthorized admin accounts
* Unapproved scheduled tasks

These events mapped to MITRE ATT&CK tactics:

* **Execution** (TA0002)
* **Persistence** (TA0003)
* **Privilege Escalation** (TA0004)
* **Defense Evasion** (TA0005)

Full narrative and examples: [Medium Article →](https://medium.com/@cyberxaman/building-a-siem-homelab-with-wazuh-sysmon-and-active-directory-ff90ff37fbcb)

---

## 🎓 Student Attendance Lab Extension

The Active Directory and Wazuh telemetry can also support an educational attendance dashboard based on interactive student sign-ins and sign-outs.

[Student Attendance Tracking Guide →](docs/attendance-tracking.md)

The guide covers Windows auditing, Wazuh event collection, saved searches, dashboard panels, late and absent status calculation, validation, and privacy safeguards.

---

## 🛡️ Safety & Ethics

* All simulations were run in an **isolated lab environment**
* Snapshots were taken before risky experiments
* No artifacts escaped the lab
* Always follow **ethical and legal guidelines**

---

## 📸 Screenshots / Artifacts

![Wazuh Dashboard](wazuhdashboard.png)

---

## 📚 References & Further Learning

* Grant Collins — *Build a Cybersecurity Homelab*
  [ProjectSecurity Course →](https://projectsecurity.teachable.com/p/build-a-cybersecurity-homelab-a-practical-guide-to-offense-defense-enterprise-101)

* Detailed Medium walkthrough:
  [https://medium.com/@cyberxaman/building-a-siem-homelab-with-wazuh-sysmon-and-active-directory-ff90ff37fbcb](https://medium.com/@cyberxaman/building-a-siem-homelab-with-wazuh-sysmon-and-active-directory-ff90ff37fbcb)

---

## 👤 Credits

* **Author:** Aman Parihar (@cyberxaman)
* **Course Reference:** Grant Collins — *Build a Cybersecurity Homelab*

---

## 📄 License

Educational and demonstrative purposes only.
Do **not** use examples here for illegal activities. Credit the author and follow applicable laws & ethics.
