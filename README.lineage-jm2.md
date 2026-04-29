# jm2/android_vendor_lineage — Lineage notes

This fork exists for four downstream patches needed by the
jm2/android_kernel_oneplus_sm8850* tree to build cleanly under
`mka kernel` with vendor's external module wrappers. It tracks
`LineageOS/android_vendor_lineage` (`github` remote) and adds these
commits on top of `lineage-23.2`, oldest first:

- `2f99de7a kernel.mk: pass 'modules' explicitly to external module wrappers`
- `2ba2ed0e kernel.mk: merge BOARD_VENDOR_KERNEL_MODULES prebuilts into the source-build flow`
- `4e196512 kernel.mk: wipe stale oem/ before repopulating from BOARD_VENDOR_KERNEL_MODULES`
- `e8a47255 kernel.mk: respect BOARD_KERNEL_MODULES_LOAD_ALLOW_MISSING for BOOT/RECOVERY/SYSTEM lists`

(plus `3ab7c8ac` adding this README.)

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

## Upstream status

None of these are proposed to LineageOS upstream yet. All four are
small, self-contained, and reproduce on any LineageOS build using the
relevant `BOARD_*` knobs against a Qualcomm-shaped module tree.

## Companion repos

- [`jm2/android_kernel_oneplus_sm8850`](https://github.com/jm2/android_kernel_oneplus_sm8850)
- [`jm2/android_kernel_oneplus_sm8850-modules`](https://github.com/jm2/android_kernel_oneplus_sm8850-modules)
- [`jm2/android_device_oneplus_sm8850-common`](https://github.com/jm2/android_device_oneplus_sm8850-common)
