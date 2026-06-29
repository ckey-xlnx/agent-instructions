---
name: simnow-launch
description: >
  Launch SimNow for SLT testing, start QEMU, verify IFoE device enumeration,
  and apply required register bodges. Use when asked to launch SimNow, start
  the simulation, start the sim, bring up the test environment, start SimNow,
  run SimNow, check if SimNow has booted, or wait for SimNow to reach steady
  state.
---

# SimNow Launch Workflow

## Host requirement

Must run on `xcbl-rtl01` (or any host in the script's `ALLOWED_HOSTS` list).

## Current release (v7)

```
Release: simnow-linux64-rhel8-gcc10-mi450_ifoe-20260514-0P1G-M222-release-combined-v7-a85ee03f90
Debug:   simnow-linux64-rhel8-gcc10-mi450_ifoe-20260514-0P1G-M222-debug-combined-v7-a85ee03f90
```

Both installed under `/proj/smartnic/xcb/ifoe/simnow/`.

## Step 1: Launch SimNow

From a scratch directory (e.g. `/scratch/ckey/simnow`), run inside a `screen` session:

```bash
cd /scratch/ckey/simnow
/tool/pandora64/bin/screen -dmS simnow bash -c \
  '/home/ckey/hg/mpifoe-fw/scripts/simnow-launch/sim222.IFoE_link_0_to_1.sh \
  --slt \
  -r /proj/smartnic/xcb/ifoe/simnow/simnow-linux64-rhel8-gcc10-mi450_ifoe-20260514-0P1G-M222-release-combined-v7-a85ee03f90 \
  -fw /home/ckey/hg/mpifoe-fw/build/fw.mi450-a0.simnow_eftest/zephyr/mpifoe_fw.hbin'
```

**Use `screen`, not `tmux`** — killing a tmux server kills all sessions within it.
**Do not pipe SimNow through `tee`** — it breaks the interactive console.

Key flags:
- `--slt` — selects the SLT topology and uses the IFWI bundled inside the release
- `-r` — explicit release directory (use release build, not debug)
- `-fw` — ToT mpifoe-fw firmware (`.hbin` format, injected into bundled IFWI)

Script location: `scripts/simnow-launch/sim222.IFoE_link_0_to_1.sh` in the `mpifoe-fw` repo (`origin/main`).

## Step 2: Wait for steady state

SimNow takes approximately 20 minutes to boot. Steady state is indicated by the mpifoe postcode monitor showing `0x800f0000`.

Check programmatically:

```bash
/tool/pandora64/bin/screen -S simnow -X hardcopy /tmp/simnow-screen.txt
grep -c '800f0000' /tmp/simnow-screen.txt
```

## Step 3: Apply register bodges

After reaching steady state but **before** launching QEMU, apply these `mid_cf` register writes via the SimNow console. Send each non-interactively via screen (note the literal newline inside the single quotes):

```bash
/tool/pandora64/bin/screen -S simnow -X stuff 'mid_cf:0.write 0x00001200 0x8000000F
'
```

Full list of writes:

```
mid_cf:0.write 0x00001200 0x8000000F
mid_cf:0.write 0x00896000 0x8000000F
mid_cf:0.write 0x00896400 0x8000000F
mid_cf:0.write 0x00896800 0x8000000F
mid_cf:0.write 0x00896C00 0x8000000F
mid_cf:0.write 0x00897000 0x8000000F
mid_cf:0.write 0x00897400 0x8000000F
mid_cf:0.write 0x00897800 0x8000000F
mid_cf:0.write 0x00897C00 0x8000000F
mid_cf:0.write 0x00898000 0x8000000F
mid_cf:0.write 0x00898400 0x8000000F
mid_cf:0.write 0x00898800 0x8000000F
mid_cf:0.write 0x00898C00 0x8000000F
mid_cf:0.write 0x00899000 0x8000000F
mid_cf:0.write 0x00899400 0x8000000F
mid_cf:0.write 0x00899800 0x8000000F
mid_cf:0.write 0x00899C00 0x8000000F
mid_cf:0.write 0x0089A000 0x8000000F
mid_cf:1.write 0x00001200 0x8000000F
mid_cf:1.write 0x00896000 0x8000000F
mid_cf:1.write 0x00896400 0x8000000F
mid_cf:1.write 0x00896800 0x8000000F
mid_cf:1.write 0x00896C00 0x8000000F
mid_cf:1.write 0x00897000 0x8000000F
mid_cf:1.write 0x00897400 0x8000000F
mid_cf:1.write 0x00897800 0x8000000F
mid_cf:1.write 0x00897C00 0x8000000F
mid_cf:1.write 0x00898000 0x8000000F
mid_cf:1.write 0x00898400 0x8000000F
mid_cf:1.write 0x00898800 0x8000000F
mid_cf:1.write 0x00898C00 0x8000000F
mid_cf:1.write 0x00899000 0x8000000F
mid_cf:1.write 0x00899400 0x8000000F
mid_cf:1.write 0x00899800 0x8000000F
mid_cf:1.write 0x00899C00 0x8000000F
mid_cf:1.write 0x0089A000 0x8000000F
```

Each write should be acknowledged with `CF Access OK on SMN Addr 0x...`.

When all writes are done, send `go` to resume SimNow:

```bash
/tool/pandora64/bin/screen -S simnow -X stuff 'go
'
```

## Step 4: Launch QEMU

```bash
cd /scratch/ckey/simnow
/tool/pandora64/bin/screen -dmS qemu bash -c './launch-qemu.sh'
```

QEMU uses a randomly chosen SSH port forwarded to port 22 inside the VM. Find it:

```bash
ps aux | grep qemu-system | grep $USER | grep -oP 'hostfwd=tcp::\K[0-9]+'
```

The VM boots Ubuntu 22.04. Credentials: `root` / `l1admin`.

SSH in via xcbl-rtl01:

```bash
ssh xcbl-rtl01 "sshpass -p l1admin ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -p <PORT> root@127.0.0.1 '<command>'"
```

## Step 5: Verify IFoE enumeration

Run `lspci` inside the VM. The IFoE device appears as:

```
04:00.1 Processing accelerators: Advanced Micro Devices, Inc. [AMD] Device 1747
```

AMD device ID `1747` confirms successful enumeration.
