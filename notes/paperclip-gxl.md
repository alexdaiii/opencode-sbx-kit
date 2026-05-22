# Paperclip (GXL.ai) — install, internals, auth

**Investigated:** 2026-05-22
**Wheel version inspected:** `gxl_paperclip-0.4.2` from `https://paperclip.gxl.ai/paperclip.whl`
**Method:** Downloaded the install script and the wheel, unpacked locally with `python3 -m zipfile -e`, read sources. Nothing was executed; no `paperclip login` performed.

The point of this note is so the next person planning a sandbox kit for Paperclip can skip the read-the-source step and decide on an auth approach. Re-verify if the version on `paperclip.gxl.ai/paperclip.whl` has bumped past 0.4.2.

---

## What Paperclip is

A research/literature search service from GXL.ai. Sample command shown in the installer footer: `paperclip search "CRISPR base editing"`. Self-hosted distribution (not on PyPI — wheel served directly from `paperclip.gxl.ai`).

## Install script — `curl -fsSL https://paperclip.gxl.ai/install.sh | bash`

Script is 191 lines of bash. Flow:

1. Requires Python 3.8+.
2. Downloads `https://paperclip.gxl.ai/paperclip.whl` to a temp dir.
3. Extracts the wheel into `~/.paperclip/lib/`, with a path-traversal guard (rejects entries whose resolved path escapes `~/.paperclip/lib/`).
4. If `requests` and `click` aren't importable system-wide, runs `pip install --target ~/.paperclip/lib` for whichever is missing.
5. Writes a wrapper at `~/.local/bin/paperclip`:
   ```python
   #!/usr/bin/env python3
   import sys, os
   sys.path.insert(0, os.path.join(os.path.expanduser("~"), ".paperclip", "lib"))
   from gxl_paperclip.cli import main
   main()
   ```
6. Detects shell, prompts interactively (`read -r REPLY < /dev/tty`) to append `export PATH="$HOME/.local/bin:$PATH"` to `~/.bashrc` / `~/.zshrc` / `~/.profile`.
7. Runs `paperclip login --no-install < /dev/tty` (interactive — needs TTY for the "Press Enter" prompt; the actual device-code flow does not need TTY beyond that).
8. Runs `paperclip install < /dev/tty` — TUI to install the "skill" for Claude Code / Cursor / Codex (see **`paperclip install` behavior** below).

Notable: both `< /dev/tty` calls will hard-fail if the script runs in a fully non-interactive environment with no TTY. The CLI's `login` and `install` subcommands each have their own non-interactive paths, so the cleaner approach for a sandbox kit is to invoke them directly rather than via `install.sh`.

## Wheel layout (`gxl_paperclip-0.4.2`)

```
gxl_paperclip/
├── __init__.py
├── __main__.py
├── config.py                  # env vars, paths
├── cli/
│   ├── __init__.py
│   ├── app.py                 # click entry points
│   ├── login.py               # device-code OAuth flow
│   ├── skill.py               # `paperclip install` — writes SKILL.md
│   ├── pkce.py                # PKCE helpers (only used by dead localhost flow)
│   ├── sources.py
│   ├── repo.py
│   ├── updater.py
│   ├── _bibparse.py
│   └── _citations.py
└── client/
    ├── __init__.py
    ├── client.py              # PaperclipClient — transport + typed methods
    ├── auth.py                # APIKeyAuth / BearerAuth / FileCredentialsAuth
    ├── credentials.py         # ~/.paperclip/credentials.json + token refresh
    ├── models.py
    └── errors.py
```

Dependencies (runtime): `requests`, `click`. That's it.

## Authentication

Three strategies in `client/auth.py`, all return a header dict that the client merges into every request.

| Class | Header sent | Where the secret comes from |
|---|---|---|
| `APIKeyAuth(api_key)` | `X-API-Key: <key>` | passed in; CLI reads from `PAPERCLIP_API_KEY` env var |
| `BearerAuth(token)` | `Authorization: Bearer <token>` | string or zero-arg callable (callable lets you refresh externally) |
| `FileCredentialsAuth(credentials=None)` | `Authorization: Bearer <firebase-id-token>` | reads `~/.paperclip/credentials.json` (from `paperclip login`); auto-refreshes via `POST {base}/api/oauth/token` with the stored refresh token |

`PaperclipClient.from_env()` resolution order (`client/client.py:118-149`):

1. `PAPERCLIP_API_KEY` env var → `APIKeyAuth`
2. `~/.paperclip/credentials.json` exists → `FileCredentialsAuth`
3. Otherwise raise `AuthError`

