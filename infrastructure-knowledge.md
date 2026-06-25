# Infrastructure Knowledge

This document contains general infrastructure information that applies across projects, including service mappings, tool configurations, and other operational knowledge.

## GitHub Instance and Repository Mapping

Multiple GitHub instances are used across different projects. SSH access is configured
in `~/.ssh/config` with per-instance identity files.

### GitHub Instances

| Instance URL | Username | SSH Host | MCP (Claude Code) | MCP (Cline) | Notes |
|-------------|----------|----------|-------------------|-------------|-------|
| `https://github.com` | `ckey-xlnx` | `github.com` | `github-ckey-xlnx` | not configured | Personal/Xilinx account |
| `https://github.com` | `ckey_amdeng` | `amdeng%github.com` | `github-ckey-amdeng` | not configured | AMD org account on public GitHub |
| `https://github.com` | `cjk32` | `cjk32%github.com` | none | none | Personal account |
| `https://gitenterprise.xilinx.com` | `ckey` | `gitenterprise.xilinx.com` | `github-ckey-xlnx-ghe` | not configured | Xilinx GHE; AMD platform `gitenterprise` endpoint |
| `https://github.amd.com` | `ckey` | `github.amd.com` | none | none | AMD GHE; no AMD platform endpoint |
| `https://er.github.amd.com` | `ckey` | `er.github.amd.com` | none | none | AMD GHE (external-restricted); endpoint URL unknown |

### Repository to Instance Mapping

**`https://github.com` — `ckey-xlnx`:**
- `dotfiles`, `agent-instructions`, `vscode-workspaces`, `cline-mcp`
- `util`, `btl_extract`, `model` (simlater), `ifoe_ss_model` (personal fork/copy)

**`https://github.com` — `ckey_amdeng` (AMD orgs):**
- `AMD-SW-Simnow/simnow`
- `AMD-GOQDIAGS/diag_tng`
- `AMD-SLAI/SLAI.Cline`

**`https://github.com` — `pensando` org (read access):**
- `mpifoe-fw` (mirror; origin is `github.amd.com`), `ifoe-ts`, `ifoe-emu-chip`
- `ifoe-arch-model`, `ifoe-drv`, `rtos-sw`, `rtos-shared`, `ualoe_config_tests`
- `diag_tng` (upstream of AMD fork)

**`https://github.com` — `Xilinx-CNS` org:**
- `smartnic-runbench`, `xcb-serverpower-config`, `ol-git-helpers`

**`https://gitenterprise.xilinx.com` — `SmartNIC` org:**
- `smartnic_fw`, `smartnic_tools`, `smartnic_registry`, `smartnic_hwdefs`
- `smartnic_internal_tools`, `x2_fw`, `dcs-arch`, `ksb_apu_fw`
- `cdxconf` (under `ckey/`)

**`https://github.amd.com` — `PFO` org:**
- `mpifoe-fw` (origin), `mpio`, `asp-fmc`

**`https://er.github.amd.com` — `PFO` / `NTSG` orgs:**
- `amd-tee3.0`, `sw-security-tools`, `dcgpu-esid` (PFO)
- `ifoe_ss_model` (NTSG — origin)

**`https://github.com` — `cjk32` (personal):**
- `cobra`

## Jira Instance Mapping

Multiple Jira instances are used across different projects. When referencing Jira tickets or using Jira tools, use the following mapping:

### Project to Instance Mapping

**Pensando Atlassian Instance** (`pensando`)
- URL: https://pensando.atlassian.net
- Projects:
  - **IFOESW-*** - IFOE Software (legacy and Oktek interaction)

**OnTrack Internal Instance** (`ontrack_internal`)
- URL: https://ontrack-internal.amd.com
- Purpose: Normal AMD internal tickets
- Projects:
  - (Internal AMD projects not yet migrated to cloud)
- Note: Many projects have been migrated to the cloud AMD Atlassian instance
  (`amd`), including **FWDEV-***. Check the `amd` instance first for projects
  that were historically here.

**OnTrack External Instance** (`ontrack_external`)
- URL: https://ontrack.amd.com
- Purpose: Tickets shared with AMD customers/suppliers
- Projects:
  - (Customer/supplier collaboration projects)

**AMD Atlassian Instance** (`amd`)
- URL: https://amd.atlassian.net
- Projects:
  - **DMFPMSN-*** - AMD-specific project
  - **FWDEV-*** - Firmware Development (migrated from `ontrack_internal`)
  - (Other projects migrated to cloud from `ontrack_internal`)

### Projects Requiring Investigation

- **SIEEMU-*** - Instance location to be determined

### Usage Notes

- Check the ticket prefix to determine which Jira instance and MCP server to use
- For automated reviews, check the ticket prefix to determine the correct instance
- If you encounter a project prefix not listed here, ask for clarification on which instance to use

### MCP Server to Use by Instance

