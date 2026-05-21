# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of [`sbx`](https://docs.anthropic.com/) sandbox **kits** that configure OpenCode (Go) and Claude Code inside a Docker microVM. There is no application code, build step, or test suite — the repo is YAML + JSON + a small tree of files dropped into the sandbox at runtime. Changes are validated by running the kit, not by a CI build.

## Layout

```
opencode/
  base/          # OpenCode Go auth + PyCharm MCP + DeepWiki MCP
  paper_search/  # + Semantic Scholar (AllenAI) MCP
  playwright/    # + Playwright local MCP
  superpowers/   # + obra/superpowers plugin (autoinstalled by OpenCode)
claude/
  base/          # Claude Code + PyCharm MCP + DeepWiki MCP
  paper_search/  # + Semantic Scholar (AllenAI) MCP
  playwright/    # + Playwright local MCP
  superpowers/   # + obra/superpowers plugin (via claude-plugins-official marketplace)
```

The `opencode/` and `claude/` trees mirror each other deliberately — same coverage per row, expressed in each tool's native config schema. Add or remove rows in lock-step. Note the `superpowers/` row uses different mechanisms on each side: OpenCode's `plugin: [...]` array vs Claude Code's `enabledPlugins` + marketplace pair (see the integration section below).

Each kit directory has two parts:

- `spec.yaml` — the kit manifest (`kind: mixin`). Defines proxy network allowlist, credential sources, env vars, and the host-tool config. OpenCode kits write `opencode.json` outright via `commands.initFiles`. Claude Code kits drop a **project-scoped `.mcp.json`** into the workspace via `commands.initFiles` with `onlyIfMissing: true` — they do NOT touch `~/.claude.json` (see below).
- `files/...` — a tree that mirrors the in-sandbox filesystem. The path `files/home/.config/opencode/agents/flash.md` lands at `/home/agent/.config/opencode/agents/flash.md` in the running sandbox.

The kits within each tree are near-duplicates. They differ only in (a) the MCP server list (or plugin / marketplace declaration) inside the embedded config and (b) sometimes the `allowedDomains` / `credentials` they require. When changing shared behavior within a tree, change all siblings; when changing behavior shared across the OpenCode and Claude Code stacks, change both rows.

## How a kit gets used

```bash
# Local — OpenCode kit
sbx run --kit ./opencode/base opencode
# Local — Claude Code kit
sbx run --kit ./claude/base claude

# From GitHub (any kit dir)
sbx run --kit "git+https://github.com/alexdaiii/opencode-sbx-kit.git#ref=main&dir=opencode/base" opencode
```

The final positional argument is the agent name (`opencode` or `claude`) and must match the toolchain the kit configures — pointing a `claude/*` kit at `opencode` (or vice versa) will start the wrong CLI with no config.

Before first use, host-side secrets must be set so the proxy can inject auth:

```bash
sbx secret set -g opencode   # required for any opencode/* kit
sbx secret set -g allenai    # required for any paper_search kit (both stacks)
```

Inside the sandbox, `OPENCODE_API_KEY` and `ALLEN_AI_KEY` literally read as the string `proxy-managed` — this is expected. The real values live on the host and are injected by the proxy as HTTP headers (`Authorization: Bearer …`, `x-api-key: …`) on requests to the matching `serviceDomains`.

### Claude Code authentication (`claude/*` kits)

**Mechanism:** Claude Code stores its OAuth state in `~/.claude/.credentials.json` (Linux, mode 0600). The sbx proxy treats this file the same way it treats `OPENCODE_API_KEY=proxy-managed` in the opencode kits — when the credentials file contains the literal placeholder `"sk-ant-oat01-proxy-managed"`, the proxy intercepts outbound calls to the Anthropic API and swaps the placeholder for the real OAuth credential held on the host. The kit doesn't need to declare anything in `serviceDomains` / `serviceAuth` / `credentials.sources` for this to work — the proxy recognizes the placeholder by string match. Confirmed empirically 2026-05.

