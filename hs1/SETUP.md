# hs1 – fresh server setup

Snapshot of the configuration of **hs1** (`192.168.1.2`) taken on 2026-11-07,
plus the steps to rebuild it from a clean install.

| Item        | Value                                                     |
|-------------|-----------------------------------------------------------|
| OS          | Debian 13 (trixie), amd64, kernel 6.12                    |
| Disk        | NVMe: EFI 976M, ext4 `/` 460G, swap 15.7G                 |
| LAN         | `enp2s0`, static `192.168.1.2/24`, gateway `192.168.1.1`  |
| Tailscale   | `100.86.181.14` (MagicDNS on, no routes/exit node/SSH)    |
| Docker      | Docker CE 29.x + Compose plugin v5 (Docker apt repo)      |
| Domain      | `*.home.campbellsoftware.co.uk` → `192.168.1.2` (AdGuard rewrite) |
| Backups     | Restic → S3 (`eu-west-2`), daily 03:00 via systemd timer  |

## Repo layout

Paths mirror the server filesystem, so `hs1/etc/foo` is `/etc/foo` on the box.

```
hs1/
├── etc/
│   ├── apt/apt.conf.d/20auto-upgrades        unattended security upgrades
│   ├── apt/sources.list.d/{docker,tailscale}.list
│   ├── default/tailscaled
│   ├── hosts
│   ├── network/interfaces                    static IP (ifupdown)
│   ├── restic/hs1.env.example                → /etc/restic/hs1.env (secrets!)
│   ├── restic/hs1.files                      paths included in backup
│   ├── ssh/sshd_config                       key-only auth
│   └── systemd/system/restic-hs1-backup.{service,timer}
├── usr/local/sbin/restic-hs1                 restic wrapper that loads env
├── srv/                                      one Docker Compose project per dir
│   ├── adguard/        DNS + DHCP (host network)       — see its README.md
│   ├── caddy/          TLS reverse proxy, custom build with DigitalOcean DNS plugin
│   ├── homeassistant/  Home Assistant (host network)
│   └── mosquitto/      MQTT broker (host network, 192.168.1.2:1883)
└── ufw-rules.txt                             firewall rules (`ufw show added`)
```

### Not in git (secrets / state)

These live only on the server and in the Restic backup. Recreate or restore them:

| Path on server                                 | What                                   |
|------------------------------------------------|----------------------------------------|
| `/srv/caddy/.env`                              | `DO_AUTH_TOKEN` (DigitalOcean API token, DNS write) |
| `/srv/caddy/data`, `/srv/caddy/config`         | ACME account + issued certificates     |
| `/srv/mosquitto/config/passwd`                 | MQTT users `homeassistant`, `mqtt-admin` |
| `/srv/mosquitto/data`                          | Retained messages / persistence        |
| `/srv/adguard/conf/AdGuardHome.yaml` (`users[].password`) | Admin bcrypt hash (redacted in repo) |
| `/srv/adguard/work`                            | Filters, stats, query log, DHCP leases |
| `/srv/homeassistant/config/.storage`           | **All UI config**: integrations, devices, entities, users, `http` settings (trusted proxy) |
| `/srv/homeassistant/config/home-assistant_v2.db` | Recorder history                     |
| `/srv/homeassistant/backup-key.txt`            | HA backup encryption key               |
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
with `ip -br link` – it is `enp2s0` on the current hardware), and `etc/hosts`
to `/etc/hosts`. Then:

```bash
sudo systemctl restart networking   # or reboot
ip -br addr show enp2s0              # expect 192.168.1.2/24
```

> `dns-nameservers` in `interfaces` is effectively ignored once Tailscale takes
> over `/etc/resolv.conf` (MagicDNS, `100.100.100.100`).

Ensure the router has `192.168.1.2` outside its DHCP pool / reserved (or that
AdGuard is the DHCP server – see §8).

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

## 6. Tailscale

```bash
sudo curl -fsSL https://pkgs.tailscale.com/stable/debian/trixie.noarmor.gpg \
  -o /usr/share/keyrings/tailscale-archive-keyring.gpg
sudo cp etc/apt/sources.list.d/tailscale.list /etc/apt/sources.list.d/tailscale.list
sudo apt update && sudo apt install -y tailscale
sudo tailscale up            # authenticate via the printed URL
```

Current prefs are defaults: accept DNS on, no subnet routes, no exit node,
no Tailscale SSH. If re-adding the machine, remove the old `hs1` from the
Tailscale admin console first so the name stays `hs1`.

