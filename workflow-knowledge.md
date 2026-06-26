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

From a scratch directory (e.g. `/scratch/ckey/simnow`), run SimNow inside a `screen` session so it survives SSH disconnection and its interactive console remains accessible:

```bash
cd /scratch/ckey/simnow
/tool/pandora64/bin/screen -dmS simnow bash -c \
  '/home/ckey/hg/mpifoe-fw/scripts/simnow-launch/sim222.IFoE_link_0_to_1.sh \
  --slt \
  -r /proj/smartnic/xcb/ifoe/simnow/simnow-linux64-rhel8-gcc10-mi450_ifoe-20260514-0P1G-M222-release-combined-v7-a85ee03f90 \
  -fw /home/ckey/hg/mpifoe-fw/build/fw.mi450-a0.simnow_eftest/zephyr/mpifoe_fw.hbin'
```

**Use `screen`, not `tmux`**: killing a tmux server kills all sessions within it. Use `screen` — sessions survive independently. Also do not pipe SimNow through `tee` — it breaks the interactive console (responses are not echoed back).

**Key flags:**
- `--slt` — selects the SLT topology (`mi450_222_slt_ifoe.listen.now`) and uses the IFWI bundled inside the release; no separate `-if` needed
- `-r` — explicit release directory (use the release build, not debug, for normal testing)
- `-fw` — ToT mpifoe-fw firmware (`.hbin` format, boshed into the bundled IFWI automatically)

#### Boot time and steady state

SimNow takes approximately 20 minutes to boot. It has reached steady state when the mpifoe postcode monitor shows `0x800f0000`.

To check programmatically via screen:

```bash
/tool/pandora64/bin/screen -S simnow -X hardcopy /tmp/simnow-screen.txt
grep -c '800f0000' /tmp/simnow-screen.txt
```

#### Launching QEMU

Once SimNow has reached steady state, launch QEMU in its own `screen` session:

```bash
cd /scratch/ckey/simnow
/tool/pandora64/bin/screen -dmS qemu bash -c './launch-qemu.sh'
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

#### Register bodges required before launching QEMU

After SimNow reaches steady state (`0x800f0000`) but **before** launching QEMU, a set of `mid_cf` register writes must be applied. These unlock MMIO access needed by the diags test.

These are applied interactively via the SimNow console (SimNow continues running — no Ctrl-C needed). To send commands non-interactively via screen:

```bash
/tool/pandora64/bin/screen -S simnow -X stuff 'mid_cf:0.write 0x00001200 0x8000000F
'
```

Note the literal newline inside the single quotes — `screen -X stuff` requires it to submit the command.

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

When done, send `go` to resume SimNow, then launch QEMU.

---

### Running the SLT Diags Test

#### Two devices in the VM

Inside the QEMU VM, two AMD PCI devices are relevant:

| Device ID | Description | Used by |
|-----------|-------------|---------|
| `75c1` (`04:00.0`) | GPU device | `tng_executor --location` |
| `1747` (`04:00.1`) | IFoE device | `xncmdclient tlp=0` |

Find the GPU device address dynamically:

```bash
lspci -d :75c1 -D | awk '{print $1}'
```

#### Step 1: IFoE firmware configuration via xncmdclient

A setup script will be committed to `mpifoe-fw` (not yet merged at time of writing). It configures the IFoE firmware through the `xncmdclient` tool, which targets the IFoE device via `tlp=0`:

```sh
#!/bin/sh
XNCMDCLIENT="/simnow/xncmdclient --force-enable-mmap"
$XNCMDCLIENT -c "ifoe_bios 4x200 0 ffc003ff f; quit" tlp=0
$XNCMDCLIENT -c "ifoe_next_phase PROVIDER; quit" tlp=0
$XNCMDCLIENT -c "ifoe_cfg BAREMETAL ETHERNET DISABLED IFOE; quit" tlp=0
$XNCMDCLIENT -c "ifoe_active_accelerators 0xffffffff 0xffffffff 0xff 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0; quit" tlp=0
$XNCMDCLIENT -c "ifoe_identity 0; quit" tlp=0
$XNCMDCLIENT -c "ifoe_next_phase TENANT; quit" tlp=0
$XNCMDCLIENT -c "ifoe_enabled_accelerators 0x3 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0 0x0; quit" tlp=0
$XNCMDCLIENT -c "ifoe_next_phase SHOWTIME; quit" tlp=0
```

#### Step 2: Run the diags test

```bash
cd /simnow/diags/diag_tng-535f67186d25dbc2d621f1ed27b1cdd197dfe55c
GPU_ADDR=$(lspci -d :75c1 -D | awk '{print $1}')
./package/tng_executor \
    --location ${GPU_ADDR} \
    @package/suites/mi450/slt/soc/bu_config_vm.opt \
    @package/suites/mi450/slt/soc/bu_config.opt \
    -c @/myhome/diags/die_full_config-cjk.json \
    -c +/diag/mi450/npa/gpuId=1 \
    -x ifoe.loopback.2.2 \
    -p importer_id=0 \
    -p exporter_id=1 \
    -p vmid=3 \
    -p size=1_KiB \
    -p copy_engine=0 \
    -c +/diag/gpu/sdma/ucodeFrontdoorLoaded=true \
    -c +/diag/gpu/vml2/global/gcvm/silenceTimeouts=true \
    -c +/diag/gpu/vml2/global/mmvm/silenceTimeouts=true \
    -c +/diag/gpu/vml2/global/timeoutInterval=500ms \
    -c +/log/root/level=Debug \
    2>&1 | tee /myhome/diags/tng_executor.log
```

**Important:** Run the xncmdclient setup script once per SimNow boot. The tng_executor test can then be run multiple times without repeating setup.

**Notes:**
- Inside the VM, `/myhome` is the virtfs mount of `/home/ckey` on the host
- The `-c @/myhome/diags/die_full_config-cjk.json` config file location is temporary — a better home for this file is TBD
- `--location` targets the GPU device (`75c1`), not the IFoE device
- The diags binary is in `/simnow/diags/` inside the VM (mounted from the host)

---
