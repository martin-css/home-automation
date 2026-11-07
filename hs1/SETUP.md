# hs1 – fresh server setup

Snapshot of the configuration of **hs1** (`192.168.1.2`) taken on 2026-11-07,
plus the steps to rebuild it from a clean install.

| Item        | Value                                                     |
|-------------|-----------------------------------------------------------|
| OS          | Debian 13 (trixie), amd64, kernel 6.12                    |
| Disk        | NVMe: EFI 976M, ext4 `/` 460G, swap 15.7G                 |
| LAN         | `enp2s0`, static `192.168.1.2/24`, gateway `192.168.1.1`  |
| DNS/DHCP    | Router (`192.168.1.1`); server uses static `/etc/resolv.conf` |
| Docker      | Docker CE 29.x + Compose plugin v5 (Docker apt repo)      |
| VPN         | WireGuard `wg0` on hs1, UDP 51820 forwarded by router, `10.20.0.0/24`, endpoint `vpn.home.campbellsoftware.co.uk` |
| Domain      | `*.home.campbellsoftware.co.uk`, public DNS at DigitalOcean (also used for ACME DNS-01) |
| Backups     | Restic → S3 (`eu-west-2`), daily 03:00 via systemd timer  |

## Repo layout

Paths mirror the server filesystem, so `hs1/etc/foo` is `/etc/foo` on the box.

```
hs1/
├── etc/
│   ├── apt/apt.conf.d/20auto-upgrades        unattended security upgrades
│   ├── apt/sources.list.d/docker.list
│   ├── ddns-vpn.env.example                  → /etc/ddns-vpn.env (DO token!)
│   ├── hosts
│   ├── network/interfaces                    static IP (ifupdown)
│   ├── resolv.conf                           static DNS resolvers
│   ├── restic/hs1.env.example                → /etc/restic/hs1.env (secrets!)
│   ├── restic/hs1.files                      paths included in backup
│   ├── ssh/sshd_config                       key-only auth
│   ├── systemd/system/restic-hs1-backup.{service,timer}
│   ├── systemd/system/ddns-vpn.{service,timer}  public IP → vpn.home A record
│   └── wireguard/peers                       VPN peers: name + tunnel IP
├── usr/local/sbin/restic-hs1                 restic wrapper that loads env
├── usr/local/sbin/wg-hs1                     builds wg0.conf + phone configs/QR codes
├── usr/local/sbin/ddns-vpn                   DigitalOcean dynamic DNS updater
├── srv/                                      one Docker Compose project per dir
│   ├── caddy/          TLS reverse proxy, custom build with DigitalOcean DNS plugin
│   ├── homeassistant/  Home Assistant (host network)
│   │   └── patches/    temporary library patch (Ring push events), bind-mounted
│   └── mosquitto/      MQTT broker (host network, 192.168.1.2:1883)
└── ufw-rules.txt                             firewall rules (`ufw show added`)
```

### Not in git (secrets / state)

These live only on the server and in the Restic backup. Recreate or restore them:

| Path on server                                 | What                                   |
|------------------------------------------------|----------------------------------------|
| `/srv/caddy/.env`                              | `DO_AUTH_TOKEN` (DigitalOcean API token, DNS write) |
| `/srv/caddy/data`, `/srv/caddy/config`         | ACME account + issued certificates     |
| `/srv/mosquitto/config/passwd`                 | MQTT users `homeassistant`, `mqtt-admin`, `nuki` |
| `/srv/mosquitto/data`                          | Retained messages / persistence        |
| `/srv/homeassistant/config/.storage`           | **All UI config**: integrations, devices, entities, users, `http` settings (trusted proxy) |
| `/srv/homeassistant/config/home-assistant_v2.db` | Recorder history                     |
| `/srv/homeassistant/backup-key.txt`            | HA backup encryption key               |
| `/etc/wireguard/server.key`, `clients/*.key`, `wg0.conf`, `clients/*.conf` | WireGuard private keys and generated configs |
| `/etc/ddns-vpn.env`                            | DigitalOcean token for the dynamic DNS updater |
| `/etc/restic/hs1.env`, `/etc/restic/hs1.password` | AWS keys + Restic repo password — **keep a copy off-server (password manager)** |

