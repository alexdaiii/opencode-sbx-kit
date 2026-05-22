# Docker Sandbox (SBX) Kits

This project provides a sandbox configuration for Claude Code/OpenCode within
a Docker environment using `sbx`. It includes pre-configured MCP servers and
authentication settings.

## Prerequisites

Before running the sandbox, you should set up your API keys. These keys are
managed by the `sbx` host and are injected into the sandbox via a proxy.
Run the following commands to set your secrets:

OpenCode Go/Zen:

``` bash
sbx secret set -g opencode
```

For AllenAI/Semantic Scholar,

```bash
sbx secret set -g allenai
```

Note: Inside the sandbox environment, if you try to inspect these environment
variables
(e.g., echo $OPENCODE_API_KEY), they will literally print as `proxy-managed`.
This is expected behavior as the actual keys are managed by the sandbox proxy
and never directly exposed to the guest microVM.

**Claude Code: no `/login` needed (kits handle it).** Each `claude/*`
kit ships a proxy-managed `~/.claude/.credentials.json` placeholder via
`commands.initFiles` — the literal string `"sk-ant-oat01-proxy-managed"`
in the access-token field tells the sbx proxy to swap in your real
host-side OAuth credential on every outbound API call. Same pattern as
opencode's `OPENCODE_API_KEY=proxy-managed`. You don't need
`sbx secret set -g anthropic` or any host-side claude-specific setup —
your existing host login is what the proxy reuses.

If for any reason it stops working (e.g. a future sbx version changes
the proxy behavior), fall back to one of:

1. `/login` inside the sandbox once per fresh sandbox. OAuth flow,
   `sk-ant-oat01-*` token, persists for that sandbox's lifetime.
2. `sbx secret set -g anthropic <sk-ant-api03-…>` on the host. The
   proxy injects `X-Api-Key` instead of `Authorization`. Switches from
   Claude.ai subscription OAuth to Console API-key billing.

