# Infrastructure Knowledge

This document contains agent-agnostic infrastructure facts: which instances exist, what
they are for, and which projects/repos live on each. It does **not** describe how to access
these services via MCP — see `mcp-configuration.md` for that.

## GitHub

Multiple GitHub instances are used across different projects. SSH access is configured
in `~/.ssh/config` with per-instance identity files.

### Accounts

| Instance URL | Username | SSH Host | Org(s) | Notes |
|-------------|----------|----------|--------|-------|
| `https://github.com` | `ckey-xlnx` | `github.com` | personal, `Xilinx-CNS`, `pensando` (read) | Personal/Xilinx account |
| `https://github.com` | `ckey_amdeng` | `amdeng%github.com` | `AMD-SW-Simnow`, `AMD-GOQDIAGS`, `AMD-SLAI` | AMD org account on public GitHub |
| `https://github.com` | `cjk32` | `cjk32%github.com` | personal | Personal account, rarely used |
| `https://gitenterprise.xilinx.com` | `ckey` | `gitenterprise.xilinx.com` | `SmartNIC` | Xilinx GHE |
| `https://github.amd.com` | `ckey` | `github.amd.com` | `PFO` | AMD GHE; no AMD platform MCP endpoint |
| `https://er.github.amd.com` | `ckey` | `er.github.amd.com` | `PFO`, `NTSG` | AMD GHE (external-restricted) |

### Repository to Account Mapping

**`https://github.com` — `ckey-xlnx` (personal):**
- `dotfiles`, `agent-instructions`, `vscode-workspaces`, `cline-mcp`
- `util`, `btl_extract`, `model` (simlater), `ifoe_ss_model` (personal fork/copy)

**`https://github.com` — `ckey-xlnx` via `Xilinx-CNS` org:**
- `smartnic-runbench`, `xcb-serverpower-config`, `ol-git-helpers`

**`https://github.com` — `ckey-xlnx` via `pensando` org (read access):**
- `mpifoe-fw` (mirror; origin is `github.amd.com`), `ifoe-ts`, `ifoe-emu-chip`
- `ifoe-arch-model`, `ifoe-drv`, `rtos-sw`, `rtos-shared`, `ualoe_config_tests`
- `diag_tng` (upstream of AMD fork)

**`https://github.com` — `ckey_amdeng` via AMD orgs:**
- `AMD-SW-Simnow/simnow`
- `AMD-GOQDIAGS/diag_tng`
- `AMD-SLAI/SLAI.Cline`

**`https://github.com` — `cjk32` (personal):**
- `cobra`

**`https://gitenterprise.xilinx.com` — `SmartNIC` org:**
- `smartnic_fw`, `smartnic_tools`, `smartnic_registry`, `smartnic_hwdefs`
- `smartnic_internal_tools`, `x2_fw`, `dcs-arch`, `ksb_apu_fw`
- `cdxconf` (under `ckey/`)

**`https://github.amd.com` — `PFO` org:**
- `mpifoe-fw` (origin), `mpio`, `asp-fmc`

**`https://er.github.amd.com` — `PFO` / `NTSG` orgs:**
- `amd-tee3.0`, `sw-security-tools`, `dcgpu-esid` (PFO)
- `ifoe_ss_model` (NTSG — origin)

## Jira

Multiple Jira instances are used across different projects. Check the ticket prefix to
determine which instance a ticket belongs to. If a prefix is not listed here, ask for
clarification.

| Instance | URL | Purpose | Known Projects |
|----------|-----|---------|---------------|
| AMD cloud (`amd`) | `https://amd.atlassian.net` | Primary AMD internal Jira (cloud) | `FWDEV-*`, `DMFPMSN-*`, and others migrated from OnTrack |
| OnTrack Internal (`ontrack_internal`) | `https://ontrack-internal.amd.com` | AMD internal (legacy/unmigrated projects) | Various — check `amd` first for projects that were historically here |
| OnTrack External (`ontrack_external`) | `https://ontrack.amd.com` | Customer/supplier collaboration | Various |
| Pensando cloud (`pensando`) | `https://pensando.atlassian.net` | IFOE/Pensando-lineage work | `IFOESW-*` |

**Projects requiring investigation:**
- `SIEEMU-*` — instance location not yet determined

## ReviewBoard

| Instance | URL | Purpose |
|----------|-----|---------|
| AMD ReviewBoard | `https://reviewboard.amd.com` | Code review (primary) |

## Other Services

| Service | URL | Purpose |
|---------|-----|---------|
| Sourcegraph | `https://sourcegraph.amd.com` | AMD internal code search |
| AMD MCP Platform | `https://mcp-platform.amd.com` | Gateway for AMD-hosted MCP services (see `mcp-configuration.md`) |

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
- Development environment setup
- Testing infrastructure
- Deployment procedures)