**What the kit does:** every `claude/*/spec.yaml` ships a second `initFiles` entry that writes `/home/agent/.claude/.credentials.json` with the proxy-managed placeholder and a far-future `expiresAt`:

```yaml
- path: /home/agent/.claude/.credentials.json
  onlyIfMissing: true
  content: |
    {
      "claudeAiOauth": {
        "accessToken": "sk-ant-oat01-proxy-managed",
        "refreshToken": "sk-ant-ort01-proxy-managed",
        "expiresAt": 9999999999999,
        "scopes": [
          "user:file_upload",
          "user:inference",
          "user:mcp_servers",
          "user:profile",
          "user:sessions:claude_code"
        ]
      }
    }
```

Result: every fresh kit-launched sandbox boots already logged-in, no `/login` prompt. `onlyIfMissing: true` is critical — if the user has somehow already logged in (or a previous sandbox left state), don't clobber the real token.

**Required fields at bootstrap (verified by stripping and retesting):** `accessToken`, `refreshToken`, `expiresAt`, `scopes`. Stripped without consequence: `subscriptionType`, `rateLimitTier`.

**Self-population behavior:** once Claude Code makes its first authenticated API call, the file gets rewritten with real values fetched from the upstream identity response. Observed in 2026-05 testing: `expiresAt` updates from our far-future `9999999999999` placeholder to a real ~28-hour expiry, and `subscriptionType` + `rateLimitTier` appear (populated from the server). The kit's payload is therefore just a *bootstrap seed* — enough to satisfy Claude Code's startup validation so it'll make its first API call, after which the proxy hands it real identity data. `onlyIfMissing: true` on the kit's `initFiles` entry is essential: we don't want to clobber the now-real values on a sandbox restart.

**Caveat from self-population:** once `expiresAt` is the real ~28h value, the token will eventually go stale. What happens then is not yet tested — Claude Code presumably tries to refresh via the OAuth refresh endpoint, and whether the sbx proxy handles that or punts back to `/login` is unknown. If a long-lived sandbox starts demanding `/login`, that's the cause. Workaround: `sbx rm` + re-run to get a fresh bootstrap.

**If it stops working in a future sbx version:** the fallback is one of (a) `/login` inside the sandbox once per fresh sandbox, or (b) `sbx secret set -g anthropic <sk-ant-api03-…>` on the host (proxy injects `X-Api-Key`, switches to Console API-key billing instead of OAuth). The proxy-managed pattern is somewhat undocumented and could change.

**Token storage (Linux), for reference:**
- `~/.claude/.credentials.json` — OAuth access + refresh tokens (or the proxy-managed placeholder our kits inject). Mode `0600`. Hidden by plain `ls` — always use `ls -la`.
- `~/.claude.json` — `oauthAccount` block (UUID, email, organization, billing tier) plus recent-project state. NOT the token bytes, but Claude needs this for identity. **Don't write this file from a kit** — it'll erase `oauthAccount` and leave the user in a broken-identity state (recoverable from `~/.claude/backups/.claude.json.backup.<timestamp>` that Claude Code auto-creates before mutations, but disruptive).
- macOS uses the Keychain instead.

**History — things claimed earlier in this session that turned out wrong:**
- "sbx auto-bridges the host's `/login` to every Claude sandbox" → wrong; sbx does it via the proxy-managed placeholder pattern, which the kits weren't using until 2026-05.
- "`--kit` mode loses login carryover via some mechanism we can't identify" → wrong; the actual issue was the kit not shipping a credentials.json with the placeholder.
- "A Docker volume named `docker-claude-sandbox-data` holds the credentials" → hallucinated identifier from a polluted blog. Do not cite.
- "Can't write `~/.claude.json` because it races with sbx boot setup" → no evidence of a race. The real reason to avoid writing it is `oauthAccount` preservation.

