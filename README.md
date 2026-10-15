# SIEM-Based SSH Brute-Force Detection

> ⚠️ **Disclaimer:** This project was conducted entirely within an isolated, self-owned lab environment for educational purposes as part of a formal cybersecurity training program (Bincom). No unauthorized, third-party, or production systems were targeted at any stage. All tools and techniques described here (Hydra, PassGAN, etc.) are documented for defensive/educational purposes only.

## Overview

This project simulates and detects an SSH brute-force attack end-to-end: standing up an isolated attacker/victim lab, deploying a SIEM, launching a real brute-force attack with Hydra, analyzing the resulting detections, and separately assessing the risk profile of an adversarial AI tool relevant to the same attack category.

**Objectives (as assigned):**
- Deploy Security Onion or ELK Stack
- Simulate a brute-force SSH attack using Hydra
- Detect and analyze the attack using a SIEM
- Perform an AI risk assessment on an adversarial ML tool

## Why Wazuh (rather than a literal Security Onion or ELK Stack install)

Wazuh was chosen as the SIEM platform for this project. This is not a deviation from the assignment but a direct architectural match:

- Wazuh was, for years, Security Onion's own host-based intrusion detection component (monitoring SSH logs and host activity), before being replaced by Elastic Agent in Security Onion's 2025 release.
- Security Onion's own documentation has historically listed Elasticsearch, Logstash, Kibana, and Wazuh together as core components of the platform, as they were never fully separate categories.
- Architecturally, Wazuh is built directly on the Elastic/OpenSearch lineage: its indexer is a fork of OpenSearch (itself forked from Elasticsearch), its dashboard is a fork of OpenSearch Dashboards (forked from Kibana), and it uses Filebeat, the same log-shipping component used in classic ELK deployments.
- Given a hardware-constrained lab (two 8GB machines), Wazuh's single-node "all-in-one" installer offered a significantly lighter footprint than a full Security Onion deployment (Suricata + Zeek + Elasticsearch), while still satisfying both named options in the assignment.

## Lab Architecture

```
 Kali (Attacker, PC1)                               PC2 (Windows Host)
 192.168.1.10                                       192.168.1.20
 ┌────────────────┐        Bridged LAN         ┌───────────────────────────────┐
 │   Kali Linux   │  ─────────────────────────▶  IP forwarding enabled        │
 │  Hydra, tcpdump│                            │  (registry: IPEnableRouter)   │
 └────────────────┘                            │                               │
                                               │  VMnet2 (internal, isolated)  │
                                               │  gateway 192.168.50.1         │
                                               │  ┌────────────────────────┐   │
                                               │  │  Victim + SIEM VM      │   │
                                               │  │  192.168.50.10         │   │
                                               │  │  Ubuntu Server 22.04   │   │
                                               │  │  sshd + Wazuh          │   │
                                               │  │  (all-in-one)          │   │
                                               │  └────────────────────────┘   │
                                               └───────────────────────────────┘
```

The attacker sits on a bridged LAN adapter, separate from the isolated internal network (VMnet2) hosting the combined victim/SIEM VM. PC2 (the physical host machine) was configured to route traffic between the two, simulating a segmented network on constrained two-machine hardware. Full routing configuration and troubleshooting is documented in [`01-lab-network-setup/routing-config.md`](01-lab-network-setup/routing-config.md).

## Repository Structure

```
├── README.md                                 you are here
├── 01-lab-network-setup/
│   └── routing-config.md                     Windows routing setup + troubleshooting log
├── 02-wazuh-deployment/
│   └── install-notes.md                      Wazuh install steps + issues encountered
├── 03-attack-simulation/
│   ├── hydra-commands.md                     wordlist derivation + attack commands/timeline
│   ├── hydra_output.txt                      Hydra terminal output
│   ├── custom-wordlist-bincom-lab.txt        
│   └── ssh_bruteforce_capture.pcap           Packet capture file
├── 04-detection-analysis/
│   ├── incident-analysis-report.md           Full incident report (Markdown version)
│   └── siem-dashboard-screenshot.png         
└── 05-ai-risk-assessment/
    └── ai-risk-assessment.md                 AI tool justification and risk assessment (Markdown version)
```

Word (`.docx`) versions of the two reports are also available on request/locally, while this repo carries the Markdown versions for readability directly on GitHub.

## Reproduction Summary

1. **Network setup**: bridged Kali + Windows-host routing into an isolated internal VM network. See [`01-lab-network-setup/routing-config.md`](01-lab-network-setup/routing-config.md).
2. **SIEM deployment**: Wazuh all-in-one install on a combined victim/SIEM Ubuntu VM. See [`02-wazuh-deployment/install-notes.md`](02-wazuh-deployment/install-notes.md).
3. **Attack simulation**: THC-Hydra brute-force against a deliberately weak SSH account, using a small custom wordlist derived from `rockyou.txt`. See [`03-attack-simulation/hydra-commands.md`](03-attack-simulation/hydra-commands.md).
4. **Detection and analysis**: Wazuh correlation rules (notably Rule 40112) identified the compromise pattern; a 4-panel dashboard was built to visualize it. See [`04-detection-analysis/incident-analysis-report.md`](04-detection-analysis/incident-analysis-report.md).
5. **AI risk assessment**: a parallel assessment of PassGAN, a GAN-based password-guessing tool, connecting its risk profile back to the detection controls built in this same project. See [`05-ai-risk-assessment/ai-risk-assessment.md`](05-ai-risk-assessment/ai-risk-assessment.md).

## Deliverables

| Deliverable | Location |
|---|---|
| SIEM dashboard screenshot | `04-detection-analysis/siem-dashboard-screenshot.png` |
| Incident analysis report | `04-detection-analysis/incident-analysis-report.md` |
| AI tool justification and risk assessment | `05-ai-risk-assessment/ai-risk-assessment.md` |

## Key Findings (Summary)

- Wazuh detected the full attack pattern, escalating from individual authentication failures to a single correlated compromise alert (**Rule 40112**, severity level 12: *"Multiple authentication failures followed by a success"*).
- Detection was effectively real-time, with the correlation alert firing at the same timestamp as the successful login itself.
- The target SSH daemon's own connection throttling (`MaxStartups`) incidentally disrupted the first, higher-concurrency Hydra attempt, a free, built-in control worth deliberately tuning rather than leaving at defaults.
- AI-driven password guessing (assessed via PassGAN) does not require new detection infrastructure: because it operates through the same authentication protocols as traditional brute-force attacks, the same SIEM correlation rules built in this project remain a valid defense.
