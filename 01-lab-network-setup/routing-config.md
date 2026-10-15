# Lab Network Setup: Routing Configuration

## Goal

Connect an attacker machine (Kali, on a bridged LAN) to a victim/SIEM VM sitting on an isolated internal network (VMnet2), by routing traffic through the Windows host machine (PC2) that hosts both networks.

## Topology

- **PC1 (Kali attacker):** `192.168.1.10`, bridged network adapter, same physical LAN as PC2.
- **PC2 (Windows host):** `192.168.1.20` on the bridged LAN; `192.168.50.1` on the internal VMnet2 (acting as gateway).
- **Victim/SIEM VM:** `192.168.50.10`, attached only to VMnet2 (isolated, host-only).

## Setup Steps

### 1. Enable IP routing on the Windows host

```
HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters
  IPEnableRouter (DWORD) = 1
```
**Requires a full reboot of PC2** to take effect - the flag is only read at boot.

### 2. Configure VMnet2 as host-only

Using VMware's Virtual Network Editor (`vmnetcfg.exe`, run as admin):
- VMnet2 - Host-only
- Subnet: `192.168.50.0/24`
- DHCP: **disabled** (static IPs assigned manually)

### 3. Static IP on the host's VMnet2 adapter

`VMware Network Adapter VMnet2` → `192.168.50.1 / 255.255.255.0` (this becomes the VM's gateway).

### 4. Windows Firewall rules

**Known issue:** Windows 11's Settings app does not expose a network-category toggle for host-only VMware adapters, even though it lists them. `Get-NetConnectionProfile` also does **not** return an entry for a purely host-only adapter with no internet-reachable gateway: it never receives a Network Location Awareness (NLA) profile at all. Attempting `Set-NetConnectionProfile -InterfaceAlias "VMware Network Adapter VMnet2"` fails with `No MSFT_NetConnectionProfile objects found`.

**Workaround:** scope firewall rules by `-Profile Any` and restrict them tightly by `-RemoteAddress` to the lab subnet instead of relying on the Private/Public profile mechanism:

```powershell
New-NetFirewallRule -DisplayName "Lab - Allow ICMP VMnet2" -Protocol ICMPv4 -IcmpType 8 `
  -RemoteAddress 192.168.50.0/24 -Direction Inbound -Action Allow -Profile Any

New-NetFirewallRule -DisplayName "Lab - Allow SSH VMnet2" -Protocol TCP -LocalPort 22 `
  -RemoteAddress 192.168.50.0/24 -Direction Inbound -Action Allow -Profile Any
```
Since these rules are scoped by `RemoteAddress` to the internal lab subnet only, `-Profile Any` here does not expose anything to the wider network.

### 5. Route on Kali (attacker)

```bash
sudo ip route add 192.168.50.0/24 via 192.168.1.20
```

### 6. Dual-NIC setup on the victim/SIEM VM

The VM needed two interfaces: one static IP on the isolated lab network, one on NAT for outbound internet access (required for `apt`/package installs). Netplan config (`/etc/netplan/00-installer-config.yaml`):

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.50.10/24
      routes:
        - to: 192.168.1.0/24
          via: 192.168.50.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
    ens37:
      dhcp4: yes
```

**Important:** the route under `ens33` is scoped specifically to `192.168.1.0/24` (Kali's subnet), **not** a blanket `default` route. An earlier version of this config set `ens33` as the default route, which silently broke internet access by sending all general traffic (including `apt`) toward the isolated gateway instead of the NAT interface. Linux prefers the route with the lowest metric; the NAT interface (`ens37`, DHCP-assigned metric 100) needed to remain the sole default route.

## Issues Encountered and Resolved

| Issue | Root Cause | Fix |
|---|---|---|
| `Set-NetConnectionProfile` fails: no NLA profile found | Host-only VMware adapters never receive an NLA profile | Scope firewall rules with `-Profile Any` + `-RemoteAddress` instead |
| VM has no internet access after adding a NAT NIC | New NIC wasn't declared in netplan | Added `ens37` block with `dhcp4: yes` |
| `apt update` fails despite NAT NIC having an IP | Blanket `default via 192.168.50.1` route on `ens33` was hijacking all outbound traffic | Replaced with a route scoped only to `192.168.1.0/24` |
| Kali cannot reach `192.168.50.10` | VM had no route back to Kali's subnet | Added scoped route: `to: 192.168.1.0/24 via: 192.168.50.1` |
| `ping` command not found on VM | Minimal Ubuntu Server image excludes `iputils-ping` | `sudo apt install -y iputils-ping` |

## Final Verification

```bash
# From Kali:
ping -c 3 192.168.50.10   # succeeded
ssh testuser@192.168.50.10 # succeeded
```
