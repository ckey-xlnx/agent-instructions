# MCP Configuration

This document describes how to access infrastructure services via MCP servers.
For the list of infrastructure instances and what they contain, see `infrastructure-knowledge.md`.
For Cline-specific command execution tooling (`mcp-cli-exec`), see `agent-tooling-instructions.md`.

## AMD MCP Platform

Most services are accessed via AMD's hosted MCP gateway at `mcp-platform.amd.com`. The
goal is to use platform endpoints everywhere and only fall back to local stdio servers
for services not yet on the platform.

| Platform Endpoint | Service | Auth |
|-------------------|---------|------|
| `https://mcp-platform.amd.com/mcp/atlassian_gateway/mcp` | AMD cloud Jira (`amd.atlassian.net`) | OAuth (hardwired to AMD Jira server-side) |
| `https://mcp-platform.amd.com/mcp/internal_atlassian` | OnTrack internal (`ontrack-internal.amd.com`) | Token PAT in `Authorization` header |
| `https://mcp-platform.amd.com/mcp/github_new` | Public GitHub (`github.com`) | Bearer PAT; one server instance per account |
| `https://mcp-platform.amd.com/mcp/gitenterprise` | Xilinx GHE (`gitenterprise.xilinx.com`) | Bearer PAT |
| `https://mcp-platform.amd.com/mcp/cloud_atlassian/` | AMD cloud Jira (basic auth alternative) | Basic auth (email:token); prefer `atlassian_gateway` |

## Common Configuration

This is the canonical, target configuration. All agents should use these servers.

### Jira

| Instance | Server | Auth |
|----------|--------|------|
| `amd.atlassian.net` | `atlassian-gateway` | OAuth via `/mcp` (auto-refresh) |
| `ontrack-internal.amd.com` | `ontrack-internal` | Token PAT |
| `ontrack.amd.com` | — | No platform endpoint yet |
| `pensando.atlassian.net` | `jira` (local stdio, stop-gap) | Token PAT via `cline-mcp/jira-server` |

**Note:** `atlassian_gateway` is preferred over `cloud_atlassian` (OAuth vs basic auth).
Pensando cloud Jira has no AMD platform path: `atlassian_gateway` is hardwired to
`amd.atlassian.net` server-side (confirmed in source). The local `jira-server` is a
stop-gap until a platform endpoint supporting Pensando is deployed.

### GitHub

| Instance / Account | Server |
|--------------------|--------|
| `github.com` — `ckey-xlnx` | `github-ckey-xlnx` |
| `github.com` — `ckey_amdeng` | `github-ckey-amdeng` |
| `gitenterprise.xilinx.com` — `ckey` | `github-ckey-xlnx-ghe` |
| `github.com` — `cjk32` | — |
| `github.amd.com` | — (no AMD platform endpoint) |
| `er.github.amd.com` | — (no AMD platform endpoint) |

### ReviewBoard

| Instance | Server |
|----------|--------|
| `reviewboard.amd.com` | `reviewboard` (local stdio, `cline-mcp/reviewboard-server`) |

### Setup Commands (new machine)

