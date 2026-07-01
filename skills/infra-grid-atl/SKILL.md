---
name: infra-grid-atl
description: >
  Reference for the compute grids at the atl site: the Slurm emulation grid
  (manage emulation/Veloce sessions with squeue and scancel) and the separate
  LSF batch farm (submit generic jobs and interactive shells with lsf_bsub).
  Use when asked to launch a grid/emulation job, get an interactive shell on
  the grid, run something on the farm/cluster, or query/kill grid jobs at atl;
  or when you see lsf_bsub, squeue, scancel, mi_veloce, or velocetool. The grid
  exists only at atl (there is no grid at xcb).
---

# ATL Grids

The atl site has **two distinct schedulers**. This is the key thing to get
right — they do not share job IDs, and a job submitted to one is invisible to
the other.

| Scheduler | Used for | Submit | Query | Kill |
|-----------|----------|--------|-------|------|
| **Slurm** | Emulation / Veloce sessions | (launched internally by `velocetool-ifoe`) | `squeue` | `scancel` |
| **LSF**   | Generic batch jobs, interactive shells | `lsf_bsub` | `bjobs` / `lsf_bjobs` | `bkill` / `lsf_bkill` |

So: **manage emulation runs with `squeue`/`scancel`; submit generic batch jobs
and interactive shells with `lsf_bsub` (and query/kill those with
`bjobs`/`bkill`).** Do not expect `squeue` to show an `lsf_bsub` job, or
`bjobs` to show an emulation job — that mismatch is the scheduler split, not a
bug.

## Where the grids exist

Only at the **`atl` site**. There is **no grid at `xcb`** and these commands are
not installed there. Run everything on **your** atl machine. atl is
etx-allocated: there is no fixed or shared host — each user connects to their
own dynamically-allocated machine with an active etx session (e.g. ssh as
`ckey@atletx8-reg111` <!-- personal -->; the host is per-user and transient).
See `infra-amd-sites` for how etx allocation works and its caveats.

Binaries verified on atl:
- `lsf_bsub` → `/tool/pandora64/bin/lsf_bsub`
- `squeue` / `scancel` → `/proj/emu/slurm/wrapper/bin/` (Slurm wrappers)

## Prerequisites (environment setup)

Same two setup steps used for building (see `build-ifoe-arch-model-ss-emu`).
Shell state does not persist across separate ssh invocations, so source and
bootenv in the **same** shell as the grid command.

```bash
# Use bash (not zsh)
/tool/pandora64/bin/bash

# Source cbwa init — puts lsf_bsub on PATH
source /proj/verif_release_ro/cbwa_initscript/current/cbwa_init.bash

# Boot a workarea — puts squeue / scancel on PATH
bootenv -C <workarea>
```

- `lsf_bsub` is on PATH after **cbwa** alone.
- `squeue` / `scancel` require **bootenv** (verified: absent after cbwa alone,
  present after bootenv).
- `bootenv -C <dir>` needs a real workarea (one set up by `mkwa`/`p4_mkwa`);
  bare `bootenv` fails with "Can't boot this workspace". `-C` accepts the
  top-level workarea or a directory below it. bootenv works via
  `module load wa_boot/...`.
- Example workarea (atl) <!-- personal -->:
  `/proj/vulcano_dump2_ner/$USER/ifoe_test/EPGM_ifoe_ss_156478_v3_xcb_20250822T144931Z`

## Slurm: emulation / Veloce sessions

Emulation sessions are **Slurm** jobs (partition `mi_veloce`). They are
launched indirectly: `velocetool-ifoe` submits to Slurm internally and prints
`SLURM job_id is <id>`. (A personal bodge wrapper <!-- personal -->
`/proj/vulcano_dump2_ner/ckey/ifoe_test/goall.sh` drives velocetool for a given
workarea; it keys off `basename $(pwd)`, so run it from the workarea dir. This is
a personal one-off, not shared tooling — to be digested/cleaned up later.)

Manage the session with Slurm:

```bash
squeue -u <user>     # your jobs (also: squeue --me)
squeue -j <jobid>    # a specific job
scancel <jobid>      # cancel a job
```

Verified flow: a launched emulation job shows `ST=R` (running) on partition
`mi_veloce` under `squeue`, and `scancel <jobid>` moves it to `CG` then clears
it.

## LSF: generic batch jobs and interactive shells

`lsf_bsub` is an AMD wrapper using LSF-style `bsub` syntax. On submit it prints
`ESUB-INFO: Since you are in ATL, adding "atl" site resource to bsub` and
`Job <NNNNNNN> is submitted to queue <...>`.

### Interactive bash shell (SimNow example)

```bash
lsf_bsub -q normal -Is -P simnow-perf \
  -R "select[(type==RHEL8_64)] rusage[mem=64000]" \
  /tool/pandora64/bin/bash
```

| Arg | Meaning |
|-----|---------|
| `-q normal` | Queue. |
| `-Is` | **Interactive** session with a pseudo-terminal — gives a usable shell. Drop it for a batch job. |
| `-P simnow-perf` | **Project** (required; the name **varies** by use case). |
| `-R "select[(type==RHEL8_64)] rusage[mem=64000]"` | Resource req: a `RHEL8_64` host, reserve ~64 GB RAM (`mem` in MB). Tune per job. |
| `/tool/pandora64/bin/bash` | Command to run — here the pandora bash. For a batch job, pass a command/script instead. |

Query / kill LSF jobs (NOT squeue/scancel):

```bash
bjobs <jobid>        # or lsf_bjobs
bkill <jobid>        # or lsf_bkill
```

Caveat observed: LSF query/kill can fail with `Failed in an LSF library call:
Master LIM is down; try later` when the LSF master daemon is down — an infra
outage independent of the submit succeeding.
