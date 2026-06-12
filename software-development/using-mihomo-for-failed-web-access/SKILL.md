---
name: using-mihomo-for-failed-web-access
description: Use when external web pages, APIs, downloads, browser navigation, curl, fetchers, dashboards, or documentation lookups fail with timeout, connection reset, DNS, 403/502, or unreachable errors in WSL/Linux and proxy access may be required.
---

# Using Mihomo For Failed Web Access

## Overview

When a normal web access step fails, keep the original task flow intact and insert recovery paths without pre-routing traffic just because a domain looks foreign or domestic. For Git operations, run the normal Git command first; if it fails, try a suitable mainland mirror or accelerated source before testing Mihomo. Only after direct Git and mirror paths both fail should you re-test the same target through Mihomo at `127.0.0.1:7890`. If Mihomo is missing or misconfigured, install/configure it using `software-development/mihomo-wsl-local-proxy` when the needed subscription/config is available, otherwise ask for only that missing secret.

## When To Use

Use this for failed access to:
- External webpages, docs, package registries, GitHub releases, Git clone/fetch/submodule downloads, APIs, RSS feeds, dashboards, and data sources.
- Tool failures from `curl`, `git clone`, `git fetch`, `git submodule update`, browser navigation, Python `httpx/requests`, Node `fetch`, Playwright, fetcher services, or crawler jobs.
- Errors such as timeout, connection reset, DNS failure, TLS handshake failure, `502`, proxy needed, or region/network blocking.

Do not use it for localhost-only failures unless the app itself fetches external resources.

## Quick Decision

1. Run or preserve the original direct access step exactly as the task intended: browser navigation, `curl`, package install, API call, Git command, fetcher job, scraper, or service endpoint.
2. Use this skill only after the original step fails with evidence such as timeout, DNS failure, connection reset, TLS failure, unreachable host, or a network-path-like `403/502`. Record the URL/domain, tool, and exact failure.
3. For Git clone/fetch/submodule failures, try a suitable mainland mirror or acceleration source before Mihomo, unless the user explicitly requires the canonical upstream remote. Examples include project-approved mirrors, Gitee mirrors, GitCode mirrors, TUNA/USTC mirror documentation for dependencies, or a temporary `https://gh.llkk.cc/https://github.com/...`-style accelerator when appropriate. Do not rewrite the repository remote permanently unless the user asks.
4. Check whether Mihomo is already available:
   ```bash
   pgrep -a mihomo || true
   systemctl --user is-active mihomo.service 2>/dev/null || true
   ss -ltnp | grep -E ':(7890|9090)\b' || true
   curl -sS --max-time 5 http://127.0.0.1:9090/version || true
   ```
5. If `127.0.0.1:7890` is listening, prefer a US-region subscription node before retrying targets that may be region-sensitive:
   ```bash
   python3 - <<'PY'
import json, re, urllib.parse, urllib.request, yaml, pathlib
base='http://127.0.0.1:9090'
def get(path):
    with urllib.request.urlopen(base+path, timeout=8) as r:
        return json.loads(r.read().decode())
def put(path, payload):
    req=urllib.request.Request(base+path, data=json.dumps(payload).encode(), method='PUT', headers={'Content-Type':'application/json'})
    with urllib.request.urlopen(req, timeout=8) as r:
        return r.status
cfg=yaml.safe_load((pathlib.Path.home()/'.config/mihomo/config.yaml').read_text(encoding='utf-8')) or {}
pat=re.compile(r'(美国|美國|美区|美區|US|USA|United States|America|洛杉矶|圣何塞|硅谷|纽约|西雅图|达拉斯|芝加哥|Los Angeles|San Jose|New York|Seattle|Dallas|Chicago)', re.I)
nodes=[p.get('name') for p in cfg.get('proxies', []) if p.get('name') and pat.search(p.get('name'))]
proxy_map=get('/proxies').get('proxies', {})
valid=[]
for name in nodes:
    if name not in proxy_map:
        continue
    enc=urllib.parse.quote(name, safe='')
    try:
        delay=get(f'/proxies/{enc}/delay?timeout=5000&url=https%3A%2F%2Fapi.ipify.org').get('delay')
        if isinstance(delay, int) and delay > 0:
            valid.append((delay, name))
    except Exception:
        pass
if not valid:
    raise SystemExit('No usable US-region node found in current Mihomo subscription')
node=min(valid)[1]
for group in ['Ghelper', '🌐 全球智能', 'AI专用']:
    item=proxy_map.get(group)
    if item and node in (item.get('all') or []):
        try:
            put('/proxies/'+urllib.parse.quote(group, safe=''), {'name': node})
        except Exception:
            pass
print('selected_us_node', node)
PY
   ```
6. Retry the exact same target through proxy, using the smallest possible change:
   ```bash
   curl -x http://127.0.0.1:7890 -I -sS --max-time 30 'https://example.com/'
   curl -x http://127.0.0.1:7890 -sS --max-time 30 'https://example.com/' | sed -n '1,20p'
   ```