## Editing conventions

- **OpenCode tool config** is embedded as a YAML literal block inside `spec.yaml` under `commands.initFiles[].content`. Must be valid JSON; mind trailing commas (they break the in-sandbox config silently). Writes `/home/agent/.config/opencode/opencode.json`.
- **Claude Code MCP config — project-scoped `.mcp.json` written to the workspace.** The kits drop `.mcp.json` into `/home/agent/workspace/.mcp.json` via `commands.initFiles` with `onlyIfMissing: true`:

  ```yaml
  - path: /home/agent/workspace/.mcp.json
    onlyIfMissing: true
    content: |
      {
        "mcpServers": {
          "pycharm":  { "type": "sse",  "url": "http://host.docker.internal:64342/sse", "headers": {} },
          "deepwiki": { "type": "http", "url": "https://mcp.deepwiki.com/mcp" }
        }
      }
  ```

  **Possible-but-unverified issue:** Claude Code shows a one-time trust prompt the first time it sees a new project-scope `.mcp.json`. If the prompt is dismissed or skipped, the servers won't load until the user explicitly accepts. If `claude mcp list` shows nothing after launching a kit, check whether claude is waiting on a trust prompt. Two ways to bypass:
  - Ship `~/.claude/settings.json` via a third `initFiles` entry with `enableAllProjectMcpServers: true` (not implemented yet).
  - Or move the MCP block into `~/.claude.json` (user-scope) instead of `/workspace/.mcp.json` (project-scope) — but `~/.claude.json` also holds `oauthAccount` (see auth section), so a kit write would have to merge, not overwrite. With the credentials.json proxy-managed pattern in place, the kit could in principle do this safely; not implemented yet.

  **`path` is literal — no substitution.** sbx hard-validates `initFiles[].path` as an absolute string. `${WORKDIR}` works inside `content` (per docs) but is rejected in `path` with `must be absolute (got "${WORKDIR}/.mcp.json")`. So the path is hard-coded to `/home/agent/workspace/.mcp.json`, which is the sbx default workspace location (confirmed via `sbx exec -it <vm> bash` → `pwd`). If a future sbx release changes this default, update all 4 kits.

  `onlyIfMissing: true` matters — without it, the kit would clobber any `.mcp.json` the user already has committed in their repo. To add/change servers, edit the JSON in `content` — must be valid JSON, no trailing commas.
