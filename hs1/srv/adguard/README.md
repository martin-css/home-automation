# AdGuard Home on hs1

AdGuard Home runs in Docker Compose with host networking.  Its persistent
configuration and state are entirely below `/srv/adguard`, which is included
by the existing Restic backup of `/srv`.

## Deployment

- Compose file: `/srv/adguard/compose.yaml`
- Configuration: `/srv/adguard/conf/AdGuardHome.yaml`
- Runtime state, filters, statistics, sessions, and leases:
  `/srv/adguard/work/data/`
- Admin UI: <https://dns.home.campbellsoftware.co.uk>
- DNS listener: `192.168.1.2:53` over UDP and TCP
- DHCP interface: `enp2s0`
- DHCP pool: `192.168.1.3` through `192.168.1.150`
- DHCP gateway: `192.168.1.1`
- DHCP DNS option: `192.168.1.2`

The Docker image is deliberately pinned.  There is no automatic container
updater.

## Routine operations

Run these commands on hs1:

```bash
cd /srv/adguard
sudo docker compose ps
sudo docker compose logs -f --tail=200 adguard
sudo docker compose restart adguard
sudo docker compose stop adguard
sudo docker compose start adguard
```

Inspect listeners and test DNS:

```bash
sudo ss -lntup | grep -E ':(53|3000)\b'
sudo ss -lunp | grep -E ':(67)\b'
dig @192.168.1.2 example.com A
dig @192.168.1.2 example.com A +tcp
dig @192.168.1.2 ha.home.campbellsoftware.co.uk A
dig @192.168.1.2 arbitrary.home.campbellsoftware.co.uk A
```

## DHCP handoff and control

Do not enable AdGuard DHCP until the router DHCP server has been disabled.
Before the handoff, check the router lease/reservation list for static devices
inside `192.168.1.3-192.168.1.150`.

Enable DHCP after disabling router DHCP:

```bash
sudo docker compose -f /srv/adguard/compose.yaml stop adguard && sudo sed -i '/^dhcp:/,/^filtering:/ s/^  enabled: false$/  enabled: true/' /srv/adguard/conf/AdGuardHome.yaml && sudo docker compose -f /srv/adguard/compose.yaml up -d adguard && sudo ss -lunp | grep -E ':(67)\b'
```

Disable AdGuard DHCP:

```bash
sudo docker compose -f /srv/adguard/compose.yaml stop adguard && sudo sed -i '/^dhcp:/,/^filtering:/ s/^  enabled: true$/  enabled: false/' /srv/adguard/conf/AdGuardHome.yaml && sudo docker compose -f /srv/adguard/compose.yaml up -d adguard && ! sudo ss -lunp | grep -E ':(67)\b'
```

Verify the persisted DHCP configuration:

```bash
sudo sed -n '/^dhcp:/,/^filtering:/p' /srv/adguard/conf/AdGuardHome.yaml
sudo cat /srv/adguard/work/data/leases.json
```

The lease file is created only after DHCP has been enabled and leases exist.

## Reset the administrator password

This procedure does not require the old password.  It uses a disposable
official Apache container to generate a bcrypt hash without putting the new
plain-text password in the command line.

1. Generate a bcrypt hash.  Enter the new password twice when prompted:

   ```bash
   sudo docker run --rm -i httpd:2.4-alpine htpasswd -niB admin
   ```

2. Copy only the hash following `admin:` (it starts with `$2y$`).
3. Stop AdGuard and edit its configuration:

   ```bash
   cd /srv/adguard
   sudo docker compose stop adguard
   sudoedit conf/AdGuardHome.yaml
   ```

4. Under `users`, retain `name: admin` and replace the associated `password`
   value with the new hash.  Quote the hash with single quotes in YAML.
5. Start AdGuard and check the logs:

   ```bash
   sudo docker compose up -d adguard
   sudo docker compose logs --tail=100 adguard
   ```

## Upgrade to a later pinned stable release

Read the upstream release notes first.  Replace `vX.Y.Z` below with the exact
stable tag being installed and replace `CURRENT_PIN` with the tag currently in
`compose.yaml`.

```bash
cd /srv/adguard
grep 'image:' compose.yaml
sudo cp compose.yaml "compose.yaml.before-vX.Y.Z"
sudo sed -i 's|adguard/adguardhome:CURRENT_PIN|adguard/adguardhome:vX.Y.Z|' compose.yaml
sudo docker compose config
sudo docker compose pull adguard
sudo docker compose up -d adguard
sudo docker compose ps
sudo docker exec adguard /opt/adguardhome/AdGuardHome --version
sudo docker compose logs --tail=100 adguard
dig @192.168.1.2 example.com A
dig @192.168.1.2 example.com A +tcp
dig @192.168.1.2 upgrade-check.home.campbellsoftware.co.uk A
```

If validation fails, restore the saved Compose file and recreate only AdGuard:

```bash
cd /srv/adguard
sudo cp "compose.yaml.before-vX.Y.Z" compose.yaml
sudo docker compose pull adguard
sudo docker compose up -d adguard
sudo docker compose logs --tail=100 adguard
```

Do not use `latest`, Watchtower, or another automatic image updater.

## Backup notes

`/etc/restic/hs1.files` includes `/srv`.  Important paths are:

- `conf/AdGuardHome.yaml`: DNS, DHCP, filters, rewrites, and administrator hash
- `work/data/filters/`: downloaded filter contents
- `work/data/leases.json`: DHCP leases, once present
- `work/data/`: statistics, sessions, query-log state, and other runtime data

No unnamed Docker volume is used.
