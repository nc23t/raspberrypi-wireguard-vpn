# Raspberry Pi WireGuard VPN

This repository documents the setup of a self-hosted WireGuard VPN running on a Raspberry Pi 4, along with DNS-level ad blocking via Pi-hole and multi-client support.

The goal was to build a lightweight, secure VPN server that lets me tunnel traffic through the Raspberry Pi and access the internet securely from any of my devices, while getting hands-on experience with modern VPN cryptography, Linux routing, NAT, and DNS filtering.

---

## Why WireGuard

WireGuard was chosen over OpenVPN or IPsec for a few specific reasons:

- **Small attack surface.** The WireGuard codebase is roughly 4,000 lines of code, compared to hundreds of thousands for OpenVPN. Less code means fewer bugs and easier auditing.
- **Modern, fixed cryptography.** WireGuard uses Curve25519 for key exchange, ChaCha20-Poly1305 for authenticated encryption, BLAKE2s for hashing, and SipHash24 for hashtable keys. There is no cipher negotiation, which eliminates downgrade attacks.
- **Kernel-native performance.** On Linux, WireGuard runs as a kernel module. That gives it lower overhead and higher throughput than user-space VPNs.
- **Stateless, key-based identity.** WireGuard uses "cryptokey routing" where each peer is identified by its public key rather than a session, username, or password. There is no login state to compromise.

## Threat Model

The VPN is designed to protect against:

- Passive eavesdropping on untrusted networks (public Wi-Fi, cellular carriers).
- DNS-based tracking and ad networks profiling my devices.
- ISP-level traffic inspection when connecting from a remote network.

It is not designed to defend against:

- Endpoint compromise on any client or the server itself.
- Active nation-state adversaries with the ability to compromise the Pi's supply chain or firmware.

Being explicit about scope matters because a self-hosted VPN is not a substitute for network segmentation, endpoint hardening, or a real IDS.

---

## Environment

- Raspberry Pi 4 Model B (4GB RAM)
- Raspberry Pi OS Lite (64-bit), Debian Bookworm base
- WireGuard (`wireguard-tools` package)
- Pi-hole for DNS-level ad blocking
- `iptables` for NAT and packet forwarding
- Clients: iPhone (WireGuard iOS app) and MacBook (WireGuard desktop app)

---

## Network Design

| Element | Value | Purpose |
|---|---|---|
| VPN subnet | `10.86.12.0/24` | Private tunnel network, RFC 1918 space |
| Server tunnel IP | `10.86.12.1` | WireGuard endpoint on the Pi |
| Client 1 (MacBook) | `10.86.12.2` | Static peer assignment |
| Client 2 (iPhone) | `10.86.12.3` | Static peer assignment |
| WireGuard listen port | UDP `51820` | Default WireGuard port |
| Pi-hole DNS | `10.86.12.1:53` | Bound only to the `wg0` interface |

The `/24` subnet gives room for 254 usable clients, which is more than I will ever need for personal use, and choosing `10.86.12.0/24` avoids collisions with the default LAN ranges (`192.168.1.0/24`, `10.0.0.0/24`) I have seen on other networks I connect from.

The Raspberry Pi serves three roles simultaneously:

1. **VPN endpoint** that terminates encrypted tunnels from clients.
2. **NAT router** that rewrites source addresses so client traffic can egress to the public internet.
3. **DNS resolver** where Pi-hole filters ads and trackers before forwarding to an upstream resolver.

---

## Phase 1: Base VPN Setup

### 1. System update and upgrade

Before installing WireGuard, I updated the Pi to make sure all packages and kernel components were current. This matters because WireGuard runs as a kernel module on modern Debian, and mismatched kernel headers or stale packages can cause the module to fail to load.

```bash
sudo apt update && sudo apt upgrade -y
```

Breakdown:

- `sudo` runs the command as root, required to modify system packages.
- `apt update` refreshes the local package index from the configured repositories in `/etc/apt/sources.list` and `/etc/apt/sources.list.d/`. It does not install anything, it just checks what versions are available.
- `&&` runs the next command only if the first one exits successfully (exit code 0). If `apt update` fails, `apt upgrade` will not run.
- `apt upgrade` installs the newer versions of already-installed packages.
- `-y` auto-confirms the "Do you want to continue?" prompt.

