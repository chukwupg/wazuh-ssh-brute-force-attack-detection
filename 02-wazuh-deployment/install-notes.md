# Wazuh Deployment Notes

## Target Host

- Ubuntu Server 22.04 (minimal install)
- 2 vCPU, allocated 3–4GB RAM (below Wazuh's officially recommended 4 vCPU / 8GB, but workable for a single-agent training lab)
- Combined victim + SIEM role (SSH server and Wazuh all-in-one on the same VM, to conserve resources across two 8GB physical machines)

## Install Command

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a -i
```
- `-a`: all-in-one install (indexer + manager + dashboard on a single node)
- `-i`: ignore the hardware requirement check (needed given the VM's specs are below Wazuh's stated minimum)

## Issues Encountered

### 1. Disk space exhaustion mid-install

**Symptom:** dashboard package install failed with `dpkg: error ... failed to write ... No space left`.

**Root cause:** the Ubuntu guided installer had allocated only ~19GB of a 40GB virtual disk to the root logical volume, leaving the remainder unassigned inside the LVM volume group.

**Fix:**
```bash
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```
(Extending the logical volume does **not** automatically grow the filesystem: `resize2fs` is required as a separate step.)

### 2. dpkg lock contention

**Symptom:** dashboard install failed fast (~3 seconds) with:
```
E: Could not get lock /var/lib/dpkg/lock-frontend. It is held by process ... (apt-get)
```

**Root cause:** Ubuntu Server's background `unattended-upgrades` service (and its associated `apt-daily` timers) started a background `apt-get` run at the same moment the Wazuh installer tried to install the dashboard package, and both processes contended for the same dpkg lock.

**Fix:**
```bash
sudo systemctl stop unattended-upgrades
sudo systemctl disable unattended-upgrades
sudo systemctl stop apt-daily.timer apt-daily-upgrade.timer
sudo systemctl disable apt-daily.timer apt-daily-upgrade.timer
```

After both fixes, the install completed successfully on the third attempt.

## Post-Install

- Dashboard accessible at `https://192.168.50.10` (self-signed certificate: browser warning expected and safe to accept in this isolated lab context).
- Admin credentials are printed once at the end of the install script's summary output. **Do not commit these to the repository.** Recover them later if needed via:
  ```bash
  sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
  ```
- Since manager and agent are co-located on the same host (all-in-one), local log ingestion (including `/var/log/auth.log`) begins automatically with no separate agent-enrollment step required.
- Verified ingestion by performing one manual successful SSH login from Kali and confirming the event appeared in the **Threat Hunting** module within roughly a minute.

## Service Recovery After VM Restart

Wazuh's four services (`wazuh-indexer`, `wazuh-manager`, `filebeat`, `wazuh-dashboard`) are enabled to start automatically on boot. After powering the VM back on:

```bash
sudo systemctl status wazuh-indexer wazuh-manager filebeat wazuh-dashboard
```
If any are not running, start them **in this order** (indexer first, since the others depend on it):
```bash
sudo systemctl start wazuh-indexer
sudo systemctl start wazuh-manager
sudo systemctl start filebeat
sudo systemctl start wazuh-dashboard
```

**Note:** after a restart, the dashboard's filter dropdowns may not immediately show all field values that exist in the data (a cached-field-values quirk). If an expected value is missing from a filter dropdown, either type the value manually into the filter instead of selecting from the list, or refresh the index pattern's field list under Index Patterns management.
