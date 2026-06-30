---
name: dev-knowledge-repo-ifoe-emu-overlays
description: >
  Repository reference for the three IFoE subsystem-emulation overlay repos that
  layer IFoE's changes onto a released ss-emu model: chip (pensando/ifoe-emu-chip),
  _ip (pensando/ifoe-emu-ip), and _env (pensando/ifoe-emu-env). Explains the
  unusual branch-per-release structure (default branch = chronological release
  snapshots; per-release branch = snapshot + IFoE overlay commits rebased forward),
  why a workarea sits in detached HEAD on an EPGM_ifoe_ss_* branch, the
  self-documenting import commit messages, and the chip .extras import/strip
  scripts — enough to infer the branch+rebase procedure for onboarding a new
  release. Use when working in or reasoning about chip/ _ip/ _env, when confused by
  the detached HEAD / weird branch names, when importing a new ss-emu release, or
  when locating IFoE's overlay changes (e.g. the single-process-tree code in _ip).
  See build-ifoe-arch-model-ss-emu for the build that consumes these, and
  dev-knowledge-ss-emu-environment for the runtime they shape.
---

# IFoE ss-emu overlay repos: chip / _ip / _env

When you clone ifoe-arch-model into a copy of a released subsystem-emulation
(ss-emu) model, three sibling directories are git checkouts that **overlay IFoE's
changes onto the released model**:

| Dir     | GitHub repo               | Default branch |
|---------|---------------------------|----------------|
| `chip`  | `pensando/ifoe-emu-chip`  | `master`       |
| `_ip`   | `pensando/ifoe-emu-ip`    | `main`         |
| `_env`  | `pensando/ifoe-emu-env`   | `main`         |

All three are **ordinary git repos** (see `.git/config`). Their *content*
originally comes from an ss-emu release (whose own storage may be Perforce/NFS),
but for our purposes they are git — you interact with them with git.

This is the home of IFoE's overlay code, including the single-process-tree
launch logic in `_ip/emu_global/chip/lib/emuEnvVLRunChip.pm` (see
`dev-knowledge-ss-emu-environment`).

## The branch-per-release structure

This is the part that surprises people. Each repo uses branches to track
**model releases**, not features:

- **The default branch (`master`/`main`) is a chronological stack of release
  snapshots.** Every ss-emu release adds one commit that imports that release's
  copy of the directory *exactly as released* (no IFoE changes). Because they are
  in release order, you can `git diff` two releases to see what the vendor
  changed.
- **There is one branch per release**, named after the release:
  `<LSx_Design>/EPGM_ifoe_ss_<num>[_<ver>]_<who>_<timestamp>Z`, e.g.
  `LSE_Design/EPGM_ifoe_ss_156478_v3_xcb_20250822T144931Z`. This branch is the
  release's snapshot commit **plus IFoE's overlay commits on top**.
- A populated workarea checks each repo out in **detached HEAD at
  `origin/<that release branch>`**. So "why is this repo in detached HEAD on a
  weird branch?" is expected — it is pinned to the overlay branch for the model
  release you are running. The release branch name matches the workarea/RTL
  release name (see `build-ifoe-arch-model-ss-emu`).

### The overlay commits are rebased forward, release to release

A release branch is *not* independent work — it is the **same logical stack of
IFoE overlay commits rebased onto each new release snapshot**. The same commit
subjects reappear on every release branch with **different hashes** (the
giveaway that they were rebased). For example `_ip`'s overlay stack is the same
~9 commits on every release branch:

```
IFOESW-410: cleanup: use strict; use warnings
DMEMUEML-5159: correctly encode content added to ctbconf.xml
DMEMUEML-5160: handle -ctb_var var=value pairs where contains '='
IFOESW-410: launch the EPGM processes as part of the main job
IFOESW-429: Comment out the line in run_emu that requires access to ER region
IFOESE-430: Use tbxepgm from the design release instead of the one from ER region
IFOESW-410: also remove /tmp/.TestBench/.TestBenchlock_${VVED_DOMAIN_ID}
IFOESW-410: correctly escape '$' in weak-quoted strings
IFOESW-410: IFOESW-439: send the processes SIGTERM instead of SIGHUP
```

`_env`'s overlay is smaller (typically the import commit plus an
`IFOESW-431: Updated MY_SIMNOW_DIR path in local/modulefile`); `chip`'s overlay
varies. The point is the *shape*: import snapshot at the base, IFoE overlay
commits above, carried forward by rebase.

## Import commits are self-documenting

The snapshot/import commit's **message records the exact command and source
release**, so the history is the documentation:

- **chip**: `​.extras/import.sh /proj/.../Release/<LSx_Design>/EPGM_ifoe_ss_…`
- **_ip** / **_env**: `rsync -ra /proj/.../Release/<LSx_Design>/EPGM_ifoe_ss_…/_<repo>/ .`
  (older ones may read `Snapshot from …/_ip/`)

So to see how any release was brought in, read that release branch's base
commit message.

## chip's maintenance scripts (.extras)

Only `chip` ships helper scripts and a README (`chip/README.md`):

- `​.extras/import.sh <release-dir>` — `rsync`s `<release-dir>/chip/` into the
  repo (excluding `.git`, `.extras`, `README.md`), `--delete` so the working
  tree matches the release exactly.
- `​.extras/strip-cruft.sh` — `git rm`s editor leftovers (`*~`, `.*.sw*`).

`_ip` and `_env` have no `.extras/`; their import commits show the raw `rsync`
that does the same job for `_ip/` and `_env/`.

## Onboarding a new release (the procedure, inferable from the above)

For each repo, the pattern documented by `chip/README.md` and visible in all
three histories is:

1. **Import the new release onto the default branch.** Check out the default
   branch (`master`/`main`); copy the new release's matching subdir in (chip:
   `​.extras/import.sh <release>`; `_ip`/`_env`: `rsync -ra <release>/_<repo>/ .`).
   Commit with the command as the message; for chip also run
   `​.extras/strip-cruft.sh` and commit. Push the default branch.
2. **Create the per-release overlay branch by rebasing the previous release's
   overlay onto the new snapshot.** Check out the previous release branch, rebase
   it onto the new default-branch commit (dropping any strip-cruft commits), and
   push the result to a new branch named after the new release
   (`<LSx_Design>/EPGM_ifoe_ss_…`).

`chip/README.md` gives the literal command sequence (an `import.sh` →
`git commit` → `git branch` → `git checkout <prev-release>` → `git rebase -i` →
`git push origin HEAD:refs/heads/<new-release>` worked example). `_ip`/`_env`
follow the same two-phase shape with `rsync` standing in for `import.sh`.

> Caveat: the README and `.extras` scripts are chip-specific. The `_ip`/`_env`
> procedure is inferred from their commit history (import commit + rebased
> overlay stack), which is consistent with the chip recipe but is not separately
> scripted or documented in those repos.

## Cross-references

- **Build that consumes these overlays**: `build-ifoe-arch-model-ss-emu`
- **Runtime they shape (single process tree, EPGM launch)**: `dev-knowledge-ss-emu-environment`
