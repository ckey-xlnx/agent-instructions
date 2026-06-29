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

## Other Infrastructure Information

(Additional infrastructure knowledge will be added here as needed, such as:
- Build system configurations
- CI/CD pipeline details
- Development environment setup
- Testing infrastructure
- Deployment procedures)
