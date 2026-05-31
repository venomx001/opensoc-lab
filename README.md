# 🛡️ OpenSOC Lab

> **Open-Source Security Operations & Detection Engineering Home Lab**
> Built by a fresher SOC analyst · 100% free tools · Real enterprise-grade architecture

![Wazuh](https://img.shields.io/badge/SIEM-Wazuh_4.7.5-blue)
![Windows](https://img.shields.io/badge/Endpoint-Windows_11_LTSC-0078D6?logo=windows)
![Sysmon](https://img.shields.io/badge/Telemetry-Sysmon-red)
![MITRE](https://img.shields.io/badge/Framework-MITRE_ATT%26CK-orange)
![License](https://img.shields.io/badge/License-MIT-green)

---

## 📌 What Is This Project?

OpenSOC Lab is a fully functional, production-inspired Security Operations Center built entirely on a personal laptop using free and open-source tools. It demonstrates real SOC analyst skills including log ingestion, detection engineering, alert triage, and MITRE ATT&CK mapping.

This project was built from scratch — including troubleshooting 13 real-world errors that occur in actual enterprise deployments.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Arch Linux Host                     │
│                                                      │
│  ┌─────────────────┐    ┌────────────────────────┐  │
│  │   wazuh-siem    │    │      win-target         │  │
│  │  Ubuntu 22.04   │◄───│  Windows 11 LTSC 2021  │  │
│  │  Wazuh 4.7.5    │    │  Sysmon + Wazuh Agent  │  │
│  │  3GB RAM        │    │  2GB RAM               │  │
│  └─────────────────┘    └────────────────────────┘  │
│                                                      │
│  Network: soclab NAT (192.168.100.0/24)             │
│  Host access: vboxnet0 (192.168.56.0/24)            │
└─────────────────────────────────────────────────────┘
```

---

## 🛠️ Tools & Technologies

| Tool | Version | Role | License |
|------|---------|------|---------|
| Wazuh | 4.7.5 | SIEM, EDR, Log Analysis, Dashboard | GPL v2 — Free |
| Sysmon | Latest | Deep Windows endpoint telemetry | Free (Microsoft) |
| VirtualBox | 7.x | Hypervisor | GPL v2 — Free |
| SwiftOnSecurity Config | Latest | Sysmon detection ruleset | MIT — Free |
| Ubuntu Server | 22.04 LTS | SIEM host operating system | Free |
| Windows 11 LTSC | 2021 Eval | Target endpoint VM | Free evaluation |

**Total cost: $0**

---

## 📋 What This Lab Detects

| Detection | Event Source | Event ID | MITRE Technique |
|-----------|-------------|----------|-----------------|
| Failed login attempts | Windows Security | 4625 | T1110 — Brute Force |
| Successful logon | Windows Security | 4624 | T1078 — Valid Accounts |
| New user account created | Windows Security | 4720 | T1136 — Create Account |
| User added to admin group | Windows Security | 4732 | T1078.003 |
| Scheduled task created | Windows Security | 4698 | T1053 — Scheduled Task |
| Privilege escalation | Windows Security | 4672 | T1078.003 |
| Process execution | Sysmon | 1 | T1059 — Scripting |
| Encoded PowerShell | Sysmon | 1 | T1059.001 |
| Network connections | Sysmon | 3 | T1071 — App Layer Protocol |
| DNS queries | Sysmon | 22 | T1071.004 |
| File creation | Sysmon | 11 | T1105 — Ingress Tool Transfer |

---

## 🚀 Log Flow Pipeline

```
Windows Security Events
        +
Sysmon Deep Telemetry
        │
        ▼
Wazuh Agent (endpoint)
        │
        ▼ (encrypted, port 1514)
Wazuh Manager (SIEM server)
        │
        ▼
Wazuh Indexer (OpenSearch)
        │
        ▼
Wazuh Dashboard (web browser)
        │
        ▼
Real-time Alerts + MITRE ATT&CK Mapping
```

---

## ⚡ Quick Start

### Prerequisites
- Linux host (guide written for Arch — adaptable to any distro)
- VirtualBox installed
- 8 GB RAM minimum
- 100 GB free storage

### Step-by-Step Guides
| Step | Guide |
|------|-------|
| 1 | [VirtualBox Setup](docs/phase1/01-virtualbox-setup.md) |
| 2 | [Isolated Lab Network](docs/phase1/02-network-setup.md) |
| 3 | [Wazuh SIEM VM](docs/phase1/03-wazuh-siem-setup.md) |
| 4 | [Windows Target VM](docs/phase1/04-windows-target-setup.md) |
| 5 | [Sysmon Installation](docs/phase1/05-sysmon-setup.md) |
| 6 | [Wazuh Agent Setup](docs/phase1/06-wazuh-agent-setup.md) |
| 7 | [First Alerts](docs/phase1/07-verification-and-first-alerts.md) |

---

## 🔍 Wazuh Search Queries (Cheat Sheet)

```
# All failed login attempts (brute force detection)
data.win.system.eventID: 4625

# New user accounts created
data.win.system.eventID: 4720

# All Sysmon process creation events
data.win.system.eventID: 1

# Encoded PowerShell (T1059.001)
data.win.eventdata.commandLine: *EncodedCommand*

# All Sysmon network connections
data.win.system.eventID: 3

# All DNS queries via Sysmon
data.win.system.eventID: 22

# Filter by MITRE technique
rule.mitre.technique: T1110
```

---

## 📁 Repository Structure

```
opensoc-lab/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── docs/
│   ├── phase1/          # Complete Phase 1 setup guides
│   ├── phase2/          # Attack simulation (coming)
│   └── phase3/          # Advanced detection (coming)
├── configs/
│   ├── sysmon/          # Sysmon XML config
│   └── wazuh-rules/     # Custom detection rules
├── detection-rules/
│   ├── windows/         # Sigma format rules
│   └── linux/           # Linux detection rules
├── playbooks/
│   ├── alert-triage-workflow.md
│   └── incident-response-template.md
└── screenshots/         # Lab screenshots
```

---

## 📖 Build Journal

This project involved solving **13 real-world errors** during the build process.
Every error, root cause, and solution is documented in:
[📋 Complete Build Journal](docs/build-journal.md)

This documentation style mirrors professional SOC runbooks.

---

## 🗺️ Roadmap

### Phase 1 ✅ (Complete)
- Wazuh SIEM + Windows endpoint + Sysmon
- Real-time log ingestion and alerting
- MITRE ATT&CK mapping

### Phase 2 🔧 (In Progress)
- Kali Linux attacker VM
- Atomic Red Team attack simulation
- Sigma rules integration
- TheHive case management

### Phase 3 📋 (Planned)
- Active Directory attack scenarios
- Threat hunting exercises
- Custom detection rule library
- Full incident response simulations

---

## 👤 Author

**[Your Name]**
Fresher SOC Analyst | Blue Team Enthusiast
[LinkedIn Profile] | [GitHub Profile]

> ⭐ Found this useful? Star the repo to help others find it!

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.