Token storage on Linux (matches [Claude Code authentication docs](https://code.claude.com/docs/en/authentication)):

* `~/.claude/.credentials.json` — OAuth access + refresh tokens (or
  the proxy-managed placeholder our kit injects). Mode `0600`. Hidden
  by plain `ls`; use `ls -la`.
* `~/.claude.json` — `oauthAccount` identity metadata (UUID, email,
  org, billing tier) plus recent-project state. NOT the token itself,
  but Claude Code needs it for identity. Kits avoid writing this file
  to preserve the user's `oauthAccount`.
* `~/.claude/backups/.claude.json.backup.<timestamp>` — Claude Code
  auto-snapshots `~/.claude.json` before mutations. Recoverable if a
  kit ever clobbers it.
* macOS uses the Keychain instead.

## Usage

To run the sandbox, use the following command if this kit is downloaded:

``` bash
sbx run --kit <PATH_TO_KIT_DIR>  opencode
```

If you are getting this kit from GitHub:

```bash
sbx run --kit "git+https://github.com/alexdaiii/slopify.git#ref=main&dir=opencode/base" opencode
```

You can replace the directory after `&dir=` with the preferred directory.
The protocol can also be `ssh` instead of `https`.

For a Claude Code kit, swap the final agent argument from `opencode` to
`claude` and point `&dir=` at a `claude/*` directory, e.g.
`&dir=claude/base`.

## OpenCode Configuration Details

The sandbox is configured with:

Model: `opencode-go/kimi-k2.6`

### Base Kit

MCP Servers:

* PyCharm: Connected via `host.docker.internal` for local IDE integration.
* DeepWiki: Remote MCP server for code documentation.
* Environment: `OPENCODE_ENABLE_EXA` is enabled for web search capabilities.

### Paper Search

Adds:

* Semantic Scholar: Remote MCP server (AllenAI) for paper search.

Requires the AllenAI secret to be set before running the sandbox.

### Playwright

Adds:

* Playright: Local MCP server

### Superpowers (`opencode/superpowers`)

Enables the [Superpowers](https://github.com/obra/superpowers) plugin via
OpenCode's native `plugin: ["superpowers@git+..."]` declaration in
`opencode.json`. OpenCode auto-installs the plugin on first launch.

## YOLO Mode (Auto-Approve)

All OpenCode kits are configured in **YOLO mode** with full permissions:

* `permission: "allow"` — all tool actions run without prompting
* `yolo: true` — auto-approves any remaining `"ask"` permission prompts

This means OpenCode will not stop to ask for approval before editing files,
running commands, searching the web, or accessing directories outside the project.
Explicit `"deny"` rules are still respected.

> **Warning:** This is intentionally permissive. Use these kits in isolated
> sandboxes where destructive actions are bounded by the microVM. Do not copy
> this permission config to your local `~/.config/opencode/opencode.json` unless
> you understand the risks.

To temporarily disable YOLO for a single session, override the env var:

```bash
OPENCODE_YOLO=false sbx run --kit ./opencode/base opencode
```

## Planning with Files (OpenCode)

All OpenCode kits ship the [planning-with-files](https://github.com/OthmanAdi/planning-with-files)
skill pre-installed. It provides Manus-style persistent markdown planning
(`task_plan.md`, `findings.md`, `progress.md`) with built-in hooks that:

* Re-inject the current plan before each prompt
* Survive `/clear` and context compaction
* Prompt the agent to update progress after every Write/Edit

The skill is auto-discovered via `skills.paths` in `opencode.json` and
auto-invoked for any task requiring multiple tool calls — no manual
`use_skill` needed. If the [Superpowers](https://github.com/obra/superpowers)
plugin is also enabled, the skill integrates with the `use_skill` tool.

## Claude Code Configuration Details

Each `claude/*` kit ships three things to the running sandbox:

1. **`commands.initFiles` → `~/.claude/.credentials.json`** — proxy-managed
   OAuth placeholder (`sk-ant-oat01-proxy-managed`). Lets each fresh
   sandbox boot already-logged-in without a `/login` prompt; the sbx
   proxy swaps the placeholder for your real host-side OAuth token on
   every API call. See Prerequisites for the fallback paths if it stops
   working.
2. **`files/home/.config/agents/`** — a stash containing:
   - `settings.json` — kit policy (defaultMode, bypassPermissions,
     `enabledPlugins`, `extraKnownMarketplaces`).
   - `merge_settings.py` — the script the startup hook invokes.
   - `kit-plugin/` — a local Claude Code plugin marketplace named `sbx`,
     containing a single plugin `sbx-plugin` that ships the kit's MCP
     servers via its own `.mcp.json`.
3. **`commands.startup` → `python3 -B …/merge_settings.py`** — runs on
   every sandbox boot. Deep-merges `…/agents/settings.json` into
   `~/.claude/settings.json`, recursively merging dicts and overwriting
   leaf conflicts (kit wins). This preserves keys Claude Code writes
   back into its own settings file (telemetry IDs, accepted trust
   prompts, user-installed plugins) while keeping kit policy current.

Why a startup-time merge instead of a one-shot `initFiles` write: Claude
Code rewrites `~/.claude/settings.json` itself during runtime, so a blind
overwrite on every restart would clobber its work. Why a local
plugin marketplace instead of a project-scoped `~/workspace/.mcp.json`:
project-scope MCP files trigger a one-time trust prompt that silently
suppressed our MCP servers until accepted. Plugin-supplied MCPs don't
trigger that prompt.

The kits do **not** write `/home/agent/.claude.json` — that file holds
the user's `oauthAccount` identity block, and Claude Code populates it
itself on first authenticated API call.

**All `claude/*` kits also bundle the
[planning-with-files](https://github.com/OthmanAdi/planning-with-files)
plugin** — Manus-style persistent markdown planning (`task_plan.md`,
`findings.md`, `progress.md`) with built-in hooks that re-inject the plan
before each prompt, survive `/clear` and context compaction, and prompt
Claude to update progress after every Write/Edit. The plugin is declared
via `extraKnownMarketplaces` + `enabledPlugins` in `settings.json` and
fetched from GitHub on first launch.

### Base Kit (`claude/base`)

MCP Servers (via `sbx-plugin`):

* PyCharm: SSE MCP server connected via `host.docker.internal` for local
  IDE integration.
* DeepWiki: Remote HTTP MCP server for code documentation.

### Paper Search (`claude/paper_search`)

Adds:

* Semantic Scholar: Remote HTTP MCP server (AllenAI) for paper search.

Requires the AllenAI secret to be set before running the sandbox.

### Playwright (`claude/playwright`)

Adds:

* Playwright: Local stdio MCP server (`npx @playwright/mcp@latest`).

### Superpowers (`claude/superpowers`)

Enables the [Superpowers](https://github.com/obra/superpowers) plugin via
Claude Code's `enabledPlugins` key in `settings.json`, pointing at
the official Claude Code plugin marketplace (`claude-plugins-official`,
auto-added in every Claude Code install). Claude Code installs the plugin
on first launch and may show a one-time trust prompt; accept it and
subsequent runs are silent.
