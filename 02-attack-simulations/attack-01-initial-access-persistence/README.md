# Attack Path 01 — Reconnaissance, Brute Force and Persistence Baseline

## Purpose

This attack path was created to test how the lab reacts to a simple attacker workflow across Linux and Windows targets. The goal was to perform advanced exploitation, review what Wazuh detects by default, and identify where detection needs to be improved.

## Lab Systems Involved

| System | IP Address | Role |
|---|---|---|
| Kali Linux | `192.168.56.40` | Attacker VM |
| Wazuh Server | `192.168.56.10` | SIEM manager, indexer and dashboard |
| Linux Victim | `192.168.56.30` | SSH target with Wazuh agent |
| Windows Victim | `192.168.56.20` | RDP target with Wazuh agent |

## Techniques Covered

| Technique | Description | Target | Result |
|---|---|---|---|
| T1046 Network Service Scanning | Nmap reconnaissance to discover live hosts and exposed services | Lab network | Scan worked, but Wazuh did not detect it by default |
| T1110.001 SSH Brute Force | Hydra password guessing against SSH | Linux Victim | Detected by Wazuh |
| T1110.001 RDP Brute Force | Hydra password guessing against RDP | Windows Victim | Detected by Wazuh |
| T1078 Valid Accounts | Successful login after password guessing | Linux + Windows victims | Detected through successful authentication events |
| T1053.005 Scheduled Task | Scheduled task persistence simulation on Windows | Windows Victim | Task worked locally, but default Wazuh visibility was incomplete |


## Outcome

Attack 01 was useful for baseline tests. Wazuh detected the authentication activity of SSH and RDP and the brute-force behaviour well. The strongest detections came from failed succesfull login events on the Linux and Windows Victims.

The main visibility gap was around schedule task persistence. The scheduled task was created sucessfully on Windows, but Wazuh did not show any direct scheduled task creation event during the initial test. On the other hand, the nmap scan for the network reconnaissance attack was visible, but some improvements such as IDS/IPS stronger detection and prevention network must be added.

## Files in This Folder

## Files in This Folder

| File | Purpose |
|---|---|
| `attack-log.md` | Commands used, attack steps, and attacker-side results |
| `detection-review.md` | What Wazuh detected, what was missed, and alert evidence |
| `soc-triage.md` | Analyst-style triage notes and response recommendations |
| `mitre-mapping.md` | MITRE ATT&CK mapping and coverage notes |
| `evidence/` | Screenshots, raw logs, and supporting evidence |
