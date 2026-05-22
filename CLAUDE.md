# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of [`sbx`](https://docs.anthropic.com/) sandbox **kits** that configure OpenCode (Go) and Claude Code inside a Docker microVM. There is no application code, build step, or test suite — the repo is YAML + JSON + a small tree of files dropped into the sandbox at runtime. Changes are validated by running the kit, not by a CI build.

## Layout

```
opencode/
  base/          # OpenCode Go auth + PyCharm MCP + DeepWiki MCP
  paper_search/  # + Semantic Scholar (AllenAI) MCP + Paperclip MCP + research-paper-workflow skill
  playwright/    # + Playwright local MCP
  superpowers/   # + obra/superpowers plugin (autoinstalled by OpenCode)
claude/
  base/          # Claude Code + PyCharm MCP + DeepWiki MCP
  paper_search/  # + Semantic Scholar (AllenAI) MCP + Paperclip MCP + research-paper-workflow skill
  playwright/    # + Playwright local MCP
  superpowers/   # + obra/superpowers plugin (via claude-plugins-official marketplace)
```

Standalone notes for involved third-party integrations live under `notes/` — e.g. `notes/paperclip-gxl.md` documents the Paperclip install script, wheel internals, auth strategies, and the sandbox-kit wiring decision tree. Add a `notes/<topic>.md` whenever an integration requires source-reading or non-trivial reverse-engineering that doesn't fit naturally in `CLAUDE.md`.

The `opencode/` and `claude/` trees mirror each other deliberately — same coverage per row, expressed in each tool's native config schema. Add or remove rows in lock-step. Note the `superpowers/` row uses different mechanisms on each side: OpenCode's `plugin: [...]` array vs Claude Code's `enabledPlugins` + marketplace pair (see the integration section below).

Each kit directory has two parts:

- `spec.yaml` — the kit manifest (`kind: mixin`). Defines proxy network allowlist, credential sources, env vars, the host-tool config, and (for `claude/*` kits) a `commands.startup` hook that runs on every sandbox boot.
- `files/...` — a tree that mirrors the in-sandbox filesystem. The path `files/home/.config/opencode/agents/flash.md` lands at `/home/agent/.config/opencode/agents/flash.md` in the running sandbox.

**Claude kits use a `commands.startup` + `files/` pattern.** Each `claude/*/files/home/.config/agents/` ships:
- `settings.json` — kit policy (defaultMode, bypassPermissions, enabledPlugins, extraKnownMarketplaces…). The startup hook deep-merges this into `~/.claude/settings.json` on every boot.
- `merge_settings.py` — the deep-merge script that the startup hook invokes (`python3 -B`). Lives in the kit so it's auditable.
- `kit-plugin/` — a local Claude Code plugin marketplace named `sbx`, registered in `settings.json` via `extraKnownMarketplaces`. Its sole plugin (`sbx-plugin`) ships the kit's MCP servers via `.mcp.json`. See **Claude Code marketplace layout** below for the directory structure.

OpenCode kits keep the original pattern: `commands.initFiles` writes `opencode.json` outright (no startup hook, no plugin marketplace).

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

- **OpenCode tool config** lives at `/home/agent/.config/opencode/opencode.json` (or `.jsonc`) in the running sandbox. Two patterns coexist in this repo — both valid, pick the one that fits the kit:
  - **Inline `initFiles` pattern** (used by `opencode/base`, `playwright`, `superpowers`): embedded as a YAML literal block inside `spec.yaml` under `commands.initFiles[].content`. Must be valid JSON; mind trailing commas (they break the in-sandbox config silently).
  - **Static file pattern** (used by `opencode/paper_search`): the file is shipped at `files/home/.config/opencode/opencode.jsonc` and gets dropped into place by sbx's static-files injection. Reach for this when the config is long enough that inline-in-YAML becomes painful to edit, or when the file is `.jsonc` (with comments) which doesn't round-trip well through `initFiles[].content`. `paper_search` drifted to this 2026-05 — its `spec.yaml` no longer has an `initFiles` entry for `opencode.json` at all.
