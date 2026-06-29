---
name: build-mpifoe-fw
description: >
  Build the mpifoe-fw Zephyr firmware (IFoE management CPU firmware) for a
  given platform (silicon, simnow, or emu) and analyze its memory usage. Use
  when asked to build mpifoe-fw, compile the management CPU firmware, produce
  zephyr.elf or the SimNow .hbin, or check firmware memory/size.
---

# Build mpifoe-fw Firmware

**Repository**: `mpifoe-fw` (Zephyr-based IFoE management CPU firmware)
**Build tool**: `scripts/build.py` (Python script wrapping CMake/west/Ninja)

Run all commands from the repository root.

## Choose the platform (`-p`) — IMPORTANT

`scripts/build.py` builds for a specific **target platform**, selected with
`-p`. You MUST pick the platform that matches where the firmware will run.
The artifact *formats* are the same across platforms — but the **build itself
differs**, because each environment presents the hardware differently and the
firmware is compiled to cope with that environment. In particular, SimNow has
some mismodelled or missing hardware that the firmware must work around, so a
`silicon` build will misbehave under SimNow (and vice versa) even though the
output files look identical.

The three valid platforms for mpifoe-fw are:
- `-p silicon` — real silicon (standard production target)
- `-p simnow` — **for running under SimNow** (see the `dev-workflow-simnow-launch` skill)
- `-p emu` — emulation

(`build.py --help` also lists `ss-emu` and `ss-c-model`, but there is no
mpifoe-fw for those subsystem targets — don't use them.)

The platform determines the build directory:
`build/fw.mi450-a0.<platform>_eftest/zephyr/`.

## Build command

```bash
PATH=${PATH}:/tool/pandora/.package/ccache-4.9.1/bin \
  /tools/pandora/.package/python-3.14.0/bin/python3 ./scripts/build.py \
  -p <platform> -s mi450-a0 --jobs 1 --eftest
```

Key arguments:
- `-p <platform>` — target platform: `silicon`, `simnow`, or `emu` (see above)
- `-s mi450-a0` — MI450 A0 SoC
- `--eftest` — include EFTest infrastructure (test commands, debug support)
- `--jobs 1` — single build job (avoids parallel build race conditions)

Environment:
- ccache: `/tool/pandora/.package/ccache-4.9.1/bin/`
- Python: `/tools/pandora/.package/python-3.14.0/bin/python3`

## Build output

Build directory: `build/fw.mi450-a0.<platform>_eftest/zephyr/` (created in the
repo root). Each build produces the same set of artifacts, e.g.:
- `zephyr.elf` — firmware ELF binary (used for memory analysis)
- `mpifoe_fw.hbin` — the `.hbin` image (e.g. `dev-workflow-simnow-launch` consumes the one
  from a `-p simnow` build via its `-fw` flag)

Substitute the platform you built into the path, e.g.
`build/fw.mi450-a0.simnow_eftest/zephyr/mpifoe_fw.hbin` for a simnow build.

## Memory analysis

To check firmware memory usage (text/data/bss sizes), run `size` on the
`zephyr.elf` from your build:

```bash
size build/fw.mi450-a0.<platform>_eftest/zephyr/zephyr.elf
```

To compare memory between commits, build at each commit and capture the `size`
output.