| Instance | MCP Server (Claude Code) | MCP Server (Cline) |
|----------|--------------------------|---------------------|
| `amd.atlassian.net` | `atlassian-gateway` (OAuth, auto-refresh) | not configured |
| `pensando.atlassian.net` | none (AMD platform hardwired to AMD Jira) | none |
| `ontrack-internal.amd.com` | `ontrack-internal` (Token PAT) | `jira` (local, `get_issue` with `instance="ontrack_internal"`) |
| `ontrack.amd.com` | none | `jira` (local, `get_issue` with `instance="ontrack_external"`, no token yet) |

**Note on AMD platform Atlassian endpoints:** Both `cloud_atlassian` and `atlassian_gateway`
are hardwired to `amd.atlassian.net` server-side. `atlassian_gateway` is preferred as it
uses OAuth with auto-refresh. Pensando requires a separately deployed platform instance —
no working path currently exists.

### Example Usage

```
# Fetch an AMD cloud Jira ticket (Claude Code)
jira_get_issue(issue_key="FWDEV-1234")                          # via atlassian-gateway

# Fetch an OnTrack internal ticket (Claude Code)
jira_get_issue(issue_key="PROJ-123")                            # via ontrack-internal

# Fetch an OnTrack internal ticket (Cline)
get_issue(instance="ontrack_internal", issue_key="PROJ-123")    # via jira
```

## MCP Server Configuration

### Overview

MCP servers are configured for two clients: **Claude Code** (CLI) and **VS Code Cline**. The
goal is to use official AMD MCP platform servers (`mcp-platform.amd.com`) wherever possible,
and only fall back to local custom servers for services not on the platform.

### AMD MCP Platform Endpoints

| Endpoint | Purpose | Notes |
|----------|---------|-------|
| `https://mcp-platform.amd.com/mcp/atlassian_gateway/mcp` | AMD cloud Jira (`amd.atlassian.net`) | OAuth (Claude Code). Hardwired to AMD Jira server-side |
| `https://mcp-platform.amd.com/mcp/internal_atlassian` | OnTrack internal (`ontrack-internal.amd.com`) | Token PAT in Authorization header |
| `https://mcp-platform.amd.com/mcp/github_new` | Public GitHub (`github.com`) | Bearer token (PAT); add once per account |
| `https://mcp-platform.amd.com/mcp/gitenterprise` | Xilinx GHE (`gitenterprise.xilinx.com`) | Bearer token (PAT) |
| `https://mcp-platform.amd.com/mcp/cloud_atlassian/` | AMD cloud Jira (Basic auth alternative) | Basic auth (email:token); hardwired to AMD Jira |

### Claude Code MCP Configuration

**Config file:** `~/.claude.json` (user-scope `mcpServers` key)

**Gotcha:** `~/.claude.json` also stores project-scoped servers under a `projects` key.
If a server appears under both user scope and a project scope, the project entry takes
precedence and may lack auth credentials, showing as "connected · no tools". Check for
duplicates with:
```bash
python3 -c "import json; d=json.load(open('/home/ckey/.claude.json')); print(d.get('projects',{}).get('/home/ckey',{}).get('mcpServers',{}).keys())"
```

**Policy-managed servers** (system-installed, read-only,
`/usr/lib/engineering-ai-suite/resources/apr_setup/claude/.mcp.json`):
- `github-mcp` — GitHub, tool-filtered
- `jira-internal` — OnTrack Jira (policy-managed, do not use — no working auth)
- `codegen`, `sourcegraph`, `similar-tickets`

**User-added servers** (stored in `~/.claude.json`, require manual setup on a new machine):

| Server | Endpoint | Auth |
|--------|----------|------|
| `atlassian-gateway` | `mcp-platform.amd.com/mcp/atlassian_gateway/mcp` | OAuth via `/mcp` (AMD Jira only) |
| `ontrack-internal` | `mcp-platform.amd.com/mcp/internal_atlassian` | Token PAT from `ontrack-internal.amd.com` |
| `github-ckey-xlnx` | `mcp-platform.amd.com/mcp/github_new` | PAT for `ckey-xlnx` on `github.com` |
| `github-ckey-amdeng` | `mcp-platform.amd.com/mcp/github_new` | PAT for `ckey_amdeng` on `github.com` |
| `github-ckey-xlnx-ghe` | `mcp-platform.amd.com/mcp/gitenterprise` | PAT for `ckey` on `gitenterprise.xilinx.com` |

**To recreate on a new machine:**