- **Claude Code MCP config — declared inside the kit's local plugin (`kit-plugin/sbx-plugin/.mcp.json`)**. The plugin is auto-enabled at every boot via `~/.claude/settings.json.enabledPlugins["sbx-plugin@sbx"]`, and the `sbx` marketplace is registered via `extraKnownMarketplaces.sbx` pointing at `/home/agent/.config/agents/kit-plugin`. To add or change MCP servers, edit `files/home/.config/agents/kit-plugin/sbx-plugin/.mcp.json` in the kit. Standard Claude Code `mcpServers:` schema (transport types `sse`, `http`, `stdio`). This sidesteps the project-scope `.mcp.json` trust-prompt issue that bit us before, because plugin-supplied MCPs don't trigger that prompt.

  **Inconsistency to clean up:** base / paper_search / playwright all load their full MCP list via the plugin's `.mcp.json` (no `~/.claude.json` initFiles entry — paper_search ships semanticscholar there alongside pycharm+deepwiki; playwright ships the playwright stdio server). superpowers still ships a redundant `~/.claude.json` initFiles entry that duplicates pycharm+deepwiki. Future cleanup: move superpowers' MCP list into its own plugin `.mcp.json` and drop the `~/.claude.json` initFiles entry. Until then, MCPs are deduplicated by name at load time.
- **Claude Code `settings.json` is deep-merged at every boot**, not written once. Source of truth: `files/home/.config/agents/settings.json`. The startup hook `python3 -B /home/agent/.config/agents/merge_settings.py` reads it and merges into `~/.claude/settings.json`, recursively merging dicts and overwriting leaf conflicts (kit wins). This preserves keys Claude Code writes back into its own settings file (telemetry IDs, accepted trust prompts, user-installed plugins) while keeping kit policy current.

  To change kit policy (model, defaultMode, enabledPlugins, marketplace path, hooks…): edit `settings.json`. To change merge behavior: edit `merge_settings.py` — both live in `files/home/.config/agents/`. `merge_settings.py` is byte-identical across all four claude kits; keep it that way. `settings.json` is identical across base / paper_search / playwright (sbx-plugin@sbx + planning-with-files@planning-with-files), and superpowers' `settings.json` adds one additional plugin (`superpowers@claude-plugins-official`) — keep the three baseline kits in lock-step, and apply common changes to superpowers too.
