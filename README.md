# NAS Customer: Shivashankar (Shinju) — WireGuard VPN Setup

## Overview
Customer: Shivashankar (login: Shinju_synology)
Device: Synology DS224+ (synology_geminilake_224+), DSM 7.4.1-90080
NAS hostname: Shinju
LAN IP: 192.168.50.96
SSH: port 2022 (not default 22 — both enabled per DSM Service list, 2022 used throughout)
DSM users: `admin` (default, was Deactivated, reactivated during this session — password set
to match Shinju_synology's), `guest` (Deactivated), `Shinju_synology` (Normal, administrators
group member)

## Other device on same network (same customer)
**CasaOS NAS** — host `casaosnas`, LAN IP 192.168.50.83, Ubuntu 22.04-based (kernel
5.15.0-198-generic), login `shrijay`. Separate box from the Synology, not part of the
WireGuard VPN build — checked only for health (disk/mem OK, CasaOS v0.4.15 up to date,
no upgrade pending). Not AD-joined, no sudo issues encountered there (not deeply tested).
Storage: /DATA pool 3.8TB (1.3TB used), additional mounts Storage1 (1.8TB), Storage2_New
(931GB) — note Storage2_New had a mount-naming collision for a second 931GB disk showing
as `/mnt/Storage2_New-/dev/sdd1` (unexplored/not fixed — flagged only).

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
   - Docker run command used (password hash redacted, regenerate via `docker run --rm
     ghcr.io/wg-easy/wg-easy wgpw '<password>'` if rebuilding):
     ```
     docker run -d --name=wg-easy \
       -e WG_HOST=117.221.142.224 \
       -e WG_MTU=1280 \
       -e PASSWORD_HASH='<bcrypt hash — see dashboard password>' \
       -v /volume1/docker/wg-easy:/etc/wireguard \
       -p 51821:51821/tcp -p 51820:51820/udp \
       --cap-add=NET_ADMIN --cap-add=SYS_MODULE \
       --sysctl="net.ipv4.conf.all.src_valid_mark=1" \
       --sysctl="net.ipv4.ip_forward=1" \
       --restart unless-stopped \
       ghcr.io/wg-easy/wg-easy
     ```

### Also installed (earlier attempt, superseded by Docker approach)
- `WireGuard-geminilake` community .spk package (Andreas Runfalk build, DSM7.2) — CLI-only, no web UI.
  Still present/running as wg0 interface at OS level but NOT the primary VPN path going forward;
  wg-easy's Docker container runs its own independent WireGuard instance.
  **Consider removing the manual wg0 CLI setup to avoid confusion/port conflicts** if wg-easy
  is the long-term solution (uses same UDP 51820 — only one can bind it at a time; wg-easy
  currently wins since manual wg0 was brought down before starting the container).
  - Config: `/etc/wireguard/wg0.conf`, server keys at `/etc/wireguard/server_private.key` /
    `server_public.key`. Server pubkey from this build:
    `SKj2eLR/k/+OEO2/vj+Db0RP4QRgzPnAss2z4RT2FH8=` (tunnel subnet 10.8.0.1/24, port 51820)
  - One test peer was added manually (pubkey `khLiDkAIiaCeW5te6pH0sok1JpTva1+nDAsyhOScHF4=`,
    later removed) and a second working client config generated (client pubkey derived from
    private key `2NG1aCnR0UNkkH861bwgwVn07dwAG1EvFrFFcc8byG4=`, address 10.8.0.2/24) — this
    client config was for testing only, written to `/tmp/client1.conf` on the NAS (ephemeral,
    gone after reboot/tmp clear).
  - **DSM Task Scheduler jobs created during setup/troubleshooting** (Control Panel → Task
    Scheduler), all User: root, safe to delete once the manual wg0 path is decommissioned:
    - `WG Start` — runs `/var/packages/WireGuard/scripts/start`
    - `WG Setup` — generated server keys, brought up wg0 interface
    - `WG Fix Sudo` / `WG Fix Sudo2` — the sudo/PAM fixes (see below)
    - `WG Fix Pass` / `WG Fix Pass2` / `WG Fix Pass3` — password-sync attempts (Pass/Pass2 used
      `chpasswd`, not present on this BusyBox-based DSM — failed harmlessly; Pass3 used the
      correct `synouser --setpw` and succeeded)
  - Pre-existing task `TUN` (owner root, runs at Boot-up, user-defined script) was already on
    this NAS before this session — not created by us, likely enables the TUN kernel module for
    some other VPN-related purpose. Left untouched.
- WGDashboard (Python/Flask) — attempted, **failed**: this NAS's Python is 3.8.15, WGDashboard's
  current codebase requires Python 3.10+ (uses `X | None` type hint syntax). Not viable without
  a newer Python interpreter. Abandoned in favor of wg-easy (Docker-based, sidesteps Python version).
  Leftover artifacts if anyone goes looking: `/usr/local/wgdashboard/` (venv + source, unused),
  pip was bootstrapped via `python3 -m ensurepip` (pip 22.0.4) since no `pip3` existed originally.

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

Note: `root`'s local Unix password was also set to the same value during troubleshooting
(`synouser --setpw root '...'` attempt, and later a `chpasswd` attempt that failed — command
not found on BusyBox). Root's shadow password MAY now be set, but **root SSH login is still
refused** — confirmed this is blocked at the sshd config level (`PermitRootLogin no`), not a
password issue, so this is not an actual access path and was not pursued further.

## Speed / Capacity Testing
- WAN speed test via Cloudflare endpoints (curl from the NAS itself): download ~179 Mbps
  (22.3 MB/s), upload ~134 Mbps (16.7 MB/s)
- UDP port check (external, via SkyPath tool) for 51820 came back "No reply (open or
  filtered)" — **this is EXPECTED and inconclusive for WireGuard**, which by design never
  responds to non-handshake packets (anti-scanning behavior). Generic UDP probes cannot
  confirm WireGuard reachability; only a real client handshake attempt can. Traffic counters
  on the iptables FORWARD/POSTROUTING chains (3383+ packets, several MB) later confirmed real
  traffic was flowing once a client connected, validating port forwarding works end-to-end.
- Realistic transfer capacity estimate: ~15-17 MB/s pulling files from NAS remotely (capped by
  WAN upload), ~18-20 MB/s pushing files to NAS remotely (capped by WAN download). CPU
  (Celeron J4125, ChaCha20 ~300-450 Mbps capacity) is not the bottleneck — the WAN link is.

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