- **Plugin / marketplace / hook config** for Claude Code goes in `/home/agent/.claude/settings.json` (different file, different schema — [`claude-code-settings.json`](https://json.schemastore.org/claude-code-settings.json) has no `mcpServers` key). Can be written via `initFiles` since OAuth doesn't live there.
- **MCP schemas differ between the two stacks** — don't paste OpenCode entries into a Claude kit:
  - OpenCode uses a top-level `mcp:` block. `type: remote` covers HTTP and SSE servers; `type: local` covers stdio. Stdio servers take a combined `command: ["argv", "array"]`.
  - Claude Code uses `mcpServers:` with explicit transport types `sse`, `http`, or `stdio`. Stdio servers take `command: "executable"` (string) plus `args: [...]`.
- **Adding a new MCP server**: edit the appropriate `mcp:` block (OpenCode `initFiles` → `opencode.json`) or the `mcpServers` block inside the `.mcp.json` `content` (Claude `initFiles`). If it needs network access, add the host to `network.allowedDomains` and (if it needs proxy-injected auth) wire `serviceDomains` → `serviceAuth` → `credentials.sources` and list any in-sandbox env var under `environment.proxyManaged`. The proxy only injects HTTP **headers** — services that authenticate via URL query params (e.g. NCBI E-utilities `?api_key=…`) can't be wired through this path.
- **Adding a new kit**: copy a sibling directory under `opencode/` or `claude/`, edit `spec.yaml`, and place any sandbox-side files under `files/<absolute path>`. Then update the README's kit list — both OpenCode and Claude Code sections when applicable.
- **Subagent definitions** live in `files/home/.config/opencode/agents/` (OpenCode only; Claude Code uses a different agent mechanism). Plain markdown with YAML frontmatter (`mode`, `model`, `permission`). The current `flash.md` is identical across the four opencode kits — keep it in sync if you change one.

## Model defaults

Each `opencode/*` kit pins `model: opencode-go/kimi-k2.6` and `small_model: opencode-go/deepseek-v4-flash` in the embedded `opencode.json`. The `flash` subagent uses the small model. The `claude/*` kits don't pin a model — Claude Code uses whatever model the user's `/login` selects.

## Integrating external skills / plugins

**sbx docs first — read these before adding anything, don't infer from this file:**

- [Kit `spec.yaml` schema](https://docs.docker.com/ai/sandboxes/customize/kits/)
- [`files/` static injection](https://docs.docker.com/ai/sandboxes/customize/kits/#static-files)
- [`commands.install`](https://docs.docker.com/ai/sandboxes/customize/kits/#install)
- [`commands.initFiles`](https://docs.docker.com/ai/sandboxes/customize/kits/#initfiles)
- [`network.allowedDomains`](https://docs.docker.com/ai/sandboxes/customize/kits/#network)
- [`credentials`](https://docs.docker.com/ai/sandboxes/customize/kits/#credentials)

Sbx sandboxes are **microVMs**, not Docker containers, and sbx network policy is not Docker networking. Don't reach for `docker exec`, compose-style networking, or implicit host reachability when reasoning about a kit — only the fields documented above exist. (`host.docker.internal` is the one host-side bridge available, and it has to be allowlisted explicitly, the same as any other host.)

**Use the target agent's native plugin/skill mechanism when it has one** — let the agent handle install/update/cache, and let the kit just write the right config. OpenCode: `"plugin": ["<name>@git+<url>"]` in `opencode.json` (see [`opencode/superpowers/spec.yaml`](opencode/superpowers/spec.yaml)). Claude Code: `enabledPlugins` (and optionally `extraKnownMarketplaces`) in `~/.claude/settings.json` — schema [`claude-code-settings.json`](https://json.schemastore.org/claude-code-settings.json), docs [Configure team marketplaces](https://code.claude.com/docs/en/discover-plugins#configure-team-marketplaces). Plugins in `claude-plugins-official` (auto-added in every install) need only `enabledPlugins`; others need both keys. All `claude/*` kits ship the [planning-with-files](https://github.com/OthmanAdi/planning-with-files) plugin as a worked example of the two-key pattern (self-hosted marketplace at the same repo); [`claude/superpowers/spec.yaml`](claude/superpowers/spec.yaml) shows both patterns in one file (`superpowers` from the official marketplace + `planning-with-files` from its own). Both flavors fetch from upstream on first launch, so `github.com` + `codeload.github.com` must be in `network.allowedDomains`. Claude Code shows a one-time trust prompt unless the same JSON is moved to managed settings at `/etc/claude-code/managed-settings.json` (write via `commands.install`).

## Handoff convention

When the user says **"handoff"** (or a clear synonym like "hand off to next claude"), update **this `CLAUDE.md`** and the auto-memory in lock-step before ending the turn:

1. **Refresh the Layout block** so it lists what's actually on disk — new kit directories, removed ones, renamed paths.
2. **Refresh any sections describing current behavior** that changed during the session (new MCP schemas, new auth patterns, new conventions).
3. **Write project / feedback / reference entries to auto-memory** for decisions, deferred work, constraints, and user preferences that aren't recoverable from code or git history. Update `memory/MEMORY.md` to index them.
4. **Don't create a separate `HANDOFF.md` or session-notes file** — `CLAUDE.md` plus the memory index are the canonical handoff channels. Avoid duplicating across files.