7. If proxy succeeds, continue the original task from the failed access step. Inject proxy settings only into that command, isolated browser, fetcher process, or service start path. Preserve `NO_PROXY` for local services and do not change unrelated global browser/system networking.
8. If proxy fails, continue the normal fallback path for the original task, such as alternate mirror, cached data, different endpoint, clearer user-facing error, or deeper debugging. Report the direct, mirror if applicable, and Mihomo results.
9. If Mihomo is missing, inactive, or config is invalid, load `software-development/mihomo-wsl-local-proxy` and follow it to install/configure/validate before retrying the failed target, provided a subscription URL or existing config file is available.

## Recovery Contract

Mihomo is a recovery step, not a replacement for the user's original workflow.

| Original flow step | Failure evidence | Mihomo recovery | Resume rule |
|---|---|---|---|
| Browser navigates to a page | Browser/network timeout or unreachable | Isolated proxy-bound browser or `curl -x` evidence | Continue the browser task after the page is reachable |
| `curl`/download/API call | DNS, timeout, reset, TLS, blocked response | Repeat the same URL with `curl -x http://127.0.0.1:7890` | Use proxy only for that command or process |
| Python/Node fetcher | Upstream fetch fails | Inject proxy env into that process and restart if needed | Verify the real endpoint/output |
| Git HTTPS clone/fetch/submodule | Normal Git command fails with timeout/reset/TLS/DNS failure | First try a suitable mainland mirror/accelerator; if that fails, prefix only that command with `HTTPS_PROXY=http://127.0.0.1:7890 HTTP_PROXY=http://127.0.0.1:7890` | Do not write global Git proxy or permanently rewrite remotes unless user asks |
| Git SSH clone/fetch | SSH to `github.com:22` times out or is blocked | First consider an HTTPS mirror/accelerator; if that fails, use `GIT_SSH_COMMAND='ssh -o ProxyCommand="nc -X 5 -x 127.0.0.1:7890 %h %p"'` if `nc` supports proxy mode, or switch remote to HTTPS and use HTTP proxy | Keep the change command-scoped unless configuring a project service |
| Package/doc lookup | Direct registry/docs unreachable | Retry through proxy, then mirror/cache if needed | Keep the chosen source explicit |

Do not skip a user's intended access method. Do not classify all `.com` as external or all mainland services by suffix. The only reliable trigger is observed failure from the current environment.

## Mainland Direct-Access Rule

Do not route mainland China networks through Mihomo by default. In practice, many mainland services use `.com`, so suffix-based routing is only a hint. Test and use direct access first for every target unless the user explicitly asks for proxy-first behavior.

Treat these as direct-access unless the user explicitly asks otherwise:
- `.cn` domains and common mainland services such as `baidu.com`, `qq.com`, `tencent.com`, `aliyun.com`, `taobao.com`, `jd.com`, `163.com`, `zhihu.com`, `bilibili.com`, `feishu.cn`, `larksuite.com.cn`.
- Mainland package mirrors such as TUNA, USTC, Aliyun, Tencent, Huawei, npm/yarn/pip mirrors with China mirror hostnames.
- Local/private networks: `127.0.0.1`, `localhost`, `::1`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, WSL/Windows host bridge addresses, and project dev servers.

Use Mihomo for any target, mainland or overseas, only when direct access fails with evidence and the failure is plausibly network-path related. Keep `NO_PROXY=127.0.0.1,localhost,::1,.local,.lan` for services and shells.

## Browser Access Modes

Keep two browser paths conceptually separate:

| Need | Use |
|---|---|
| Normal mainland/direct browsing | Built-in browser tools or direct `curl` without proxy |
| External site failed in normal browser | Mihomo proxy retry with `curl -x http://127.0.0.1:7890` |
| Browser-rendered external site must use Mihomo | Launch an isolated Playwright/Chromium session with `--proxy-server=http://127.0.0.1:7890` |
| Compare direct vs proxy behavior | Run two isolated checks: direct browser/tool first, proxy-bound Playwright second |

The built-in browser tool may not expose per-session proxy configuration. Do not assume it can be rebound globally. For proxy-bound browser verification, use a specialized isolated browser worker that runs Playwright/Chromium with an explicit proxy. Give that worker terminal/file tools rather than relying on a generic browser tool, because generic browser sessions may keep their normal direct network path.

If Playwright/Chromium is missing, do one of these before testing:
- Use an existing project Playwright dependency if present.
- Install Playwright in a temporary or project-local environment only when dependency installation is acceptable.
- If installation is not acceptable, fall back to `curl -x http://127.0.0.1:7890` evidence and report that browser-rendered proxy verification was not available.

For proxy-bound browser verification, prefer an isolated Playwright script:

```bash
node - <<'JS'
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch({
    headless: true,
    proxy: { server: 'http://127.0.0.1:7890' }
  });
  const page = await browser.newPage();
  await page.goto('https://example.com/', { waitUntil: 'domcontentloaded', timeout: 30000 });
  console.log(await page.title());
  console.log(await page.locator('body').innerText({ timeout: 5000 }).catch(() => ''));
  await browser.close();
})();
JS
```

Do not change global browser/network settings just to test one external site. Keep direct and proxy browser checks isolated so mainland access remains direct.