Header construction is dead simple (`client/client.py:163-166`):

```python
def _headers(self):
    h = {"Content-Type": "application/json", "User-Agent": self._user_agent}
    h.update(self._auth.apply())   # adds X-API-Key OR Authorization
    return h
```

So every request gets the auth header automatically; no other code path touches auth headers.

## Env vars (`config.py`)

| Env var | Default | Effect |
|---|---|---|
| `PAPERCLIP_BASE_URL` | `https://paperclip.gxl.ai` | Server base URL |
| `PAPERCLIP_OAUTH_URL` | `{base}/api/oauth` | OAuth endpoint base |
| `PAPERCLIP_API_KEY` | `""` | If non-empty, `from_env()` picks `APIKeyAuth` |
| `PAPERCLIP_CONFIG_DIR` | `~/.paperclip/` | Where credentials.json and config.json live |

`config.get_base_url()` checks `~/.paperclip/config.json` for a user-configured `base_url` before falling back to the env-var default.

## Routing logic (`client/client.py:241-310`)

Where commands actually get sent depends on auth strategy AND base URL:

- Base URL hostname is `localhost` or `127.0.0.1` → REST `{base}/api/cli/execute` (server can route to multiple backends).
- Remote base URL + `APIKeyAuth` → MCP JSON-RPC at `{base}/papers` (API-key middleware lives at that path).
- Remote base URL + Bearer auth (OAuth) → REST `{base}/api/cli/execute`, falls back to `{base}/mcp` on HTTP 404.
- Hostname starts with `mcp.` → `{base}/papers` regardless.

For the streaming `map` command, the API-key path is non-streaming (returns one result event); Bearer auth streams NDJSON from `/api/cli/execute?stream=1`.

## Login flow (`cli/login.py`)

**`run_login()` always calls `_run_device_code_login()` now** (line 76-77). The PKCE-with-localhost-callback flow (`_run_pkce_login`) exists but is dead code — kept around for reference.

Device-code flow (sandbox-friendly):

1. CLI generates a code verifier locally.
2. POST `{base}/api/oauth/device` with the code verifier → server returns `{user_code, device_code}`.
3. CLI prints the `user_code` and the URL `{base}/login?device=true`.
4. User opens the URL in a browser on the host, types the code, authenticates with Firebase.
5. CLI polls `GET {base}/api/oauth/device/poll?device_code=<...>` every 2 seconds, up to 5 minutes.
6. On `status == "complete"`, server returns `{refresh_token, id_token, email, uid, expires_in}`. CLI writes to `~/.paperclip/credentials.json` with mode 0600.

The `/dev/tty` "Press Enter to log into GXL..." prompt at line 88 is wrapped in `try/except OSError` — gracefully skipped without a TTY, so device-code login works in any environment that lets the user read stdout.

`_is_remote_env()` (line 57) suppresses the auto-`webbrowser.open()` when it detects Codespaces, devcontainers, or `/.dockerenv`. The user just visits the URL manually.

## Credential storage and refresh (`credentials.py`)

`~/.paperclip/credentials.json` (mode 0600) contains:

```json
{
  "refresh_token": "...",
  "email": "...",
  "uid": "...",
  "id_token": "...",
  "id_token_expires_at": <unix-float>,
  "created_at": "<ISO 8601>"
}
```

ID tokens are refreshed transparently via `POST {base}/api/oauth/token` with `{grant_type: "refresh_token", refresh_token: ...}`. 5-minute pre-expiry margin (`_TOKEN_REFRESH_MARGIN = 300`). Refresh updates `id_token`, `id_token_expires_at`, and may rotate `refresh_token` if the server returns a new one.

## `paperclip install` behavior (`cli/skill.py`)

**Important for kit design — does NOT touch `~/.claude/settings.json`, `enabledPlugins`, MCP config, or anything our `merge_settings.py` deep-merges.**

What it does: downloads `{base}/skills/skill.md` and writes it to one of:

| Agent target | File path written (relative to user-provided `target` directory, default cwd) |
|---|---|
| Claude Code | `.claude/skills/paperclip/SKILL.md` |
| Cursor | `.cursor/skills/paperclip/SKILL.md` |
| Codex | `.agents/skills/paperclip/SKILL.md` (or legacy `AGENTS.md` with `<!-- paperclip-skill-start -->` / `<!-- paperclip-skill-end -->` sentinel comments) |

The TUI prompts (multi-select agent + install directory). Both flows can be bypassed; the underlying writer functions take `(content, target: Path)`.

