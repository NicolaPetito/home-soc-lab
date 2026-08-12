# Home SOC Lab — Attack Simulation & Detection Engineering

## Overview

This repository documents the design, build, and operation of a local SOC (Security Operations Centre) home lab, created as a practical environment for developing blue team skills. The lab is designed to simulate a small enterprise setup, with victim machines acting as targets, an attacker machine used to execute controlled techniques, and a SIEM platform collecting logs, generating alerts, and helping me practise basic incident analysis and reporting.

The project is structured both as a way for me to learn through hands-on deployment and as a reproducible guide for anyone who wants to try it.

---

## Context and Motivation

This lab is a direct extension of my dissertation project:

> *"Simulating Cyber-Attack Scenarios in a Cloud-Based Cyber Range and Mapping them to the MITRE ATT\&CK Framework"*

The dissertation focused on **attack simulation**, building cloud-native infrastructure with OpenStack, Kubernetes, Helm, and OpenTofu to provision cyber range environments and study attacker behaviour at a systems level. It also explored cyber ranges as an alternative environment for education, training and research.

This lab shifts the perspective into the **defensive side**. Instead of only asking how an attack can be simulated, this lab asks how the activity can be detected, investigated, documented, and improved from a SOC analyst and detection engineering point of view. The same MITRE ATT\&CK framework used in my dissertation is used here to map detection coverage and visibility gaps. This helps me connect both sides of the same reality: how attacks work, and how defenders can observe and respond to them.

---

