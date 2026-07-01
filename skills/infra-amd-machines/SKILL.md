---
name: infra-amd-machines
description: >
  Reference for which machine to use for which IFoE task: the well-known fixed
  hosts (e.g. xcbl-rtl01 for SimNow) and the allocated-machine classes (LSF
  build servers, etx/Slurm emulation hosts, Pandora-env hosts for diags). Use
  when asked where to run something — SimNow, a SimNow/diags build, an ss-emu
  emulation job — which host to ssh to, why a command "isn't installed here" or
  fails on the wrong host/OS, or what a named host like xcbl-rtl01 /
  dfcetx8-emu001 is for. For sites and how allocation works see
  infra-amd-sites; for the filesystems these machines see, see infra-amd-storage.
---

# AMD IFoE machines: which host for what

Where to run each kind of IFoE task. Machines come in two flavours: a few
**fixed named hosts** for specific jobs, and **allocated-machine classes** where
you don't get a fixed host but request one (LSF, etx/Slurm). This grows as hosts
are confirmed; record only what is known for a fact.

Two axes matter and are documented in the companion skills:
- **Site** — which office/site a machine is at, and how you get one there
  (VDI vs etx). See `infra-amd-sites`.
- **Storage/OS** — machine-local vs shared `/proj`, and per-machine OS build
  (RHEL7/RHEL8) affecting tool availability. See `infra-amd-storage`.

## By task

| Task | Where to run | Kind | Notes |
|------|--------------|------|-------|
| **Launch SimNow** (+ QEMU) | `xcbl-rtl01` (or a host in the launch script's `ALLOWED_HOSTS`) | fixed named host(s) | required by `dev-workflow-simnow-launch`; the launch script hard-fails on other hosts |
| **Build SimNow** | an LSF-allocated **RHEL8** build server | allocated (LSF) | `lsf_bsub -q normal -Is -P simnow-perf -R "select[(type==RHEL8_64)] rusage[mem=64000]" ...`; see `build-simnow` |
| **Build diag_tng (XCE/Pandora)** | a host with the Pandora environment on XCE | allocated / enclave | `build-diag-tng` (XCE path); XCE can't reach atlartifactory (drives the Conan-cache workaround). XCE is a separate enclave — TODO: document in `infra-amd-sites` |
| **Build diag_tng (DFC)** | a DFC host, e.g. `dfcetx8-emu001` | example host | `build-diag-tng` (DFC path); can auth to atlartifactory directly |
| **ss-emu emulation / Veloce job** | an **atl** etx-allocated host; the job itself runs via Slurm (`mi_veloce`) on the emulator | allocated (etx + Slurm) | grid is **atl-only**; see `infra-grid-atl`. Physical emulator selected via `--emu-list` (e.g. MUGELLO) |
| **ss-emu build** | an atl machine (in your workarea) | allocated (etx) | atl-only; see `build-ifoe-arch-model-ss-emu` |

## Fixed named hosts (verified/stated — grow this)

| Host | Purpose | Site | Notes |
|------|---------|------|-------|
| `xcbl-rtl01` | SimNow launch host | xcb | plus other hosts in the launch script's `ALLOWED_HOSTS` list (not enumerated here) |

## Allocated-machine classes

You don't ssh to a fixed host for these — you request one:

- **LSF build servers** — `lsf_bsub` submits an interactive/batch job to an
  LSF-selected host matching a resource spec (e.g. `type==RHEL8_64`, memory).
  Used for the SimNow build. `lsf_bsub` is at `/tool/pandora64/bin/lsf_bsub`.
- **etx-allocated hosts** — at atl (and sac/cac/dfc), you get a dynamically
  allocated machine rather than a fixed one; the specific host is per-user and
  transient. See `infra-amd-sites` for how etx allocation works and its caveats.
- **Slurm / emulation grid** — emulation (Veloce) sessions are Slurm jobs on the
  atl grid (partition `mi_veloce`), launched indirectly by `velocetool-ifoe`.
  The grid exists **only at atl**. See `infra-grid-atl`.

## Example / illustrative hosts

These appear in skills as examples, not fixed requirements — the *class* is what
matters, not the specific name:
- `dfcetx8-emu001` — a DFC host, for diag_tng builds.
- `atletx8-reg111` — an atl etx host (per-user, transient).

## See also

- `infra-amd-sites` — sites, and how you get a machine at each (VDI vs etx).
- `infra-amd-storage` — the filesystems these machines see; per-machine OS/tool variance.
- `infra-grid-atl` — the atl emulation grid + LSF farm in detail.
- `dev-workflow-simnow-launch` — the SimNow host requirement.
- `build-simnow`, `build-diag-tng` — the build-host requirements.
