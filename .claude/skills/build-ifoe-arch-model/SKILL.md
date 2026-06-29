---
name: build-ifoe-arch-model
description: >
  Configure and build ifoe-arch-model on subsystem emulation (meson + ninja +
  Makefile post-processing), including environment setup and rebuild rules.
  Use when asked to build ifoe-arch-model, build the arch model on subsystem
  emulation, run etx-meson/etx-ninja/emu-post-ninja, or figure out what to
  rebuild after changing model/chip/ip/env code.
---

# Build ifoe-arch-model (subsystem emulation)

**Build tool**: Meson + Ninja, with additional Makefile-based post-processing.

This covers the semi-standalone build of ifoe-arch-model on a subsystem
emulation model. (Use with the TE / Test Environment is a separate flow.)

## Subsystem emulation setup

Directory structure:
- Clone ifoe-arch-model into a copy of a released subsystem emulation model.
  Example: `/proj/vulcano_dump2_ner/ckey/ifoe_test/EPGM_ifoe_ss_156478_v3_xcb_20250822T144931Z/ifoe-arch-model`
- Alongside ifoe-arch-model, clone additional repos that replace content within
  the released model:
  - `chip` repo at `${MODEL}/chip`
  - `_ip` repo at `${MODEL}/_ip`
  - `_env` repo at `${MODEL}/_env`
- These additional repos have a branch per model release
  (e.g. `LSE_Design/EPGM_ifoe_ss_156478_v3_xcb_20250822T144931Z`).
- ifoe-arch-model itself follows a normal branch policy.

`${MODEL}` below is the path to the subsystem emulation model.

## Environment setup

Prerequisites: must use bash (not zsh), and set up the environment before building.

```bash
# Step 0: Use bash
/tool/pandora64/bin/bash

# Step 1: Source initialization script
source /proj/verif_release_ro/cbwa_initscript/current/cbwa_init.bash

# Step 2: Boot environment for emulation
bootenv -u emu_epgm -C ${MODEL}
```

Example with full path:

```bash
bootenv -u emu_epgm -C /proj/vulcano_dump2_ner/ckey/ifoe_test/EPGM_ifoe_ss_156478_v3_xcb_20250822T144931Z
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
