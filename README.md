# OpenCode Docker Sandbox (SBX)

This project provides a sandbox configuration for running OpenCode tools within
a Docker environment using `sbx`. It includes pre-configured MCP servers and
authentication settings.

## Prerequisites

Before running the sandbox, you must set up your API keys. These keys are
managed by the `sbx` host and are injected into the sandbox via a proxy.
Run the following commands to set your secrets:

``` bash
sbx secret set -g opencode
```

For AllenAI,

```bash
sbx secret set -g allenai
```

Note: Inside the sandbox environment, if you try to inspect these environment
variables
(e.g., echo $OPENCODE_API_KEY), they will literally print as proxy-managed.
This is expected behavior as the actual keys are managed by the sandbox proxy
and never directly exposed to the guest microVM.
Usage

To run the sandbox, use the following command if this kit is downloaded:

``` bash
sbx run --kit <PATH_TO_KIT_DIR>  opencode
```

If you are getting this kit from GitHub:

```bash
sbx run --kit "git+https://github.com/alexdaiii/opencode-sbx-kit&dir=base"
```

You can replace the directory after `&dir=` with your preferred directory.

## Configuration Details

The sandbox is configured with:

Model: `opencode-go/kimi-k2.6`

### Base Kit

MCP Servers:

* PyCharm: Connected via `host.docker.internal` for local IDE integration.
* DeepWiki: Remote MCP server for code documentation.
* Environment: OPENCODE_ENABLE_EXA is enabled for web search capabilities.

### Paper Search

Adds:

* Semantic Scholar: Remote MCP server (AllenAI) for paper search.

Requires the allenai secret to be set before running the sandbox.

### Playwright

Adds:

* Playright: Local MCP server