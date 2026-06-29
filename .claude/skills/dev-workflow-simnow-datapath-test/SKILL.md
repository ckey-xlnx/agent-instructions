---
name: dev-workflow-simnow-datapath-test
description: >
  Configure IFoE firmware and run the SLT datapath (loopback) diagnostics test
  inside the SimNow QEMU VM. Use when asked to run the diags test, run the
  datapath test, run SLT diagnostics, run the loopback test, run tng_executor,
  configure xncmdclient, or set up the IFoE firmware for testing.
  Requires SimNow and QEMU to already be running (see dev-workflow-simnow-launch skill).
---

# SimNow Datapath Test Workflow

Requires SimNow and QEMU to be up and the IFoE device enumerated.
Run the `dev-workflow-simnow-launch` skill first if not already done.

## Devices in the VM

| Device ID | PCI address | Used by |
|-----------|-------------|---------|
| `75c1` (`04:00.0`) | GPU device | `tng_executor --location` |
| `1747` (`04:00.1`) | IFoE device | `xncmdclient tlp=0` |

Find the GPU device address dynamically:

```bash
lspci -d :75c1 -D | awk '{print $1}'
```

## Step 1: Configure IFoE firmware via xncmdclient

Run this setup script once per SimNow boot (safe to run before each test run):

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

A committed version of this script will land in `mpifoe-fw` (not yet merged at time of writing).

## Step 2: Run the diags test

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
    2>&1 | tee /tmp/tng_executor.log
```

The xncmdclient setup (Step 1) must be run once per SimNow boot. `tng_executor` can be re-run multiple times without repeating setup.

## File system notes (important)

- **`/myhome` is read-only inside the VM** — it is a virtfs mount of `/home/ckey` on the host. Use it for reading inputs (e.g. the config JSON) only. Any write attempt is silently dropped.
- **Write all logs and temp files to `/tmp`** inside the VM (e.g. `tee /tmp/tng_executor.log`).
- To retrieve logs to the host: `cat` over SSH or `scp` — `/tmp` in the VM is not shared with the host.
- `/simnow` inside the VM is a host-mounted path — treat as read-only.
- The `-c @/myhome/diags/die_full_config-cjk.json` config file location is temporary; a better home is TBD.