Auto-refresh: `maybe_refresh_skills()` runs on every command, refreshes every 4 hours, gated by `~/.paperclip/.last_skill_refresh`. Failures are swallowed silently. Only updates files that *already exist* at those paths — won't create new ones.

For a sandbox kit we'd either:
- Invoke `paperclip install` non-interactively with `--target=$HOME` (if such a flag exists; not visible from skill.py — would need to check app.py) so the SKILL lands at `~/.claude/skills/paperclip/SKILL.md` (user-scope), OR
- Bypass the CLI and just `curl {base}/skills/skill.md > ~/.claude/skills/paperclip/SKILL.md` ourselves in `commands.install`.

## Sandbox kit implications

Three viable auth paths for embedding Paperclip in an `sbx` kit. Pick based on whether Paperclip issues API keys.

### Option A — API key via sbx proxy (cleanest, if API keys are offered)

Exact same pattern as the `allenai` (Semantic Scholar) wiring in `paper_search`:

```yaml
network:
  allowedDomains: [paperclip.gxl.ai]
  serviceDomains: {paperclip.gxl.ai: paperclip}
  serviceAuth:
    paperclip: {headerName: X-API-Key, valueFormat: "%s"}
credentials:
  sources: {paperclip: {env: [PAPERCLIP_API_KEY]}}
environment:
  proxyManaged: [PAPERCLIP_API_KEY]
```

Inside the sandbox: `PAPERCLIP_API_KEY=proxy-managed` literal. `PaperclipClient.from_env()` builds `APIKeyAuth("proxy-managed")`, sends `X-API-Key: proxy-managed`, the sbx proxy swaps in the real key on outbound traffic to `paperclip.gxl.ai`. **No `paperclip login` ever runs in the sandbox.**

Host side: `sbx secret set -g paperclip <key>`.

### Option B — device-code login inside the sandbox

User runs `paperclip login` once after sandbox boot. Reads the user code from sandbox stdout, opens `https://paperclip.gxl.ai/login?device=true` in their host browser, types the code. Credentials persist in the sandbox at `~/.paperclip/credentials.json`. Refresh tokens are long-lived.

Pros: works without GXL needing to issue API keys.
Cons: every fresh sandbox needs an interactive login.

### Option C — pre-stage `credentials.json` (mirror of the Claude Code OAuth pattern)

User authenticates once on the host (`paperclip login` outside the sandbox), then the kit's `initFiles` injects the host's `~/.paperclip/credentials.json` into the sandbox with `onlyIfMissing: true`:

```yaml
- path: /home/agent/.paperclip/credentials.json
  onlyIfMissing: true
  content: |
    { ... contents of host's ~/.paperclip/credentials.json ... }
```

`FileCredentialsAuth` auto-refreshes from the stored refresh token, so the sandbox stays authenticated transparently.

Cons: the credential content is kit-specific (per-user). Can't be committed to a public kit. Refresh token expiry could eventually require a re-login — same caveat as the Claude Code OAuth bootstrap pattern documented in `CLAUDE.md`.

## Open questions (not resolvable from the wheel)

- **Does Paperclip's `gxl.ai` dashboard actually issue API keys to users?** The client-side wiring is there (`APIKeyAuth`, `PAPERCLIP_API_KEY` env var, the `/papers` MCP middleware route), but client code can't tell us whether the product offers key issuance. If yes → Option A. If no → Option B or C.
- **Is `paperclip install` invocable non-interactively** (e.g. `paperclip install --agent claude-code --target $HOME --yes`)? `skill.py` exposes the writers as functions but doesn't show CLI flags; need to read `cli/app.py` to see how `@click.command` wraps `install_skill`.
- **What does the skill content actually instruct the agent to do?** Fetching `https://paperclip.gxl.ai/skills/skill.md` would answer this without running anything.

## Security notes

- Wheel is self-hosted, no PyPI signature. Trust model is "whatever is served at `paperclip.gxl.ai/paperclip.whl` is correct." Fine for a sandbox (contained blast radius); host installs should weigh this.
- The install script's wheel-extraction has a proper path-traversal guard (`install.sh:71-73`), so a malicious wheel can't escape `~/.paperclip/lib/`. Good hygiene.
- Credentials file is `chmod 0600` after write (`credentials.py:55-62`), with a warning logged if `chmod` fails.
- ID tokens are JWTs parsed without verification (`login.py:_parse_jwt_claims`) — comment notes "we trust the token because we just received it from our own server." Fine in context (paperclip's server is the issuer the CLI is talking to), but worth knowing.
