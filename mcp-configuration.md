# MCP Configuration

This document describes how to access infrastructure services via MCP servers, per agent.
For the list of infrastructure instances and what they contain, see `infrastructure-knowledge.md`.

## AMD MCP Platform

Most services are accessed via AMD's hosted MCP gateway at `mcp-platform.amd.com`. The
goal is to use platform endpoints wherever possible and only fall back to local stdio
servers for services not on the platform.

| Platform Endpoint | Service | Auth |
|-------------------|---------|------|
| `https://mcp-platform.amd.com/mcp/atlassian_gateway/mcp` | AMD cloud Jira (`amd.atlassian.net`) | OAuth (hardwired to AMD Jira server-side) |
| `https://mcp-platform.amd.com/mcp/internal_atlassian` | OnTrack internal (`ontrack-internal.amd.com`) | Token PAT in `Authorization` header |
| `https://mcp-platform.amd.com/mcp/github_new` | Public GitHub (`github.com`) | Bearer PAT; one server instance per account |
| `https://mcp-platform.amd.com/mcp/gitenterprise` | Xilinx GHE (`gitenterprise.xilinx.com`) | Bearer PAT |
| `https://mcp-platform.amd.com/mcp/cloud_atlassian/` | AMD cloud Jira (basic auth alternative) | Basic auth (email:token); prefer `atlassian_gateway` |

## Service to MCP Server Mapping

### Jira

| Instance | Claude Code Server | Cline Server |
|----------|-------------------|--------------|
| `amd.atlassian.net` | `atlassian-gateway` (OAuth, auto-refresh) | not configured |
| `pensando.atlassian.net` | none — AMD platform hardwired to AMD Jira; no Pensando path | none |
| `ontrack-internal.amd.com` | `ontrack-internal` (Token PAT) | `jira` (local, `get_issue` with `instance="ontrack_internal"`) |
| `ontrack.amd.com` | none | `jira` (local, `get_issue` with `instance="ontrack_external"`, no token yet) |

**Note:** Both `cloud_atlassian` and `atlassian_gateway` platform endpoints are hardwired
to `amd.atlassian.net` server-side. `atlassian_gateway` is preferred (OAuth with
auto-refresh). Pensando cloud Jira requires a separately deployed platform instance —
no working path currently exists.

### GitHub

| Instance | Claude Code Server | Cline Server |
|----------|-------------------|--------------|
| `github.com` — `ckey-xlnx` | `github-ckey-xlnx` | not configured |
| `github.com` — `ckey_amdeng` | `github-ckey-amdeng` | not configured |
| `github.com` — `cjk32` | none | none |
| `gitenterprise.xilinx.com` | `github-ckey-xlnx-ghe` | not configured |
| `github.amd.com` | none | none |
| `er.github.amd.com` | none | none |

### ReviewBoard

| Instance | Claude Code Server | Cline Server |
|----------|-------------------|--------------|
| `reviewboard.amd.com` | none | `reviewboard` (local stdio) |

---

## Claude Code MCP Configuration

**Config file:** `~/.claude.json` (user-scope `mcpServers` key)

**Gotcha:** `~/.claude.json` also stores project-scoped servers under a `projects` key.
If a server appears under both user scope and a project scope, the project entry takes
precedence and may lack auth credentials, showing as "connected · no tools". Check for
duplicates with:
```bash
python3 -c "import json; d=json.load(open('/home/ckey/.claude.json')); print(d.get('projects',{}).get('/home/ckey',{}).get('mcpServers',{}).keys())"
```

### Policy-Managed Servers

System-installed, read-only. Config at
`/usr/lib/engineering-ai-suite/resources/apr_setup/claude/.mcp.json`.

| Server | Purpose | Notes |
|--------|---------|-------|
| `github-mcp` | GitHub (tool-filtered) | Read-only subset of tools |
| `jira-internal` | OnTrack internal | Do not use — token is a placeholder, no working auth |
| `codegen` | AMD code generation | — |
| `sourcegraph` | AMD Sourcegraph | — |
| `similar-tickets` | Similar ticket search | — |

### User-Added Servers

Stored in `~/.claude.json`. Must be set up manually on a new machine.

| Server | Platform Endpoint | Auth |
|--------|------------------|------|
| `atlassian-gateway` | `mcp/atlassian_gateway/mcp` | OAuth via `/mcp` (AMD Jira only) |
| `ontrack-internal` | `mcp/internal_atlassian` | Token PAT from `ontrack-internal.amd.com` |
| `github-ckey-xlnx` | `mcp/github_new` | PAT for `ckey-xlnx` on `github.com` |
| `github-ckey-amdeng` | `mcp/github_new` | PAT for `ckey_amdeng` on `github.com` |
| `github-ckey-xlnx-ghe` | `mcp/gitenterprise` | PAT for `ckey` on `gitenterprise.xilinx.com` |

### Setup Commands (new machine)

```bash
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
jira_get_issue(issue_key="FWDEV-1234")        # via atlassian-gateway

# OnTrack internal
jira_get_issue(issue_key="PROJ-123")          # via ontrack-internal
```

---

## Cline MCP Configuration

**Config file:** `~/.dotfiles/.cline/data/settings/cline_mcp_settings.json`
(symlinked to `~/.cline/data/settings/cline_mcp_settings.json`)

All Cline servers are local stdio servers from the `cline-mcp` repo
(`/home/ckey/hg/cline-mcp`).

| Server | Source | Covers |
|--------|--------|--------|
| `jira` | `cline-mcp/jira-server` | OnTrack internal and external (no AMD cloud Jira) |
| `reviewboard` | `cline-mcp/reviewboard-server` | `reviewboard.amd.com` |
| `mcp-cli-exec` | `cline-mcp/mcp-cli-exec` | Shell command execution fallback |

AMD cloud Jira (`amd.atlassian.net`) and Pensando are not configured for Cline.

### Building Local Servers

All local servers are TypeScript-based:

```bash
cd /home/ckey/hg/cline-mcp/<server-name>
npm install
npm run build
```

The build output goes to the `build/` directory and is referenced in the Cline config.

### Example Tool Calls

```
# OnTrack internal (Cline)
get_issue(instance="ontrack_internal", issue_key="PROJ-123")

# OnTrack external (Cline) — no token configured yet
get_issue(instance="ontrack_external", issue_key="PROJ-456")
```