```bash
# AMD cloud Jira via OAuth gateway — then authenticate via /mcp
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

After adding `atlassian-gateway`, open Claude Code and run `/mcp` to authenticate via OAuth.

### VS Code Cline MCP Configuration

**Config file:** `~/.dotfiles/.cline/data/settings/cline_mcp_settings.json`
(symlinked to `~/.cline/data/settings/cline_mcp_settings.json`)

**Servers:**

| Server | Type | Source/URL |
|--------|------|-----------|
| `jira` | stdio (local) | `/home/ckey/hg/cline-mcp/jira-server` (OnTrack instances only; no AMD cloud) |
| `reviewboard` | stdio (local) | `/home/ckey/hg/cline-mcp/reviewboard-server` |
| `mcp-cli-exec` | stdio (local) | `/home/ckey/hg/cline-mcp/mcp-cli-exec` |

AMD cloud Jira and Pensando are not configured for Cline.

### Building Local MCP Servers

Local servers (reviewboard, mcp-cli-exec) in the `cline-mcp` repo are TypeScript-based:

```bash
cd /home/ckey/hg/cline-mcp/<server-name>
npm install
npm run build
```

The build output goes to the `build/` directory and is referenced in the Cline config.

## VS Code Server Storage Cleanup

The home directory quota can be exhausted by accumulated VS Code Server data under
`~/.vscode-server/` (total can easily reach 6GB+). The main offenders are described
below, along with how to identify and clean them up safely.

### Directory Overview

```
~/.vscode-server/
├── cli/
│   └── servers/              ← Full VS Code Server installations (~500MB each)
├── data/
│   ├── CachedExtensionVSIXs/ ← Downloaded VSIX packages (~300MB)
│   └── User/
│       ├── workspaceStorage/ ← Per-workspace extension state (can be multi-GB)
│       └── globalStorage/    ← Per-extension global state
├── extensions/               ← Installed server-side extensions
└── code-<hash>               ← CLI tunnel binaries (~30MB each)
```

### Identifying the Current Server Version

The **active/current** server commit hash is recorded as the first entry in:

```bash
cat ~/.vscode-server/cli/servers/lru.json
```

The first entry (e.g. `Stable-fcf604774b9f2674b473065736ee75077e256353`) is the most
recently used server and must be kept. All others are safe to delete.

### cli/servers/ — Old Server Installations

Each `Stable-<hash>/` directory is a complete VS Code Server installation (~450–510MB).
VS Code does not automatically prune these. Only keep the current one (first in lru.json).

Directories ending in `.staging` are temporary install artefacts and are always safe
to delete regardless of which version they belong to.

```bash
# Check sizes and dates to identify old ones
ls -la ~/.vscode-server/cli/servers/
du -sh ~/.vscode-server/cli/servers/*

# Delete all old server dirs and all staging dirs, keeping only the current one.
# Replace CURRENT_HASH with the hash from lru.json before running.
CURRENT="Stable-CURRENT_HASH"
for d in ~/.vscode-server/cli/servers/Stable-*; do
    name=$(basename "$d")
    if [[ "$name" != "$CURRENT" ]]; then
        echo "Removing: $name"
        rm -rf "$d"
    fi
done
```

### code-<hash> — CLI Tunnel Binaries

Several `code-<hash>` binaries accumulate in `~/.vscode-server/` (~30MB each). Keep
only the one whose hash matches the current server (from lru.json); delete the rest.

```bash
# List all CLI binaries with sizes and dates
ls -lh ~/.vscode-server/code-*

# Delete old ones (replace CURRENT_HASH with hash from lru.json)
CURRENT_HASH="fcf604774b9f2674b473065736ee75077e256353"  # example
for f in ~/.vscode-server/code-*; do
    if [[ "$f" != *"$CURRENT_HASH"* ]]; then
        echo "Removing: $f"
        rm -f "$f"
    fi
done
```

Note: `.nfs00000000...` files alongside binaries indicate NFS-locked open file
handles. These are safe to leave — they disappear when the holding process exits.

### data/CachedExtensionVSIXs/ — Cached Extension Packages

Downloaded VSIX packages are cached here and can be deleted safely; VS Code will
re-download them if needed:

```bash
du -sh ~/.vscode-server/data/CachedExtensionVSIXs/*  # see what's there
rm -rf ~/.vscode-server/data/CachedExtensionVSIXs/*
```

### data/User/workspaceStorage/ — Per-Workspace State

This is often the largest area (can be 2GB+). Large entries here are typically
GitHub Copilot Chat history. To identify what each opaque directory belongs to:

```bash
du -sh ~/.vscode-server/data/User/workspaceStorage/* | sort -rh | head -10
# Check what extension owns the large ones:
du -sh ~/.vscode-server/data/User/workspaceStorage/<hash>/*
```

Entries without a `workspace.json` are for workspaces that no longer exist and
are safe to delete. Entries with large `GitHub.copilot-chat/` subdirectories are
Copilot Chat history caches and can be cleared if the history is not needed.

### Typical Space Recovery

| Item | Typical Size | Notes |
|------|-------------|-------|
| Old `cli/servers/` directories | 450–510 MB each | Keep only current |
| `.staging` directories | 35–350 MB each | Always safe to delete |
| Old `code-<hash>` binaries | ~30 MB each | Keep only current |
| `CachedExtensionVSIXs/` | 100–300 MB total | Safe to clear entirely |
| `workspaceStorage/` Copilot Chat | 400 MB–1.2 GB each | Optional — loses chat history |

## Other Infrastructure Information

(Additional infrastructure knowledge will be added here as needed, such as:
- Build system configurations
- CI/CD pipeline details
- Repository locations and purposes
- Development environment setup
- Testing infrastructure
- Deployment procedures)