## 7. Restore data from Restic (rebuild only)

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
sudo restic-hs1 restore latest --host hs1 --target / --include /srv
```

Only restore `/etc/*` paths selectively (e.g. compare with `--target /tmp/r`),
as the new install's `/etc/systemd`, `/etc/ssh` host keys etc. may differ.
Restoring `/etc/ssh/ssh_host_*` keeps the old host key (no client warnings).

Fresh server with no backup: just copy `srv/` from this repo to `/srv` and
create the secrets in §8.

## 8. Services in `/srv`

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

Routes (see `Caddyfile`): `ha.` → `192.168.1.2:8123`, `dns.` → `192.168.1.2:3000`,
`test.` → static response, anything else aborted. Binds only `192.168.1.2:80/443`.

### Mosquitto

```bash
cd /srv/mosquitto
sudo chown root:1883 config && sudo chmod 750 config
sudo chown root:1883 config/mosquitto.conf && sudo chmod 640 config/mosquitto.conf
sudo install -d -o 1883 -g 1883 -m 750 data
# create users (skip if passwd was restored); -c only for the first one
sudo docker run --rm -it -v "$PWD/config:/mosquitto/config" eclipse-mosquitto:2.1.2-alpine \
  mosquitto_passwd -c /mosquitto/config/passwd homeassistant
sudo docker run --rm -it -v "$PWD/config:/mosquitto/config" eclipse-mosquitto:2.1.2-alpine \
  mosquitto_passwd /mosquitto/config/passwd mqtt-admin
sudo chown 1883:1883 config/passwd && sudo chmod 600 config/passwd
sudo docker compose up -d
```

### Home Assistant

```bash
cd /srv/homeassistant
sudo docker compose up -d
```

Fresh install (no restore): onboard at `http://192.168.1.2:8123`, then in
*Settings → System → Network* / via `.storage/http` set:

- `use_x_forwarded_for: true`
- `trusted_proxies: [<caddy container IP>/32]` (currently `172.18.0.2/32`)

Then add the MQTT integration with the `homeassistant` user.

### AdGuard Home (DNS + DHCP)

```bash
cd /srv/adguard
sudo install -d -m 750 conf work
sudo chmod 600 conf/AdGuardHome.yaml
```

Set the admin password hash in `conf/AdGuardHome.yaml` (`users[0].password`)
following *Reset the administrator password* in `srv/adguard/README.md`, then:

```bash
sudo docker compose up -d
dig @192.168.1.2 ha.home.campbellsoftware.co.uk A    # expect 192.168.1.2
```

**DHCP:** the committed config has `dhcp.enabled: true`. Only start AdGuard with
that setting once the router's DHCP server is off, otherwise there will be two
DHCP servers on the LAN. See `srv/adguard/README.md` for enable/disable steps.

All containers use `restart: unless-stopped`, so they return after reboot.

## 9. Firewall (ufw)

Defaults: deny incoming, allow outgoing, deny routed. Two rules depend on the
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
sudo ufw allow in on $BR from $CADDY_IP to 192.168.1.2 port 3000 proto tcp comment 'AdGuard UI from Caddy only'
sudo ufw allow in on enp2s0 from 192.168.1.0/24 to 192.168.1.2 port 53 proto tcp comment 'AdGuard DNS LAN only'
sudo ufw allow in on enp2s0 from 192.168.1.0/24 to 192.168.1.2 port 53 proto udp comment 'AdGuard DNS LAN only'
sudo ufw allow in on enp2s0 from 0.0.0.0 port 68 to 255.255.255.255 port 67 proto udp comment 'AdGuard DHCP discovery LAN only'
sudo ufw allow in on enp2s0 from 192.168.1.0/24 port 68 to 192.168.1.2 port 67 proto udp comment 'AdGuard DHCP renewal LAN only'
sudo ufw enable
```

If the Caddy bridge or IP differs from before, also update Home Assistant's
`trusted_proxies`. Reference copy of the live rules: `ufw-rules.txt`.

> Note: Docker-published ports (Caddy 80/443) are inserted by Docker's own
> iptables rules and bypass ufw; they are limited by binding to `192.168.1.2`.

## 10. Backups (Restic → S3)

If not already done in §7, set up `/etc/restic/` and the wrapper as shown there.
For a brand-new repository: `sudo restic-hs1 init`.

```bash
sudo cp etc/systemd/system/restic-hs1-backup.{service,timer} /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now restic-hs1-backup.timer
sudo systemctl start restic-hs1-backup.service      # first run
journalctl -u restic-hs1-backup.service -n 50
sudo restic-hs1 snapshots
```

Backs up `/srv`, `/etc/{ssh,ufw,systemd,docker,restic}`; retention 7 daily /
4 weekly / 12 monthly / 3 yearly, S3 Intelligent-Tiering.

## 11. Verify

```bash
docker ps                                            # caddy, homeassistant, mosquitto, adguard
sudo ss -lntup | grep -E ':(53|67|80|443|1883|3000|8123)\b'
curl -sS https://test.home.campbellsoftware.co.uk    # "Caddy on hs1 is working"
curl -sSI https://ha.home.campbellsoftware.co.uk | head -1
tailscale status
sudo ufw status verbose
systemctl list-timers restic-hs1-backup.timer
```

## Updating this snapshot

After changing the server, re-copy the changed files into `hs1/` (never the
secrets listed above), and redact the AdGuard `users[].password` hash.
