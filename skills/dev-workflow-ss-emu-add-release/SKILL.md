---
name: dev-workflow-ss-emu-add-release
description: >
  Add a new IFoE subsystem-emulation (ss-emu) release to the overlay repos
  (chip / _ip / _env) so a workspace can be created on it: import the release
  snapshot onto the default branch, then rebase the previous release's IFoE
  overlay commits onto it and push a new per-release branch. Use when asked to
  onboard / import / add a new ss-emu release to the overlay repos, create the
  per-release branch, or "make a new release available for emulation work".
  Covers why releases are snapshotted read-only. Runs at the atl site. For the
  branch model this maintains see dev-knowledge-repo-ifoe-emu-overlays; to then
  build a workspace on the release see dev-workflow-ss-emu-workspace.
---

# Add a new ss-emu release to the overlay repos

To work on a new ss-emu release you first make it available in the three overlay
repos (`chip`, `_ip`, `_env`) by adding (a) a snapshot of the release on the
default branch and (b) a per-release branch carrying IFoE's overlay commits. See
`dev-knowledge-repo-ifoe-emu-overlays` for the branch model this maintains.

**Site: atl.** Releases live on the atl filesystem; run this on an atl machine.

> Confidence: the procedure below is taken from the per-repo `import-release-*.sh`
> scripts found in a TE work area (`ifoe_emu_ci/scripts/`) and `chip/README.md`,
> so it is **script-derived, not pure guesswork**. It has **not been re-run and
> verified end-to-end here**, and the exact base refs differ per repo (see
> below), so treat it as a guide to confirm against the scripts, not a guaranteed
> recipe. Mark any divergence you find.

## Why releases are read-only snapshots

The team producing ss-emu releases **keeps modifying them in place**. To get a
stable base to work from, we **snapshot** a release and mark the snapshot
read-only for everyone (`chmod ugo-w`, i.e. no write bits — observed on disk as
`dr-xr-x---`). That snapshot is what lives under
`/proj/vulcano_dump2_ner/Release/<LSx_Design>/<DESIGN_NAME>` and what the import
below copies from. (It is also why creating a workspace from a release starts
with `chmod -R u+w` — the source is deliberately unwritable; see
`dev-workflow-ss-emu-workspace`.)

`<DESIGN_NAME>` is the release name (e.g.
`EPGM_ifoe_ss_156478_v3_xcb_20250822T144931Z`) and `<LSx_Design>` its design dir
(e.g. `LSE_Design`); together `<LSx_Design>/<DESIGN_NAME>` is the per-release
branch name in each overlay repo.

## The shared shape

For every repo the procedure is two phases:

1. **Import the release snapshot** onto a base (the default branch for `chip`; a
   fixed import-base commit for `_ip`/`_env`), committing the snapshot with the
   import command as the message.
2. **Rebase the previous release's overlay commits onto that new snapshot** and
   push the result as the new per-release branch
   (`git rebase -i --onto HEAD HEAD <prev-release-ref>`), then push.

`<NEW_RELEASE>` and `<BASE_RELEASE>` below are `<LSx_Design>/<DESIGN_NAME>`
strings (new one being added, and the previous release whose overlay you carry
forward).

## chip

```bash
cd chip
git checkout origin/master

.extras/import.sh /proj/vulcano_dump2_ner/Release/<NEW_RELEASE>
git add .
git status                       # check nothing unwanted under .extras got staged
git commit -m ".extras/import.sh /proj/vulcano_dump2_ner/Release/<NEW_RELEASE>"
NEW_MASTER_HASH=$(git rev-parse HEAD)

.extras/strip-cruft.sh           # remove editor leftovers
git add .
git commit -m ".extras/strip-cruft.sh"

# carry the previous release's overlay forward onto the new snapshot,
# dropping the strip-cruft commit(s) during the interactive rebase
git rebase -i --onto HEAD HEAD origin/<BASE_RELEASE>
NEW_RELEASE_HASH=$(git rev-parse HEAD)

git push origin ${NEW_MASTER_HASH}:refs/heads/master
git push origin ${NEW_RELEASE_HASH}:refs/heads/<NEW_RELEASE>
```

## _ip and _env

Same two-phase shape, but the import base is a **fixed commit** (the repo's
original import-base, hardcoded in the scripts) rather than the default branch,
and the snapshot is brought in with `rsync` (these repos have no `.extras/`):

```bash
cd _ip          # or _env
git checkout <import-base-commit>      # the fixed base SHA from import-release-ip.sh / -env.sh

rsync -ra /proj/vulcano_dump2_ner/Release/<NEW_RELEASE>/_ip/ .   # _env/ for the env repo
chmod +w -R .
git add .
git commit -m "rsync -ra /proj/vulcano_dump2_ner/Release/<NEW_RELEASE>/_ip/ ."

# drop the previous release's rsync (import) commit during this rebase,
# replaying only the overlay commits onto the new snapshot
git rebase -i --onto HEAD HEAD origin/<BASE_RELEASE>
NEW_RELEASE_HASH=$(git rev-parse HEAD)

git push origin ${NEW_RELEASE_HASH}:refs/heads/<NEW_RELEASE>
```

Notes / cautions:
- The `import-release-ip.sh` / `import-release-env.sh` scripts each hardcode their
  own base commit SHA — read the current script for the right value rather than
  copying one from here.
- The interactive rebase requires you to **drop the previous release's import
  commit** (the script literally pauses and asks you to remove it) so the new
  branch is "new import + overlay", not two stacked imports.
- The scripts `echo` the final `git push` commands rather than running them —
  review the resulting hashes/branch name before pushing.
- `_ip` may be cloned via either `ifoe-emu-ip` or its `ifoe_emu_ip` alias.

## Cross-references

- **Branch model being maintained**: `dev-knowledge-repo-ifoe-emu-overlays`
- **Create a workspace on the release once it is added**: `dev-workflow-ss-emu-workspace`
- **Build**: `build-ifoe-arch-model-ss-emu`