- **`commands.startup` is a distinct sbx command type** ([docs](https://docs.docker.com/ai/sandboxes/customize/kits/)), separate from `install` (once at sandbox creation) and `initFiles` (writes files at sandbox creation). `startup` runs on **every boot**. Each entry is an object with:

  ```yaml
  - command: ["python3", "-B", "/home/agent/.config/agents/merge_settings.py"]
    user: "1000"           # optional, defaults to the sandbox agent user
    background: false      # optional
    description: ...       # optional, shows up in sbx run output
  ```

  `command:` is a **string array** — argv, NOT shell-interpreted. To run shell syntax, wrap with `["sh", "-c", "..."]` explicitly. Must be **idempotent** — they run every boot. Sbx docs say to prefer `initFiles` for file writes; we use `startup` here specifically because we *want* the deep-merge re-applied every boot.
- **Don't put binary files in `files/`** — even as a transient side effect. sbx's file injector breaks on binary content (`sh: 2: Syntax error: Unterminated quoted string`). The way this bit us: an `importlib.exec_module(...)` dry-run of `merge_settings.py` silently created `__pycache__/merge_settings.cpython-313.pyc` inside the kit tree; on the next `sbx run` the `.pyc` broke the injector. Defenses: (a) `python3 -B` in the startup command so the running sandbox never generates `.pyc`, (b) when dry-running a python script that lives under `files/`, run it with `subprocess.run(["python3", "-B", "<path>"])` not `importlib`. Reference: 2026-05.
- **Claude Code marketplace layout — plugin must live in a subdirectory of the marketplace root.** The walkthrough convention in the [Claude marketplace docs](https://code.claude.com/docs/en/plugin-marketplaces) puts `marketplace.json` at the marketplace root and each plugin in `./<plugin-name>/`. Co-locating them (e.g. `source: "./"` with marketplace.json and plugin.json sharing the same `.claude-plugin/`) fails — Claude reports "Plugin 'X' not found in marketplace 'Y'" even when both names look right. Layout we ended up with for the kit-plugin:

  ```
  files/home/.config/agents/kit-plugin/
    .claude-plugin/marketplace.json       # name: "sbx", plugins: [{name:"sbx-plugin", source:"./sbx-plugin"}]
    sbx-plugin/
      .claude-plugin/plugin.json          # name: "sbx-plugin"
      .mcp.json                           # the kit's MCP servers
  ```

  **The marketplace's `name:` field must match the key used in `extraKnownMarketplaces`.** We register the marketplace under key `"sbx"`, so `marketplace.json.name` must also be `"sbx"`. When they disagreed (`name: "sbx-plugin"` registered under `"sbx"`), `/plugins` couldn't resolve `sbx-plugin@sbx`. Fixed 2026-05.

  **`extraKnownMarketplaces` for a local directory uses `path`, not `repo`.** Schema for a directory-typed marketplace source:

  ```json
  "extraKnownMarketplaces": {
    "sbx": {
      "source": {
        "source": "directory",
        "path": "/home/agent/.config/agents/kit-plugin"
      }
    }
  }
  ```

  `repo` is for `"source": "github"`. The kits' first iteration used `repo` and `/doctor` reported `extraKnownMarketplaces.sbx.source.path: Expected string, but received undefined`.
- **MCP schemas differ between the two stacks** — don't paste OpenCode entries into a Claude kit:
  - OpenCode uses a top-level `mcp:` block. `type: remote` covers HTTP and SSE servers; `type: local` covers stdio. Stdio servers take a combined `command: ["argv", "array"]`.
  - Claude Code uses `mcpServers:` with explicit transport types `sse`, `http`, or `stdio`. Stdio servers take `command: "executable"` (string) plus `args: [...]`.
- **Adding a new MCP server**: edit the appropriate `mcp:` block (OpenCode — either `initFiles` writing `opencode.json` OR the static `files/home/.config/opencode/opencode.jsonc`, depending on which pattern the kit uses) or the `mcpServers` block inside `files/home/.config/agents/kit-plugin/sbx-plugin/.mcp.json` (Claude). If it needs network access, add the host to `network.allowedDomains` and (if it needs proxy-injected auth) wire `serviceDomains` → `serviceAuth` → `credentials.sources` and list any in-sandbox env var under `environment.proxyManaged`. The proxy only injects HTTP **headers** — services that authenticate via URL query params (e.g. NCBI E-utilities `?api_key=…`) can't be wired through this path.

  **Worked example — Paperclip MCP (added 2026-05).** The `paper_search` kits route through `https://paperclip.gxl.ai/mcp` with header `X-API-Key: proxy-managed`. Host secret name: `gxl-paperclip`. In-sandbox env var name: `PAPERCLIP_API_KEY` (the canonical name per paperclip's own SDK, even though we don't read it from the env directly when going MCP-only — keeping the env var matches paperclip's docs and lets users add the CLI later without renaming). The naming chain — `network.serviceDomains.<host>: <service-key>` ⇒ `network.serviceAuth.<service-key>` ⇒ `credentials.sources.<service-key>` — must use the same `<service-key>` throughout (we use `gxl-paperclip`). Header case matters to some upstreams: paperclip's SDK sends `X-API-Key` (capital), allenai's takes lowercase `x-api-key`. Don't normalize. Full source-level analysis of paperclip's auth, install script, and skill mechanism in `notes/paperclip-gxl.md`.
- **Adding a new kit**: copy a sibling directory under `opencode/` or `claude/`, edit `spec.yaml`, and place any sandbox-side files under `files/<absolute path>`. Then update the README's kit list — both OpenCode and Claude Code sections when applicable. For claude kits, copy from `base` (or any of paper_search / playwright, which share an identical `settings.json`) when adding the next baseline kit; copy from `superpowers` only if the new kit also wants the `superpowers@claude-plugins-official` plugin. `merge_settings.py` is identical across all four claude kits — keep it that way.
- **Python in the sandbox is `python3`** (not `python`). Use that explicitly in startup commands and scripts; relying on a `python` symlink will break.
- **Subagent definitions** live in `files/home/.config/opencode/agents/` (OpenCode only; Claude Code's subagent mechanism is separate from skills). Plain markdown with YAML frontmatter (`mode`, `model`, `permission`). The current `flash.md` is identical across the four opencode kits — keep it in sync if you change one.
- **Skills (both stacks)** — kit-shipped skills are markdown files with YAML frontmatter that the agent loads on demand by description match. The on-disk filename is `SKILL.md` (all caps; OpenCode enforces this case-sensitively).
  - **Claude Code**: plugin-scoped at `files/home/.config/agents/kit-plugin/sbx-plugin/skills/<skill-name>/SKILL.md`. Frontmatter: `description` is the only recommended field; `name` is optional and defaults to the directory name. The plugin's skills are namespaced as `sbx-plugin:<skill-name>` in `/` autocomplete and in `/mcp` listings.
  - **OpenCode**: at `files/home/.config/opencode/skills/<skill-name>/SKILL.md`. Frontmatter: **both `name` AND `description` are required** (unlike Claude Code). OpenCode loads skills via a `skill()` tool — the model calls `skill({name: "..."})` when the description matches.
  - When the same workflow rule applies to both kits, ship parallel `SKILL.md` files with identical content modulo tool-name casing (Claude uses `WebFetch`/`Read`/`Edit`; OpenCode uses `webfetch`/`read`/`edit`). The current `research-paper-workflow` skill in `claude/paper_search` and `opencode/paper_search` is the reference example — keep them in sync.
  - Use skills for **rules + workflow patterns** that decide *which tool* to reach for (e.g. "paperclip MCP before WebFetch for indexed papers"). Don't use them to duplicate documentation that ships with an upstream tool — the upstream `paperclip` SKILL.md (CLI-focused, 505 lines, auto-refreshing every 4hrs from `paperclip.gxl.ai/skills/skill.md`) is intentionally NOT shipped by our kits because we're MCP-only.

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

**Use the target agent's native plugin/skill mechanism when it has one** — let the agent handle install/update/cache, and let the kit just write the right config. OpenCode: `"plugin": ["<name>@git+<url>"]` in `opencode.json` (see [`opencode/superpowers/spec.yaml`](opencode/superpowers/spec.yaml)). Claude Code: `enabledPlugins` (and optionally `extraKnownMarketplaces`) in `~/.claude/settings.json` — schema [`claude-code-settings.json`](https://json.schemastore.org/claude-code-settings.json), docs [Configure team marketplaces](https://code.claude.com/docs/en/discover-plugins#configure-team-marketplaces). Plugins in `claude-plugins-official` (auto-added in every install) need only `enabledPlugins`; others need both keys. All `claude/*` kits ship the [planning-with-files](https://github.com/OthmanAdi/planning-with-files) plugin (its own self-hosted marketplace, `name: "planning-with-files"`, plugin source `./` co-located with marketplace.json). `claude/superpowers` additionally enables [`obra/superpowers`](https://github.com/obra/superpowers) via the official marketplace — `enabledPlugins["superpowers@claude-plugins-official"] = true`, no `extraKnownMarketplaces` entry needed. (Confirmed 2026-05: the official marketplace contains 200+ plugins including `superpowers`, pinned to a specific SHA of `obra/superpowers.git`. WebFetch summaries of files this long silently drop entries — verify presence by parsing the raw JSON, not by asking the summarizer.) Both flavors fetch from upstream on first launch, so `github.com` + `codeload.github.com` must be in `network.allowedDomains`. Claude Code shows a one-time trust prompt unless the same JSON is moved to managed settings at `/etc/claude-code/managed-settings.json` (write via `commands.install`).

## Handoff convention

When the user says **"handoff"** (or a clear synonym like "hand off to next claude"), update **this `CLAUDE.md`** and the auto-memory in lock-step before ending the turn:

1. **Refresh the Layout block** so it lists what's actually on disk — new kit directories, removed ones, renamed paths.
2. **Refresh any sections describing current behavior** that changed during the session (new MCP schemas, new auth patterns, new conventions).
3. **Write project / feedback / reference entries to auto-memory** for decisions, deferred work, constraints, and user preferences that aren't recoverable from code or git history. Update `memory/MEMORY.md` to index them.
4. **Don't create a separate `HANDOFF.md` or session-notes file** — `CLAUDE.md` plus the memory index are the canonical handoff channels. Avoid duplicating across files.
