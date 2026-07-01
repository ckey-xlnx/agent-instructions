---
name: dev-knowledge-velocetool
description: >
  velocetool-ifoe — the launcher that submits an IFoE ss-emu emulation job (it
  wraps run_emu and Slurm). Where it lives in version control, the common
  checked-out copy, and its arguments. Use when asked what
  velocetool / velocetool-ifoe is, where to get it, or how to invoke it / what
  its args mean. For the grid job it submits see infra-grid-atl; for the --test
  directory it selects see dev-knowledge-ss-emu-environment.
---

# velocetool-ifoe

`velocetool-ifoe` is the Python launcher that stands up an IFoE ss-emu emulation
session: it turns high-level options into a `run_emu` invocation and submits it
to Slurm (partition `mi_veloce`), returning a job id. (For the grid side see
`infra-grid-atl`.)

## Where it lives

- **Version control (source of truth):** `git@github.com:Xilinx-CNS/smartnic-vbu`
  (the tool is `velocetool-ifoe` at the repo root).
- **Recommended: a per-user clone** at `/proj/vulcano_dump2_ner/<user>/smartnic-vbu`.
  One clone, shared across all your ss-emu workspaces, that you own and can
  `git pull`. Create it once:

  ```bash
  cd /proj/vulcano_dump2_ner/<user>
  git clone git@github.com:Xilinx-CNS/smartnic-vbu.git
  ```

- **Do NOT rely on `/proj/vulcano_dump2_ner/common/smartnic-vbu`.** It is a stale
  shared snapshot: it **hardcodes `--partition=veloce`** and ignores the
  `--partition` argument, so submissions fail with `Invalid partition specified:
  veloce`. Upstream (and a fresh clone) honour `--partition` correctly.

> The **TE flow is different** — its `ifoe_emu_ci/scripts/checkout.sh` clones
> `smartnic-vbu` into the workspace itself and calls it by workspace-relative
> path. That is owned by the TE scripts; leave it alone. The per-user clone above
> is for the **self-contained** flow. See `dev-knowledge-ss-emu-te`.

## Arguments

Typical invocation shape:

```
velocetool-ifoe --debug setup \
  --rtl <er-release-path> \          # the er (RTL) release
  --skip-rtl-rsync \
  --allocID MI450_P1_VEL_AINIC_AP \  # Veloce allocation id
  --emu-list MUGELLO \               # emulator(s) to use
  --constraint RHEL8 \               # host constraint
  --partition mi_veloce \            # Slurm partition
  --test <name> \                    # selects the test directory (see below)
  --libdpi <path>/libdpi.so \        # shared object loaded into the primary emulation process
  --run-epgm-binary <path>/<app> \   # the application run on the epgm/VVED comodel host
  [--port <n>] [--gui]
```

- `setup` is the subcommand; `--debug` raises verbosity.
- **`--test <name>`** selects the test directory `chip/runtime/vulcano_ifoe/<name>`
  — the directory of files that control the run (`run.veloce`, `ctbconf.xml`, …).
  It becomes `run_emu -test <name>` under `-model vulcano_ifoe`, and is surfaced
  at run time as the `test_source` symlink. What those files do is documented in
  `dev-knowledge-ss-emu-environment` ("The test directory").
- **`--run-epgm-binary <app>`** names the application to run on the epgm/VVED
  comodel host; it surfaces downstream as the `epgm.app` CTB config option.
- **`--libdpi <path>`** is the shared object loaded into the primary emulation
  process.

For the exact, current option set, read the tool itself (it is short) rather than
trusting this list.

For deriving the `--rtl` er-release path from a workspace name, see
`dev-workflow-ss-emu-workspace` ("Deriving the er (RTL) path from a workspace").

### Environment vs structural arguments

When adapting the command line, distinguish two kinds of argument:

- **Environment (site/hardware/allocation) — known-good for atl+Veloce+MI450, but
  would change elsewhere, and override wrong-for-this-system tool defaults:**
  `--allocID MI450_P1_VEL_AINIC_AP`, `--emu-list MUGELLO` (physical emulator),
  `--constraint RHEL8` (Slurm node OS; MUGELLO needs it, the default is not RHEL8),
  `--partition mi_veloce`. See `infra-grid-atl` for the grid side.
- **Structural — identify the specific run** (`--test`, `--run-epgm-binary`,
  `--libdpi`, `--rtl`). These are workspace/variant-specific; the caller must set
  them correctly. For the self-contained variant's structural values see
  `dev-knowledge-ss-emu-selfcontained-app`.
