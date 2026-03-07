# sing-box Lessons Learned in China

Hard-won lessons from running sing-box across multiple devices in China — iPhone, Linux laptop, Debian home server, and Tokyo VPS. Each lesson cost me hours of debugging.

## DNS: Use DoH, Not Raw TCP

**Problem**: I originally used `tcp://1.1.1.1` for foreign DNS resolution. This worked fine through VPS proxies (VLESS, vmess) but silently broke when `urltest` rotated to my Cloudflare Workers edgetunnel backup. The edgetunnel uses WebSocket transport, which can't relay raw TCP DNS streams — queries returned `unexpected EOF`.

The worst part: this only triggered when urltest *happened* to pick edgetunnel as the fastest outbound. It appeared as a random overnight DNS failure that I couldn't reproduce on demand.

**Fix**: Use HTTPS (DoH) instead of TCP for foreign DNS:

```json
{
  "tag": "google",
  "address": "https://8.8.8.8/dns-query",
  "detour": "proxy"
}
```

DoH uses standard HTTPS, which works through *every* outbound type — VLESS, vmess, WebSocket, you name it.

**Why Google, not Cloudflare**: Cloudflare DoH at `1.1.1.1` has TLS certificate issues when routed through the edgetunnel — `x509: cannot validate certificate for 1.1.1.1 because it doesn't contain any IP SANs`. This is a Cloudflare Workers internal routing quirk. Google DoH at `8.8.8.8` works reliably through all outbound types.

**Critical**: The `detour: "proxy"` is essential. Without it, the DoH connection goes direct to `1.1.1.1:443`, which is blocked in China. I learned this the hard way when I accidentally stripped it during a config update — every device using that DNS server lost foreign DNS resolution.

## urltest: Enable `interrupt_exist_connections`

**Problem**: With default urltest settings, when the active outbound goes down, sing-box keeps routing all traffic through it until the next health check (default: every 3 minutes). During that window, everything times out.

I noticed this when my VLESS proxy had a brief outage — sing-box had vmess and edgetunnel available as backups, but stubbornly kept sending traffic to the dead VLESS connection for minutes.

**Fix**:

```json
{
  "type": "urltest",
  "tag": "proxy",
  "outbounds": ["vless-tokyo", "vmess-tokyo", "cf-edgetunnel"],
  "url": "https://www.gstatic.com/generate_204",
  "interval": "1m",
  "tolerance": 50,
  "interrupt_exist_connections": true
}
```

- `interval: "1m"` — check every minute instead of 3
- `interrupt_exist_connections: true` — when a better outbound is found, immediately kill stale connections and switch

## `strict_route` and `route_exclude_address`

**Problem**: With `strict_route: true` on Linux, sing-box sets up kernel policy routing that captures *all* outbound traffic — including traffic that sing-box itself needs to send directly (like connections to proxy servers).

This bit me twice:
1. **LAN connections blocked** — devices on my LAN couldn't reach the sing-box machine. The `ip_is_private` route rule only handles traffic *inside* sing-box, not kernel-level policy routing.
2. **Edgetunnel routing loop** — the outbound connection to Cloudflare Workers IPs was captured by TUN, creating a loop.

**Fix**: Add all IPs that must bypass TUN to `route_exclude_address`:

```json
{
  "type": "tun",
  "strict_route": true,
  "route_exclude_address": [
    "10.0.0.0/16",
    "YOUR_VPS_IP/32",
    "CLOUDFLARE_IP_1/32",
    "CLOUDFLARE_IP_2/32"
  ]
}
```

Find your Cloudflare IPs with `dig +short your-worker-domain.com`.

## TUN Breaks After Suspend/Resume (Linux)

**Problem**: On my Linux laptop, sing-box's TUN interface and routes go stale after suspend/resume. The service stays "active" but all traffic fails with `no route to internet`.

**Fix**: Create a systemd service that auto-restarts sing-box on wake:

