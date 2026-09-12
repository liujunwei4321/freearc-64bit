# FreeArc Build Guide (64-bit Linux, all fixes applied)

This guide documents how to rebuild every FreeArc artifact from the fixed source trees, with the exact toolchains and commands used and verified during the porting effort.

Applies to the fixed source packages:

- `freearc-0.67-fork-fixed-src.tar.gz` → **fork 0.67** (Haskell console `freearc` + C++ `clibs`)
- `freearc-master-0.67-fixed-src.tar.gz` → **master 0.67** (C++ `unarc` + SFX stubs on Linux)
- `freearc-0.51-fixed-src.tar.gz` → **0.51** (Haskell `arc` + `unarc`/SFX)

---

## 1. Toolchains (exact versions used)

### Linux aarch64 (Kylin V10 — "ssh20")

| Component | Version / location |
|---|---|
| g++ | 9.4.2 (`g++ --version`) |
| GHC | 9.4.2 at `/home/test/ghc-9.4.2/bin/ghc` |
| cabal-install | 3.12.1.0 at `/home/test/cabal312/cabal` |
| cabal store / package db | `/home/test/.local/state/cabal/store/ghc-9.4.2/package.db` (provides `old-time`, `old-locale`) |
| make, gdb, strace | system packages |
| AddressSanitizer | supported by the system g++ |

### Linux x86_64 (Kylin V10 SP1 — "ssh1")

| Component | Version / location |
|---|---|
| g++ | 9.3.0 (Kylin build) |
| GHC | 9.4.8 at `/mnt/bigdisk/ssh1-freearc/ghc-9.4.2/bin/ghc` (the directory name is misleading; the binaries are 9.4.8) |
| cabal-install | 3.12.1.0 at `/mnt/bigdisk/ssh1-freearc/cabal313/cabal` |
| **CABAL_DIR (required)** | `export CABAL_DIR=/mnt/bigdisk/ssh1-freearc/cabal-dir` — without it cabal reports *"The package list for 'hackage.haskell.org' does not exist"* |
| package db for 0.51 | `/mnt/bigdisk/ssh1-freearc/cabal-dir/store/ghc-9.4.8/package.db` |
| libcurl | no `libcurl.so` dev symlink → build 0.51 without `-lcurl` (clibs are compiled with `-DFREEARC_NOURL`) |

### Linux loongarch64 (Kylin V10 — "ssh19")

| Component | Version / location |
|---|---|
| g++ | 8.3.0 (Kylin/Loongson build, `8.3.0-6.lnd.vec`) — AddressSanitizer works |
| GHC | **not available** (no loongarch64 backend) → only C++ targets are built here: `unarc` + 6 SFX stubs |
| extra define | the Unarc `common.mak` needs `-DFREEARC_64BIT` added to `DEFINES` (the architecture check in `Common.h` knows only x86/aarch64) |

### Windows (reference-archive generator only, not a build machine)

| Component | Version |
|---|---|
| FreeArc console 0.666 | May 20 2010 (`...\freearc-0.666-win32\bin\Arc.exe`) |
| FreeArc console 0.51 | Apr 28 2009 (`...\FreeArc-console-0.51-win32\bin\Arc.exe`) |

### Control machine (Windows) — file transfer

Python 3 + `paramiko` (script `sshr.py` with `run`/`put`/`get`). **Important:** when invoking Python from Git Bash with POSIX-looking remote paths, set `MSYS_NO_PATHCONV=1`, otherwise Git Bash rewrites `/home/...` arguments into Windows paths and uploads land in the wrong place.

---

## 2. Source tree layout (after extraction)

```
fork 0.67:  freearc-main/
              clibs/Compression/…        (all C++ codecs, PPMD/, _Encryption/, GRZip/, …)
              app/*.hs                   (Haskell console)
              compile                    (root script → clibs/Compression/compile + clibs make)
              freearc.cabal
master:     FreeArc-master/
              Compression/…              (C++ codecs)
              Unarc/                     (unarc + SFX stubs, C++ only on Linux)
              Arc.hs, Utils.hs, …       (Haskell, not built on Linux here)
0.51:       FreeArc-0.51-sources/
              Compression/…              (C++ codecs incl. Compression/PPMD)
              Unarc/
              Arc.hs + Compression/*.hs  (Haskell, ghc --make build)
              unix-common.mak
```

