# Why I Switched to sing-box on iOS

## Background

I use a proxy on my iPhone daily in China — both for accessing international services and for normal domestic apps that need to coexist with the proxy. After months of frustration with other popular iOS proxy apps, I switched to the [sing-box](https://sing-box.sagernet.org/) app and every issue disappeared.

This is purely my personal experience. No offense to any other apps — they're all great projects. But for my specific use case (split routing in China), sing-box has been flawless.

> **Side note**: sing-box actually runs on *every* device in my setup — iPhone (client), Linux laptop (TUN client), Debian home server (TUN proxy router for LAN), and Tokyo VPS (proxy server). One unified platform, one config format, across all devices. That consistency is a huge advantage over juggling different software on each device.

## The Problems I Had with Other Apps

With other popular iOS proxy apps, I consistently ran into:

- **Randomly slow domestic apps** — Apps that should go direct (Alipay, Taobao, banking apps) would occasionally become sluggish or unresponsive for seconds
- **WeChat mini-programs not loading** — Mini-programs (小程序) would hang or fail to load, sometimes requiring multiple retries
- **DNS leaks and conflicts** — Domestic domains occasionally resolved through the foreign DNS, causing timeouts or wrong CDN routing
- **Inconsistent split routing** — Even with GeoIP/GeoSite rules, some traffic would go through the wrong path

I spent a lot of time tweaking DNS settings, adding domain rules, trying different routing configurations. Nothing fully fixed it — the issues would always come back randomly.

## Why sing-box Works Better (for me)

After switching to sing-box on iOS, all of the above issues vanished. Zero friction with domestic apps, WeChat mini-programs load instantly, and international services work perfectly.

I believe the key differences are:

- **Sniff + override destination** — sing-box can sniff the actual protocol/domain from traffic and override the destination. This means even if DNS returns a wrong IP, the routing decision is based on the real domain name.
- **Clean DNS architecture** — Separate DNS servers for domestic (AliDNS) and foreign (Cloudflare via proxy) with rule-based routing. No DNS leaks.
- **TUN mode done right** — The gVisor-based TUN stack handles all traffic transparently without the quirks I experienced in other apps.
- **GeoIP/GeoSite rule sets** — Binary rule sets from [SagerNet](https://github.com/SagerNet/sing-geoip) are comprehensive and actively maintained.

## My Configuration

The sing-box iOS app supports importing JSON configs directly. Here's my working config (sanitized):

```json
{
  "log": {
    "level": "info",
    "timestamp": true
  },
  "dns": {
    "servers": [
      {
        "tag": "cloudflare",
        "address": "tcp://1.1.1.1",
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
      {
        "outbound": "any",
        "server": "local"
      },
      {
        "rule_set": "geosite-cn",
        "server": "local"
      }
    ],
    "final": "cloudflare",
    "independent_cache": true
  },
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "tun0",
      "address": ["198.18.0.1/16"],
      "mtu": 1500,
      "auto_route": true,
      "stack": "gvisor",
      "sniff": true,
      "sniff_override_destination": true
    }
  ],
  "outbounds": [
    {
      "type": "urltest",
      "tag": "proxy",
      "outbounds": ["your-proxy-1", "your-proxy-2"],
      "url": "https://www.gstatic.com/generate_204",
      "interval": "3m",
      "tolerance": 50
    },
    {
      "type": "vless",
      "tag": "your-proxy-1",
      "server": "YOUR_SERVER",
      "server_port": 443,
      "uuid": "YOUR_UUID",
      "flow": "xtls-rprx-vision",
      "tls": {
        "enabled": true,
        "server_name": "www.microsoft.com",
        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        },
        "reality": {
          "enabled": true,
          "public_key": "YOUR_PUBLIC_KEY",
          "short_id": "YOUR_SHORT_ID"
        }
      }
    },
    {
      "type": "direct",
      "tag": "direct"
    }
  ],
  "route": {
    "rules": [
      {
        "protocol": "dns",
        "action": "hijack-dns"
      },
      {
        "ip_is_private": true,
        "outbound": "direct"
      },
      {
        "rule_set": "geoip-cn",
        "outbound": "direct"
      },
      {
        "rule_set": "geosite-cn",
        "outbound": "direct"
      }
    ],
    "rule_set": [
      {
        "tag": "geoip-cn",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geoip/rule-set/geoip-cn.srs",
        "download_detour": "proxy"
      },
      {
        "tag": "geosite-cn",
        "type": "remote",
        "format": "binary",
        "url": "https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set/geosite-cn.srs",
        "download_detour": "proxy"
      }
    ],
    "final": "proxy",
    "auto_detect_interface": true
  },
  "experimental": {
    "cache_file": {
      "enabled": true
    }
  }
}
```

### Key design decisions

- **DNS**: Domestic domains → AliDNS (119.29.29.29, direct). Everything else → Cloudflare (1.1.1.1, via proxy). `independent_cache: true` prevents cache pollution between the two.
- **Routing**: GeoIP-CN and GeoSite-CN → direct. Private IPs → direct. Everything else → proxy via `urltest` auto-selection.
- **Sniffing**: `sniff: true` + `sniff_override_destination: true` on the TUN inbound — this is critical for correct routing when DNS returns unexpected IPs.
- **urltest**: Automatically picks the lowest-latency proxy from your list. Add multiple outbounds for failover.

## How to Set Up

1. Install [sing-box](https://apps.apple.com/app/sing-box/id6451272673) from the App Store
2. Create a JSON config file with your proxy server details (use the template above)
3. In the app: Profiles → New Profile → import the JSON file
4. Enable the VPN toggle

That's it. No subscription services, no complex UI — just a clean JSON config and it works.

## Tips

- **Battery life** is comparable to other proxy apps — the gVisor TUN stack is efficient
- **On-Demand Rules** can be configured in iOS VPN settings to auto-connect on specific Wi-Fi networks
- **Updates**: The sing-box app on the App Store stays reasonably up to date with the core project
- **Multiple proxies**: Use `urltest` with multiple outbounds for automatic failover — if your primary VPS goes down, traffic switches to the backup seamlessly
