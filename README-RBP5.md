# LANtern 🎃

A Raspberry Pi 5 set up from scratch as a dedicated, locked-down DNS box for a home network. It runs Pi-hole and Unbound (from my [pihole-unbound-docker](https://github.com/JCBRobeson/pihole-unbound-docker) repo) as the primary DNS server for the house, with a second Pi-hole on another machine as the backup.

This is the full build, start to finish: hardware, flashing, first boot, networking, hardening, Docker, deploying the DNS stack, testing, and failover. Every step has the reasoning behind it, because "run these commands" stops being useful the first time something goes sideways. I wrote it so I can rebuild the box from nothing, and so anyone else can build the same thing. It's also the outline for automating this with Ansible later.

## Why a separate box for DNS

DNS is the one service everyone in the house feels. If it goes down, "the internet is broken," even though it isn't.

My Pi-hole first lived on a VM on my main home server. That machine is where I experiment, rebuild things and break them on purpose, so every reboot or bad change took DNS down with it. A dedicated Pi fixes that:

- It does one job, so there's very little reason to touch it.
- It's separate hardware, so the main server can be down for hours and DNS keeps working.
- The old Pi-hole stays on the main server as the backup. Two Pi-holes on two machines means either one can fail.

A UPS doesn't solve this on its own. It only covers power cuts, not crashes, bad updates, or me taking a box apart.

## Hardware

| Part | Why |
|---|---|
| Raspberry Pi 5 (I used the 8GB; 2GB is plenty for DNS alone) | Pi-hole and Unbound together use a few hundred MB. Extra RAM is headroom for monitoring and other small infrastructure services later. |
| Official 27W USB-C power supply | The Pi 5 is picky about power. Cheap chargers cause undervoltage and random instability, which is the last thing a DNS server needs. |
| A case with cooling | It runs 24/7. A fan case or a passive aluminum case both work. Passive has no moving parts to fail; a fan gives more margin in a hot spot. Pick one cooling approach. Aluminum cases usually can't take the Active Cooler or stick-on heatsinks because the case itself is the heatsink. |
| microSD card, ideally "high endurance" | Pi-hole writes to its database constantly, which wears out normal cards. A starter-kit card is fine to begin with. Swap in a high-endurance card before the Pi becomes the house's main DNS. |
| Ethernet cable | A DNS server should be wired. Wi-Fi drops are exactly the kind of failure this whole setup is trying to avoid. |

Optional: the official RTC battery (keeps the clock running while unplugged) and an M.2 HAT+ with an NVMe SSD (more durable storage). Neither is required. See [the clock](#a-note-on-the-clock) below.

Why a Pi 5 and not a 3 or 4: a Pi 3B+ can technically run Pi-hole and Unbound, but 1GB gets tight once you add anything else, and it'll stop getting updates sooner. After the 2026 price increases, the Pi 4 costs nearly as much as a Pi 5 and gives you less. The Pi 5 is the one that'll still be supported years from now.

Don't bother with an inline power switch. The Pi 5 has its own power button, and on an always-on DNS box a switch is just something to bump by accident.

## Before you start

**Pick the Pi's address.** A DNS server needs a fixed address, or every device pointing at it breaks when it changes. Log into your router and find its DHCP range (often under something like Home Network → DHCP). Anything below the start of that range is safe to assign by hand.

My convention: infrastructure gets static addresses counting up from `.10`, below the router's DHCP range, set on the machine itself. Floating IPs (for failover later) get their own block so you can tell what an address is just by its number.

| Example address | Use |
|---|---|
| `.10`, `.11`, … | Servers (hypervisor, main server, this Pi) |
| `.50`–`.63` | Floating IPs, later |
| `.64` and up | The router's DHCP range: phones, laptops, TVs |

Check your plan against what's already in use. I nearly assigned a floating IP to the address my hypervisor already had.

Throughout this README, `192.168.1.12` is the Pi, `192.168.1.254` is the router, and `youruser` is your username. Replace them with your own.

## 1. Flash the SD card

Use **Raspberry Pi Imager**, downloaded from raspberrypi.com only. Flashing tools from random sites are a classic way to pick up malware.

1. **Device:** Raspberry Pi 5
2. **OS:** Raspberry Pi OS (other) → **Raspberry Pi OS Lite (64-bit)**. Lite has no desktop. This is a headless box managed over SSH, and a desktop would only add software to patch and things that can break. 64-bit uses all the RAM and matches the ARM64 Docker images.
3. **Storage:** the SD card. Imager erases whatever you pick here, so unplug any other USB drives first if you're not sure which is which.

Then the customisation settings. This is where most of the security happens, because it's all baked into the card before the Pi ever boots.

| Setting | What I set | Why |
|---|---|---|
| Hostname | `lantern` | How the Pi identifies itself on the network. Shows up in the router and in Pi-hole's dashboard. Not sensitive; have fun with it. |
| Localisation | Your time zone, keyboard `us` | Matching time zones across machines makes comparing logs at 3am much less painful. Keyboard matters if you ever plug one in. (The time zone only changes how times are displayed; Linux keeps time in UTC internally.) |
| User | A non-default username, a strong unique password | Bots try `pi` and `admin` endlessly. The password still matters with key-only SSH: it's what `sudo` asks for, and it's how you log in at the console if SSH ever breaks. Save it in a password manager. |
| Wi-Fi | **Nothing.** Clear both fields. | Wired only. It also keeps your Wi-Fi password off the card. Whatever you type here gets written to it. |
| SSH | Enabled, **public key authentication** | No password logins over the network, so password guessing can't work. |
| Raspberry Pi Connect | **Off** | It's a cloud remote-access service. That's another way into your network, tied to another account you'd have to protect. I use SSH at home and Tailscale for remote access. |

**Gotcha: Imager refills the Wi-Fi fields from your PC every time you open that page.** Clear them, click next, and don't go back. The summary screen before writing is the real check: Wi-Fi should not be listed under customisations.

For the SSH key, paste your **public** key, the one-line file ending in `.pub`:

```powershell
Get-ChildItem $env:USERPROFILE\.ssh
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

It should be one line starting with `ssh-ed25519`. If you ever see `-----BEGIN OPENSSH PRIVATE KEY-----`, that's the private key. Stop and don't paste it anywhere. I copy the key from PowerShell rather than using Imager's Browse button, because Windows hides file extensions and thinks `.pub` files are Publisher documents, which makes it easy to grab the wrong file.

Reusing the key you already use for your other servers is normal. A key identifies the computer you connect from, not the server you connect to.

When it's done, Windows may say the disk needs formatting. Click **Cancel**. Windows just can't read a Linux filesystem. Formatting would wipe what you just wrote.

## 2. First boot

Order matters:

1. SD card in (underside of the board, label facing away)
2. **Ethernet in.** There's no Wi-Fi, so without the cable the Pi has no network at all.
3. Power last. The Pi boots as soon as it gets power.

Red light means power, flickering green means it's reading the card. Give the first boot two or three minutes. It expands the filesystem, applies your settings, and may reboot itself once.

## 3. Find the Pi on the network

The router handed the Pi an address and remembers it by hostname, so you can ask the router's DNS from any Linux machine:

```bash
dig @192.168.1.254 lantern.lan +short
```

- `@192.168.1.254` asks the router directly.
- `lantern.lan` is the hostname plus your router's local domain. Find yours with `nmcli device show | grep -i domain` on a NetworkManager machine, or in the router's settings.
- `+short` prints just the answer.

This works because your router does both jobs: DHCP gives out the address, and its small built-in DNS server remembers which name got which address. If it comes back empty, check the router's connected-devices page instead.

## 4. First SSH connection

```powershell
ssh youruser@192.168.1.12
```

The first time, SSH asks whether you trust the Pi's host key. Your PC has never seen this machine before. Type `yes` and your PC saves the fingerprint in `known_hosts`. From then on it checks automatically. Trusting a key the first time you see it is called "trust on first use," which is a reasonable trade on your own network with a Pi you set up five minutes ago.

If you ever see **WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED**, take it seriously. At home it usually means you reflashed the Pi, or a different machine used to have that address. Before you remove the old entry, prove the new key is legitimate. Compare it against a fingerprint you already trust:

```powershell
ssh-keygen -lF 192.168.1.134     # the fingerprint saved for the Pi's old address
ssh-keygen -R 192.168.1.12       # remove the stale entry (backs up known_hosts first)
```

If the new fingerprint matches one you already trusted, remove the stale entry and reconnect.

## 5. Verify the baseline

Don't assume the Imager settings took. Check them:

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|kbdinteractiveauthentication|permitrootlogin'
nmcli connection show
vcgencmd get_throttled
vcgencmd measure_temp
```

| Command | You want | What it proves |
|---|---|---|
| `sshd -T` | `passwordauthentication no`, `kbdinteractiveauthentication no`, `permitrootlogin` set to `without-password` or `prohibit-password` | Key-only SSH, and no password route in through the side door. `-T` prints the SSH server's final settings after every config file is combined. |
| `nmcli connection show` | Only an `ethernet` connection (plus `lo`) | No Wi-Fi config snuck onto the card |
| `get_throttled` | `throttled=0x0` | No undervoltage or overheating since boot. `0x50000` or `0x50005` means the power supply couldn't keep up. |
| `measure_temp` | Roughly 40–55°C idle | It's cooling properly. The Pi starts slowing itself down around 80°C. |

`vcgencmd` is a Pi-specific tool for asking the firmware about power, heat and clocks. For something that works on any Linux box, `cat /sys/class/thermal/thermal_zone0/temp` gives the temperature in thousandths of a degree.

## 6. Static IP and the Pi's own DNS

Two things in one NetworkManager profile:

1. **A static address**, so devices can always find it.
2. **The Pi's own DNS set to a public resolver, not to itself.** The machine running Pi-hole must never use its own Pi-hole for its own lookups. If Pi-hole breaks, the host can't resolve anything, so it can't pull images or install packages to fix it. I also tell it to ignore DNS from DHCP: once the router hands out Pi-hole as the network's DNS, the Pi would otherwise pick up its own address and quietly create that loop.

I use Quad9: it validates DNSSEC and blocks known malware domains, which matters because the host deliberately bypasses Pi-hole's own blocklists. It also doesn't send your network's address to every website's nameserver (EDNS Client Subnet), and its addresses never change, so this never needs touching when you change routers.

First, make sure your chosen address is free:

```bash
ping -c 3 192.168.1.12
```

You want `Destination Host Unreachable` or `100% packet loss`. Some devices ignore pings, so also check the router's device list.

Find your connection name (`nmcli connection show`; on the Pi it's usually `Wired connection 1`), then save the settings. This only changes the saved profile, not the live network:

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.12/24 \
  ipv4.gateway 192.168.1.254 \
  ipv4.dns "9.9.9.9,149.112.112.112" \
  ipv4.ignore-auto-dns yes \
  ipv6.dns "2620:fe::fe" \
  ipv6.ignore-auto-dns yes
```

- `ipv4.method manual` is the important one: no DHCP at all. If you keep `auto` and add a manual address on top, the machine ends up with two IPv4 addresses, the static one and a DHCP one. That causes confusing problems later, and it happened to my main server.
- `ipv4.gateway` has to be set by hand now that DHCP isn't supplying it.
- Three DNS servers: Quad9's primary, its backup on a separate IP range, and its IPv6 address as a last resort. Linux uses at most three `nameserver` lines and silently ignores the rest. An IPv4 DNS server still answers IPv6 (AAAA) lookups, so the IPv6 entry is only there for redundancy.
- IPv6 addressing stays automatic. Only IPv6 DNS is overridden.

Check what was saved:

```bash
nmcli -g ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns,ipv4.ignore-auto-dns,ipv6.dns,ipv6.ignore-auto-dns connection show "Wired connection 1"
```

(The backslashes in the IPv6 address are just display escaping. `:` is the field separator in this output mode.)

Then apply it. **Your SSH session will freeze**, because the Pi is moving to a new address mid-conversation. NetworkManager finishes the change regardless. To close a frozen session, press Enter, then type `~.`.

```bash
sudo nmcli connection up "Wired connection 1"
```

Reconnect at the new address. If it doesn't come back after a minute: ping the new address from your PC, and as a last resort plug in a monitor (micro-HDMI) and keyboard and log in with your password.

Verify:

```bash
ip -4 addr show eth0          # exactly one inet line: 192.168.1.12/24
cat /etc/resolv.conf          # only the three Quad9 servers
ping -c 3 9.9.9.9             # the route out works (no DNS involved)
getent hosts example.com      # name lookups work, through the same path every program uses
```

Testing the route and DNS separately is a classic troubleshooting split: if the ping works but the lookup fails, the problem is DNS, not the network.

## 7. Update everything

The image is months old, and security fixes have come out since. Patch the base before building anything on it.

```bash
sudo apt update
apt list --upgradable
sudo apt full-upgrade
sudo reboot
```

- `apt update` only refreshes the list of available versions. It installs nothing.
- `full-upgrade` installs the updates and is allowed to add or remove packages when an update needs it, which plain `upgrade` won't do. Read its summary before answering `Y`. If it wants to remove anything, stop and look first.
- The reboot loads the new kernel and firmware, and it doubles as a test that the static IP survives a reboot.

## 8. Hardening

Anything the box doesn't need should go. Software that isn't installed can't be misconfigured, switched on by accident, or have a security bug.

**Remove Raspberry Pi Connect.** Turning it off in Imager stops it running, but the software is still installed. `purge` removes it and its config:

```bash
sudo apt purge rpi-connect-lite
dpkg -l | grep rpi-connect      # no output = gone
```

Check the removal list before saying yes. It should only be that one package.

**Switch off the Wi-Fi and Bluetooth radios.** A radio that's on is a way onto the device that doesn't need a cable. `/boot/firmware/config.txt` is read by the firmware at power-on, before Linux starts, and it decides what hardware gets switched on.

```bash
sudo cp /boot/firmware/config.txt /boot/firmware/config.txt.bak
sudo nano /boot/firmware/config.txt
```

Add this at the very end:

```
[all]
dtoverlay=disable-wifi
dtoverlay=disable-bt
```

The `[all]` header matters. Lines under a header like `[cm5]` only apply to that board model, so this guarantees the lines apply to yours. `dtoverlay` edits the device tree, the hardware list Linux uses to know what's on the board. Reboot, then check:

```bash
ip link                       # lo and eth0 only, no wlan0
ls /sys/class/bluetooth       # "No such file or directory" is the pass
```

Nothing is deleted. The chips are just never switched on, and `/sys` is generated live by the kernel, so the Bluetooth folder only exists when there's Bluetooth hardware running. Remove those three lines and reboot to undo it. `config.txt` sits on a partition Windows can read, so if an edit ever stops the Pi booting, you can fix the file from your PC.

**Automatic security updates.** You won't remember to SSH in every week. `unattended-upgrades` checks daily and installs updates with nobody logged in.

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades   # answer Yes
cat /etc/apt/apt.conf.d/20auto-upgrades           # both lines should end in "1"
```

I leave it exactly as Debian ships it:

- **It only installs from Debian's sources:** security fixes and Debian's tested point releases. Raspberry Pi's own repository (kernel, firmware, bootloader) isn't on its list. I considered adding it and decided against it. That repository mixes security fixes with everything else, so there's no "security only" option, and the kernel and firmware don't take effect until a reboot anyway, which I do by hand. Debian's OpenSSL security fixes still arrive automatically, because apt installs whichever version is highest.
- **It never reboots on its own.** A surprise reboot of the house's DNS server is exactly what I don't want.

Rebuilding this is just those two commands. Nothing custom to remember.

## 9. Docker

Where Docker comes from matters:

| Source | Verdict |
|---|---|
| Debian's `docker.io` package | No. Usually older, and it's on unattended-upgrades' list, so Docker would update itself, and that restarts every container (including Pi-hole) at a random time of day. |
| Docker's convenience script (`curl ... \| sh`) | No. It runs a script from the internet as root without you seeing what it does. Docker's own docs say not to use it on production machines. |
| **Docker's official apt repository** | Yes. Current versions, packages signed by Docker, and not on the auto-update list, so Docker updates happen when I choose. |

Raspberry Pi OS 64-bit uses Docker's Debian instructions. First make sure nothing conflicting is installed:

```bash
dpkg -l | grep -E 'docker|containerd|runc|podman'    # expect no output
```

Add Docker's signing key and repository (straight from Docker's docs):

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

What these do:

- `install -m 0755 -d` creates the keyrings folder with sensible permissions.
- `curl -fsSL`: `-f` fails on an HTTP error instead of saving an error page as the "key", `-s` hides the progress bar, `-S` still shows errors, `-L` follows redirects.
- `chmod a+r` makes the key readable by everyone. apt checks signatures as a low-privilege user.
- `sudo tee` writes the file. `sudo echo ... > file` wouldn't work, because your own shell opens the file, not sudo.
- `$(...)` fills in your OS codename (`trixie`) and CPU architecture (`arm64`) automatically.
- **`Signed-By`** is the line that matters: apt only accepts packages from this source if Docker's key signed them, and that key can't vouch for anything else.

The final `apt update` should show `download.docker.com` with no signature errors. Then install and verify:

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker run --rm hello-world     # "Hello from Docker!" and (arm64v8)
docker compose version
systemctl is-enabled docker          # must say enabled, so it starts at boot
```

**I don't add my user to the `docker` group.** Being in that group is the same as being root without a password: anyone who can run `docker` can mount the whole filesystem into a container. My setup has two layers, the SSH key to get in and the `sudo` password for admin powers, and the `docker` group quietly removes the second one. So Docker always runs with `sudo` on this box. `sudo` remembers the password for about 15 minutes, so it's one prompt per session.

## 10. Git and the project folder

```bash
sudo apt install git
git config --global user.useConfigOnly true
sudo mkdir -p /srv/network-stack
sudo chown youruser:youruser /srv/network-stack
```

- `user.useConfigOnly true` stops Git from guessing an identity. Without it, a commit from a machine with no Git identity set gets stamped with `user@hostname.your-isp-domain`, which leaks your username, hostname and ISP into a public history. It happened to me once. With this set, Git refuses to commit instead.
- `/srv` is the Linux convention for data a machine serves. Using the same path on every box keeps commands and docs identical.
- `chown` hands the folder to your user so Git works without `sudo`. Only Docker needs `sudo`.

## 11. Deploy Pi-hole and Unbound

Clone over HTTPS. The repo is public, so no credentials are needed, and the Pi never holds anything that can write to GitHub. Changes get made and pushed from another machine, then pulled here.

```bash
cd /srv/network-stack
git clone https://github.com/JCBRobeson/pihole-unbound-docker.git .
ls -la
```

The `.` at the end clones into the current (empty) folder instead of creating a subfolder. Note what isn't there: no `.env` and no `pihole/` data folder. Passwords and query history never leave the machine they belong to.

From here, follow that repo's README. In short:

```bash
cp .env.example .env
chmod 600 .env
openssl rand -base64 24       # generate a password; save it in your password manager
nano .env
```

- Give each Pi-hole its **own** admin password, so one leak doesn't expose the other.
- **Avoid `$` in the password.** Docker Compose reads `$` in `.env` as the start of a variable and silently mangles it. `openssl rand -base64` output never contains `$`.

Then check, without printing the whole config (it contains the password):

```bash
ls -l .env                                              # -rw------- 
sudo docker compose config | grep -E 'revServers|TZ'    # variables filled in, no ",,"
```

Pre-flight checks before starting anything:

```bash
sudo ss -tulpn | grep ':53 '        # no output: port 53 is free
ls -l unbound/custom.conf.d/        # must be readable by "others" (last r)
sudo docker compose pull            # both images pull for arm64
```

The pull is where you'd find out if an image had no ARM build: `no matching manifest for linux/arm64`. Both of these images support it.

Launch:

```bash
sudo docker compose up -d
sudo docker compose ps     # both "(healthy)" after a minute or so
```

Pi-hole waits for Unbound to pass its health check first. Never forward port 53 from your router to this box; it would become an open DNS server for the whole internet. Port 53 is also published on IPv6, so it relies on your router's IPv6 firewall blocking inbound traffic, the same as any other machine on the network.

## 12. Test it

`dig` isn't included in Lite:

```bash
sudo apt install bind9-dnsutils
```

Then run the test suite from the pihole-unbound-docker README:

```bash
dig @127.0.0.1 example.com                         # NOERROR; second run ~1 ms (cached)
dig @127.0.0.1 ad.doubleclick.net +short           # 0.0.0.0 (blocked)
dig @127.0.0.1 dnssec-failed.org | grep status     # SERVFAIL (forged signatures rejected)
dig @127.0.0.1 whoami.akamai.net +short            # your own public IP (true recursion)
sudo docker exec unbound drill -D @127.0.0.1 example.com | grep flags   # includes "ad"
sudo docker exec unbound unbound-checkconf -o do-ip6                    # no
sudo docker exec pihole pihole-FTL --config dns.revServers
sudo docker exec pihole pihole-FTL --config database.maxDBdays          # 14
```

Then the test that matters most, from a **different machine** on the network, because that's how your devices will reach it:

```bash
dig @192.168.1.12 example.com +short
dig @192.168.1.12 dnssec-failed.org | grep status
```

Reverse lookups (IP to name) only return names for devices that got their address from the router's DHCP. Servers with static addresses come back blank, because the router never learned their names. Test with a phone or laptop's address instead. Pi-hole's Local DNS records can name the static servers.

## 13. Blocklists

A fresh Pi-hole only has the default list. Blocklists live in Pi-hole's database, not in the repo, so add them in the dashboard (`http://192.168.1.12:8053/admin`) under **Lists**, as **blocklists** (not allowlists, which would do the opposite):

| List | URL |
|---|---|
| HaGeZi Multi Pro++ | `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/pro.plus.txt` |
| HaGeZi Threat Intelligence Feeds | `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/tif.txt` |

Then rebuild:

```bash
sudo docker exec pihole pihole -g
```

The first line should be `[✓] DNS resolution is available` with no `✗` before it. That confirms the host DNS from step 6 is right. My main server showed a `✗` first for weeks because of a typo in its DNS settings. You should end up with around 2.6 million unique domains.

I added the lists by hand instead of importing a Teleporter backup from the other Pi-hole. A Teleporter export contains your whole Pi-hole setup, likely including a hash of the admin password, so that's one fewer sensitive file to handle and clean up. If you do use Teleporter, delete the export afterwards and never commit it.

The dashboard is plain HTTP, so the password crosses your LAN unencrypted. That's a common tradeoff at home. A reverse proxy with HTTPS (Caddy) is on the roadmap.

## 14. Point devices at it, and test failover

Test the new Pi-hole on its own first. If you give a device both servers straight away and the new one has a problem, the device quietly uses the other one and you never find out.

1. On one device, set DNS to the Pi only (on an iPhone: Settings → Wi-Fi → ⓘ → Configure DNS → Manual).
2. Browse, then check the Pi's Query Log: the device should show up by name, with some queries blocked.
3. Run the **Extended test** at dnsleaktest.com. The only IP listed should be your home's public IP.

Then add the backup Pi-hole as the second server. Both run the same lists, so it doesn't matter which one answers. **The backup has to be a Pi-hole too.** Phones and computers send queries to either server, so a public DNS as the "backup" would let ads through all the time.

**Then prove the failover works:**

```bash
sudo docker compose stop pihole     # on the Pi
```

Browse to sites you haven't visited today, check that the backup Pi-hole's log picks up the device, then:

```bash
sudo docker compose start pihole
```

What I saw: during the switch, one site briefly showed "you're using an ad blocker." Its scripts asked the stopped Pi-hole for addresses, got no answer, and failed while the phone was still deciding that server was dead, and the site's ad-blocker check read failed scripts as blocked ones. A refresh fixed it. Devices do fail over, but not instantly. Smoothing that out is what a floating IP is for (see the roadmap).

## 15. Keep the backup in sync (nebula-sync)

Two Pi-holes don't share anything on their own. Add an allow entry on one and the other still blocks that site, so your devices get different answers depending on which server they happen to ask. [nebula-sync](https://github.com/lovelaze/nebula-sync) fixes that by copying the lists from this Pi to the backup every night.

It runs here, on the Pi, in its own folder. The Pi is the source of truth, and if the Pi is dead there's nothing to sync from anyway. It's not part of the pihole-unbound-docker repo because most people running that have one Pi-hole and don't need it.

**It's one-way.** Make every change on this Pi. Each sync replaces the backup's lists, so anything added only on the backup disappears. Before the first sync, open **Domains** on both dashboards and copy anything that exists only on the backup over to the Pi.

### App passwords

nebula-sync has to log into both Pi-holes. Instead of giving it your real logins, make each Pi-hole an app password: **Settings → Web interface / API**, switch from **Basic** to **Expert**, then **Configure app password**.

- It's shown once. Put it straight into a password manager, named so you know what it's for (`<box> – Pi-hole app password (nebula-sync)`).
- Don't paste it into chats, notes or commands. A password typed into a command ends up in your shell history.
- If it ever leaks, generate a new one. That kills the old one, and your real login doesn't change.

It doesn't make a leak less bad, since an app password can do everything your login can through the API. The point is separation: the robot gets its own credential, which you can revoke on its own.

By default an app password can read everything but change nothing. The backup gets written to, so it needs app passwords allowed to make changes. In the pihole-unbound-docker repo that's one variable, set in the **backup's** `.env` only:

```
PIHOLE_APP_SUDO=true
```

Then `sudo docker compose up -d` on the backup and confirm with `sudo docker exec pihole pihole-FTL --config webserver.api.app_sudo` (`true`). Leave it unset on this Pi: it defaults to `false`, and this Pi only ever gets read. I skipped this the first time and the sync failed with a `403` (forbidden) every run.

This also means the backup's app password lives on this Pi. That's the price of a sync tool, and the Pi is the more locked-down of the two machines.

### The folder and `.env`

```bash
sudo mkdir /srv/nebula-sync
sudo chown youruser:youruser /srv/nebula-sync
cd /srv/nebula-sync
touch .env
chmod 600 .env
nano .env
```

`touch` and `chmod 600` come **before** `nano`, so the file is private before any password goes into it.

```
PRIMARY_URL=http://192.168.1.12:8053
PRIMARY_APP_PASSWORD=
REPLICA_URL=http://192.168.1.11:8053
REPLICA_APP_PASSWORD=
TZ=America/Chicago
```

The URLs are the dashboard address without `/admin`; nebula-sync talks to the API underneath it. Fill in each app password after its `=`, with no spaces and no quotes. My backup is `192.168.1.11`; use your own.

Check it without printing the passwords:

```bash
ls -l .env                          # -rw-------
grep -c '_APP_PASSWORD=.\+' .env    # 2, meaning both are filled in
cut -d= -f1 .env                    # just the variable names
```

### The compose file

```yaml
services:
  nebula-sync:
    image: ghcr.io/lovelaze/nebula-sync:v0.11.2
    container_name: nebula-sync
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    logging:
      driver: json-file
      options:
        max-size: '10m'
        max-file: '3'
    environment:
      PRIMARY: '${PRIMARY_URL}|${PRIMARY_APP_PASSWORD}'
      REPLICAS: '${REPLICA_URL}|${REPLICA_APP_PASSWORD}'
      TZ: '${TZ}'
      CRON: '0 2 * * *'
      FULL_SYNC: 'false'
      RUN_GRAVITY: 'true'
      # Lists, allow/deny entries, groups, clients: primary -> backup
      SYNC_GRAVITY_GROUP: 'true'
      SYNC_GRAVITY_AD_LIST: 'true'
      SYNC_GRAVITY_AD_LIST_BY_GROUP: 'true'
      SYNC_GRAVITY_DOMAIN_LIST: 'true'
      SYNC_GRAVITY_DOMAIN_LIST_BY_GROUP: 'true'
      SYNC_GRAVITY_CLIENT: 'true'
      SYNC_GRAVITY_CLIENT_BY_GROUP: 'true'
      # Settings stay owned by each box's compose file (SYNC_CONFIG_* default false).
      # Enable once Local DNS records exist, to sync ONLY those records:
      # SYNC_CONFIG_DNS: 'true'
      # SYNC_CONFIG_DNS_INCLUDE: 'hosts,cnameRecords'
```

| Setting | Why |
|---|---|
| Pinned version (`v0.11.2`), not `:latest` | Same image on every rebuild. Updates happen when I change this line, after reading the release notes. |
| `restart: unless-stopped` | Comes back after reboots and crashes, stays down if I stop it on purpose. |
| `no-new-privileges` | Nothing inside the container can gain more privileges than it started with. nebula-sync never needs to, so it's free protection. |
| Log limits | Docker's default log has no size cap. On an SD card, an ever-growing log is wasted wear and eventually a full disk. |
| No `ports:` | nebula-sync only connects out to the two Pi-holes. Nothing can connect to it. |
| Passwords glued together from `.env` | nebula-sync wants `url\|password`. Keeping the pieces separate in `.env` keeps every site-specific value and secret out of the compose file. |
| `CRON: '0 2 * * *'` | Minute, hour, day, month, weekday: 2:00 AM every day. The backup being up to a day behind is fine. Daylight saving skips 2 AM one night each March, so that night's sync may run late or not at all, which is harmless. |
| `FULL_SYNC: 'false'` | Selective sync. A full sync clones every setting, including ones the compose file locks on each box. Each setting should have one owner: compose owns settings, nebula-sync owns lists. |
| `RUN_GRAVITY: 'true'` | The backup rebuilds its list database after a sync, so new lists actually take effect. It runs on the backup, not on this SD card. |
| `SYNC_GRAVITY_*` | Groups have to sync, because lists and devices are attached to groups. Then blocklists, allow/deny entries and device assignments, each with their group links. |
| Left off | DHCP leases (Pi-hole isn't the DHCP server) and every `SYNC_CONFIG_*` section. DHCP settings are the dangerous one: syncing them could switch DHCP on for two servers at once, and two DHCP servers on one network hand out conflicting addresses. |

The commented-out lines are for later. Pi-hole v6 keeps Local DNS records in its settings, not with the lists, so when I add names for my servers, that filter syncs those records and nothing else.

Validate without printing the passwords. The full `docker compose config` output includes them, so only grep it:

```bash
sudo docker compose config --quiet && echo OK
sudo docker compose config | grep -E 'image|CRON|FULL_SYNC|RUN_GRAVITY|TZ'
```

### Run it

```bash
sudo docker compose pull
sudo docker compose up -d
sudo docker logs -f nebula-sync     # Ctrl+C stops watching; the container keeps running
```

It syncs once on startup, then waits for the schedule. Success ends with `Sync completed`. A line starting with `FTL` is a fatal error, and with `restart: unless-stopped` it'll crash, restart and retry every few seconds. Stop it with `sudo docker compose stop` while you fix the problem.

Since it syncs on startup, `sudo docker compose restart` is the "sync now" button.

### Prove it works both ways

"Sync completed" is nebula-sync's opinion. This proves it:

1. On this Pi's dashboard, **Domains** → add `nebula-test.example.com` as an exact **deny** entry. `example.com` is reserved for testing, so it can't be a real site.
2. `sudo docker compose restart` here.
3. On the backup: `dig @127.0.0.1 nebula-test.example.com +short` → `0.0.0.0`. The backup is blocking something you only added on the Pi.
4. Delete the entry on the Pi, `restart` again, and rerun the `dig` on the backup → empty answer.

Step 4 matters as much as step 3. A sync that only adds would slowly fill the backup with entries you'd deleted.

The dashboards are plain HTTP, so every sync sends both app passwords across the LAN unencrypted. I'm accepting that on a home network until Caddy puts HTTPS in front of them.

## Maintenance

**Automatic, daily:** Debian security fixes and point releases. Never reboots.

**By hand, about monthly, at a quiet time:**

```bash
sudo apt update && sudo apt full-upgrade && sudo reboot
```

This picks up everything the automatic updates skip: the Pi's kernel and firmware, and Docker. A Docker update restarts every container, including Pi-hole, so do it when a short DNS blip won't bother anyone.

nebula-sync is pinned, so it never updates by itself. Every few months, check its [releases page](https://github.com/lovelaze/nebula-sync/releases), read the notes, change the version in the compose file, then `sudo docker compose pull && sudo docker compose up -d`.

Occasionally check power and heat with `vcgencmd get_throttled` and `vcgencmd measure_temp`.

When swapping to a new SD card, reflash and rebuild with this README instead of cloning the old card. It proves the setup really is reproducible. The Pi will have a new host key, so expect the SSH warning from step 4.

## A note on the clock

DNSSEC signatures have valid-from and valid-until dates, so a Pi with a badly wrong clock would reject everything. The Pi 3 and 4 have no real-time clock, and the Pi 5 has one but no battery by default. In practice this is covered:

- Raspberry Pi OS saves the time to disk regularly and restores it at boot, so after a power cut the clock is only slightly behind.
- The Pi's own DNS goes to Quad9 (step 6), which validates with its own correct clock, so the Pi can reach a time server and fix its clock without depending on its own Unbound.
- The second Pi-hole answers during the few seconds after boot.

The RTC battery is a nice extra, mainly for a Pi that's been unplugged for days. It isn't required.

## Security summary

- Key-only SSH; password and keyboard-interactive login off; root can't log in with a password
- `sudo` needs a password; no `docker` group membership
- No Wi-Fi or Bluetooth, at the firmware level
- Raspberry Pi Connect removed
- Daily automatic security updates, manual reboots
- Docker from its signed official repository
- Static address and a public validating resolver for the host's own DNS, independent of the router and of Pi-hole
- Secrets in `.env` (mode 600), never committed; a unique admin password per Pi-hole
- nebula-sync uses revocable app passwords, not the real logins; app passwords can only change settings on the backup, and stay read-only on this Pi
- nebula-sync publishes no ports, can't gain privileges, and runs a pinned version
- Git can't stamp a guessed identity onto commits
- Nothing forwarded from the router

## Roadmap

- Floating IPs with keepalived: one shared address for DNS, so devices only know one server and failover takes seconds. Separate floating IPs per service (DNS, and Caddy later), each with its own health check, so each one fails over independently.
- Local DNS records for the statically addressed servers
- Uptime Kuma, to monitor the other servers from outside them
- A Tailscale node, as a second way in when the main server is down
- A UPS with NUT, so everything shuts down cleanly on low battery, with the Pi as the NUT server since it outlasts everything else
- High-endurance SD card, or NVMe
- Caddy for HTTPS dashboards
- Turn this README into an Ansible playbook