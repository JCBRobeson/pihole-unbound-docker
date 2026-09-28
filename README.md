# Pi-hole + Unbound

Pi-hole and Unbound in Docker Compose, for network-wide ad blocking and private DNS on a home network.

Pi-hole blocks ads, trackers and known-malicious domains for every device that uses it. Instead of passing lookups to your ISP or to Google or Cloudflare, it hands them to Unbound. Unbound resolves each name itself, starting from the internet's root servers, and checks DNSSEC signatures along the way. No single outside company ends up with a list of every site your household visits.

This is the setup I run in my home lab (a RHEL VM on Proxmox), written up so anyone can reproduce it.

| Service | Image | What it does |
|---|---|---|
| Pi-hole | `pihole/pihole` | The DNS server your devices talk to. Blocks, caches, and gives you a dashboard. |
| Unbound | `klutchell/unbound` | Resolves names from the root servers down and rejects forged answers. Only Pi-hole can reach it. |

## How a lookup flows

```
Your device
    │
    ▼
Pi-hole (port 53 on the host)
    │   on a blocklist? → answers 0.0.0.0 and stops
    │   already cached? → answers right away
    ▼
Unbound (172.28.0.2, internal to Docker)
    │   asks the root servers, then .com, then the site's own nameservers
    │   checks DNSSEC signatures; anything forged gets SERVFAIL
    ▼
The internet
```

## What you need

- A Linux machine with Docker Engine and the Docker Compose plugin (`docker compose`, not the old `docker-compose`)
- Port 53 free on that machine
- A static IP for the machine, so your devices can find it
- Access to your router's settings, or the ability to set DNS on each device

## Quick start

```bash
git clone <this repo> network-stack
cd network-stack

cp .env.example .env
chmod 600 .env        # only your user can read it, since it holds the password
nano .env             # fill in your values (see the table below)
```

Check that nothing else is using port 53:

```bash
sudo ss -tulpn | grep ':53 '
```

It should print nothing. On Ubuntu and some other distros, `systemd-resolved` holds port 53 by default, and you'll need to turn off its stub listener first.

Check the compose file and your variables before starting anything:

```bash
docker compose config --quiet              # no output means the file is valid
docker compose config | grep revServers    # confirm the variables were filled in
```

Only grep for specific lines here. The full output of `docker compose config` includes your password. A misspelled variable name quietly becomes an empty string, and this is where you'd catch it: `true,,#53,` means something didn't match.

Start it up:

```bash
docker compose up -d
docker compose ps     # both containers should show "healthy"
```

The dashboard is at `http://<your-server-ip>:8053/admin`.

Two more things aren't in the compose file and need doing by hand: adding the blocklists (next section), then pointing your devices at the server.

### `.env` values

| Variable | Example | What it's for |
|---|---|---|
| `PIHOLE_PASSWORD` | `change-me` | Password for the web dashboard |
| `TZ` | `Etc/UTC` | Time zone for Pi-hole's logs ([list of names](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)) |
| `LAN_SUBNET` | `192.168.1.0/24` | Your home network range, in CIDR notation |
| `LAN_GATEWAY` | `192.168.1.254` | Your router, which hands out IP addresses and knows device names |
| `LAN_DOMAIN` | `lan` | The local domain your router gives out. On NetworkManager systems: `nmcli device show \| grep -i domain`. Otherwise, check your router's settings. |

Everything specific to your network lives in `.env`, so the compose file itself works as-is on any network.

## Host DNS: don't point the server at itself

The machine running this stack should never use Pi-hole for its own DNS lookups. If it does, a broken Pi-hole container takes the host's DNS down with it, and then the host can't pull images, install packages or reach GitHub to fix the problem. The thing that's broken is the thing you'd need to fix it.

It's easy to create this loop without meaning to. Once your router hands out Pi-hole as the DNS server for your whole network, the host picks that up over DHCP like every other device.

So I set the host's DNS manually to a public resolver and tell it to ignore DNS from DHCP. I use Quad9, which validates DNSSEC and blocks known malware domains. That matters here because the host deliberately bypasses Pi-hole's own blocklists. The host gets three servers: `9.9.9.9`, then `149.112.112.112` as a backup on a separate IP range, then `2620:fe::fe` over IPv6 as a last resort. Three is the most Linux will use; any extra `nameserver` lines are silently ignored.