All object files for the Haskell-linked builds go to the **shared** directory `/tmp/out/FreeArc/`; the `Unarc` makefile uses `/tmp/out/FreeArc-unarc/`. These directories are shared between trees — every `compile` step wipes and rebuilds its own set (see §5, ordering notes).

---

## 3. Build recipes

### 3.1 fork 0.67 — full `freearc` console (aarch64 / x86_64)

```bash
# ssh20:
export PATH=/home/test/ghc-9.4.2/bin:/home/test/cabal312:$PATH
# ssh1 (note the extra CABAL_DIR):
export CABAL_DIR=/mnt/bigdisk/ssh1-freearc/cabal-dir
export PATH=/mnt/bigdisk/ssh1-freearc/ghc-9.4.2/bin:/mnt/bigdisk/ssh1-freearc/cabal313:$PATH

cd freearc-main
rm -f /tmp/out/FreeArc/*.o            # wipe the shared object dir
bash compile                          # builds all clibs objects → /tmp/out/FreeArc/ (22 objects)
find dist-newstyle -type d -name cache -exec rm -rf {} +
rm -f $(find dist-newstyle -name freearc -type f)
cabal build --ghc-options=-O2

# deploy (unstripped first, then stripped):
BIN=$(find dist-newstyle -name freearc -type f | head -1)
cp "$BIN" ~/freearc-bins/freearc-0.67-fork-O2-unstripped
strip "$BIN" && cp "$BIN" ~/freearc-bins/freearc-0.67-fork-O2-stripped
```

Notes:
- The cabal file links the C++ objects by absolute path from `/tmp/out/FreeArc/*.o` — the `compile` step above is mandatory whenever any clibs source (e.g. `PPMD/`, `_Encryption/`) changes. Running only `cabal build` after a C++ change silently relinks stale objects.
- Result sizes after all fixes: aarch64 ≈ 24.5 MB stripped, x86_64 ≈ 7.9 MB stripped.

### 3.2 master 0.67 — `unarc` + 3 SFX stubs (all three Linux machines)

```bash
cd FreeArc-master/Unarc
# loongarch64 only — add the explicit 64-bit define:
#   cp unix-common.mak common.mak
#   sed -i 's/^DEFINES  = /DEFINES  = -DFREEARC_64BIT /' common.mak
rm -rf /tmp/out/FreeArc-unarc && mkdir -p /tmp/out/FreeArc-unarc
rm -f unarc arc*.linux.sfx
make unarc arc.linux.sfx arc-mini.linux.sfx arc-tiny.linux.sfx
```

Deploy names (per machine `freearc-bins/`): `unarc-0.67-stripped`, `sfx-<arch>/arc-067-full|mini|tiny.linux.sfx`.

### 3.3 0.51 — full `arc` console (aarch64 / x86_64)

