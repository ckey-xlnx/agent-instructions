---
name: build-mpifoe-fw
description: >
  Build the mpifoe-fw Zephyr firmware (IFoE management CPU firmware) and
  analyze its memory usage. Use when asked to build mpifoe-fw, compile the
  management CPU firmware, produce zephyr.elf, or check firmware memory/size.
---

# Build mpifoe-fw Firmware

**Repository**: `mpifoe-fw` (Zephyr-based IFoE management CPU firmware)
**Build tool**: `scripts/build.py` (Python script wrapping CMake/west/Ninja)

Run all commands from the repository root.

## Build command

```bash
PATH=${PATH}:/tool/pandora/.package/ccache-4.9.1/bin \
  /tools/pandora/.package/python-3.14.0/bin/python3 ./scripts/build.py \
  -p silicon -s mi450-a0 --jobs 1 --eftest
```

Key arguments:
- `-p silicon` — silicon platform (standard production target)
- `-s mi450-a0` — MI450 A0 SoC
- `--eftest` — include EFTest infrastructure (test commands, debug support)
- `--jobs 1` — single build job (avoids parallel build race conditions)

Environment:
- ccache: `/tool/pandora/.package/ccache-4.9.1/bin/`
- Python: `/tools/pandora/.package/python-3.14.0/bin/python3`

## Build output

Build directory: `build/fw.mi450-a0.silicon_eftest/` (created in the repo root)

Key output files:
- `build/fw.mi450-a0.silicon_eftest/zephyr/zephyr.elf` — firmware ELF binary

## Memory analysis

To check firmware memory usage (text/data/bss sizes):

```bash
size build/fw.mi450-a0.silicon_eftest/zephyr/zephyr.elf
```

To compare memory between commits, build at each commit and capture the `size`
output.
