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
| `https://gitenterprise.xilinx.com` | `ckey` | `gitenterprise.xilinx.com` | none | none | Xilinx GitHub Enterprise |
| `https://github.amd.com` | `ckey` | `github.amd.com` | none | none | AMD GitHub Enterprise |
| `https://er.github.amd.com` | `ckey` | `er.github.amd.com` | none | none | AMD GHE (external-restricted) |

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
| `amd.atlassian.net` | `amd-atlassian` (AMD platform, full tools) | `amd-atlassian` (AMD platform, full tools) |
| `pensando.atlassian.net` | `pensando-atlassian` (AMD platform, full tools) | `pensando-atlassian` (AMD platform, full tools) |
| `ontrack-internal.amd.com` | not supported | `jira` (local, `get_issue` with `instance="ontrack_internal"`) |
| `ontrack.amd.com` | not supported | `jira` (local, `get_issue` with `instance="ontrack_external"`) |

**Important:** Both `amd-atlassian` and `pensando-atlassian` connect to the same AMD platform
endpoint (`cloud_atlassian/`) but are authenticated to different Atlassian instances via OAuth.
OnTrack tickets are only accessible via the `jira` local server in Cline.

### Example Usage

```
# Fetch a FWDEV or DMFPMSN ticket (amd.atlassian.net)
jira_get_issue(issue_key="FWDEV-1234")                          # via amd-atlassian

# Fetch a Pensando ticket
jira_get_issue(issue_key="IFOESW-205")                          # via pensando-atlassian

# Fetch an OnTrack internal ticket (Cline only)
get_issue(instance="ontrack_internal", issue_key="PROJ-123")    # via jira

# Fetch an OnTrack external ticket (Cline only)
get_issue(instance="ontrack_external", issue_key="PROJ-123")    # via jira
```

## MCP Server Configuration

### Overview

MCP servers are configured for two clients: **Claude Code** (CLI) and **VS Code Cline**. The
goal is to use official AMD MCP platform servers (`mcp-platform.amd.com`) wherever possible,
and only fall back to local custom servers for services not on the platform.

### AMD MCP Platform Endpoints

| Endpoint | Purpose | Notes |
|----------|---------|-------|
| `https://mcp-platform.amd.com/mcp/cloud_atlassian/` | AMD and Pensando Jira | OAuth — instance (`amd.atlassian.net` or `pensando.atlassian.net`) chosen at auth time |
| `https://mcp-platform.amd.com/mcp/github_new` | Public GitHub (`github.com`) | Bearer token (PAT); add once per account |
| `https://mcp-platform.amd.com/mcp/internal_atlassian` | OnTrack (`ontrack-internal.amd.com`) | Policy-only, not used (no working auth) |

### Claude Code MCP Configuration

**Config file:** `~/.claude.json` (user-scope `mcpServers` key)

**Policy-managed servers** (system-installed, read-only,
`/usr/lib/engineering-ai-suite/resources/apr_setup/claude/.mcp.json`):
- `github-mcp` — GitHub, tool-filtered
- `jira-internal` — OnTrack Jira (policy-managed, do not use — no working auth)
- `codegen`, `sourcegraph`, `similar-tickets`

**User-added servers** (stored in `~/.claude.json`, require manual setup on a new machine):

| Server | Endpoint | Auth |
|--------|----------|------|
| `amd-atlassian` | `mcp-platform.amd.com/mcp/cloud_atlassian/` | OAuth via `/mcp` — choose AMD instance |
| `pensando-atlassian` | `mcp-platform.amd.com/mcp/cloud_atlassian/` | OAuth via `/mcp` — choose Pensando instance |
| `github-ckey-xlnx` | `mcp-platform.amd.com/mcp/github_new` | PAT for `ckey-xlnx` on `github.com` |
| `github-ckey-amdeng` | `mcp-platform.amd.com/mcp/github_new` | PAT for `ckey_amdeng` on `github.com` |

**To recreate on a new machine:**

```bash
# Add AMD and Pensando Atlassian — authenticate each via /mcp, choosing the correct instance
claude mcp add --transport http "amd-atlassian" \
  "https://mcp-platform.amd.com/mcp/cloud_atlassian/" -s user
claude mcp add --transport http "pensando-atlassian" \
  "https://mcp-platform.amd.com/mcp/cloud_atlassian/" -s user

# Add GitHub accounts (PATs stored separately — do not commit)
claude mcp add --transport http "github-ckey-xlnx" \
  "https://mcp-platform.amd.com/mcp/github_new" -s user \
  --header "Authorization: Bearer <ckey-xlnx-pat>"
claude mcp add --transport http "github-ckey-amdeng" \
  "https://mcp-platform.amd.com/mcp/github_new" -s user \
  --header "Authorization: Bearer <ckey-amdeng-pat>"
```

After adding each Atlassian server, open Claude Code and run `/mcp` to authenticate via
OAuth. The OAuth page asks "use app on" with options `amd.atlassian.net` and
`pensando.atlassian.net` — choose `amd.atlassian.net` for `amd-atlassian` and
`pensando.atlassian.net` for `pensando-atlassian`.

### VS Code Cline MCP Configuration

**Config file:** `~/.dotfiles/.cline/data/settings/cline_mcp_settings.json`
(symlinked to `~/.cline/data/settings/cline_mcp_settings.json`)

**Servers:**

| Server | Type | Source/URL |
|--------|------|-----------|
| `amd-atlassian` | streamableHttp | `mcp-platform.amd.com/mcp/cloud_atlassian/` (AMD instance) |
| `pensando-atlassian` | streamableHttp | `mcp-platform.amd.com/mcp/cloud_atlassian/` (Pensando instance) |
| `jira` | stdio (local) | `/home/ckey/hg/cline-mcp/jira-server` (OnTrack only) |
| `reviewboard` | stdio (local) | `/home/ckey/hg/cline-mcp/reviewboard-server` |
| `mcp-cli-exec` | stdio (local) | `/home/ckey/hg/cline-mcp/mcp-cli-exec` |

Cline handles OAuth for the `streamableHttp` servers itself.

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
