# Home SOC Lab — Blue Team Detection

## Overview

This repository documents the design, build, and operation of a local SOC (Security Operations Centre) home lab, created as a practical environment for developing blue team skills. The lab is designed to simulate a small enterprise setup, with victim machines acting as targets, an attacker machine used to execute controlled techniques, and a SIEM platform collecting logs, generating alerts, and helping me practise basic incident analysis and reporting.

The project is structured both as a way for me to learn through hands-on deployment and as a reproducible guide for anyone who wants to try it.

---

## Context and Motivation

This lab is a direct extension of my dissertation project:

> *"Simulating Cyber-Attack Scenarios in a Cloud-Based Cyber Range and Mapping them to the MITRE ATT\&CK Framework"*

The dissertation focused on **attack simulation**, building cloud-native infrastructure (OpenStack, Kubernetes, Helm, OpenTofu) to provision cyber range environments and study attacker behaviour at a systems level.

This lab shifts perspective to the **defensive side**: when attacks occur, how can they be detected, investigate them, investigated, and documented from a SOC analyst’s point of view? The same MITRE ATT\&CK framework used to map attack scenarios in my dissertation is used here to map detection coverage, showing two differetn sides of the same reality.

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
│   │                  │  │  Debian 12   │  │   Attacker   │  │
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
| Linux Victim | Debian 12 | 2GB | 25GB | Wazuh agent, SSH target |
| Kali Attacker | Kali Linux (latest) | 4GB | 80GB | Attack simulation and validation using tools such as Hydra, Nmap, and Metasploit |

**Total resource usage: ~14GB RAM, ~215GB disk**

---

## Attack Scenarios To Be Covered

| Scenario | MITRE Technique | Target | Detection Method |
|---|---|---|---|
| RDP & SSH Brute Force | T1110 — Brute Force | Windows + Linux | Failed auth log analysis |
| Network Reconnaissance | T1046 — Network Service Scanning | All | Port scan pattern detection |
| Scheduled Task Persistence | T1053 — Scheduled Task/Job | Windows | Process monitoring, Windows event logs, and file integrity monitoring |

---

## Tools Used

| Tool | Purpose | Why This Tool |
|---|---|---|
| VMware Workstation Pro | Hypervisor | Free for personal use, strong snapshot support, and suitable for isolated lab environments |
| Wazuh | SIEM & XDR | Open source, no data cap, and native MITRE mapping |
| Kali Linux | Attack platform | Industry standard for penetration testing, pre-loaded toolset |
| Hydra | Brute force simulation | Well-documented tool for controlled authentication attack testing and T1110 validation |
| Nmap | Network reconnaissance | Standard network scanning tool used to simulate and detect T1046-style activity |
| Metasploit | Attack validation framework | Useful for controlled exploitation and post-exploitation testing in isolated environments |
| Atomic Red Team | Attack simulation library | MITRE-mapped test cases for validating detection coverage in a controlled ways (https://github.com/redcanaryco/atomic-red-team) |
| MITRE ATT\&CK Navigator | Detection coverage visualisation | Useful for mapping, communicating, and reviewing detection coverage (https://attack.mitre.org/) |
| Sigma | Detection rule format | SIEM-agnostic community standard for sharing detection logic |

---

## How to Use This Repository

- **Building the lab from scratch**: Start at [`01-lab-setup/architecture.md`](01-lab-setup/architecture.md) and follow the sections in order.
- **Understanding the attack scenarios**: Each folder in `02-attack-simulations/` is self-contained with the attack methodology, commands used, and resulting logs.
- **Reviewing detection logic**: [`03-detection-rules/`](03-detection-rules/) contains all custom Wazuh rules with explanations of why each rule is written the way it is.
- **Seeing SOC analyst workflow**: [`04-soc-analyst-simulation/`](04-soc-analyst-simulation/) contains formal incident reports and alert triage documentation as a practitioner would produce them.
- **Checking MITRE coverage**: [`05-mitre-mapping/`](05-mitre-mapping/) shows which techniques are detected and the confidence level of each detection.

---

## Status

| Phase | Status |
|---|---|
| Lab design and documentation | Complete |
| VM setup and networking | Complete |
| Wazuh deployment | Complete |
| T1110 — Brute Force scenario | Complete |
| T1046 — Network Scan scenario | Complete |
| T1053 — Scheduled Task scenario | Complete |
| SOC analyst simulation | Pending |
| MITRE ATT\&CK Navigator layer | Pending |