On a host that uses NetworkManager (RHEL, Fedora and most desktop distros):

    nmcli -g NAME,DEVICE connection show --active       # find your connection name
    sudo nmcli connection modify <name> \
        ipv4.dns "9.9.9.9,149.112.112.112" ipv6.dns "2620:fe::fe" \
        ipv4.ignore-auto-dns yes ipv6.ignore-auto-dns yes
    sudo nmcli connection up <name>

`connection up` briefly drops the network while it reconnects. I use it instead of `nmcli device reapply` because reapply adds the new servers but doesn't forget ones already learned from the router.

Then check it:

    cat /etc/resolv.conf                          # only the three Quad9 addresses
    dig dnssec-failed.org | grep status           # SERVFAIL

Restart the stack afterwards (`docker compose restart`). Containers copy the host's DNS settings when they start.

If your host uses something other than NetworkManager, like netplan or systemd-networkd, the idea is the same: set the DNS servers explicitly and ignore the ones DHCP hands out.

This only applies to the machine running Pi-hole. Every other device on the network should use Pi-hole normally.

## Blocklists

Pi-hole keeps its blocklists in its own database (`pihole/gravity.db`), not in the compose file. So you add them once through the dashboard, and again after any fresh rebuild. To avoid redoing it, back up your setup from **Settings → Teleporter** and restore from the same page.

In the dashboard, go to **Lists** and add each of these as a **blocklist**:

| List | URL | Notes |
|---|---|---|
| StevenBlack hosts | `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` | Pi-hole's default list, already there on first start |
| HaGeZi Multi Pro++ | `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/pro.plus.txt` | Ads, trackers and telemetry. Fairly aggressive. |
| HaGeZi Threat Intelligence Feeds | `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/tif.txt` | Phishing, malware, scam and command-and-control domains. Security, not ads. |

Then rebuild Pi-hole's list database so it downloads them:

```bash
docker exec pihole pihole -g
```

With all three lists it takes about a minute.

Use the `adblock/` versions of the HaGeZi lists. Older guides link to a `domains/` folder that no longer exists in the repo, and HaGeZi recommends the Adblock format for Pi-hole anyway. Each line like `||example.com^` blocks a domain and all its subdomains.

**Why Pro++ and not Pro?** In HaGeZi's own benchmark, Pro++ blocked about 40% of all DNS queries versus about 33% for Pro. The tradeoff is more breakage: some affiliate links from shopping emails or blogs won't open, and some device telemetry is blocked, which can switch off minor features. When something breaks, find the red entry in the **Query Log** and click **Allow**. If you're running this for people who won't want to troubleshoot, Pro is the safer pick.

**Why the full Threat Intelligence list?** HaGeZi also offers medium and mini versions, but those are for blockers that hold everything in RAM. Pi-hole keeps its lists in an on-disk database, so adding the full list (about 2.3 million domains) moved Pi-hole's memory from 19.8 MiB to 20.1 MiB. The real costs are disk and time: building the database wrote about 320 MB, Pi-hole keeps the previous copy as a backup, and the rebuild takes about a minute. I measured it with `docker stats --no-stream pihole` and `time docker exec pihole pihole -g`.

With all three lists you'll end up at roughly 2.65 million entries, about 2.59 million of them unique. That number overstates things a bit. StevenBlack lists exact hostnames, while HaGeZi uses wildcard rules, and Pi-hole counts lines. So much of StevenBlack is already covered by HaGeZi without being counted as a duplicate.

One built-in behavior worth knowing: Pi-hole answers `mask.icloud.com` as "doesn't exist," with or without any lists. That's Apple's signal for "don't use iCloud Private Relay on this network," so Safari's lookups go through Pi-hole while at home instead of around it.

## Testing it

Run these on the server. `127.0.0.1` is Pi-hole on the same machine.

```bash
dig @127.0.0.1 example.com                        # NOERROR. Run it twice: the second answer is ~1 ms (cached)
dig @127.0.0.1 ad.doubleclick.net +short          # 0.0.0.0, i.e. blocked
dig @127.0.0.1 dnssec-failed.org | grep status    # SERVFAIL: this domain's signatures are deliberately broken
dig @127.0.0.1 whoami.akamai.net +short           # your own public IP, which proves Unbound resolves itself instead of forwarding
```

