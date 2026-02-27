# Bypass Google Scholar Redirect Blocking

## The Problem

Google Scholar email alerts contain redirect URLs like:

```
https://scholar.google.com/scholar_url?url=https://example.com/paper.pdf&hl=en&sa=X&...
```

Clicking these links takes you through `scholar.google.com`, which frequently blocks access with:

> *"We're sorry... but your computer or network may be sending automated queries."*

This happens when you're behind a VPN, proxy, or datacenter IP — which is common for researchers working remotely or in restricted network environments. The irony: the actual paper is hosted on a completely accessible server, but Google Scholar's redirect stands in the way.

## The Solution

The real paper URL is embedded in the `url` parameter. We can extract it and skip Google Scholar entirely.

### Clipboard Watcher (Linux — fully automatic)

A background service that monitors your clipboard. Copy a Scholar link, and the paper opens automatically in your browser.

**Script** — save as `~/.local/bin/scholar-watch` and `chmod +x`:

```bash
#!/bin/bash
last=""
while true; do
  clip=$(xclip -selection clipboard -o 2>/dev/null)
  if [[ "$clip" != "$last" && "$clip" == *"scholar.google.com/scholar_url"* ]]; then
    url=$(echo "$clip" | grep -oP 'url=\K[^&]+' | python3 -c "import sys,urllib.parse; print(urllib.parse.unquote(sys.stdin.read().strip()))")
    if [[ -n "$url" ]]; then
      xdg-open "$url"
      echo -n "$url" | xclip -selection clipboard
    fi
    last="$url"
  else
    last="$clip"
  fi
  sleep 0.5
done
```

**Systemd user service** — save as `~/.config/systemd/user/scholar-watch.service`:

```ini
[Unit]
Description=Scholar clipboard watcher
After=graphical-session.target

[Service]
ExecStart=%h/.local/bin/scholar-watch
Restart=on-failure
RestartSec=3
Environment=DISPLAY=:0

[Install]
WantedBy=default.target
```

Enable it:

```bash
systemctl --user enable --now scholar-watch
```

**Usage**: Right-click a Scholar link in your email client (Gmail, Protonmail, etc.) → Copy Link → paper opens automatically.

**Requirements**: `xclip`, `python3`, Linux with X11.

> **Note**: If your `DISPLAY` is different (e.g., Wayland), check with `echo $DISPLAY` and update the service file. For Wayland, replace `xclip` with `wl-copy`/`wl-paste` from `wl-clipboard`.

### Shell Function (CLI)

Add to your `~/.bashrc` or `~/.zshrc`:

```bash
scholar() {
  local url=$(echo "$1" | grep -oP 'url=\K[^&]+' | python3 -c "import sys,urllib.parse; print(urllib.parse.unquote(sys.stdin.read().strip()))")
  echo "$url"
  xdg-open "$url"  # macOS: use "open" instead
}
```

Usage (wrap URL in single quotes to avoid shell `&` interpretation):

```bash
scholar 'https://scholar.google.com/scholar_url?url=https://example.com/paper.pdf&hl=en&...'
```

## Why Not Just Proxy Google Scholar?

I tested routing `scholar.google.com` through various proxies:

- **VPS proxy** — blocked (datacenter IP)
- **Cloudflare Workers** — blocked (CDN IP)

Google Scholar is very aggressive about blocking non-residential IPs. The redirect bypass is the only reliable solution that doesn't require a residential proxy service.

## macOS / Windows

The clipboard watcher concept works on any OS:

- **macOS**: Replace `xclip` with `pbpaste`/`pbcopy`, use `open` instead of `xdg-open`, and run as a LaunchAgent instead of systemd.
- **Windows**: Use PowerShell with `Get-Clipboard`/`Set-Clipboard` and `Start-Process`, run as a scheduled task.

PRs with platform-specific implementations are welcome!

## License

MIT
