# Workflow Knowledge

This file contains knowledge for end-to-end multi-component workflows: tasks that span multiple repositories or artifacts, such as launching SimNow, connecting the arch model, running diagnostics, and collecting results.

For knowledge about building and developing *within* a specific repository, see `repo-specific-knowledge.md`.
For knowledge about operational/administrative tasks (release installation, artifact management), see `operational-knowledge.md`.

---

## SimNow Test Workflows

### Launching SimNow for SLT Diags Testing

#### Host requirement

Must run on a capable host — currently `xcbl-rtl01` (or any host in the script's `ALLOWED_HOSTS` list).

#### Current release (v7)

```
Release: simnow-linux64-rhel8-gcc10-mi450_ifoe-20260514-0P1G-M222-release-combined-v7-a85ee03f90
Debug:   simnow-linux64-rhel8-gcc10-mi450_ifoe-20260514-0P1G-M222-debug-combined-v7-a85ee03f90
```

Both installed under `/proj/smartnic/xcb/ifoe/simnow/`.

#### Launch command

From a scratch directory (e.g. `/scratch/ckey/simnow`):

```bash
cd /scratch/ckey/simnow
/home/ckey/hg/mpifoe-fw/scripts/simnow-launch/sim222.IFoE_link_0_to_1.sh \
  --slt \
  -r /proj/smartnic/xcb/ifoe/simnow/simnow-linux64-rhel8-gcc10-mi450_ifoe-20260514-0P1G-M222-release-combined-v7-a85ee03f90 \
  -fw /home/ckey/hg/mpifoe-fw/build/fw.mi450-a0.simnow_eftest/zephyr/mpifoe_fw.hbin
```

**Key flags:**
- `--slt` — selects the SLT topology (`mi450_222_slt_ifoe.listen.now`) and uses the IFWI bundled inside the release; no separate `-if` needed
- `-r` — explicit release directory (use the release build, not debug, for normal testing)
- `-fw` — ToT mpifoe-fw firmware (`.hbin` format, boshed into the bundled IFWI automatically)

#### Boot time and steady state

SimNow takes approximately 20 minutes to boot. It has reached steady state when the mpifoe postcode monitor shows `0x800f0000`.

#### Launching QEMU

Once SimNow has reached steady state, launch QEMU from the same directory SimNow was run from:

```bash
cd /scratch/ckey/simnow
./launch-qemu.sh
```

QEMU uses a randomly chosen SSH port forwarded to port 22 inside the VM. Find the port from the running process:

```bash
ps aux | grep qemu-system | grep $USER | grep -oP 'hostfwd=tcp::\K[0-9]+'
```

The VM boots Ubuntu 22.04. Credentials: `root` / `l1admin`.

SSH in via xcbl-rtl01 (the port is on localhost there):

```bash
ssh xcbl-rtl01 "sshpass -p l1admin ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -p <PORT> root@127.0.0.1 '<command>'"
```

#### Verifying IFoE device enumeration

Run `lspci` inside the VM. The IFoE device appears as:

```
04:00.1 Processing accelerators: Advanced Micro Devices, Inc. [AMD] Device 1747
```

AMD device ID `1747` is the IFoE device. Its presence confirms successful enumeration.

#### Script location

`scripts/simnow-launch/sim222.IFoE_link_0_to_1.sh` in the `mpifoe-fw` repo (`origin/main`). The `--slt` flag was added in IFOESW-1254.

---
