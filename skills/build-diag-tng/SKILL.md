---
name: build-diag-tng
description: >
  Build the diag_tng diagnostics test software for MI450 using Conan + CMake/
  Ninja, including the Pandora "bodge" Conan profile and cache workaround. Use
  when asked to build diag_tng, build the diags, build tng_executor, set up the
  Conan bodge profile, or build diagnostics in the XCE/Pandora or DFC
  environment. (To run the diags, use dev-workflow-simnow-datapath-test instead.)
---

# Build diag_tng (Diagnostics Test Software)

**Repository**: https://github.com/pensando/diag_tng (private, AMD access required)
**Purpose**: Diagnostics test software for MI450 — test cases that run on the
device (in QEMU/SimNow) to validate hardware functionality.
**Known working revision**: `f6a5f363b3d945a65c43644a98c4d23142432dc1`
**Build system**: Conan (package manager) + CMake/Ninja
**Related tickets**: IFOESW-782 (build in Pandora), IFOESW-805 (mhub.opt)

## Key challenges

1. **No artifactory access from XCE**: Oktet (XCE) cannot reach
   `https://atlartifactory.amd.com/` where Conan packages are hosted.
2. **Package incompatibility**: prebuilt Conan packages target the diags build
   server (`/opt/gcc11-for-tng/lib64`) and don't work in Pandora.
3. **"Bodge" profile**: the working approach uses a Conan profile
   (`gcc13-glibc-linux5.4-pandora-bodge-dbg`) that targets Pandora glibc but
   keeps the same configuration hash as the existing packages so they can be
   pulled from artifactory.
4. **Conan cache workaround**: since XCE can't reach artifactory, use a
   pre-populated Conan data cache:
   `/proj/smartnic/xcb/ifoe/diags/conan-20260318-121400`

## Build instructions — XCE / Pandora environment

Host requirement: a host with the Pandora environment (e.g. `xce-cmod-10`).

```bash
# 1. Clone the repo
git clone https://github.com/pensando/diag_tng tng
git -C tng checkout f6a5f363b3d945a65c43644a98c4d23142432dc1

# 2. Set up environment variables
export CONAN_USER_HOME=/home/$USER/
export CONAN=/tools/pandora64/.package/conan-1.59.0/bin/conan
export CCACHE_DIR=/home/$USER/.ccache

# 3. Configure Conan
$CONAN config set storage.download_cache=$CONAN_USER_HOME/.conan/cache
$CONAN config set general.read_only_cache=True
$CONAN config set general.revisions_enabled=True
$CONAN config set general.conan_cmake_program=/tools/pandora64/.package/cmake-3.30.0/bin/cmake
$CONAN config set general.cmake_generator=Ninja
$CONAN remote remove conan-center          # old bintray, deprecated
$CONAN remote remove conancenter           # current upstream

# 4. Copy the pre-populated Conan cache (instead of using artifactory)
cp -r /proj/smartnic/xcb/ifoe/diags/conan-20260318-121400/* $CONAN_USER_HOME/.conan/data/

# 5. Create pandora-gcc-13.2.0.specs file (needed for the bodge profile)
#    Create this file at a path you control (e.g., /home/$USER/pandora-gcc-13.2.0.specs)
#    Content:
cat > /home/$USER/pandora-gcc-13.2.0.specs << 'EOF'
*startfile:
%{!mandroid|tno-android-ld:%{shared:;      pg|p|profile:%{static-pie:grcrt1.o%s;:gcrt1.o%s};      static:/tools/pandora64/.package/glibc-2.35/lib/crt1.o%s;      static-pie:rcrt1.o%s;      pie:Scrt1.o%s;      :/tools/pandora64/.package/glibc-2.35/lib/crt1.o%s} /tools/pandora64/.package/glibc-2.35/lib/crti.o%s    %{static:crtbeginT.o%s;      shared|static-pie| pie:crtbeginS.o%s;      :crtbegin.o%s}    %{fvtable-verify=none:%s;      fvtable-verify=preinit:vtv_start_preinit.o%s;      fvtable-verify=std:vtv_start.o%s} ;:%{shared: crtbegin_so%O%s;:  %{static: crtbegin_static%O%s;: crtbegin_dynamic%O%s}}}
EOF

# 6. Update path to specs file in the bodge profile (gcc13-glibc-linux5.4-pandora-bodge-dbg)
#    Update the LDFLAGS line to use the standard ld-linux-x86-64.so.2 (NOT a fixed patchelf'd version):
#    LDFLAGS=-Wl,-rpath -Wl,$standalone_toolchain_libc/lib -Wl,-dynamic-linker $standalone_toolchain_libc/lib/ld-linux-x86-64.so.2 -B$standalone_toolchain_binutils/bin -L$standalone_toolchain_libc/lib
#    Also update the pandora-gcc-13.2.0.specs path to point to your file.

# 7. Install Conan config from the repo
$CONAN config install tng/build/config/conan/linux/

# 8. Build
mkdir builddir
cd builddir
$CONAN install ../tng --profile=gcc13-glibc-linux5.4-pandora-bodge-dbg -r artifactory \
  -o enable_ccache=True -o static_executables=True -o device=mi450
$CONAN build ../tng
```

**Note**: do NOT use `--update` in the `$CONAN install` step (it would try to
pull updated packages from artifactory, which is inaccessible from XCE).

## Build instructions — DFC / AMD internal

On DFC hosts (e.g. `dfcetx8-emu001`), use `/proj/mi450_runs/users/$USER/` as
`CONAN_USER_HOME` and authenticate to artifactory directly:

```bash
export CONAN_USER_HOME=/proj/mi450_runs/users/$USER/
export CONAN=/tools/pandora64/.package/conan-1.59.0/bin/conan
export CCACHE_DIR=/proj/mi450_runs/users/$USER/.ccache
# ... (same Conan config steps as above) ...
$CONAN remote add artifactory https://atlartifactory.amd.com/artifactory/api/conan/diags-pkg/
$CONAN user $USER -p -r artifactory
$CONAN config install tng/build/config/conan/linux/

mkdir builddir && cd builddir

# Bodge approach (working):
$CONAN install ../tng --profile=gcc13-glibc-linux5.4-pandora-bodge-dbg -r artifactory --update \
  -o enable_ccache=True -o static_executables=True -o device=mi450
$CONAN install ../tng --profile=gcc13-glibc-linux5.4-pandora-bodge-dbg -r artifactory --update \
  -o enable_ccache=True -o static_executables=True -o device=mi450 \
  --build=protobuf_compiler --build=glslc --build=grpc_plugin \
  --build=gtest --build=grpc --build=benchmark
$CONAN build ../tng
```

## Output

Build artifacts land in `builddir/package/`. These are also published to:

```
/proj/smartnic/xcb/ifoe/diags/diag_tng-${GIT_HASH}
```

## Deploying for QEMU

- Diags binaries are large; don't include them in QEMU images.
- Instead, mount a pre-built diags directory that XCE can access.
- Also requires reserved memory in the QEMU kernel that the kernel doesn't touch
  (see IFOESW-782).
