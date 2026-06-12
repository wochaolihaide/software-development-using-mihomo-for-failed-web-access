---
name: mihomo-wsl-local-proxy
version: 1.0.0
description: Use when configuring Mihomo/Clash Meta in WSL or Linux from a subscription URL, especially when the user needs local proxy ports, shell proxy variables, systemd user autostart, and evidence that external data/API access works.
---

# Mihomo WSL Local Proxy

## Overview
Set up Mihomo as a local WSL/Linux proxy without touching global system paths: discover an existing config when present, fetch a subscription only when needed, validate YAML, install the user-local binary, run it under user systemd, export proxy variables for opt-in process use, and prove data access through the proxy.

Never store subscription URLs, tokens, or API keys in the skill or final summaries unless the user explicitly asks.

## When to use
- User provides a Mihomo/Clash subscription and asks to configure it locally.
- WSL/Linux needs `127.0.0.1:7890` HTTP/SOCKS proxy and `127.0.0.1:9090` controller.
- Direct network access fails but proxy access should work.
- Need repeatable validation for OpenAI/Anthropic/Google/YouTube or other remote APIs.

## Workflow
1. Inspect whether Mihomo/Clash and a config already exist:
   ```bash
   which mihomo || which clash || true
   systemctl --user status mihomo --no-pager 2>/dev/null || true
   ss -ltnp | awk '$4 ~ /:(7890|9090)$/ {print}' || true
   test -s ~/.config/mihomo/config.yaml && echo config_exists || true
   ```
2. If `~/.config/mihomo/config.yaml` exists, validate it first and skip subscription fetching. If it is missing, get a config from either an existing user-provided file path or a subscription URL. Ask the user only if neither is available.
3. Fetch subscription to a temporary file and parse, but do not print secrets:
   ```bash
   curl -L --max-time 30 -sS -D /tmp/mihomo_sub_headers.txt -o /tmp/mihomo_sub.yaml "$SUB_URL"
   python3 - <<'PY'
import yaml
p='/tmp/mihomo_sub.yaml'
with open(p, encoding='utf-8') as f: data=yaml.safe_load(f)
print('proxies', len(data.get('proxies', [])), 'groups', len(data.get('proxy-groups', [])), 'rules', len(data.get('rules', [])))
PY
   ```
4. Write user-local config and force safe local ports if needed:
   ```bash
   mkdir -p ~/.config/mihomo ~/.local/bin
   cp /tmp/mihomo_sub.yaml ~/.config/mihomo/config.yaml
   # Use Python/YAML to set mixed-port: 7890 and external-controller: 127.0.0.1:9090.
   ```
5. Install Mihomo user-locally if no working `mihomo` binary exists. Prefer GitHub release `.gz`; if GitHub download stalls, use retry or a mirror. Verify with `mihomo -v`.
6. Validate configuration before starting:
   ```bash
   ~/.local/bin/mihomo -t -d ~/.config/mihomo
   ```
7. Start once, then migrate to systemd user service. If a manual Mihomo process already owns ports and it is the same user-local Mihomo instance, stop it before restarting the service. Do not kill unrelated global proxy processes unless the user explicitly authorizes that scope.
8. Add `~/.config/mihomo/proxy-env.sh` for opt-in shell/service use. Source it only in contexts that need proxy recovery; do not force every shell or browser to use the proxy by default:
   ```bash
   export HTTP_PROXY="http://127.0.0.1:7890"
   export HTTPS_PROXY="http://127.0.0.1:7890"
   export ALL_PROXY="socks5://127.0.0.1:7890"
   export http_proxy="$HTTP_PROXY"
   export https_proxy="$HTTPS_PROXY"
   export all_proxy="$ALL_PROXY"
   export NO_PROXY="127.0.0.1,localhost,::1"
   export no_proxy="$NO_PROXY"
   ```
9. Verify with controller, process, ports, and real network requests:
   ```bash
   pgrep -a mihomo
   curl -sS http://127.0.0.1:9090/version
   curl -x http://127.0.0.1:7890 -sS --max-time 20 https://api.ipify.org
   curl -x http://127.0.0.1:7890 -I -sS --max-time 20 https://www.google.com/generate_204
   ```

## systemd user service
Create `~/.config/systemd/user/mihomo.service`:

```ini
[Unit]
Description=Mihomo local proxy
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=%h/.local/bin/mihomo -d %h/.config/mihomo
Restart=always
RestartSec=5

[Install]
WantedBy=default.target
```

Enable and restart:
```bash
systemctl --user daemon-reload
systemctl --user enable mihomo.service
systemctl --user restart mihomo.service
systemctl --user status mihomo.service --no-pager
```

## Common pitfalls
- GitHub release downloads may fail with curl exit 56 or time out. Switch to `wget --tries`, `.gz` asset, or a proxy mirror.
- A manually started Mihomo can occupy `7890`/`9090`, making systemd logs show bind errors. Keep only the service-managed process.
- `~/.bashrc` may contain secrets; avoid full rewrite. Append a small source block or use safe file tooling.
- Direct IP tests can fail while proxy tests pass; that is acceptable if the goal is routed access.
- A `403`/`404` from remote API roots often proves reachability when the endpoint requires auth or a specific path.

## Completion criteria
- `mihomo -t -d ~/.config/mihomo` succeeds.
- `systemctl --user is-active mihomo.service` returns `active`.
- `ss` shows `127.0.0.1:7890` and `127.0.0.1:9090` listening.
- Controller `/version` returns Mihomo version JSON.
- A proxy request to an external site succeeds and logs show a rule/provider match.
- The original failed target from the calling task is retried through `127.0.0.1:7890`, or the skill reports exactly why that retry could not be performed.
