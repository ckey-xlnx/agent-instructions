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

This build needs three things checked out **as siblings in one directory** —
`amd-tee3.0`, `sw-security-tools`, and the RISC-V `toolchain` — because the build
locates the toolchain relative to the tools (`${PSP_TOOLS_DIR}/../toolchain/...`).
Put them wherever you like; `$SRC` below is that parent directory.

```bash
SRC=/proj/smartnic/xcb/ckey/src   # <!-- personal --> the author's location; set to your own
```

So `$SRC` must end up containing:
- `$SRC/amd-tee3.0` — the ASP firmware repo
- `$SRC/sw-security-tools` — the PSP security tools (see below)
- `$SRC/toolchain/riscv/linux/riscv32-toolchain-linux/` — the RISC-V toolchain

## Prerequisites

1. **PSP Security Tools** — clone into `$SRC/sw-security-tools`:
   ```bash
   # Clone from: ssh://<user>@git.amd.com:29418/sw_security/er/tools
   # (the repo was renamed to sw-security-tools)
   git clone ssh://<user>@git.amd.com:29418/sw_security/er/tools "$SRC/sw-security-tools"
   ```

2. **RISC-V Toolchain** — the official PSP toolchain (not Zephyr SDK):
   - Located, relative to the tools, at `${PSP_TOOLS_DIR}/../toolchain/riscv/linux/riscv32-toolchain-linux/bin/` — i.e. `toolchain/` must be a sibling of `sw-security-tools` (both directly under `$SRC`)
   - Required ABI: `-march=rv32imafc -mabi=ilp32f` (single-float)
   - The Zephyr SDK's `riscv64-zephyr-elf` has an ABI mismatch (soft-float vs single-float)

3. **Git submodules** — initialize before building:
   ```bash
   cd "$SRC/amd-tee3.0"
   git submodule update --init --recursive
   ```

## Build command

Build for MI450 (drv_preesid):

```bash
cd "$SRC/amd-tee3.0/amd_tee_dr/drv_preesid"
PSP_TOOLS_DIR="$SRC/sw-security-tools" BUILD=MI450 make
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

Solution: use the official PSP RISC-V toolchain — the sibling `toolchain/` dir
under `$SRC` (see Prerequisites), not the Zephyr SDK.