```bash
# ssh20: PATH + DB as in §3.1
# ssh1: CABAL_DIR + PATH as in §3.1, DB=/mnt/bigdisk/ssh1-freearc/cabal-dir/store/ghc-9.4.8/package.db

cd FreeArc-0.51-sources
cp unix-common.mak common.mak
rm -f /tmp/out/FreeArc/*.o
(cd Compression && bash compile)     # 15 codec objects
make                                  # adds Environment/GuiEnvironment/URL → 18 objects

C_MODULES="/tmp/out/FreeArc/Environment.o /tmp/out/FreeArc/GuiEnvironment.o /tmp/out/FreeArc/URL.o \
/tmp/out/FreeArc/Common.o /tmp/out/FreeArc/CompressionLibrary.o /tmp/out/FreeArc/C_PPMD.o \
/tmp/out/FreeArc/C_LZP.o /tmp/out/FreeArc/C_LZMA.o /tmp/out/FreeArc/C_BCJ.o \
/tmp/out/FreeArc/C_GRZip.o /tmp/out/FreeArc/C_Dict.o /tmp/out/FreeArc/C_REP.o \
/tmp/out/FreeArc/C_MM.o /tmp/out/FreeArc/C_TTA.o /tmp/out/FreeArc/C_Tornado.o \
/tmp/out/FreeArc/C_Delta.o /tmp/out/FreeArc/C_External.o /tmp/out/FreeArc/C_Encryption.o"

mkdir -p /tmp/fa051-obj2
ghc --make -O2 Arc.hs -iCompression -threaded -fglasgow-exts \
  -XOverlappingInstances -XUndecidableInstances -XNoMonomorphismRestriction \
  -XBangPatterns -XNondecreasingIndentation -XBlockArguments \
  -DFREEARC_PACKED_STRINGS -DFREEARC_UNIX -DFREEARC_INTEL_BYTE_ORDER -DFREEARC_NO_LUA \
  -optc-DFREEARC_UNIX -optc-DFREEARC_INTEL_BYTE_ORDER \
  -package-db "$DB" -package old-time -package old-locale \
  $C_MODULES -optl-s -lstdc++ -lncurses [-lcurl] \
  -odir /tmp/fa051-obj2 -hidir /tmp/fa051-obj2 -o /tmp/arc051
```

- Include `-lcurl` on ssh20; **omit it on ssh1** (no `libcurl.so` symlink; the codecs are already built with `-DFREEARC_NOURL`, so the binary is functionally identical to the nocurl variant).
- Deploy: copy `/tmp/arc051` to `freearc-bins/freearc-0.51-O2-unstripped`, strip, copy as `…-O2-stripped`.

### 3.4 0.51 — 3 SFX stubs (all three Linux machines)

```bash
cd FreeArc-0.51-sources/Unarc
rm -rf /tmp/out/FreeArc-unarc && mkdir -p /tmp/out/FreeArc-unarc
rm -f arc*.linux.sfx
make arc.linux.sfx arc-mini.linux.sfx arc-tiny.linux.sfx
```

Deploy: `sfx-<arch>/arc-051-full|mini|tiny.linux.sfx`.

---

## 4. Sanitizer / regression harnesses

### 4.1 PPMd ASAN harness

```bash
M=<fork>/clibs/Compression
g++ -fsanitize=address -fsanitize-recover=address -g -O1 \
  -DFREEARC_UNIX -DFREEARC_NOURL [-DFREEARC_64BIT] -D_FILE_OFFSET_BITS=64 \
  -I$M/PPMD -I$M -include stdlib.h -include string.h -include stdio.h \
  ppmd_harness.cpp $M/PPMD/Model.cpp $M/Common.cpp -o ppmd_asan

ASAN_OPTIONS=halt_on_error=0 ./ppmd_asan <order 2..128> <t|r|b> <mr 0|1|2> <mem MB>
```

- `t` = repetitive text, `r` = 200 KB random, `b` = 3 MB mixed.
- Small memory (2–16 MB) plus `mr=1/2` forces the allocator recovery paths (`cutOff`, `GlueFreeBlocks`, `ExpandTextArea`).
- Pass criterion: `ROUNDTRIP OK` and zero `ERROR: AddressSanitizer` lines.
- On loongarch64 add `-DFREEARC_64BIT`; plain `-O2` builds without sanitizers are also part of the matrix (they caught the strict-aliasing regression).

### 4.2 Cross-platform matrix (Windows ↔ Linux)

Assets: `C:\Users\ljw\xtest\` locally, `/tmp/xtest/` on each Linux machine; scripts `linux_verify.sh`, `linux_create.sh` (Linux), `win_phase1.sh`, `win_verify.sh`, `win_phase3.sh` (Windows).

```bash
# Windows: generate 30 reference archives with the 0.666/0.51 consoles and self-verify
bash win_phase1.sh && bash win_verify.sh

# Linux (each machine): decode every Windows archive with every local tool
bash linux_verify.sh <ssh20|ssh1|ssh19>

