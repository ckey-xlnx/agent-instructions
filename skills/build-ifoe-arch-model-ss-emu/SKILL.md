---
name: build-ifoe-arch-model-ss-emu
description: >
  Build ifoe-arch-model in its self-contained subsystem-emulation
  configuration (meson + ninja + Makefile post-processing), including
  environment setup and rebuild rules. Use when asked to build ifoe-arch-model
  on subsystem emulation / ss-emu, run etx-meson/etx-ninja/emu-post-ninja, or
  figure out what to rebuild after changing model/chip/ip/env code. (This is
  the subsystem-emulation build path only; building ifoe-arch-model as part of
  TE is a separate flow not covered here.) Runs at the atl site.
---

# Build ifoe-arch-model (subsystem emulation)

**Build tool**: Meson + Ninja, with additional Makefile-based post-processing.

## Site: atl

Run everything below on an **atl** machine. The model tree and the `/proj/...`
paths in this skill are atl-local (i.e. `atl:/proj/...`; see `infra-amd-sites`).
A "not found" path usually means you are not on an atl host.

ifoe-arch-model can be built in more than one configuration; this skill covers
the **self-contained subsystem-emulation (ss-emu)** build only. Building it as
part of the TE (Test Environment) is a separate flow not documented here.

## Prerequisite: a workspace

This build runs inside an ss-emu **workspace** — a writable copy of a released
model with the `chip`/`_ip`/`_env` overlays swapped for their git clones and
`ifoe-arch-model` cloned alongside. Creating one (release sync, overlay swap,
arch-model clone) is a separate procedure: see `dev-workflow-ss-emu-workspace`.

Below, `$MODEL` is that workspace's model directory (the `$DESIGN` dir from the
workspace skill, i.e. `$WORK/$DESIGN`), and you build from the `ifoe-arch-model`
directory cloned inside it.

## Environment setup

Prerequisites: must use bash (not zsh), and set up the environment before building.

```bash
# Step 0: Use bash
/tool/pandora64/bin/bash

# Step 1: Source initialization script
source /proj/verif_release_ro/cbwa_initscript/current/cbwa_init.bash

# Step 2: Boot environment for emulation ($MODEL = your workspace's model dir)
bootenv -u emu_epgm -C "$MODEL"
```

## Build steps

Run from within the ifoe-arch-model directory:

```bash
# Step 3: Configure with meson
./etx-meson.sh

# Step 4: Build with ninja
./etx-ninja.sh

# Step 5: Post-build processing (builds additional libraries)
./emu-post-ninja.sh
```

Build outputs:
- `build/src/libs/libdpi.so` — loaded by emulation software
- `build/src/apps/emu_eth_to_arch` — connects emulation ethernet transactor to architecture model copies
- `build/src/apps/emu_eth_lb` — loops back traffic from the emulation ethernet transactor

## Rebuild rules (what to rebuild when)

1. **Meson/Ninja-built code changed**: rerun both `./etx-ninja.sh` AND `./emu-post-ninja.sh`.
2. **Makefile-built libraries changed** (invoked from emu-post-ninja.sh): rerun only `./emu-post-ninja.sh`.
3. **Code in `_chip`, `_ip`, or `_env` changed**: delete `${MODEL}/build`, then rerun `./emu-post-ninja.sh`. (Proper dependency tracking from those files to `libemu_chip.a` is infeasible.)

Notes:
- The build process is convoluted, mixing meson/ninja with Makefiles.
- Dependency tracking is incomplete; manual intervention is required for certain changes.
