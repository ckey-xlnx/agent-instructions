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

- **Version control:** `git@github.com:Xilinx-CNS/smartnic-vbu` (the tool is
  `velocetool-ifoe` at the repo root). This is the source of truth; clone it if
  you need the latest or want to modify the tool.
- **Common checked-out copy:**
  `/proj/vulcano_dump2_ner/common/smartnic-vbu/velocetool-ifoe`. Usable without
  cloning your own, but note it is a shared convenience copy that may not be
  actively kept up to date — if in doubt, clone from version control above.

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