After the upgrade finished I rebooted with `sudo reboot` to pick up any kernel updates.

### 2. Installing WireGuard

```bash
sudo apt install wireguard -y
```

- `apt install` fetches the specified package and any dependencies from the repositories.
- `wireguard` is the meta-package that pulls in `wireguard-tools` (the userspace CLI including `wg` and `wg-quick`) and, on modern Debian, ensures the kernel module is available.
- `-y` auto-confirms.

Verify the tools were available:

```bash
wg --version
```

- `wg` is the primary WireGuard CLI tool.
- `--version` prints the installed version and exits. Useful as a quick smoke test before you start configuring.

### 3. Installing iptables

```bash
sudo apt install iptables -y
```

`iptables` is the userspace tool that talks to the Linux kernel's `netfilter` framework to configure packet filtering, NAT, and forwarding rules. Without it, packets arriving on the WireGuard interface (`wg0`) cannot be translated onto the physical network interface (`eth0` or `wlan0`) and out to the internet.

Modern Debian ships with `nftables` as the default backend, but the `iptables` command still works via the `iptables-nft` translation layer, so the classic rules used by WireGuard's `PostUp` hooks continue to function.

### 4. Generating server keys

WireGuard uses public-key cryptography (Curve25519) instead of shared secrets or usernames and passwords. Each peer, server or client, has a private key that never leaves the device and a public key that gets shared with everyone it wants to talk to.

I generated the server keypair like this:

```bash
umask 077
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
```

Breakdown:

- `umask 077` sets the file creation mask so any files created by the following commands have permissions `600` (read/write for the owner only). This prevents the private key from being world-readable at the moment of creation.
- `wg genkey` generates a new Curve25519 private key and prints it to stdout as base64.
- `|` pipes stdout into the next command.
- `tee /etc/wireguard/server_private.key` writes stdin to the specified file AND passes it through to stdout, so we can save the private key and simultaneously use it to derive the public key.
- `wg pubkey` reads a private key from stdin and prints the corresponding public key to stdout.
- `> /etc/wireguard/server_public.key` redirects stdout to that file, overwriting it if it exists.

End result: two files under `/etc/wireguard/`, one holding the private key (locked to owner-only) and one holding the public key (shareable).

Sanity check that the private key is not world-readable:

```bash
sudo ls -l /etc/wireguard/server_private.key
```

Expected output should show `-rw-------` at the start of the line.

### 5. Generating client keys

Each client device gets its own unique keypair. Reusing a keypair across devices is a bad habit because if one device is compromised, you want the ability to revoke just that peer instead of everything.

For each client I ran:

```bash
umask 077
wg genkey | tee /etc/wireguard/macbook_private.key | wg pubkey > /etc/wireguard/macbook_public.key
```

Same breakdown as the server keys, just with different filenames. I generated separate pairs for `macbook` and `iphone`.

The client's **private key** eventually goes into the client's WireGuard config file. The client's **public key** goes into the server's config as an authorized peer.

### 6. Enabling IP forwarding

By default, the Linux kernel drops packets that arrive on one interface and are destined for another. That is a good security default for a workstation, but for a router or VPN gateway it has to be turned off.

```bash
sudo nano /etc/sysctl.conf
```

- `sudo` because `/etc/sysctl.conf` is root-owned.
- `nano` is a simple terminal text editor.
- `/etc/sysctl.conf` is the file where persistent kernel parameters are set at boot.

Inside the file, I uncommented (removed the leading `#` from) this line:

```
net.ipv4.ip_forward=1
```

Then applied it without rebooting:

```bash
sudo sysctl -p
```

- `sysctl` reads and modifies kernel parameters at runtime.
- `-p` (short for `--load`) reads settings from `/etc/sysctl.conf` and applies them immediately.

Verify:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

Expected output: `1`.

- `cat` prints file contents to stdout.
- `/proc/sys/net/ipv4/ip_forward` is the kernel's runtime interface for this setting. `0` means forwarding is off, `1` means on. Writing directly here would also work at runtime but would not persist across a reboot, which is why we edit `sysctl.conf`.

### 7. WireGuard interface configuration

I created the server's configuration file:

```bash
sudo nano /etc/wireguard/wg0.conf
```

The filename `wg0` determines the network interface name. If it were `wg1.conf`, the interface would be `wg1`.

