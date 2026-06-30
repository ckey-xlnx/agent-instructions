---
name: build-asp
description: >
  Build the ASP (Application Security Processor) firmware from the amd-tee3.0
  repo using the PSP RISC-V toolchain. Use when asked to build ASP firmware,
  build amd-tee3.0, build drv_preesid, build the secure-processor / PSP
  firmware, or work on P2PSync. Covers prerequisites, build commands, outputs,
  and the Zephyr-SDK ABI pitfall.
---

# Build ASP (amd-tee3.0) Firmware

The ASP (Application Security Processor) firmware lives in the `amd-tee3.0`
repository and is built with the PSP RISC-V toolchain (NOT the Zephyr SDK).

**Repository location**: `/proj/smartnic/xcb/ckey/ifoe-fw/amd-tee3.0`

## Prerequisites

1. **PSP Security Tools**:
   ```bash
   # Clone from: ssh://ckey@git.amd.com:29418/sw_security/er/tools
   # The repo was renamed to sw-security-tools
   # Expected location: /proj/smartnic/xcb/ckey/ifoe-fw/sw-security-tools
   ```

2. **RISC-V Toolchain** — the official PSP toolchain (not Zephyr SDK):
   - Expected location: `${PSP_TOOLS_DIR}/../toolchain/riscv/linux/riscv32-toolchain-linux/bin/`
   - Required ABI: `-march=rv32imafc -mabi=ilp32f` (single-float)
   - The Zephyr SDK's `riscv64-zephyr-elf` has an ABI mismatch (soft-float vs single-float)

3. **Git submodules** — initialize before building:
   ```bash
   cd /proj/smartnic/xcb/ckey/ifoe-fw/amd-tee3.0
   git submodule update --init --recursive
   ```

## Build command

Build for MI450 (drv_preesid):

```bash
cd /proj/smartnic/xcb/ckey/ifoe-fw/amd-tee3.0/amd_tee_dr/drv_preesid
PSP_TOOLS_DIR=/proj/smartnic/xcb/ckey/ifoe-fw/sw-security-tools BUILD=MI450 make
```

Supported build targets:
- `MI450`, `AT2`, `ARCD` (DGPU)
- `WH`, `RH` (CPU)
- `MTP` (Workstation)
- `MDS1`, `OLR` (APU)

## Output

- `obj/${BUILD}/` — object files
- `bin/` — final binary (e.g. `TypeId0x6A_drv_preesid_MI450.bin`)

## P2P Sync source files

The P2PSync code that handles multi-socket coordination:
- Main implementation: `amd_tee_ddk/shared_src/p2p_sync/p2p_sync.c`
- Macros: `amd_tee_ddk/inc/multicore_chiplet_defines.h` (`GET_TOTAL_NODES`, `IS_MULTI_NODE`)

## Deployment: ASP FW is NOT the same as mpifoe FW

- **mpifoe FW** (`mpifoe_fw.bin/.hbin`): runs on the IFoE management CPU (Zephyr
  RTOS), merged with `ifwi-bosh`. Can be replaced at SimNow launch time.
- **ASP FW** (`drv_preesid`): runs on the PSP/ASP secure processor, built into
  the IFWI at AGESA build time.

For ASP firmware changes (like P2PSync):
1. The ASP firmware is baked into the IFWI image during the AGESA/BIOS build.
2. It cannot be replaced at SimNow launch time like mpifoe FW.
3. It requires rebuilding the full IFWI with the modified ASP components.

For reference, mpifoe FW changes are injected at launch by the dev-workflow-simnow-launch
script when using `.hbin`:
```bash
$SIMNOW_SHARED/IFWI/ifwi-bosh $IFWI $FWNAME $SIMNOW_WORKDIR/IFWI.boshed.sbin
```

## Known issue: Zephyr SDK incompatibility

The Zephyr SDK's RISC-V toolchain cannot be used directly for ASP firmware due
to an ABI mismatch:

```
error: can't link soft-float modules with single-float modules
```

Solution: use the official PSP RISC-V toolchain from the sw-security-tools repo.