```ini
# /etc/systemd/system/sing-box-resume.service
[Unit]
Description=Restart sing-box after suspend/resume
After=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 3
ExecStart=/usr/bin/systemctl restart sing-box.service

[Install]
WantedBy=suspend.target hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

```bash
sudo systemctl enable sing-box-resume
```

The 3-second delay lets the network interface come up before sing-box restarts.

## Proxy Changes Are High-Risk

When your entire internet access depends on the proxy, a bad config change means you lose everything — internet, SSH, Claude Code, the ability to fix the problem remotely.

**Safe migration sequence**:

1. Add the new protocol on a **separate port** alongside the working setup
2. Update one client to test the new port
3. Test thoroughly
4. Only then swap ports or remove the old config

Never take down the working proxy path before the new one is confirmed working.

## VLESS+REALITY: Common Pitfalls

- Server-side TLS config **must** include `server_name` matching the client's SNI — omitting it causes "reality verification failed"
- Client **must** have `utls` enabled with a browser fingerprint
- Use a separate port (e.g., 8443) alongside your existing proxy so you have a fallback during testing

## `interrupt_exist_connections` Kills SSH Sessions

**Problem**: With `interrupt_exist_connections: true` and low `tolerance` (e.g., 50ms), urltest frequently switches outbounds when latencies fluctuate. Each switch **kills all existing connections** — including long-running SSH sessions. I was seeing SSH drops every ~2 minutes on my iPhone.

**Fix**: For devices where you run SSH sessions, either remove `interrupt_exist_connections` or increase tolerance significantly:

```json
{
  "type": "urltest",
  "tolerance": 200
}
```

With `tolerance: 200`, urltest only switches when the latency difference exceeds 200ms — reducing unnecessary switches while still handling outbound failures. Without `interrupt_exist_connections`, existing connections stay on the old outbound and only new connections use the faster path.

**Tradeoff**: `interrupt_exist_connections` is great for fast failover when an outbound dies, but bad for connection stability. Choose based on your priority — fast failover (laptop/phone browsing) vs. session stability (SSH).

## iOS Hotspot Devices and sing-box

**Problem**: Devices connected to iPhone's personal hotspot can't access blocked sites, even though sing-box is running on the iPhone. The SFI app's TUN only captures the iPhone's own traffic — **forwarded hotspot traffic bypasses the tunnel entirely**. iOS's `includeAllNetworks` VPN setting doesn't help either.

**Fix**: Use sing-box's mixed proxy inbound instead of TUN for hotspot devices. The default config already includes a mixed inbound on port 7890. On each hotspot device:

1. Wi-Fi settings → iPhone hotspot network → Configure IP: Manual
2. Set IP, subnet, and router (iPhone's gateway IP)
3. HTTP Proxy → Manual → Server: iPhone's gateway IP, Port: `7890`

This routes the device's traffic directly to sing-box's proxy listener, bypassing the TUN limitation.

## `workers.dev` SNI Blocking

Chinese ISPs block the `workers.dev` domain at the SNI level — the TLS Client Hello is RST'd during the handshake. This affects all Cloudflare Workers on the default domain.

**Fix**: Set up a custom domain for your Worker (e.g., `edge.example.com`) through Cloudflare's Workers custom domains. Any domain on Cloudflare works — the SNI will show your custom domain instead of `workers.dev`.

## Automatic Upgrades on Linux

**Problem**: sing-box is installed from a third-party repo (`deb.sagernet.org`), which unattended-upgrades ignores by default. The package's postinst script also doesn't restart the service after upgrade.

**Fix** (two parts):

1. **Allow unattended-upgrades to update sing-box** — add the sagernet origin:

   Ubuntu (`Allowed-Origins`):
   ```
   "sagernet_apt_fury_io:*";
   ```

   Debian (`Origins-Pattern`):
   ```
   "origin=sagernet_apt_fury_io";
   ```

2. **Auto-restart after upgrade** — add a dpkg hook:

   ```bash
   # /etc/apt/apt.conf.d/99sing-box-restart
   DPkg::Post-Invoke {"if systemctl is-active --quiet sing-box; then systemctl restart sing-box; fi";};
   ```

Now sing-box upgrades and restarts are fully hands-off.

## Recommended Base Configuration

After all these lessons, here's the configuration pattern I've settled on:

```json
{
  "dns": {
    "servers": [
      {
        "tag": "google",
        "address": "https://8.8.8.8/dns-query",
        "detour": "proxy"
      },
      {
        "tag": "alidns-doh",
        "address": "https://dns.alidns.com/dns-query",
        "address_resolver": "local",
        "detour": "direct"
      },
      {
        "tag": "local",
        "address": "119.29.29.29",
        "detour": "direct"
      }
    ],
    "rules": [
      { "outbound": "any", "server": "local" },
      { "rule_set": "geosite-cn", "server": "local" }
    ],
    "final": "google",
    "independent_cache": true
  }
}
```

**Why this works**:
- Foreign domains → Google DoH **via proxy** (bypasses China DNS poisoning)
- Chinese domains → AliDNS DoH **direct** (fast, correct CDN routing)
- Proxy server domains → local UDP DNS (breaks circular dependency)
- `independent_cache` prevents cross-contamination between DNS servers
