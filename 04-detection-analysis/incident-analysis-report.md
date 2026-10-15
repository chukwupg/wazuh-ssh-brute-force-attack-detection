# Incident Analysis Report

### SSH Brute-Force Attack: Detection and Analysis via Wazuh SIEM

**Bincom Training Program: Assignment 4**
Prepared by: [Your Name]
Date: September 21, 2026

---

## 1. Executive Summary

This report documents a controlled SSH brute-force attack simulation conducted against an isolated lab environment, and the detection and analysis of that attack using Wazuh, a SIEM platform built on the Elastic/OpenSearch stack. The attack was executed using THC-Hydra from a Kali Linux attacker host against a deliberately weak SSH account on an Ubuntu Server target. Over an approximately six-and-a-half-minute window, the target's SIEM logged 23 relevant security events, escalating from individual authentication failures to a correlated "multiple authentication failures followed by a success" alert (Rule 40112, severity level 12), confirming that the SIEM correctly identified the compromise as a single connected event rather than isolated log lines.

This exercise validates that a lightweight, properly configured SIEM deployment can reliably detect brute-force SSH attacks in near real time, even on modest hardware, and demonstrates the practical value of rule correlation over simple log collection.

## 2. Lab Environment and Architecture

All testing was performed entirely within an isolated lab environment across two physical machines, with the attacker segmented onto a separate network path from the victim/SIEM host. No production systems, third-party services, or unauthorized targets were involved at any stage.

| Field | Value |
|---|---|
| Attacker host | Kali Linux, 192.168.1.10 (bridged LAN, PC1) |
| Target / victim host | Ubuntu Server 22.04, agent name "target-pc", 192.168.50.10 (isolated VMnet2) |
| SIEM platform | Wazuh 4.14.7 (indexer, manager, Filebeat, dashboard), co-located on the target host |
| Attack tool | THC-Hydra v9.7 |
| Target service | SSH (TCP/22), password authentication enabled |
| Target account | testuser (deliberately weak password, planted in a custom wordlist) |
| Attack window (analyzed) | 2026-09-21 22:10:34.736 – 22:16:59.741 (approx. 6 min 25 sec) |
| Compromise timestamp | 2026-09-21 22:16:57.735 (Rule 40112) |

Network path: the attacker (Kali, 192.168.1.10) reached the target (192.168.50.10) via a bridged LAN connection to the host machine, which routed traffic into an isolated internal network (VMnet2) hosting the combined victim/SIEM virtual machine. This topology was chosen deliberately to mirror a segmented network while remaining operable on two 8GB machines.

## 3. Attack Methodology

### 3.1 Tooling and Wordlist Preparation

A custom wordlist (`custom-wordlist-bincom-lab.txt`, 301 entries) was derived from the publicly available `rockyou.txt` combining the first 150 entries, the specific entry known to match the target account's password, and the final 150 entries, placing the correct credential roughly in the middle of the list. This produced a realistic "storm of failures, then a success" pattern rather than an instant or exhaustive crack, while keeping the derived file clearly distinct from the original file.

### 3.2 Attack Execution Timeline

- **Initial attempt:** Hydra's default concurrency (16 parallel connections) triggered the target SSH daemon's own connection-throttling protections, causing Hydra to abort with "all children were disabled due too many connection errors" before any credential was tested to completion.
- **Corrected attempt:** Hydra was re-run with concurrency capped at 4 parallel connections (`-t 4`), which completed successfully without triggering SSH-level connection throttling.
- **Command used:** `hydra -l testuser -P custom-wordlist-bincom-lab.txt -t 4 ssh://192.168.50.10`
- **Result:** the correct credential was identified and a successful authenticated session was established, approximately 6 minutes 23 seconds after the first detected failed-login attempt in the analyzed window.

A packet capture (`ssh_bruteforce_capture.pcap`), scoped to traffic between the attacker and the target on TCP/22, and the full Hydra terminal output (`hydra_output.txt`) were retained as supporting evidence alongside the SIEM's own record of the incident.

### Evidence

**Threat Hunting Dashboard Unfiltered**
![all alerts](/screenshots/dashboard-filtered-to-attack-timeline.png)

**Threat Hunting Dashboard Filtered to sshd**

![Threat hunting dashboard](/screenshots/threat-hunting-dashboard.png)

## 4. Detection and SIEM Analysis

### 4.1 Detection Rules Triggered

Filtering Wazuh's alert index to the attack window (22:10:30.000 – 22:17:05.000) and to `rule.groups` containing `attacks`, `authentication_success`, or `authentication_failures` returned 23 relevant events, all originating from source IP `192.168.1.10` (the attacker) against agent `target-pc`. The breakdown by rule:

| Rule ID | Rule Description | Level | Occurrences |
|---|---|---|---|
| 5763 | sshd: brute force trying to get access to the system. Authentication failed. | 10 | 4 |
| 5551 | PAM: Multiple failed logins in a small period of time. | 10 | 5 |
| 40111 | Multiple authentication failures. | 10 | 11 |
| **40112** | **Multiple authentication failures followed by a success.** | **12** | **1** |
| 5501 | PAM: Login session opened. | 3 | 2 |