Ask Unbound directly, skipping Pi-hole:

```bash
docker exec unbound drill -D @127.0.0.1 example.com | grep flags   # includes "ad", meaning DNSSEC verified
docker exec unbound unbound-checkconf -o do-ip6                    # no, so the override file loaded
```

Confirm Pi-hole picked up the settings from `.env` and compose:

```bash
docker exec pihole pihole-FTL --config dns.revServers        # your conditional forwarding string
docker exec pihole pihole-FTL --config database.maxDBdays    # 14
dig @127.0.0.1 -x <a-device-ip> +short                        # that device's name, from your router
```

For comparison, try `dig @<your-router-ip> dnssec-failed.org | grep status`. Many ISP routers return `NOERROR`, meaning they hand out the forged answer. Mine did.

Then test from a real device. Set its DNS to your server's IP, browse for a bit, and run the **Extended test** at [dnsleaktest.com](https://www.dnsleaktest.com). The only IP listed should be your own public IP. The ISP column will show your ISP's name, which is expected: it's just who owns your IP address.

## Why it's set up this way

### Networking

- **Its own Docker network (`dns-net`, `172.28.0.0/24`) with fixed IPs.** Unbound is `.2` and Pi-hole is `.3`. Pi-hole points at an exact address rather than looking Unbound up by name, which makes it easy to test with `dig`. If `172.28.0.0/24` clashes with your LAN or a VPN, change it.
- **Unbound publishes no ports.** Only Pi-hole can reach it. Nothing on your LAN or the internet can.
- **Pi-hole publishes 53 (TCP and UDP) for DNS, and 8053 for the dashboard.** 8053 is only because other services on my host already use 8080 and 8090, and I'm keeping 80/443 free for a reverse proxy. Use whatever port suits you.
- **HTTPS (443), DHCP (67/udp) and NTP (123/udp) are not published.** None are needed here. I think DHCP belongs on your router or firewall, not on Pi-hole.
- **Never forward port 53 from your router to this server.** Pi-hole's listening mode (below) answers anyone who can reach it, which would turn it into an open resolver on the internet.

### Pi-hole settings

Every `FTLCONF_<section>_<key>` variable in the compose file maps onto Pi-hole's `pihole.toml` and locks that setting in the web dashboard. The compose file is the single source of truth, not whatever someone last clicked.

| Setting | Why |
|---|---|
| `FTLCONF_dns_upstreams: '172.28.0.2#53'` | Unbound is the only upstream. Adding a second, like `1.1.1.1`, would split queries between them and send some to a public resolver, which defeats the point. |
| `FTLCONF_dns_listeningMode: 'ALL'` | Required with Docker bridge networking. Queries reach Pi-hole through Docker's NAT, not from a "local" network, so the default mode would refuse them. |
| `FTLCONF_webserver_api_password` from `.env` | Keeps the password out of the compose file and out of Git. |
| NTP turned off (`FTLCONF_ntp_*: 'false'`) | The host already keeps time (`chronyd` in my case). Pi-hole shouldn't fight it. |
| `cap_add: SYS_NICE` only | `SYS_TIME` isn't needed with NTP off. `NET_ADMIN` would only be needed if Pi-hole did DHCP. |
| Pi-hole's own DNSSEC off | Unbound already validates and refuses forged answers. Turning it on in Pi-hole too only adds the `ad` flag to replies and BOGUS labels in the query log. There's no extra protection. |
| `FTLCONF_misc_privacylevel: '0'` | The query log shows every domain and which device asked. You need that to fix blocklist breakage. 0 is already the default; setting it explicitly records the decision and locks it. If other people use your network, tell them what's logged. |
| `FTLCONF_database_maxDBdays: '14'` | Keeps 14 days of query history instead of the default 91. That's plenty for troubleshooting, and it means less browsing history sitting on disk. |
| `FTLCONF_dns_revServers: 'true,${LAN_SUBNET},${LAN_GATEWAY}#53,${LAN_DOMAIN}'` | Conditional forwarding. Pi-hole asks your router which device owns which IP, so the dashboard shows "iPhone" instead of `192.168.1.50`. The format is `<enabled>,<range>,<server>#<port>,<domain>`. Keep the `/24` on the range: without it, Pi-hole treats it as a single IP. |
| `depends_on` with `condition: service_healthy` | On `docker compose up`, Pi-hole waits until Unbound is actually answering. Note this is a Compose feature only: after a host reboot, Docker restarts containers in no particular order. |