```bash
# Pensando cloud Jira — stop-gap local server (Token PAT from pensando.atlassian.net)
# Build first: cd /home/ckey/hg/cline-mcp/jira-server && npm install && npm run build

# Claude Code:
claude mcp add -s user --transport stdio "jira" \
  /tool/pandora/.package/node-24.5.0/bin/node \
  /home/ckey/hg/cline-mcp/jira-server/build/index.js \
  -e JIRA_INSTANCE_PENSANDO_URL=https://pensando.atlassian.net \
  -e JIRA_INSTANCE_PENSANDO_EMAIL=<amd-email> \
  -e JIRA_INSTANCE_PENSANDO_TOKEN=<pensando-pat>

# Cline: add env vars to the existing "jira" server entry in cline_mcp_settings.json:
#   "JIRA_INSTANCE_PENSANDO_URL": "https://pensando.atlassian.net",
#   "JIRA_INSTANCE_PENSANDO_EMAIL": "<amd-email>",
#   "JIRA_INSTANCE_PENSANDO_TOKEN": "<pensando-pat>"

# ReviewBoard — API token (generate at reviewboard.amd.com/account/api-tokens/)
claude mcp add -s user --transport stdio "reviewboard" \
  /tool/pandora/.package/node-24.5.0/bin/node \
  /home/ckey/hg/cline-mcp/reviewboard-server/build/index.js \
  -e REVIEWBOARD_URL=https://reviewboard.amd.com \
  -e REVIEWBOARD_TOKEN=<reviewboard-pat>

# AMD cloud Jira via OAuth gateway — then run /mcp in Claude Code to authenticate
claude mcp add --transport http "atlassian-gateway" \
  "https://mcp-platform.amd.com/mcp/atlassian_gateway/mcp" -s user

# OnTrack internal — Token PAT (generate at ontrack-internal.amd.com/secure/ViewProfile.jspa)
claude mcp add --transport http "ontrack-internal" \
  "https://mcp-platform.amd.com/mcp/internal_atlassian" -s user \
  --header "Authorization: Token <ontrack-internal-pat>"

# GitHub accounts (PATs stored separately — do not commit)
claude mcp add --transport http "github-ckey-xlnx" \
  "https://mcp-platform.amd.com/mcp/github_new" -s user \
  --header "Authorization: Bearer <ckey-xlnx-pat>"
claude mcp add --transport http "github-ckey-amdeng" \
  "https://mcp-platform.amd.com/mcp/github_new" -s user \
  --header "Authorization: Bearer <ckey-amdeng-pat>"
claude mcp add --transport http "github-ckey-xlnx-ghe" \
  "https://mcp-platform.amd.com/mcp/gitenterprise" -s user \
  --header "Authorization: Bearer <gitenterprise-xlnx-pat>"
```

### Example Tool Calls

```
# AMD cloud Jira
jira_get_issue(issue_key="FWDEV-1234")    # via atlassian-gateway

# OnTrack internal
jira_get_issue(issue_key="PROJ-123")      # via ontrack-internal
```

---

## Agent Exceptions

Deviations from the common configuration above. The goal is to shrink these over time
by migrating each exception to the common platform-based setup.

### Claude Code

No exceptions — Claude Code matches the common configuration exactly.

### Cline

Config file: `~/.dotfiles/.cline/data/settings/cline_mcp_settings.json`
(symlinked to `~/.cline/data/settings/cline_mcp_settings.json`)

The config file is tracked in `~/.dotfiles` (version controlled) but contains
placeholder tokens (`YOUR_..._TOKEN_HERE`) in place of real credentials. After
cloning dotfiles on a new machine, replace each placeholder with the real PAT
before starting Cline. Do not commit real tokens — the placeholders serve as
the documented template.

All servers below are local stdio servers from `/home/ckey/hg/cline-mcp`. They replace
the corresponding common-config servers until migrated.

| Service | Common Server | Cline Exception | Notes |
|---------|--------------|-----------------|-------|
| `ontrack-internal.amd.com` | `ontrack-internal` | `jira` (local stdio) | `get_issue(instance="ontrack_internal", ...)` |
| `ontrack.amd.com` | — | `jira` (local stdio) | `get_issue(instance="ontrack_external", ...)` — no token configured yet |

AMD cloud Jira (`amd.atlassian.net`) and ReviewBoard are not configured for Cline.

#### Building Local Servers

All local servers are TypeScript-based:

```bash
cd /home/ckey/hg/cline-mcp/<server-name>
npm install
npm run build
```

#### Example Tool Calls (Cline)

```
# OnTrack internal
get_issue(instance="ontrack_internal", issue_key="PROJ-123")

# OnTrack external
get_issue(instance="ontrack_external", issue_key="PROJ-456")
```

---

## Policy-Managed Servers (Claude Code)

System-installed by the AMD engineering suite, read-only. Config at
`/usr/lib/engineering-ai-suite/resources/apr_setup/claude/.mcp.json`.
These are always present alongside user-configured servers.

| Server | Purpose | Notes |
|--------|---------|-------|
| `github-mcp` | GitHub (tool-filtered) | Read-only subset of tools |
| `jira-internal` | OnTrack internal | Do not use — token is a placeholder, no working auth |
| `codegen` | AMD code generation | — |
| `sourcegraph` | AMD Sourcegraph | — |
| `similar-tickets` | Similar ticket search | — |
