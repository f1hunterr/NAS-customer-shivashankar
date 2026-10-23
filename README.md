# NAS Customer: Shivashankar (Shinju) — WireGuard VPN Setup

## Overview
Customer: Shivashankar (login: Shinju_synology)
Device: Synology DS224+ (synology_geminilake_224+), DSM 7.4.1-90080
NAS hostname: Shinju
LAN IP: 192.168.50.96

## Network Topology
```
ISP (Syrotech GPON ONT, 192.168.1.1, PPPoE/Router mode)
  -> Asus TUF Gaming AX4200 (WAN IP: 192.168.1.33)
    -> Synology DS224+ NAS (192.168.50.96)
```
Double-NAT setup — port forwarding required on BOTH the ISP ONT and the Asus router.

DDNS: `shinju.ddns.net` (no-ip.com, configured on Asus router, Active)
Public IP (at setup time): 117.221.142.224

## Hardware / Capacity
- CPU: Intel Celeron J4125 @ 2.0GHz, 4 cores
- RAM: 7.0GB total
- NIC: 1 Gbps
- Storage: /DATA pool 3.8TB (1.3TB used, 35%), disks sda(1.8TB)/sdc(931GB)/sdd(931GB)
- WAN speed (measured via Cloudflare speed test): ~179 Mbps download, ~134 Mbps upload
- Expected WireGuard throughput: capped by WAN (~130-180 Mbps), not CPU (~300-450 Mbps capacity)

## Software Stack Installed
1. **Container Manager** (Synology's Docker package) — installed via Package Center
2. **wg-easy** (maintained fork: `ghcr.io/wg-easy/wg-easy`) — WireGuard + Web UI in Docker
   - Web dashboard: `http://192.168.50.96:51821`
   - Auth: PASSWORD_HASH (bcrypt), not plaintext PASSWORD (required by current version)
   - WG_HOST: public IP / DDNS hostname
   - WG_MTU: 1280 (reduced from default 1420 to fix PPPoE + double-NAT fragmentation issue)
   - Config volume: `/volume1/docker/wg-easy` (persists across container recreates)
   - Ports: 51820/udp (WireGuard), 51821/tcp (web UI)

### Also installed (earlier attempt, superseded by Docker approach)
- `WireGuard-geminilake` community .spk package (Andreas Runfalk build, DSM7.2) — CLI-only, no web UI.
  Still present/running as wg0 interface at OS level but NOT the primary VPN path going forward;
  wg-easy's Docker container runs its own independent WireGuard instance.
  **Consider removing the manual wg0 CLI setup to avoid confusion/port conflicts** if wg-easy
  is the long-term solution (uses same UDP 51820 — only one can bind it at a time; wg-easy
  currently wins since manual wg0 was brought down before starting the container).
- WGDashboard (Python/Flask) — attempted, **failed**: this NAS's Python is 3.8.15, WGDashboard's
  current codebase requires Python 3.10+ (uses `X | None` type hint syntax). Not viable without
  a newer Python interpreter. Abandoned in favor of wg-easy (Docker-based, sidesteps Python version).

## Critical Issue Found & Fixed: sudo/PAM broken
**Root cause**: `/etc/pam.d/sudo` had `pam_syno_support.so` (tied to Synology's remote
SupportService/SupportForm package — likely installed during a prior remote-support session)
intercepting the auth chain and breaking sudo for all local accounts, including a reactivated
`admin`. Root SSH login is disabled by Synology default regardless of password (expected,
did not touch this).

**Fix applied** (via DSM Task Scheduler running scripts as `root` — bypasses the broken
sudo/PAM path entirely, since DSM's own job scheduler privilege escalation is separate from
SSH+sudo):
```
/etc/pam.d/sudo changed from:
  auth sufficient pam_syno_support.so sudo
  auth sufficient pam_rootok.so
  auth sufficient pam_unix.so
  auth sufficient pam_winbind.so use_first_pass
  auth sufficient pam_ldap.so
  account sufficient pam_unix.so
  account sufficient pam_winbind.so
  account sufficient pam_ldap.so
  session sufficient pam_unix.so
  session sufficient pam_winbind.so

to:
  #auth sufficient pam_syno_support.so sudo
  auth sufficient pam_rootok.so
  auth required pam_unix.so
  account required pam_unix.so
  session required pam_unix.so
```
Removed pam_winbind/pam_ldap entirely (this NAS is standalone, not AD/LDAP-joined — those
modules were failing hard on the "account" phase and blocking login even after successful
password auth, since "account" phase failures override a prior "auth" success).

Backup of original file saved to `/etc/pam.d/sudo.bak` on the NAS.

Also ran `synouser --setpw Shinju_synology 'Shriju@2022'` to resync the local Unix password
hash (SSH login uses DSM's own auth backend, not necessarily synced to the local shadow file
used by sudo/pam_unix — this was contributing to the mismatch).

## Port Forwarding Rules
### ISP ONT (Syrotech, 192.168.1.1, admin/Syro@432103)
Application → Virtual Server:
| Name | Protocol | External Port | Server IP |
|---|---|---|---|
| Wireguard | UDP | 51820 | 192.168.1.33 (Asus WAN IP — NOT the NAS LAN IP, critical distinction due to double-NAT) |

### Asus TUF Gaming AX4200 (192.168.50.1)
WAN → Virtual Server / Port Forwarding:
| Service Name | Protocol | External Port | Internal Port | Internal IP |
|---|---|---|---|---|
| wireguard | UDP | 51820 | 51820 | 192.168.50.96 |

## Known Issues / Follow-ups
1. **Full-tunnel (AllowedIPs=0.0.0.0/0) client configs caused "internet going offline"**
   — root-caused to MTU mismatch (PPPoE overhead + double NAT reduces effective path MTU
   below WireGuard's default 1420). Fixed by setting `WG_MTU=1280` on the container.
   **Existing client configs generated before this change must be deleted and regenerated**
   — MTU setting only applies to newly created peer configs, not retroactively.
2. Decide whether to decommission the manual CLI wg0 WireGuard package (`/etc/wireguard/wg0.conf`,
   Task Scheduler jobs "WG Start"/"WG Setup") since wg-easy's Docker instance is the one
   actually in use — leaving both around risks confusion on a future session.
3. Verify DDNS (`shinju.ddns.net`) stays in sync if public IP changes — WG_HOST is currently
   set to a static IP string; consider switching to the DDNS hostname for resilience.
4. wg-easy update banner seen once (referred to archived WeeJeWel/wg-easy repo) — already
   migrated to maintained fork `ghcr.io/wg-easy/wg-easy`, current and healthy as of last check.

## Credentials Note
Do NOT store real passwords in this repo. NAS admin/user passwords and wg-easy dashboard
password are known to Hemanth/customer directly — not documented here.