> Home Assistant is almost entirely UI-configured (`configuration.yaml` is just
> `default_config:`), so a real rebuild depends on restoring `.storage` from
> Restic or from an HA backup + `backup-key.txt`.

---

## 1. Install Debian

1. Install Debian 13 netinst. Select only **SSH server** and **standard system utilities**.
2. Hostname `hs1`, create user `martin`, set a root password (or leave root disabled).
3. Partitioning: guided, whole disk, ext4 + swap (as above).

## 2. Base packages, sudo, SSH

As root (or via `su -`):

```bash
apt update && apt full-upgrade -y
apt install -y sudo curl ca-certificates gnupg ufw restic unattended-upgrades \
               bind9-dnsutils lsof qrencode
usermod -aG sudo martin
```

From your workstation, install your keys (currently `martin@MARTINPC` and `oneplus9`):

```bash
ssh-copy-id martin@<install-ip>
```

Then on the server, copy `etc/ssh/sshd_config` from this repo to
`/etc/ssh/sshd_config` (disables password + keyboard-interactive auth) and:

```bash
sudo sshd -t && sudo systemctl restart ssh
```

Check you can still log in from a **second** terminal before closing the first.

## 3. Static IP

Copy `etc/network/interfaces` to `/etc/network/interfaces` (check the NIC name
with `ip -br link` – it is `enp2s0` on the current hardware), `etc/hosts`
to `/etc/hosts` and `etc/resolv.conf` to `/etc/resolv.conf`. Then:

```bash
sudo systemctl restart networking   # or reboot
ip -br addr show enp2s0              # expect 192.168.1.2/24
```

> `resolvconf` is not installed, so `dns-nameservers` in `interfaces` is not
> applied; `/etc/resolv.conf` is a plain static file (router first, then
> public resolvers). Make sure nothing else (e.g. `dhcpcd`) overwrites it.
>
> Containers copy `/etc/resolv.conf` when they are created. After changing it,
> recreate them: `cd /srv/<svc> && sudo docker compose up -d --force-recreate`.

The router provides DHCP and DNS for the LAN. Ensure `192.168.1.2` is outside
its DHCP pool or reserved for hs1.

## 4. Unattended upgrades

```bash
sudo cp etc/apt/apt.conf.d/20auto-upgrades /etc/apt/apt.conf.d/20auto-upgrades
```

The stock `50unattended-upgrades` is used unchanged (Debian + Debian-Security).