### Unbound settings

| Setting | Why |
|---|---|
| Recursive, not forwarding | Unbound asks the authoritative servers directly, so no single resolver company sees all your lookups. `whoami.akamai.net` returning your own IP confirms it. |
| `do-ip6: no` (in `unbound/custom.conf.d/no-ipv6.conf`) | The Docker network here is IPv4-only, so Unbound shouldn't waste time trying IPv6 nameservers it can't reach. |
| Healthcheck: `drill-hc @127.0.0.1 dnssec.works` | The image doesn't define a healthcheck, but it includes `drill-hc`, a small wrapper for exactly this. The image has no shell ("distroless"), so the test has to use `CMD`, not `CMD-SHELL`. |
| Image defaults otherwise | They're sensible. For example, the EDNS buffer size of 1232 avoids fragmented packets. |
| Config file permissions | Unbound runs as user 101, group 102 inside the container. Files in `custom.conf.d` must be readable by that user, or by everyone. |

Image source: <https://github.com/klutchell/unbound-docker>

### Things I tried or saw in other guides and left out

- **Pi-hole v5 variables** (`WEBPASSWORD`, `PIHOLE_DNS_1`, `DNSMASQ_LISTENING`). Pi-hole v6 ignores them. Plenty of guides still use them.
- **`NET_RAW`, `SYS_TIME`, `seccomp:unconfined`.** Extra privileges the containers don't need.
- **`chattr +i /etc/resolv.conf` and editing dhclient config.** This breaks NetworkManager on RHEL-family systems.
- **A `version:` key in the compose file.** Obsolete; current Compose ignores it.
- **`edns-buffer-size: 1472`.** Worse than the default 1232.
- **A Redis/Valkey cache behind Unbound.** Not worth it with a single Unbound instance and Pi-hole already caching in front of it.

## Layout

```
network-stack/
├── docker-compose.yaml
├── .env                  # your real values; never committed
├── .env.example          # template with placeholder values
├── pihole/               # Pi-hole's data, created on first start; never committed
└── unbound/
    └── custom.conf.d/
        └── no-ipv6.conf  # Unbound override, mounted read-only
```

Each service gets its own subfolder. I keep the project in `/srv/network-stack`. Following the Linux filesystem conventions, `/srv` is for data a machine serves, while the host's own `/etc` is for the host's own config, and would clash if you ever installed Unbound directly on the host.

## Security

- **Never commit `.env` or `pihole/`.** `.env` holds your password. `pihole/` holds the query database, a record of every domain every device looked up.
- The `.gitignore` works as an allowlist: it ignores everything, then lets through only the files meant to be public. A new file stays out of Git until you deliberately add it.
- Home network addresses like `192.168.x.x` aren't sensitive, but they still live in `.env` so the compose file works for anyone.
- Nothing in this repo should ever contain a public IP address, a MAC address or a credential.

## Known limitations

- **One server means one point of failure.** If it goes down, DNS stops working for every device using it.
- **IPv6 can bypass Pi-hole.** If your router advertises its own IPv6 DNS server, devices on automatic DNS may send some lookups there. Until your router can hand out Pi-hole's address, set DNS manually on each device and confirm with the leak test.
- **Docker's SELinux support is off on my host**, so the bind mounts don't use `:z`/`:Z` labels. If you enable it, add them.
- **Images use `:latest`.** Pin versions if you want updates to be deliberate.

## Roadmap

- Set Docker log size limits
- Enable Docker's SELinux support, with `:z`/`:Z` labels on every mount
- Pin image versions
- Consider dropping StevenBlack, since HaGeZi's wildcard rules already cover most of it
- Add a Caddy reverse proxy to this project (as `caddy/`)
- Move routing and DHCP to pfSense: put the ISP gateway in passthrough mode, have pfSense hand out Pi-hole as DNS over both IPv4 and IPv6, enable IPv6 on the Docker network, and block outbound DNS (ports 53 and 853) from everything except Pi-hole