## Lab Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    HOST MACHINE                             │
│              Ryzen 7700X │ 64GB RAM │ Windows               │
│                                                             │
│   ┌──────────────────┐  Host-Only Network: 192.168.56.0/24  │
│   │ VMwareWorkstation│                                      │
│   │                  │  ┌──────────────┐  ┌──────────────┐  │
│   │                  │  │ Wazuh Server │  │ Windows 10   │  │
│   │                  │  │ Ubuntu 22.04 │  │   Victim     │  │
│   │                  │  │192.168.56.10 │  │192.168.56.20 │  │
│   │                  │  └──────┬───────┘  └──────┬───────┘  │
│   │                  │         │ (agent logs)    │          │
│   │                  │  ┌──────┴───────┐  ┌──────┴───────┐  │
│   │                  │  │ Linux Victim │  │  Kali Linux  │  │
│   │                  │  │  Debian 13   │  │   Attacker   │  │
│   │                  │  │192.168.56.30 │  │192.168.56.40 │  │
│   │                  │  └──────────────┘  └──────────────┘  │
│   └──────────────────┘                                      │
└─────────────────────────────────────────────────────────────┘
```

Internal traffic (attacks, agent communications, SIEM logs) flows over the **Host-Only internal network (VMnet1)**. Each VM has a separate NAT adapter (VMnet8) for internet access during setup and updates.

---

## VM Stack

| VM | OS | RAM | Disk | Role |
|---|---|---|------|---|
| Wazuh Server | Ubuntu 22.04 LTS | 4GB | 50GB | SIEM manager, indexer, dashboard |
| Windows Victim | Windows 10 Pro | 4GB | 60GB | Wazuh agent, RDP target |
| Linux Victim | Debian 13 | 2GB | 25GB | Wazuh agent, SSH target |
| Kali Attacker | Kali Linux (VMware image) | 4GB | 80GB | Attack simulation and validation using tools such as Hydra, Nmap, and Metasploit |

**Total resource usage: ~14GB RAM, ~215GB disk**

---

## Current Attack Path

This first scenario is designed to test how the lab reacts to common attacker behaviour across targets.

| Attack Path | Techniques Covered | Target | Current Result |
|---|---|---|---|
| Attack Path 01 - Reconnaissance, Brute Force and Persistence Baseline | T1046, T1110/T1110.001, T1078, T1053.005 | Windows + Linux victims | Baseline testing complete; alert review and detection improvements in progress |

### Techniques covered in Attack Path 01

| Technique | Description | Target | Detection Notes |
|---|---|---|---|
| T1046 - Network Service Scanning | Nmap reconnaissance against the lab network | Windows, Linux, Wazuh Server | Useful attacker-side evidence; limited default Wazuh visibility without network IDS/IPS |
| T1110 / T1110.001 - Brute Force / Password Guessing | SSH and RDP brute-force testing using controlled lab credentials | Linux + Windows victims | Wazuh detected authentication failures and successful logons |
| T1078 - Valid Accounts | Successful login after password guessing | Linux + Windows victims | Wazuh showed valid login activity after failed attempts |
| T1053.005 - Scheduled Task | Windows scheduled task persistence simulation | Windows victim | Task creation worked locally; default Wazuh visibility was incomplete and needs improvement |

---

## Tools Used

This lab uses different categories of tools. Some are part of the current setup, while others are planned for future attack paths, detection engineering or lab expansions.

### Core Lab Infrastructure

| Tool | Purpose | Why This? |
|---|---|---|
| VMware Workstation Pro | Hypervisor | Free for personal use, strong snapshot support, and suitable for isolated lab environments |
| Wazuh | SIEM & XDR | Open source, no data cap, and native MITRE mapping |
| Kali Linux | Attack platform | Industry standard for penetration testing, pre-loaded toolset |
| Windows 10 Pro | Windows victim endpoint | Used to generate Windows telemetry |
| Debian Linux | Linux victim endpoint | Used to generate Linux telemetry |

### Attack Simulation Tools

| Tool | Purpose | Status |
|---|---|---|
| Nmap | Network reconnaissance and service scanning | Used in Attack Path 01 |
| Hydra | Controlled SSH and RDP brute-force simulation | Used in Attack Path 01 |
| Atomic Red Team | MITRE-mapped attack simulation tests | Introduced for T1053.005 testing |
| Metasploit | Controlled exploitation and post-exploitation testing | Planned |

### Detection and Mapping

| Tool | Purpose | Why This? |
|---|---|---|
| Wazuh rules | Detection logic and alerting | Useful to create custom rules that would generate better response and detection to attacks |
| MITRE ATT\&CK Navigator | Detection coverage visualisation | Useful for mapping, communicating, and reviewing detection coverage |
| Sigma | Detection rule format | SIEM-agnostic community standard for sharing detection logic |
| Sysmon | Enhanced Windows telemetry and event logs |  Planned for more detailed Windows process, network, and system activity visibility |

**References:**
- Wazuh custom rules: https://documentation.wazuh.com/current/user-manual/ruleset/rules/custom.html
- MITRE ATT&CK Navigator: https://attack.mitre.org/

### Future Expansion Areas

| Area | Purpose |
|---|---|
| Firewall | Network segmentation and traffic control |
| IDS/IPS | Network-based detection and prevention |
| SOAR | Automated response and playbook testing |
| EDR/XDR | Endpoint detection and response comparison |

---

## How to Use This Repository

- **Building the lab from scratch**: Start with [`01-lab-setup/architecture.md`](01-lab-setup/architecture.md), then follow the setup documentation for networking, VM configuration, Wazuh installation, and agent deployment.
- **Understanding the current attack path**: [`02-attack-simulations/`](02-attack-simulations/) contains the current attack path documentation, including the techniques tested, commands used, evidence collected, and detection results.
- **Reviewing detection logic**: [`03-detection-rules/`](03-detection-rules/) contains all custom Wazuh rules with explanations of why each rule is written the way it is.
- **Following the SOC analyst workflow**: [`04-soc-analyst-simulation/`](04-soc-analyst-simulation/) is used for alert triage notes, incident-style analysis, and reporting based on the evidence collected.
- **Checking MITRE coverage**: [`05-mitre-mapping/`](05-mitre-mapping/) is used to map tested techniques, detection coverage, and visibility gaps to the MITRE ATT&CK framework.

---

## Status

| Phase | Status |
|---|---|
| Lab design and documentation | Complete |
| VM setup and networking | Complete |
| Wazuh deployment | Complete |
| Attack Path 01 - Reconnaissance, brute force and persistence baseline | Complete |
| Attack Path 01 - Alert and evidence collection | In progress |
| Attack Path 01 - Custom detection engineering | Pending |
| Attack Path 01 - SOC-style incident response write up | Pending |
| MITRE ATT\&CK Navigator layer | Pending |
| Future attack paths and lab expansion | Pending |
