# using-mihomo-for-failed-web-access

Hermes skill for handling failed web access in WSL/Linux by preserving the original direct access flow first, then using Mihomo as a recovery path when direct access fails.

## Install

Copy `SKILL.md` into:

```text
~/.hermes/skills/software-development/using-mihomo-for-failed-web-access/SKILL.md
```

## Behavior

- Try the original access path first: browser, curl, fetcher, API, package registry, or documentation lookup.
- Use Mihomo only after a concrete network failure such as timeout, DNS failure, connection reset, TLS failure, or unreachable host.
- Retry the same target through `127.0.0.1:7890` with the smallest possible change.
- Keep localhost and local services on `NO_PROXY`.
- If Mihomo is missing or misconfigured, use the companion `mihomo-wsl-local-proxy` skill to install and validate it.
- Do not classify all `.com` domains as external or all mainland services by suffix; observed failure is the trigger.

## Secret Handling

Do not commit subscription URLs, tokens, or credential-bearing config files.