>Rule 40112 is the single most significant finding in this dataset: it is a correlation rule that specifically recognizes a pattern of repeated authentication failures immediately followed by a successful login: the defining signature of a successful brute-force compromise, as distinct from an isolated failed or successful login. Its elevated severity level (12, versus level 10 for the underlying individual brute-force rules) reflects that Wazuh's rule engine treats this correlated pattern as materially more serious than any single failed attempt.

### Visual Evidence

**Events filtered by attack timeline**

![filtered events](/screenshots/events-filtered-to-attack-timeline.png)

**23 Alerts filtered by detection rule**

![alerts filtered by detection rules](/screenshots/detection-rules-triggered.png)

**Compromise: repeated authentication failures followed by a successful login**

![Compromise](/screenshots/attacker-successful-auth.png)

### 4.2 Escalation Timeline

| Timestamp | Rule | Description | Lvl |
|---|---|---|---|
| 22:10:34.736 | 5551 | First PAM warning: multiple failed logins in a short window | 10 |
| 22:10:37.005 | 5763 | Wazuh's dedicated SSH brute-force rule fires | 10 |
| 22:11:01 – 22:11:23 | 40111 | Repeated "multiple authentication failures" correlations (3 events) | 10 |
| 22:14:37 – 22:16:41 | 5763 / 40111 / 5551 | Sustained brute-force pattern continues across the reduced-concurrency Hydra run | 10 |
| 22:16:57.735 | 40111 | Final pre-compromise failure correlation | 10 |
| **22:16:57.735** | **40112** | **COMPROMISE DETECTED: multiple failures followed by a success** | **12** |
| 22:16:57.735 / .738 | 5501 | PAM session opened for testuser (x2) | 3 |
| 22:16:59.741 | 5760 | One trailing failed attempt (residual parallel Hydra thread) | 5 |

> Note the trailing event at 22:16:59.741 (Rule 5760, a single discrete authentication failure) occurring after the compromise timestamp: this reflects one of Hydra's four parallel connection threads completing its remaining queued attempts after the correct credential had already been found by another thread, rather than a separate or renewed attack.

### 4.3 Dashboard Visualizations

A four-panel Wazuh Custom Dashboard was built to visualize this incident, covering: (1) event volume over time across the attack window, showing a concentrated spike of activity; (2) a breakdown of triggered rules by ID and frequency; (3) an isolated metric highlighting the single Rule 40112 compromise-detection event; and (4) the source IP address (`data.srcip`) responsible for all 23 events, confirming a single attacking host. 

**Visual evidence**

![siem dashboard](/screenshots/siem-dashboard.png)

## 5. Findings

- The SIEM successfully detected both the brute-force attempt pattern and the eventual compromise, correlating them into a single high-severity alert (Rule 40112) rather than requiring manual correlation of individual log lines.
- Detection was near-real-time: the compromise-correlation alert fired at the same timestamp as the successful login itself (22:16:57.735), with no meaningful lag.
- The target SSH daemon's own built-in connection throttling (triggered during the first, higher-concurrency Hydra attempt) acted as an incidental first line of defense, independent of the SIEM, and is worth noting as a contributing control.
- All malicious traffic originated from a single, clearly identifiable source IP (192.168.1.10), with no attempt at obfuscation, IP rotation, or evasion, consistent with an unsophisticated, high-volume brute-force approach rather than a targeted or stealthy attack.
- The weak test password was locatable within a relatively small (301-entry) custom wordlist, underscoring how quickly weak credentials can be compromised even without a large-scale dictionary attack.

## 6. Recommendations

- Enforce account lockout or progressive delay (e.g., via `fail2ban` or PAM `tally2`/`faillock`) after a small number of consecutive SSH authentication failures, to reduce the window in which a brute-force attack can succeed.
- Disable SSH password authentication in favor of key-based authentication wherever operationally feasible, removing the brute-force attack surface entirely.
- Tune SSH's `MaxStartups` and related throttling parameters deliberately (rather than relying on their defaults), since this exercise showed they can meaningfully disrupt naive high-concurrency brute-force attempts.
- Configure active response in Wazuh (e.g., automatic firewall blocking) tied to Rule 40111/40112 triggers, to move from detection to automated containment for future incidents.
- Enforce a minimum password complexity/length policy and screen new passwords against known-breached-password lists, since the compromised credential in this exercise was trivially guessable.

## 7. Conclusion

This exercise successfully demonstrated the full lifecycle of a brute-force SSH attack: from planning and execution through SIEM-based detection and analysis, within a self-contained, isolated lab environment. Wazuh's rule correlation engine proved capable of distinguishing a genuine compromise from routine failed-login noise, providing clear, actionable, near-real-time evidence of the incident. The findings and recommendations above reflect practical, low-cost hardening measures that would have meaningfully raised the difficulty of this specific attack path.
