---
name: dev-knowledge-ss-emu-selfcontained-app
description: >
  The self-contained test-app variant of IFoE subsystem-emulation (ss-emu) — the
  ifoe_test flow that drives the model via ctb_cmd, using emu_eth_to_arch as the
  epgm app, launched with velocetool-ifoe. Covers how to form the velocetool
  command line for THIS variant: which structural args select it, and why
  --skip-rtl-rsync is required for the overlay workspace. Use when working on or
  running the self-contained ss-emu test app (ifoe_test), as opposed to the TE
  variant. For velocetool and its generic args see dev-knowledge-velocetool; for
  the grid job and results see infra-grid-atl / dev-knowledge-ss-emu-environment.
---

# Self-contained ss-emu test app (ifoe_test)

The **self-contained** ss-emu variant runs the IFoE arch model as a standalone
emulation test, driven through `ctb_cmd`, with `emu_eth_to_arch` as the epgm app.
This is the `ifoe_test` work area, as opposed to the TE variant.

This skill covers **only** what is specific to this variant when forming the
`velocetool-ifoe` command line. Everything generic is owned elsewhere — do not
duplicate:

- `velocetool-ifoe` itself, its full arg set, the environment-vs-structural arg
  split: `dev-knowledge-velocetool`
- The Slurm job it submits (state, cancel, `mi_veloce`): `infra-grid-atl`
- Runtime, the `--test` directory internals, where results / logs land:
  `dev-knowledge-ss-emu-environment`
- Deriving the `--rtl` er-release path from a workspace name, and er vs ner:
  `dev-workflow-ss-emu-workspace`
- Building the artifacts (`libdpi.so`, `emu_eth_to_arch`, `emu_eth_lb`):
  `build-ifoe-arch-model-ss-emu`
- ctctb / ctb_cmd routing into test code: `dev-knowledge-arch-model-transactors`
- The TE variant: `dev-knowledge-ss-emu-te`

## The command line

Run `velocetool-ifoe` (see `dev-knowledge-velocetool` for its path) with the
`setup` subcommand, **from inside the workspace directory** (the `EPGM_ifoe_ss_*`
model tree):

```
velocetool-ifoe --debug setup \
  --rtl <ER_RELEASE> \
  --skip-rtl-rsync \
  --allocID MI450_P1_VEL_AINIC_AP \
  --emu-list MUGELLO \
  --constraint RHEL8 \
  --partition mi_veloce \
  --test ifoe_emu_test_app \
  --libdpi          <WORKSPACE>/ifoe-arch-model/build/src/libs/libdpi.so \
  --run-epgm-binary <WORKSPACE>/ifoe-arch-model/build/src/apps/emu_eth_to_arch
```

Omit `--gui` for a batch run (GUI usage: to be documented separately). For the
meaning/categories of the environment args (`--allocID/--emu-list/--constraint/
--partition`) and generic args (`setup`, `--debug`, `--libdpi`, `--rtl`), see
`dev-knowledge-velocetool`.

## What is specific to this variant

The **structural** args that make the run *this* variant rather than another:

- `--test ifoe_emu_test_app` — selects this variant's test directory
  `chip/runtime/vulcano_ifoe/ifoe_emu_test_app` (siblings include
  `vulcano_sdp_socapi_test`, `te_test`). This is *the* choice that makes it the
  self-contained IFoE test app. What the directory's files do is owned by
  `dev-knowledge-ss-emu-environment`.
- `--run-epgm-binary .../emu_eth_to_arch` — this variant's epgm app. Use
  `emu_eth_to_arch` for the arch-model test; `emu_eth_lb` is the loopback app.
- `--libdpi .../ifoe-arch-model/build/src/libs/libdpi.so` — the arch model built
  from *this* workspace (per `build-ifoe-arch-model-ss-emu`), not a released lib.

## `--skip-rtl-rsync` is required here

For the overlay workspace this flag is **not optional**. Without it,
velocetool's `prepare_rundir` runs `rsync … {rtl} .`, copying the **er release**
into the work area; the run would then use the *released* `chip` / `_ip` / `_env`
instead of your overlay clones (and needlessly copy a huge tree). Always pass
`--skip-rtl-rsync` so the run uses the workspace's overlay repos. (Overlay repos:
`dev-knowledge-repo-ifoe-emu-overlays`; workspace layout:
`dev-workflow-ss-emu-workspace`.)
