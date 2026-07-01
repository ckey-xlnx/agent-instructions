---
name: dev-knowledge-velocetool
description: >
  velocetool-ifoe — the launcher that submits an IFoE ss-emu emulation job (it
  wraps run_emu and Slurm). Where it lives in version control, the shared
  checked-out copy that should work, and its arguments. Use when asked what
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
  `velocetool-ifoe` at the repo root).
- **Shared checked-out copy that should work:**
  `/proj/vulcano_dump2_ner/ckey/ifoe_test/smartnic-vbu/velocetool-ifoe`. Use this
  rather than cloning your own unless you need to modify the tool.

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
