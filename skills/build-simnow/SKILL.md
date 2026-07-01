---
name: build-simnow
description: >
  Configure, build, and package SimNow (the mi450_ifoe simulator) with
  CMake/Ninja on a Pandora build server. Use when asked to build SimNow,
  configure a SimNow release or debug build, package a SimNow zip, or set up
  a SimNow build session on the build server. (For installing a pre-built
  release, use admin-workflow-simnow-install-release instead.)
---

# Build SimNow

**Build tool**: CMake with Ninja generator.

Tool locations:
- CMake: `/tool/pandora/.package/cmake-3.30.0/bin/cmake`
- Ninja: `/tool/pandora/.package/ninja-1.12.1/bin/ninja`
- Git: `/tools/pandora64/.package/git-2.48.1/bin/git`
- Python: `/tools/pandora64/.package/python-3.14.0/bin/python3`

## Step 1: Build server setup

All build commands must run on a build server. Set up an interactive session:

```bash
# Start bash (needed if using zsh, as the environment script requires bash)
/tool/pandora64/bin/bash

# Source the environment initialization script
source /proj/verif_release_ro/cbwa_initscript/current/cbwa_init.bash

# Submit interactive LSF job to build server (RHEL8)
lsf_bsub -q normal -Is -P simnow-perf -R "select[(type==RHEL8_64)] rusage[mem=64000]" /tool/pandora64/bin/bash
```

## Step 2: Configure

Release build:

```bash
/tool/pandora/.package/cmake-3.30.0/bin/cmake \
  --preset=linux-ninja-pandora-gcc10 \
  -S . \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_STANDARD=17 \
  -DGIT_EXECUTABLE=/tools/pandora64/.package/git-2.48.1/bin/git \
  -DCMAKE_MAKE_PROGRAM="/tool/pandora/.package/ninja-1.12.1/bin/ninja" \
  -DUSE_IFOE_SS_MODEL=1 \
  -G "Ninja" \
  -B builds/linux-ninja-pandora-gcc10-release
```

Debug build — identical except `-DCMAKE_BUILD_TYPE=Debug` and the `-B` output dir:

```bash
/tool/pandora/.package/cmake-3.30.0/bin/cmake \
  --preset=linux-ninja-pandora-gcc10 \
  -S . \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_STANDARD=17 \
  -DGIT_EXECUTABLE=/tools/pandora64/.package/git-2.48.1/bin/git \
  -DCMAKE_MAKE_PROGRAM="/tool/pandora/.package/ninja-1.12.1/bin/ninja" \
  -DUSE_IFOE_SS_MODEL=1 \
  -G "Ninja" \
  -B builds/linux-ninja-pandora-gcc10-debug
```

Notes:
- The only difference between release and debug is `-DCMAKE_BUILD_TYPE` and the `-B` output directory.
- `-DCMAKE_CXX_STANDARD=17` is required — SimNow needs C++17 or later, and this overrides any cached older standard.
- **Stale cache**: errors like "target was not found" for `OpenSSL::Crypto` mean a stale build dir. Delete and reconfigure:
  ```bash
  rm -rf builds/linux-ninja-pandora-gcc10-release  # or -debug
  ```

## Step 3: Build

```bash
/tool/pandora/.package/ninja-1.12.1/bin/ninja -C builds/linux-ninja-pandora-gcc10-debug
```

## Step 4: Package

The packaged output is large, so `--output-dir` needs somewhere with plenty of
storage — a `/proj/...` scratch/dump area, not your home directory. Set `$OUT`
to such a location:

```bash
OUT=/proj/vulcano_dump2_ner/$USER/simnow/packaging   # <!-- personal --> any big-storage dir you own
```

Standard zip release:

```bash
# Release build
/tools/pandora64/.package/python-3.14.0/bin/python3 scripts/package/package.py \
  --project mi450_ifoe \
  --simnow-root $(pwd)/builds/linux-ninja-pandora-gcc10-release \
  --output-dir "$OUT"

# Debug build
/tools/pandora64/.package/python-3.14.0/bin/python3 scripts/package/package.py \
  --project mi450_ifoe \
  --simnow-root $(pwd)/builds/linux-ninja-pandora-gcc10-debug \
  --output-dir "$OUT"
```

Produces zips named like:
- `simnow-linux64-rhel8-gcc10-mi450_ifoe-20251205-daf970901c.zip` (release)
- `simnow-linux64-rhel8-gcc10-mi450_ifoe-debug-20251204-df22f4ccdf.zip` (debug)

For iterative development (rsync deployment, no zip), use `--no-date --no-sha --no-zip` to get a consistent directory name with date/SHA zeroed:

```bash
# Release build
/tools/pandora64/.package/python-3.14.0/bin/python3 scripts/package/package.py \
  --project mi450_ifoe \
  --simnow-root $(pwd)/builds/linux-ninja-pandora-gcc10-release \
  --output-dir "$OUT" \
  --no-date --no-sha --no-zip

# Debug build
/tools/pandora64/.package/python-3.14.0/bin/python3 scripts/package/package.py \
  --project mi450_ifoe \
  --simnow-root $(pwd)/builds/linux-ninja-pandora-gcc10-debug \
  --output-dir "$OUT" \
  --no-date --no-sha --no-zip
```

Produces directories named like:
- `simnow-linux64-rhel8-gcc10-mi450_ifoe-00000000-0000000000` (release)
- `simnow-linux64-rhel8-gcc10-mi450_ifoe-debug-00000000-0000000000` (debug)

Note: `$OUT` is only a convention — any writable big-storage directory works;
it does not have to be under `/proj/vulcano_dump2_ner`.