## 5. Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo cp etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker martin     # log out/in afterwards
```

No `/etc/docker/daemon.json` is used (defaults).

## 6. Restore data from Restic (rebuild only)

Do this **before** starting containers so they come up with their old state.

```bash
sudo install -d -m 700 /etc/restic /var/cache/restic-hs1
sudo cp etc/restic/hs1.files /etc/restic/
sudo cp etc/restic/hs1.env.example /etc/restic/hs1.env     # fill in AWS keys
sudoedit /etc/restic/hs1.env
echo '<restic repo password>' | sudo tee /etc/restic/hs1.password >/dev/null
sudo chmod 600 /etc/restic/*
sudo install -m 755 usr/local/sbin/restic-hs1 /usr/local/sbin/restic-hs1

sudo restic-hs1 snapshots --host hs1
sudo restic-hs1 restore latest --host hs1 --target / --include /srv --exclude /srv/adguard
```

Only restore `/etc/*` paths selectively (e.g. compare with `--target /tmp/r`),
as the new install's `/etc/systemd`, `/etc/ssh` host keys etc. may differ.
Restoring `/etc/ssh/ssh_host_*` keeps the old host key (no client warnings).

Fresh server with no backup: just copy `srv/` from this repo to `/srv` and
create the secrets in §7.

## 7. Services in `/srv`

```bash
sudo rsync -a srv/ /srv/      # repo files on top of any restored state
```

### Caddy (reverse proxy, TLS via DigitalOcean DNS-01)

```bash
cd /srv/caddy
sudo cp .env.example .env && sudoedit .env    # DO_AUTH_TOKEN=...
sudo chmod 600 .env
sudo docker compose build
sudo docker compose up -d
```

Routes (see `Caddyfile`): `ha.` → `192.168.1.2:8123`, `test.` → static
response, anything else aborted. Plus a plain-HTTP site on `http://192.168.1.2`
that serves only the doorbell camera snapshot (for the TV alert, see *Scripts
and automations*). Binds only `192.168.1.2:80/443`.

LAN clients reach these names through **public DNS** (DigitalOcean zone
`home.campbellsoftware.co.uk`), since the router does the LAN DNS.
`ha.home.campbellsoftware.co.uk` (and any other routes) needs an A record
pointing to `192.168.1.2` – this same name/IP is used over the VPN.

**The router's DNS drops answers that point at private IPs** (DNS-rebind
protection; returns an empty `NOERROR`, no setting to change it). So devices
using the router for DNS cannot resolve `ha.home`. Work-around per device:

- **Android:** Settings → Network & internet → **Private DNS** → hostname
  `one.one.one.one` (or `dns.google`). Uses DNS-over-TLS on port 853, bypassing
  the router, on Wi-Fi and mobile data.
- **Windows/macOS/Linux:** set the DNS server to `1.1.1.1` / `1.0.0.1` for the
  home network, or add `192.168.1.2 ha.home.campbellsoftware.co.uk` to the
  hosts file.

Check: `dig @192.168.1.1 ha.home.campbellsoftware.co.uk +short` is empty;
`dig +tls @one.one.one.one ha.home.campbellsoftware.co.uk +short` gives `192.168.1.2`.

### Mosquitto

```bash
cd /srv/mosquitto
sudo chown root:1883 config && sudo chmod 750 config
sudo chown root:1883 config/mosquitto.conf config/acl && sudo chmod 640 config/mosquitto.conf config/acl
sudo install -d -o 1883 -g 1883 -m 750 data
# create users (skip if passwd was restored); -c only for the first one
sudo docker run --rm -it -v "$PWD/config:/mosquitto/config" eclipse-mosquitto:2.1.2-alpine \
  mosquitto_passwd -c /mosquitto/config/passwd homeassistant
sudo docker run --rm -it -v "$PWD/config:/mosquitto/config" eclipse-mosquitto:2.1.2-alpine \
  mosquitto_passwd /mosquitto/config/passwd mqtt-admin
sudo docker run --rm -it -v "$PWD/config:/mosquitto/config" eclipse-mosquitto:2.1.2-alpine \
  mosquitto_passwd /mosquitto/config/passwd nuki
sudo chown 1883:1883 config/passwd && sudo chmod 600 config/passwd
sudo docker compose up -d
```

**One MQTT user per device.** `config/acl` limits each user to its own topics
(anything not listed is denied); `homeassistant` and `mqtt-admin` get `#`.
To add a device: add a user as above, add a `user <name>` block to `config/acl`
(normally `readwrite <device>/#` and `write homeassistant/#` for discovery),
then `sudo docker compose restart mosquitto`.

> **Don't use `docker kill` (even `-s HUP`) to reload any container.** Docker
> treats every `docker kill` as a manual stop, so `restart: unless-stopped`
> will not start that container after the next reboot. Use
> `docker compose restart <svc>`, or for a live reload without a restart
> `docker compose exec mosquitto kill -HUP 1`.

| User            | Used by                 | Topics                                   |
|-----------------|-------------------------|------------------------------------------|
| `homeassistant` | HA MQTT integration     | all                                      |
| `mqtt-admin`    | manual debugging        | all                                      |
| `nuki`          | Nuki smart lock (Wi-Fi) | `nuki/#` rw, `homeassistant/#` write     |

### Home Assistant

```bash
cd /srv/homeassistant
sudo docker compose up -d
```

Fresh install (no restore): onboard at `http://192.168.1.2:8123`, then in
*Settings → System → Network* / via `.storage/http` set:

- `use_x_forwarded_for: true`
- `trusted_proxies: [<caddy container IP>/32]` (currently `172.18.0.2/32`)
- *Settings → System → Network*: **Home Assistant URL** – both internal and
  external = `https://ha.home.campbellsoftware.co.uk` (used for OAuth callbacks,
  notifications and the companion app).

Then add the MQTT integration with the `homeassistant` user.

#### Temporary patch: Ring push events (firebase-messaging)

HA 2026.10.4 ships firebase-messaging 0.4.5, whose push listener crashes on
Ring's current push format (`Incorrect padding, shutting down FcmPushClient`
about a second after every start). After that, Ring doorbell and motion events
(`event.front_door_ding` / `_motion`) never fire; only polled sensors update.
Upstream: [home-assistant/core#173915](https://github.com/home-assistant/core/issues/173915),
fix not yet merged: [firebase-messaging PR #46](https://github.com/sdb9696/firebase-messaging/pull/46).

`patches/fcmpushclient.py` is the 0.4.5 file with PR #46 applied (head
`3e33699`), plus one local change: header parsing moved inside the `try`, and
`http_ece.ECEException` caught too. `patches/fcmpushclient.diff` is the full
diff against the stock file. `compose.yaml` bind-mounts it over the library
file (read-only), so a recreate (`docker compose up -d`) is needed after
changing it.

**On every HA image update**, before `docker compose up -d`:

```bash
NEW=ghcr.io/home-assistant/home-assistant:<new-tag>
sudo docker run --rm --entrypoint sh $NEW -c \
  'pip show firebase-messaging | grep Version; python3 --version'
```

- Still `0.4.5` and Python 3.14: keep the patch.
- Python version changed: update the mount path in `compose.yaml`.
- Newer firebase-messaging: check its release notes for the padding/header fix.
  If it's fixed, delete the mount line and `patches/`. If not, re-apply PR #46 to
  the new file rather than mounting the old one over a different version.

Verify after a restart: the log has no `Incorrect padding`, and a doorbell press
changes `event.front_door_ding`.

#### HACS and custom integrations

HACS (community store) is installed manually from a pinned GitHub release, not
via the `get.hacs.xyz` script. Custom integrations live in
`config/custom_components/` (not in git; backed up by Restic with `/srv`).

| Custom integration | Version | Installed via | Purpose |
|--------------------|---------|---------------|---------|
| `hacs`             | 2.0.5   | release zip   | Community store |
| `octopus_energy`   | via HACS | HACS         | Octopus electricity + gas, tariffs, Home Mini |
| Bubble Card (frontend) | 3.4.3 | HACS      | Cards used by the Home dashboard (`config/www/community/`, resource registered by HACS) |
| `pipup`            | 0.3.1   | HACS          | On-screen TV notifications via the PiPup app on the Playroom TV (Sony Google TV, 192.168.1.169:7979) |

Install HACS (`unzip` isn't on hs1, so Python extracts it):

```bash
D=$(mktemp -d)
curl -fsSL -o $D/hacs.zip https://github.com/hacs/integration/releases/download/2.0.5/hacs.zip
sha256sum $D/hacs.zip   # 2.0.5: 97be6b824a4f38e683728cc6dd72367f6b8bad0a43428b1b3b987a3087adf413
sudo mkdir -p /srv/homeassistant/config/custom_components/hacs
sudo python3 -I -m zipfile -e $D/hacs.zip /srv/homeassistant/config/custom_components/hacs/
rm -rf $D
cd /srv/homeassistant && sudo docker compose restart
```

Then in HA: *Settings → Devices & services → Add integration → HACS*, accept
the notices and authorise with GitHub (device code). Update HACS itself and
its integrations from the HACS page; HA shows them as updates.

#### Scripts and automations

`scripts.yaml` and `automations.yaml` are in git (both also editable in the UI;
UI edits rewrite the file on hs1, so copy changes back to the repo).

| Name | What it does |
|------|--------------|
| `script.hot_water_boost_1h` / `_cancel` | Hive hot water boost for 1 h / cancel |
| `script.tv_alert` | Pause the Playroom TV (if on and not on the home screen) and show a PiPup message, optionally with a camera snapshot. Fields: `message`, `title`, `duration`, `pause`, `camera` |
| `automation.doorbell_alert_on_tv` | Ring doorbell press -> `tv_alert` "Someone is at the front door" with the doorbell snapshot; 30 s debounce |

Notes:
- Neither Playroom TV entity reports "playing" for native apps (Cast shows
  `idle`, Android TV Remote only on/off), so `tv_alert` sends the pause-only
  `MEDIA_PAUSE` key (a no-op if already paused), never play/pause.
- The Ring doorbell event entity (`event.front_door_ding`) uses event type
  `ring`, not `ding`.
- PiPup needs the PiPup app installed and allowed to draw over other apps on the
  TV. PiPup buttons call back to HA's internal URL, which the TV can't resolve
  (router DNS rebind protection), so the alerts don't use buttons.
- Snapshots: the TV can't use `ha.home.*` (DNS rebind) or reach port 8123
  (ufw), so it loads `http://192.168.1.2/api/camera_proxy/<camera>?token=...`
  from a plain-HTTP Caddy site that passes through **only**
  `camera.front_door_live_view`; HA still checks the camera's short-lived
  token. To allow another camera, add its path to `@doorbell_snapshot` in the
  Caddyfile. Fetching the snapshot doesn't create a Ring event; only starting
  live view does.
- The Caddyfile is bind-mounted as a single file: overwrite it in place
  (`sudo tee /srv/caddy/Caddyfile < Caddyfile`) and then
  `docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile`.
  Replacing the file (`mv`, `install`) leaves the container on the old copy.

#### Dashboards

The **Home** dashboard (`/dashboard-home`) is YAML-mode and versioned in
`config/dashboards/home.yaml`, registered under `lovelace:` in
`configuration.yaml`. It needs Bubble Card (install via HACS first, or the cards
show as errors). Edit the file, copy it to hs1 and refresh the browser; only
adding or removing a dashboard in `configuration.yaml` needs a restart.

```bash
scp hs1/srv/homeassistant/config/dashboards/home.yaml martin@192.168.1.2:/tmp/
ssh martin@192.168.1.2 'sudo install -m 644 /tmp/home.yaml /srv/homeassistant/config/dashboards/ && rm /tmp/home.yaml'
```

Layout: summary sections on the main view; per-floor heating, energy and
appliance detail and the weather forecast are Bubble pop-ups (`#ground`,
`#first`, `#energy`, `#appliances`, `#weather`) opened from the bottom bar.
Weather is the built-in Met.no integration (`weather.forecast_home`). Avoid YAML anchors/merge keys in
dashboards: HA logs a duplicate-key warning for every override. The Ring card
shows the snapshot preview only; a Ring event is created only when someone
presses play for live view.

Set it as the default (Settings -> Dashboards -> Home -> *Set as default*);
the auto-generated Overview stays as a fallback. The Map dashboard is
storage-mode (UI-edited, not in git).

#### Octopus Energy

1. HACS → search **Octopus Energy** (BottlecapDave) → Download → restart HA.
2. *Add integration → Octopus Energy*: account number (`A-XXXXXXXX`) and API key
   from octopus.energy → Account → Personal details → API access.
3. *Settings → Dashboards → Energy*: add the electricity and gas consumption
   sensors and the tariff cost entities the integration creates.

The Octopus Home Mini gives near-live electricity and ~30-min gas data via the
same integration; without it, data arrives with roughly a day's delay. It talks
to Octopus over 2.4 GHz Wi-Fi, not to HA directly. If it stops reporting, a hard
reset and re-pair in the Octopus app fixed it previously (Nov 2026). Reload the
integration afterwards so the Home Mini entities appear.

All containers use `restart: unless-stopped`, so they return after reboot.

## 8. Firewall (ufw)

Defaults: deny incoming, allow outgoing, deny routed. One rule depends on the
Docker bridge for the `caddy_ingress` network and Caddy's IP on it. Docker
assigns these when the network is created, so look them up **after** Caddy has
started:

```bash
BR=br-$(docker network inspect caddy_ingress -f '{{printf "%.12s" .Id}}')
CADDY_IP=$(docker inspect caddy -f '{{(index .NetworkSettings.Networks "caddy_ingress").IPAddress}}')
echo "$BR $CADDY_IP"     # previously br-8b5fa0e1c1e7 172.18.0.2
```

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow from 192.168.1.0/24 to any port 1883 proto tcp comment 'Mosquitto LAN only'
sudo ufw allow from 192.168.1.0/24 to 192.168.1.2 port 80 proto tcp comment 'Caddy HTTP LAN only'
sudo ufw allow from 192.168.1.0/24 to 192.168.1.2 port 443 proto tcp comment 'Caddy HTTPS LAN only'
sudo ufw allow from 192.168.1.0/24 to 192.168.1.2 port 443 proto udp comment 'Caddy HTTP3 LAN only'
sudo ufw allow in on $BR from $CADDY_IP to 192.168.1.2 port 8123 proto tcp comment 'Home Assistant from Caddy only'
sudo ufw allow 51820/udp comment 'WireGuard'
sudo ufw allow in on wg0 from 10.20.0.0/24 to 192.168.1.2 port 443 proto tcp comment 'Caddy HTTPS from VPN'
sudo ufw allow in on wg0 from 10.20.0.0/24 to 192.168.1.2 port 443 proto udp comment 'Caddy HTTP3 from VPN'
sudo ufw enable
```

If the Caddy bridge or IP differs from before, also update Home Assistant's
`trusted_proxies`. Reference copy of the live rules: `ufw-rules.txt`.

> Note: Docker-published ports (Caddy 80/443) are inserted by Docker's own
> iptables rules and bypass ufw; they are limited by binding to `192.168.1.2`.

## 9. WireGuard VPN (remote access)

Phones connect to `vpn.home.campbellsoftware.co.uk:51820/udp`, which the router
forwards to hs1. It is a **split tunnel**: the phones send only `192.168.1.2`
(Caddy → Home Assistant) and `10.20.0.1` (hs1) through it; everything else goes
straight to the internet. Because `ha.home` resolves to `192.168.1.2` in public
DNS, the same URL works at home and away.

```
Phone ──UDP 51820──> Router (port forward) ──> hs1 wg0 10.20.0.1
   https://ha.home.campbellsoftware.co.uk → 192.168.1.2:443 (Caddy) → HA :8123
```

| Peer          | Tunnel IP   |
|---------------|-------------|
| hs1 (server)  | 10.20.0.1   |
| martin-phone  | 10.20.0.2   |
| katie-phone   | 10.20.0.3   |
| ethan-phone   | 10.20.0.4   |

### Router

Forward **UDP 51820** (WAN) → `192.168.1.2:51820`. Requires a real public IPv4
on the router's WAN (no CGNAT).

### Dynamic DNS for `vpn.home`

```bash
sudo apt install -y jq
sudo install -m 755 usr/local/sbin/ddns-vpn /usr/local/sbin/
sudo install -m 600 etc/ddns-vpn.env.example /etc/ddns-vpn.env && sudoedit /etc/ddns-vpn.env
sudo cp etc/systemd/system/ddns-vpn.{service,timer} /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now ddns-vpn.timer          # every 5 min, TTL 60s
sudo systemctl start ddns-vpn.service && journalctl -u ddns-vpn.service -n 5
```

Use a DigitalOcean token with a **custom scope of `domain:read` + `domain:update`**.

### WireGuard server

```bash
sudo apt install -y wireguard-tools qrencode
sudo install -m 755 usr/local/sbin/wg-hs1 /usr/local/sbin/
sudo install -d -m 700 /etc/wireguard
sudo install -m 600 etc/wireguard/peers /etc/wireguard/peers
```

```bash
sudo wg-hs1 render     # creates any missing keys, writes wg0.conf + clients/*.conf
```

With `/etc/wireguard` restored from Restic the existing keys are reused and
the phones keep working. Without it, new keys are generated and every phone
must re-scan its QR code.

```bash
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

`wg-hs1` writes `/etc/wireguard/wg0.conf` and `clients/<name>.conf` from
`peers` + keys, and applies changes live with `wg syncconf`. Don't edit
`wg0.conf` by hand.

### Phones (Android)

1. Install **WireGuard** (by WireGuard Development Team) from the Play Store.
2. On hs1, show the phone's QR code: `sudo wg-hs1 qr martin-phone`
3. In the app: **+ → Scan from QR code**, name it `home`, turn it on.
4. Set **Private DNS** to `one.one.one.one` (see the router note in §7) –
   needed for `ha.home` to resolve on home Wi-Fi.
5. HA companion app: set both internal and external URL to
   `https://ha.home.campbellsoftware.co.uk`.

Turn the tunnel off on home Wi-Fi unless the router supports NAT loopback
(hairpin): the tunnel endpoint is the house's public IP. The third-party
*WG Tunnel* app can toggle automatically by SSID.

### Add / remove a device

```bash
sudo wg-hs1 add laptop-name          # next free IP, applied live
sudo wg-hs1 qr laptop-name
# remove: delete its line from /etc/wireguard/peers, then
sudo wg-hs1 render && sudo rm /etc/wireguard/clients/<name>.*
```

Update `etc/wireguard/peers` and the table above in this repo to match.

### Later: access to the rest of the LAN

hs1 already has IP forwarding (Docker enables it) and a NAT rule for
`10.20.0.0/24` out of `enp2s0` (in `wg0.conf` `PostUp`). Routed traffic is
blocked by ufw. To open it up:

1. `sudo ufw route allow in on wg0 out on enp2s0 from 10.20.0.0/24 to 192.168.1.0/24 comment 'VPN to LAN'`
2. In `/usr/local/sbin/wg-hs1` set `CLIENT_ALLOWED_IPS="192.168.1.0/24, 10.20.0.1/32"`
   (and in this repo), run `sudo wg-hs1 render`, re-import the QR codes on the phones.

Narrow the ufw rule to specific hosts/ports if only some devices should be reachable.

## 10. Backups (Restic → S3)

If not already done in §6, set up `/etc/restic/` and the wrapper as shown there.
For a brand-new repository: `sudo restic-hs1 init`.

```bash
sudo cp etc/systemd/system/restic-hs1-backup.{service,timer} /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now restic-hs1-backup.timer
sudo systemctl start restic-hs1-backup.service      # first run
journalctl -u restic-hs1-backup.service -n 50
sudo restic-hs1 snapshots
```

Backs up `/srv`, `/etc/{ssh,ufw,systemd,docker,restic,wireguard}`, `/etc/ddns-vpn.env`; retention 7 daily /
4 weekly / 12 monthly / 3 yearly, S3 Intelligent-Tiering.

## 11. Verify

```bash
docker ps                                            # caddy, homeassistant, mosquitto
sudo ss -lntup | grep -E ':(80|443|1883|8123)\b'
curl -sS https://test.home.campbellsoftware.co.uk    # "Caddy on hs1 is working"
curl -sSI https://ha.home.campbellsoftware.co.uk | head -1
sudo ufw status verbose
sudo wg show                                         # latest handshake per phone
dig +short vpn.home.campbellsoftware.co.uk           # = curl -4 ifconfig.me
systemctl list-timers restic-hs1-backup.timer ddns-vpn.timer
```

## Updating this snapshot

After changing the server, re-copy the changed files into `hs1/` (never the
secrets listed above).