# Linux: create archives with fork/0.51 → pull to Windows
bash linux_create.sh <ssh20|ssh1>
# Windows: decode the Linux archives and verify content
bash win_phase3.sh
```

Pass criterion: every extracted file byte-identical (sha256) to the corpus; CJK tree archives compared as content multisets (names are platform-encoding sensitive).

---

## 5. Build-order and environment pitfalls

1. **Shared `/tmp/out/FreeArc`** — the fork, 0.51 `Compression/compile` and the top-level 0.51 `make` all write here and each wipe it first. If you later relink the fork (`cabal build`), restore its objects first with `rm -f /tmp/out/FreeArc/*.o && bash compile` inside `freearc-main`, otherwise the linker fails with missing `CELS.o/C_4x4.o/…` (those modules exist only in the fork tree).
2. **`/tmp/out/FreeArc-unarc`** — clean it (`rm -rf` + `mkdir -p`) before switching between the master and 0.51 `Unarc` builds.
3. **ssh1 cabal** — keep `CABAL_DIR=/mnt/bigdisk/ssh1-freearc/cabal-dir` exported; the default cabal config on that box has no hackage index and every clean rebuild fails with *"The package list for 'hackage.haskell.org' does not exist"*.
4. **`common.mak`** — the 0.51 tree needs `cp unix-common.mak common.mak` before building; on loongarch64 add `-DFREEARC_64BIT` to `DEFINES` (already done in the shipped trees' `unix-common.mak`).
5. **Link flags** — 0.51 on ssh1: drop `-lcurl` (see §1). The fix is size-neutral; `-DFREEARC_NOURL` was applied when compiling the clibs.
6. **tiny/mini SFX + `C_Encryption.o`** — never link the encryption object into the tiny/mini stubs: they are compiled without `UNARC_DECRYPTION` and contain their own `Pbkdf2Hmac` stub (duplicate symbol at link time). Encrypted archives are rejected by tiny/mini by design (upstream behavior).
7. **`LTC_NO_TEST`** — `C_Encryption.cpp` defines it, so the cipher self-tests are compiled out. The serpent regression was caught with a standalone harness that compiles `ciphers/serpent.c` + `crypt/crypt_argchk.c` directly and calls `serpent_test()`.
8. **Git Bash on Windows** — when uploading files with `python sshr.py put/get`, prefix with `MSYS_NO_PATHCONV=1` or POSIX remote paths get rewritten (`/home/test/…` → `C:/Program Files/Git/home/test/…`) and files silently land elsewhere.
9. **Rebuild triggers** — `make` dependency lists in `Unarc/makefile` do not include the LTC headers; after changing `_Encryption/headers/*.h`, wipe `/tmp/out/FreeArc-unarc` and rebuild from scratch.

---

## 6. Verification snapshot (expected results after a clean rebuild)

| Check | Command | Expected |
|---|---|---|
| fork version | `freearc --help` | FreeArc 0.67 (March 15 2014) |
| PPMd round-trip | `freearc a -m=ppmd t.arc <data> && unarc x t.arc && cmp` | byte-identical |
| password archive | `freearc a -ptestpw p.arc f && unarc x -ptestpw p.arc && cmp` | byte-identical |
| serpent archive | `freearc a -aeserpent -pserpw s.arc f && unarc x -pserpw s.arc && cmp` | byte-identical |
| SFX methods | `cat arc-067-tiny.linux.sfx t.arc > t && printf 'y\n' | ./t` | extracts (grzip/ppmd payloads included) |
| `arc s` from foreign CWD | place `arc.sfx` next to the executable, run `arc s -sfxarc.sfx a.arc` from any directory | SFX archive produced and self-extractable |
| CJK under `LC_ALL=C` | `LC_ALL=C freearc a t.arc 中文文件.txt` | rc=0, file archived |
| ASAN matrix | §4.1 | all `ROUNDTRIP OK`, zero ASAN reports |
| cross matrix | §4.2 | P1 30/30, P2 66+66+30, P3 30+30 — zero failures |