## Git Recovery Order

For Git operations, use this exact order:

1. Normal upstream command first, with no proxy and no mirror rewrite:
   ```bash
   git clone https://github.com/owner/repo.git
   git fetch --all --prune
   git submodule update --init --recursive
   ```
2. If normal Git fails, try a mainland mirror or accelerator without changing global Git config. Prefer project-approved mirrors first; otherwise use a temporary mirror URL or one-off clone target when appropriate:
   ```bash
   git clone https://gitee.com/mirrors/repo.git
   git clone https://gitcode.com/mirrors/repo.git
   git clone https://gh.llkk.cc/https://github.com/owner/repo.git
   ```
   Mirror URLs are examples, not guaranteed canonical mappings. Verify repository identity before trusting code from a mirror.
3. Only if both direct Git and mirror/accelerator fail, test Mihomo for that single command:
   ```bash
   HTTPS_PROXY=http://127.0.0.1:7890 \
   HTTP_PROXY=http://127.0.0.1:7890 \
   NO_PROXY=127.0.0.1,localhost \
   git clone https://github.com/owner/repo.git
   ```

Do not set `git config --global http.proxy` or rewrite existing remotes permanently unless the user explicitly asks for persistent proxy behavior.

## Route Tools Through Mihomo

Use the least invasive option first:

| Context | Preferred fix |
|---|---|
| One command | `curl -x http://127.0.0.1:7890 ...` |
| Git over HTTPS | Only after direct Git and mirror/accelerator both fail: `HTTPS_PROXY=http://127.0.0.1:7890 HTTP_PROXY=http://127.0.0.1:7890 NO_PROXY=127.0.0.1,localhost git clone https://github.com/owner/repo.git` |
| Git submodules over HTTPS | Only after direct submodule update and mirror/accelerator options fail: `HTTPS_PROXY=http://127.0.0.1:7890 HTTP_PROXY=http://127.0.0.1:7890 git submodule update --init --recursive` |
| Git over SSH | First try direct SSH, then an HTTPS mirror/accelerator. Only then use one-command `GIT_SSH_COMMAND` with a SOCKS/HTTP-aware `nc`, or temporarily use an HTTPS remote and the HTTPS proxy env. Verify the local `nc` supports `-X/-x` before relying on it. |
| Current shell | `export HTTP_PROXY=http://127.0.0.1:7890 HTTPS_PROXY=http://127.0.0.1:7890 ALL_PROXY=socks5://127.0.0.1:7890` |
| Python `requests/httpx` | pass `proxy`/`proxies`, or set env vars for the process |
| Node/fetcher service | inject `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY=127.0.0.1,localhost` into service env |
| Browser/Playwright | set browser proxy/server launch option if env vars are ignored |
| Long-running project | update start script/service file, then restart and verify real endpoint |

Always keep `NO_PROXY=127.0.0.1,localhost,::1` so local backend/frontend traffic does not loop through proxy.

## If Mihomo Is Missing Or Broken

Load and follow `software-development/mihomo-wsl-local-proxy`.

New-environment bootstrap rule:
- If `~/.config/mihomo/config.yaml` already exists, validate and start it before asking the user for anything.
- If no config exists but the user supplied a subscription URL or config path in the current task, install/configure/validate without further confirmation.
- If no subscription/config is available anywhere retrievable, ask only for that missing subscription URL or config file path. Do not invent proxy subscriptions, tokens, node names, or credentials.
- State that secrets/subscription URLs will not be printed in summaries.
- Install user-locally under `~/.local/bin/mihomo` and `~/.config/mihomo/config.yaml` when possible.
- Configure local ports: `mixed-port: 7890`, `external-controller: 127.0.0.1:9090`.
- Validate config before starting: `~/.local/bin/mihomo -t -d ~/.config/mihomo`.
- Enable `systemd --user` service if available, otherwise start a background process.
- Verify with controller and the original failed target domain.

After bootstrap succeeds, return to the original failed access step and retry it through Mihomo. The install work is not complete until the original target has been retried or a specific remaining failure is reported.

## Verification Checklist

Before saying the web access problem is solved, show evidence for at least one real target:
- Mihomo process or service is active.
- `127.0.0.1:7890` listens.
- `127.0.0.1:9090/version` returns version JSON when controller is enabled.
- The failed external URL succeeds through `curl -x http://127.0.0.1:7890` or the target app/service returns expected data after proxy env injection.
- If changing a service, restart it and verify its real endpoint, not only standalone `curl`.
- The original workflow is resumed from the failed access step, instead of ending at “proxy works”.

## Common Pitfalls

- A direct request failing while proxy succeeds is acceptable; report that distinction clearly.
- One node logging `502` does not prove Mihomo is globally broken; test actual proxy egress and target domain.
- Some websites fail `HEAD` but succeed `GET`; retry with `GET` before concluding failure.
- `HTTP_PROXY` env vars may not affect browser tools or already-running services; inject env into the service launch path and restart.
- Avoid killing global `7890/9090` unless this project owns the Mihomo instance.
- Do not print subscription URLs or credential-bearing config contents.