Sanitized contents:

```ini
[Interface]
Address = 10.86.12.1/24
ListenPort = 51820
PrivateKey = <server-private-key>
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# MacBook
PublicKey = <macbook-public-key>
AllowedIPs = 10.86.12.2/32

[Peer]
# iPhone
PublicKey = <iphone-public-key>
AllowedIPs = 10.86.12.3/32
```

Line by line on the `[Interface]` block:

- `Address = 10.86.12.1/24` assigns the Pi the address `10.86.12.1` inside a `/24` subnet.
- `ListenPort = 51820` is the UDP port WireGuard listens on. This is also the port forwarded from my home router to the Pi.
- `PrivateKey` is the server's private key (from `/etc/wireguard/server_private.key`).
- `PostUp` runs after the interface comes up:
  - `iptables -A FORWARD -i wg0 -j ACCEPT` appends (`-A`) a rule to the `FORWARD` chain that accepts (`-j ACCEPT`) any packet arriving on input interface (`-i`) `wg0`.
  - `;` separates two shell commands.
  - `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE` operates on the `nat` table (`-t nat`), appends to `POSTROUTING`, matches packets leaving on output interface (`-o`) `eth0`, and applies `MASQUERADE`, which rewrites the source address to the Pi's LAN IP so return traffic finds its way back.
- `PostDown` runs when the interface is torn down. It uses `-D` (delete) instead of `-A` to remove the rules cleanly so they do not stack up if the interface is bounced.

If your Pi is connected via Wi-Fi rather than Ethernet, replace `eth0` with `wlan0` in the `PostUp` and `PostDown` lines.

On the `[Peer]` blocks:

- `PublicKey` is the client's public key, used to authenticate the client.
- `AllowedIPs = 10.86.12.2/32` is a `/32` (single host) address. On the server side, `AllowedIPs` acts as a routing rule: any packet destined for this address gets encrypted and sent to this peer, and any packet arriving from this peer must have a source address in this range or WireGuard drops it. This is what "cryptokey routing" means in practice, and it is why WireGuard does not need a separate firewall to prevent peer address spoofing.

Bring the interface up and enable it at boot:

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

- `wg-quick up wg0` reads `/etc/wireguard/wg0.conf`, creates the interface, applies the configuration, runs `PostUp` commands, and adds routes.
- `systemctl enable wg-quick@wg0` tells `systemd` to start the `wg-quick@wg0` service at every boot. The `@wg0` part is a systemd template instance: the value after `@` gets passed to the service definition as `%i`, which `wg-quick` uses as the interface name.

Verify it is running:

```bash
sudo wg show
```

- `wg show` prints the current status of all WireGuard interfaces, listing each peer, the last handshake time, and bytes transferred in each direction. If a peer has connected recently, its endpoint and handshake timestamp will appear here.

### 8. Assigning client IPs and peers

Each client I added required two edits: appending a `[Peer]` block to `/etc/wireguard/wg0.conf` on the server (shown above), and building a client-side config (shown in step 9).

After editing the server config, I reloaded the peer list without dropping existing tunnels:

```bash
sudo wg syncconf wg0 <(wg-quick strip wg0)
```

- `wg syncconf wg0 <file>` re-reads the peer list from a config file without tearing down the interface. Bouncing the interface would drop any live connections.
- `wg-quick strip wg0` reads `/etc/wireguard/wg0.conf` and prints just the parts that `wg` understands, stripping out `PostUp`, `PostDown`, and other `wg-quick`-only directives.
- `<(...)` is bash process substitution, treating the output of the command as a file that `wg syncconf` can read.

### 9. Client tunnel configuration

Each client device gets a config file that mirrors the server's view of the tunnel:

```ini
[Interface]
PrivateKey = <client-private-key>
Address = 10.86.12.X/32
DNS = 10.86.12.1

[Peer]
PublicKey = <server-public-key>
AllowedIPs = 0.0.0.0/0
Endpoint = <server-public-ip>:51820
PersistentKeepalive = 25
```

Key points:

- `Address` uses `/32` so the client claims only its own tunnel IP, not the whole subnet.
- `DNS = 10.86.12.1` forces DNS queries through Pi-hole once the tunnel is up. On mobile clients this also sets the system resolver while the VPN is active.
- `AllowedIPs = 0.0.0.0/0` on the client side has a different meaning than on the server. Here it means "route all traffic through the tunnel." A "split-tunnel" configuration would list only specific subnets, so only that traffic goes over the VPN.
- `Endpoint` is the server's public IP and port. This is the only field that needs to point to something reachable from the outside internet.
- `PersistentKeepalive = 25` sends an empty keepalive packet every 25 seconds. Without this, NAT tables on intermediate routers can time out and the tunnel goes silent from the server's perspective. 25 seconds sits well under the typical 30-to-60-second NAT idle timeout.

For mobile clients, the fastest way to load a config is to render a QR code on the Pi:

```bash
qrencode -t ansiutf8 < /etc/wireguard/iphone.conf
```

- `qrencode` renders text or file contents as a QR code.
- `-t ansiutf8` sets the output type to Unicode block characters in the terminal, so the QR code displays directly in an SSH session.
- `<` redirects the contents of the config file into `qrencode`'s stdin.

The WireGuard iOS app has a "Scan QR code" option that reads it directly from the terminal window. After scanning, immediately clear the terminal (`clear` or `Ctrl-L`) so the config does not sit in scrollback.

### 10. Tunnel established successfully

Once the client activates its tunnel, running `sudo wg show` on the Pi shows a recent handshake time and byte counters climbing. From the client, a quick check confirms traffic is actually flowing through the VPN:

```bash
curl -s https://ifconfig.me
```

- `curl` is a command-line HTTP client.
- `-s` (silent) suppresses the progress bar so only the response body prints.
- `ifconfig.me` returns the public IP that made the request, which should now match the Pi's home ISP address, not the client's local network.

At this point:

- Encrypted traffic flows through the tunnel using ChaCha20-Poly1305.
- NAT allows outbound internet access from the tunnel subnet.
- The VPN is fully operational.

---

## Phase 2: Pi-hole Integration

After the base VPN was running, I integrated Pi-hole to add DNS-level ad blocking for all devices connected through the tunnel.

### What Pi-hole does

Pi-hole acts as a DNS sinkhole. When a device makes a DNS request, say trying to resolve `ads.example.com`, Pi-hole checks the domain against its blocklists. If the domain matches, Pi-hole returns `0.0.0.0` (or `NXDOMAIN`, depending on config) instead of the real IP. The client's browser or app then fails to load anything from that host.

For domains not on a blocklist, Pi-hole forwards the query to an upstream resolver (Cloudflare `1.1.1.1` in my setup) and caches the response, so repeat lookups are near-instant.

### Installing Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

- `curl` fetches a URL.
- `-s` silent, `-S` show errors even in silent mode, `-L` follow redirects.
- `|` pipes the downloaded script into `bash` to execute it.

Piping a `curl`'d script into `bash` is convenient but not a habit I would recommend for higher-trust environments. For a personal lab it is fine. For anything I care about I download the script first, review it, then run it.

During install I chose:

- **Upstream DNS provider:** Cloudflare (`1.1.1.1` / `1.0.0.1`).
- **Listening interface:** `wg0` only.

The listening interface choice is important. If Pi-hole is bound to `eth0` or "all interfaces," and the Pi's WAN port ever gets exposed, it becomes an open resolver. That is both a DDoS amplification risk and a violation of most ISP terms of service.

To confirm Pi-hole is bound correctly:

```bash
sudo netstat -tulnp | grep pihole-FTL
```

- `netstat` prints network connection info.
- `-t` TCP, `-u` UDP, `-l` listening sockets only, `-n` show numeric addresses instead of resolving hostnames, `-p` show the process holding each socket.
- `|` pipes the output into the next command.
- `grep pihole-FTL` filters for Pi-hole's daemon.

The output should show sockets bound to `10.86.12.1:53`, not `0.0.0.0:53`.

### Client config change

I updated each client's WireGuard config to set `DNS = 10.86.12.1`. When the client connects, its system DNS is temporarily overridden to point at Pi-hole through the tunnel.

### Troubleshooting

The main issue during setup was DNS resolution failing when traffic flowed VPN → Pi-hole → upstream. Symptoms: tunnels came up cleanly, `ping 10.86.12.1` from the client worked, but browser page loads hung.

Root cause: Pi-hole's `dnsmasq` config was still bound to loopback only, not `wg0`. Fixed by editing `/etc/pihole/pihole-FTL.conf` and setting:

```
LOCAL_IPV4=10.86.12.1
```

Then restarting:

```bash
sudo systemctl restart pihole-FTL
```

- `systemctl restart pihole-FTL` stops and starts the `pihole-FTL` service in one step, forcing it to re-read config.

Test resolution from a client while connected to the VPN:

```bash
dig @10.86.12.1 example.com
```

- `dig` is the standard DNS lookup tool.
- `@10.86.12.1` specifies which DNS server to query, overriding the system default.
- `example.com` is the name to look up.

The `ANSWER SECTION` in the response should contain a valid A record. To confirm blocking is active:

```bash
dig @10.86.12.1 doubleclick.net
```

If Pi-hole is working, the answer will be `0.0.0.0` (or `NXDOMAIN`, depending on blocking mode).

---

## Phase 3: Multi-Client Support

The original setup had a single peer, my MacBook. I expanded it to support multiple clients connecting simultaneously.

### How it works

Each client device gets:

- Its own unique keypair.
- A dedicated IP address within the VPN subnet (`/32` on the client, listed in `AllowedIPs` on the server).
- Its own `[Peer]` block in the server's WireGuard configuration.
- Its own client config file with the server's public key and endpoint.

The server distinguishes peers by public key, not by IP or connection order. Two clients can connect from the same source IP (for example, both my phone and laptop tethered to the same cellular hotspot) and WireGuard still routes them correctly.

### Current clients

| Device  | Tunnel IP  |
| ------- | ---------- |
| MacBook | 10.86.12.2 |
| iPhone  | 10.86.12.3 |

### Adding a new client

1. Generate a new keypair for the client:
   ```bash
   umask 077
   wg genkey | tee /etc/wireguard/<name>_private.key | wg pubkey > /etc/wireguard/<name>_public.key
   ```
2. Add a new `[Peer]` block to `/etc/wireguard/wg0.conf` with the client's public key and assigned `AllowedIPs`.
3. Reload the server config without bouncing the interface:
   ```bash
   sudo wg syncconf wg0 <(wg-quick strip wg0)
   ```
4. Create a client config file with the server's public key, endpoint, and the client's private key.
5. Import the config into the client app (QR code for mobile, file import for desktop).

### Example server peer block (sanitized)

```ini
[Peer]
PublicKey = <client-public-key>
AllowedIPs = 10.86.12.X/32
```

### Example client config (sanitized)

```ini
[Interface]
PrivateKey = <client-private-key>
Address = 10.86.12.X/32
DNS = 10.86.12.1

[Peer]
PublicKey = <server-public-key>
AllowedIPs = 0.0.0.0/0
Endpoint = <server-public-ip>:51820
PersistentKeepalive = 25
```

---

## Hardening Notes

A few operational practices I applied that are not obvious from the config alone:

- **SSH is not exposed to the WAN.** The Pi is only administered from the LAN or through the VPN tunnel itself. Once the VPN is up, I SSH to `10.86.12.1`, never to the public IP.
- **Only UDP 51820 is forwarded from the router.** No other ports are exposed from the WAN to the Pi.
- **`fail2ban` monitors auth logs** and drops sources that repeatedly fail to authenticate.
- **`unattended-upgrades` is enabled** so security patches apply automatically.
- **Key rotation policy.** Any keypair that touches development gets rotated before the setup is documented publicly. Old keys are removed from every peer's config and the interface is reloaded with `wg syncconf`.
- **No secrets in the repo.** All example configs use placeholders. Screenshots that would have shown key material during generation were removed intentionally.

---

## Learning Outcomes

Through this project I gained hands-on experience with:

- Modern VPN cryptography (Curve25519, ChaCha20-Poly1305) and why algorithm agility can be a liability rather than a feature.
- WireGuard's cryptokey routing model and how it differs from traditional session-based VPN authentication.
- Linux networking fundamentals: interfaces, routing tables, netfilter, NAT, and packet forwarding.
- `iptables` rule construction and the difference between the `filter`, `nat`, and `mangle` tables.
- DNS-level filtering with Pi-hole and the operational risks of running an open resolver.
- Multi-peer VPN management and how public-key identity simplifies peer revocation.
- Troubleshooting layer 3 and layer 7 issues across a tunneled network.
