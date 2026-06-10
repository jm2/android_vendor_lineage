# jm2/android_vendor_lineage — Lineage notes

This fork exists for downstream `build/tasks/kernel.mk` patches needed
by the jm2/android_kernel_oneplus_sm8850* trees. It tracks
`LineageOS/android_vendor_lineage` (`github` remote) and adds these
patches on top of `lineage-23.2`, oldest first (hashes drift on
rebase; titles are stable):

- `kernel.mk: pass 'modules' explicitly to external module wrappers`
- `kernel.mk: merge BOARD_VENDOR_KERNEL_MODULES prebuilts into the source-build flow`
- `kernel.mk: wipe stale oem/ before repopulating from BOARD_VENDOR_KERNEL_MODULES`
- `kernel.mk: respect BOARD_KERNEL_MODULES_LOAD_ALLOW_MISSING for BOOT/RECOVERY/SYSTEM lists`
- `kernel.mk: depmod-layer bridge for OEM-prebuilt-sibling-producer class`
- `kernel.mk: OEM-wrapper-driven branch for the kernel platform (Kleaf) path` (patch 5 below)
- `kernel.mk: tolerate missing first-stage modules on the Kleaf path under ALLOW_MISSING` (patch 6 below)

## Patch 1 — modules target passed explicitly

Vendor's external module wrappers in
`kernel/oneplus/sm8850-modules/vendor/qcom/opensource/*/Makefile`
typically have:

```make
all: modules
%:
	$(MAKE) -C $(KERNEL_SRC) M=$(M) $@ $(KBUILD_OPTIONS)
```

When `vendor/lineage`'s `build/tasks/kernel.mk` invokes the wrapper
with no explicit target, GNU make picks `all` (the default goal),
which depends on `modules` but has no recipe of its own. After
satisfying `modules` via the `%:` catch-all, make then needs to
"build" `all` itself — also via `%:` since there's no explicit
recipe — so it fires a SECOND `make -C $(KERNEL_SRC) M=$(M) all`
against the kernel. That second invocation lands on the kernel's
`all` target, cascades through `arch/arm64/Makefile` to build
Image, which fails because `M=` is in play and `VPATH` is empty for
external module builds (so in-tree subdir lookups like
`arch/arm64/boot/Makefile` resolve against objtree and miss).

Passing `modules` explicitly bypasses the wildcard fallthrough on
`all`. The wrapper's `%:` still catches `modules` (it does its job
once and stops), and the kernel make then runs only its `modules`
target, not `all`.

```diff
-                                       $(call $(if $(filter $(word 2,$(subst :, ,$(p))),kbuild),make-kbuild-module-target,make-external-module-target),$(word 1,$(subst :, ,$(p))),$$rpath,) || exit "$$?";  \
+                                       $(call $(if $(filter $(word 2,$(subst :, ,$(p))),kbuild),make-kbuild-module-target,make-external-module-target),$(word 1,$(subst :, ,$(p))),$$rpath,modules) || exit "$$?";  \
```

Reproduces under any LineageOS build that uses
`TARGET_KERNEL_EXT_MODULES` against an external module whose Makefile
has `all: modules` + `%:` catch-all but no explicit `modules:` rule —
which is a common shape across Qualcomm vendor module trees.

## Patch 2 — BOARD_VENDOR_KERNEL_MODULES prebuilts merge into the source-build flow

Stock `kernel.mk` has separate paths for "build kernel + ext modules
from source" vs "copy `BOARD_VENDOR_KERNEL_MODULES` prebuilt `.ko`s
into the partition." For the OnePlus 15 hybrid (source-built kernel
+ 30 source-built externals + 568 OEM prebuilt `.ko`), both paths
need to populate the same depmod staging. This patch merges the
prebuilt copy into the source-build flow so depmod sees both before
generating `modules.dep`/`modules.alias`.

## Patch 3 — wipe stale oem/ before repopulating

Companion to patch 2: when the prebuilt set changes between
incremental builds, leftover files in `kernel_modules_dir/oem/` would
shadow the new set. `rm -rf` before each populate avoids that.

## Patch 4 — soft-fail on missing modules.list entries

Vendor's `modules.list.msm.canoe` / `modules.load.recovery` /
`modules.load.system_dlkm` are bloated with entries for SoC variants
the canoe/infiniti build doesn't ship (sun, tuna, kera, niobe) plus
helpers we don't have source-built today. Stock `kernel.mk` errors
the build if any line in those lists is missing. This patch makes
the BOOT / RECOVERY / SYSTEM kernel-modules-not-found check honor
`BOARD_KERNEL_MODULES_LOAD_ALLOW_MISSING := true`, so missing entries
become warnings rather than hard errors. The same flag is already
honored by the `vendor_dlkm` path; this just brings the other three
phases in line.

## Patch 5 — OEM-wrapper-driven Kleaf platform branch

Some QC OEM kernel drops (OnePlus SM8850) keep `.repo` at the repo root
but the bazel workspace in a `kernel_platform/` subdir, driven by an
OEM wrapper script — a shape the stock `TARGET_KERNEL_PLATFORM_TARGET`
path (repo-root == workspace, bare `bazel run`) cannot consume. When
`TARGET_KERNEL_PLATFORM_BUILD_WRAPPER` is set, the recipe instead runs
that wrapper from `TARGET_KERNEL_PLATFORM_ROOT` and copies
`TARGET_KERNEL_PLATFORM_DIST` into `KERNEL_OUT`; module collection and
dtb/dtbo packaging are unchanged. Strictly opt-in; the stock branch is
preserved verbatim when the vars are unset. First consumer:
`device/oneplus/sm8850-common/kernel-build/build-canoe-kleaf.sh`.

## Patch 6 — tolerant first-stage staging on the Kleaf path

On the Kleaf path, `BOOT_KERNEL_MODULES` / `RECOVERY_KERNEL_MODULES`
are staged via a direct `cp` of `$(KERNEL_OUT)/<name>`, so one missing
.ko hard-fails the build — and ALLOW_MISSING (patch 4) only covered the
FULL_KERNEL_BUILD path's load-list checks. With
`BOARD_KERNEL_MODULES_LOAD_ALLOW_MISSING := true`, missing first-stage
modules are now skipped with a warning (shell-side filter at recipe
runtime — a make `$(wildcard)` would evaluate before the kernel build
step of the same recipe populates KERNEL_OUT). Needed because a few
stock-loaded OnePlus modules (e.g. `oplus_bsp_ex_gpio`) have no
published source anywhere; see the device tree's
`kernel-build/known-source-gaps.txt`.

## Upstream status

None of these are proposed to LineageOS upstream yet. All four are
small, self-contained, and reproduce on any LineageOS build using the
relevant `BOARD_*` knobs against a Qualcomm-shaped module tree.

## Companion repos

- [`jm2/android_kernel_oneplus_sm8850`](https://github.com/jm2/android_kernel_oneplus_sm8850)
- [`jm2/android_kernel_oneplus_sm8850-modules`](https://github.com/jm2/android_kernel_oneplus_sm8850-modules)
- [`jm2/android_device_oneplus_sm8850-common`](https://github.com/jm2/android_device_oneplus_sm8850-common)
