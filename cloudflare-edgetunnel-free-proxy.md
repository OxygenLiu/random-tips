# Free Cloudflare Workers Proxy That Actually Performs

## TL;DR

I set up a Cloudflare Workers-based VLESS proxy as a "last resort" fallback behind my VPS. Expected it to be slow. Instead, it delivers 3.5 MB/s downloads and sometimes **wins** the latency test against my Tokyo VPS. It's free, fully automated, and requires zero server maintenance.

## What is it?

[Edgetunnel](https://github.com/cmliu/edgetunnel) is a [Cloudflare Worker](https://workers.cloudflare.com/) that acts as a VLESS-over-WebSocket proxy. Your traffic goes:

```
Your device --> Cloudflare CDN (nearest edge) --> CF Worker --> Target website
```

Since Cloudflare has edge nodes everywhere, your traffic exits from a Cloudflare IP — which looks like normal CDN traffic to anyone inspecting it.

## Performance (real numbers)

Tested from Shanghai, China:

| Route | TTFB (httpbin.org) | Download (50MB) |
|-------|-------------------|-----------------|
| Edgetunnel (Cloudflare Workers) | 1.8s | 3.5 MB/s |
| VPS direct (Tokyo, VLESS+REALITY) | 2.8s | 4.7 MB/s |

The edgetunnel delivers ~75% of the VPS throughput and actually has **lower latency** — Cloudflare's edge is closer than my Tokyo VPS.

In a `urltest` group with both, the edgetunnel sometimes wins the automatic latency check and gets selected as the primary route.

## Setup (fully automated)

The entire pipeline is: fork the repo → deploy to Cloudflare → auto-update forever. Total setup time: ~15 minutes.

### 1. Fork and deploy

```bash
# Fork cmliu/edgetunnel to your GitHub account, then:
gh repo clone YourUsername/edgetunnel
cd edgetunnel

# Update wrangler.toml
cat > wrangler.toml << 'EOF'
name = "your-worker-name"
main = "_worker.js"
compatibility_date = "2025-11-04"
keep_vars = true
EOF

# Set your UUID (this is your authentication credential)
# Generate one: uuidgen or https://www.uuidgenerator.net/
wrangler deploy
wrangler secret put UUID  # paste your generated UUID
```

### 2. Custom domain (important for China)

If you're in China, `workers.dev` domains are SNI-blocked by ISPs. You need a custom domain:

1. Have any domain on Cloudflare (e.g., `example.com`)
2. Add a Workers custom domain via the Cloudflare dashboard or API:

```bash
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/workers/domains" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "hostname": "edge.example.com",
    "zone_id": "YOUR_ZONE_ID",
    "service": "your-worker-name",
    "environment": "production"
  }'
```

Now your worker is accessible at `edge.example.com` with a non-blocked SNI.

### 3. Auto-update from upstream

Add two GitHub Actions workflows to your fork:

**Upstream sync** (`.github/workflows/sync.yml`) — daily sync from the original repo:

```yaml
name: Upstream Sync
permissions:
  contents: write
on:
  schedule:
    - cron: "0 0 * * *"
  workflow_dispatch:
jobs:
  sync_latest_from_upstream:
    name: Sync latest commits from upstream repo
    runs-on: ubuntu-latest
    if: ${{ github.event.repository.fork }}
    steps:
      - name: Checkout target repo
        uses: actions/checkout@v4
      - name: Sync upstream changes
        uses: aormsby/Fork-Sync-With-Upstream-action@v3.4
        with:
          upstream_sync_repo: cmliu/edgetunnel
          upstream_sync_branch: main
          target_sync_branch: main
          target_repo_token: ${{ secrets.GITHUB_TOKEN }}
          test_mode: false
```

**Auto-deploy** (`.github/workflows/deploy.yml`) — deploy to Cloudflare after each sync:

```yaml
name: Deploy to Cloudflare Workers
on:
  workflow_run:
    workflows: ["Upstream Sync"]
    types: [completed]
  workflow_dispatch:
jobs:
  deploy:
    name: Deploy to Cloudflare Workers
    runs-on: ubuntu-latest
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Deploy with Wrangler
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

Add these as GitHub repo secrets:
- `CLOUDFLARE_API_TOKEN` — your CF API token
- `CLOUDFLARE_ACCOUNT_ID` — your CF account ID

Now whenever the upstream edgetunnel project updates, your fork syncs and redeploys automatically. Zero maintenance.

## sing-box client config

```json
{
  "type": "vless",
  "tag": "cf-edgetunnel",
  "server": "edge.example.com",
  "server_port": 443,
  "uuid": "YOUR_UUID",
  "tls": {
    "enabled": true,
    "server_name": "edge.example.com"
  },
  "transport": {
    "type": "ws",
    "path": "/",
    "headers": {
      "Host": "edge.example.com"
    }
  }
}
```

Add it to a `urltest` group alongside your VPS proxies for automatic failover:

```json
{
  "type": "urltest",
  "tag": "proxy",
  "outbounds": ["vps-proxy", "cf-edgetunnel"],
  "url": "https://www.gstatic.com/generate_204",
  "interval": "3m",
  "tolerance": 50
}
```

## Gotchas

### TUN strict_route on Linux

If you run sing-box with `strict_route: true` on Linux, the edgetunnel outbound connection gets captured by TUN, creating a routing loop. Add the Cloudflare IPs for your custom domain to `route_exclude_address`:

```bash
dig +short edge.example.com
# Add the returned IPs to route_exclude_address
```

This is not an issue on iOS or when not using `strict_route`.

### Free tier limits

Cloudflare Workers free tier: 100,000 requests/day. For personal proxy use, this is more than enough. Each HTTP request through the proxy counts as one Worker invocation.

### Not a replacement for a VPS

The edgetunnel is best as a **fallback** or **complement** to a VPS proxy:
- No UDP support (WebSocket is TCP-only)
- Slightly lower throughput than a direct VPS
- Cloudflare may update their ToS — don't rely on it as your only proxy

But as a free, zero-maintenance addition to your proxy setup? It's surprisingly good.
