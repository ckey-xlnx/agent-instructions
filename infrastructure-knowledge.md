# Infrastructure Knowledge

This document contains general infrastructure information that applies across projects, including service mappings, tool configurations, and other operational knowledge.

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

- When using the Jira MCP server tools, specify the instance explicitly based on the project prefix
- For automated reviews, check the ticket prefix to determine the correct instance
- If you encounter a project prefix not listed here, ask for clarification on which instance to use

### Example Usage

```
# Fetch an IFOESW ticket from Pensando
get_issue(instance="pensando", issue_key="IFOESW-205")

# Search for FWDEV tickets (migrated to cloud AMD instance)
search_issues(instance="amd", jql="project = FWDEV AND status = Open")

# Fetch a DMFPMSN ticket from AMD
get_issue(instance="amd", issue_key="DMFPMSN-12345")
```

## MCP Server Configuration

### Configuration File Location

The Cline MCP server configuration is located at:
```
~/.vscode-server/data/User/globalStorage/slai.claude-dev/settings/cline_mcp_settings.json
```

### Available MCP Servers

**reviewboard** - ReviewBoard integration
- Source: `/home/ckey/hg/cline-mcp/reviewboard-server`
- Provides tools for interacting with ReviewBoard code review system
- Auto-approved tools: list_review_requests, get_review_request, get_review_diffs, get_diff_content, get_reviews, get_review_comments, get_review_replies, get_diff_files

**jira** - Jira integration
- Source: `/home/ckey/hg/cline-mcp/jira-server`
- Provides tools for interacting with multiple Jira instances
- Auto-approved tools: get_issue

**mcp-cli-exec** - Terminal/CLI command execution
- Source: `/home/ckey/hg/cline-mcp/mcp-cli-exec`
- Provides tools for executing shell commands with structured output
- Tools: cli-exec-raw, cli-exec
- Timeout: 300 seconds (5 minutes)

### Building MCP Servers

MCP servers in the cline-mcp submodule are TypeScript-based and require compilation:

```bash
cd /home/ckey/hg/cline-mcp/<server-name>
npm install
npm run build
```

The build output goes to the `build/` directory and is referenced in the MCP configuration.

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
