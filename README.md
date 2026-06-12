# Mihomo Hermes Skills

This repository publishes a small Mihomo skill set for Hermes Agent in WSL/Linux environments.

## Included skills

- `using-mihomo-for-failed-web-access`: recovery rules for failed web, API, download, browser, fetcher, and Git access.
- `mihomo-wsl-local-proxy`: local Mihomo setup, validation, user systemd service, and opt-in proxy environment.

The root `SKILL.md` keeps backward compatibility with the original single-skill repository and points to `using-mihomo-for-failed-web-access`.

## Install

### Single primary skill

Copy the root skill into your Hermes skills directory:

```bash
mkdir -p ~/.hermes/skills/software-development/using-mihomo-for-failed-web-access
cp SKILL.md ~/.hermes/skills/software-development/using-mihomo-for-failed-web-access/SKILL.md
```

### Full Mihomo skill set

Copy both skill directories:

```bash
mkdir -p ~/.hermes/skills/software-development
cp -a software-development/using-mihomo-for-failed-web-access ~/.hermes/skills/software-development/
cp -a software-development/mihomo-wsl-local-proxy ~/.hermes/skills/software-development/
```

## Behavior summary

- Direct access is tried first; Mihomo is a recovery path after observed network failure.
- Git uses direct access first, then a suitable mirror/accelerator, and only then command-scoped Mihomo proxy variables.
- Mainland and local/private networks stay direct by default.
- Proxy settings are scoped to the failing command, browser session, fetcher process, or service start path.
- Subscription URLs, tokens, and credential-bearing configs must not be committed or printed.

## Expected local ports

- Mixed proxy: `127.0.0.1:7890`
- Controller: `127.0.0.1:9090`

## Verification

After installation, verify with:

```bash
hermes skills list | grep -E 'using-mihomo-for-failed-web-access|mihomo-wsl-local-proxy'
```
