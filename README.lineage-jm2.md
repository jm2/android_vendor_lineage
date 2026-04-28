# jm2/android_vendor_lineage — Lineage notes

This fork exists for one downstream patch needed by the
jm2/android_kernel_oneplus_sm8850* tree to build cleanly under
`mka kernel` with vendor's external module wrappers. It tracks
`LineageOS/android_vendor_lineage` (`github` remote) and adds
exactly one commit on top of `lineage-23.2`:

- `2f99de7a kernel.mk: pass 'modules' explicitly to external module wrappers`

## Why this patch is needed

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

## Upstream status

Not yet proposed to LineageOS upstream. The diff is two characters
on one line and self-contained; should be straightforward to PR
once someone has the time:

```diff
-                                       $(call $(if $(filter $(word 2,$(subst :, ,$(p))),kbuild),make-kbuild-module-target,make-external-module-target),$(word 1,$(subst :, ,$(p))),$$rpath,) || exit "$$?";  \
+                                       $(call $(if $(filter $(word 2,$(subst :, ,$(p))),kbuild),make-kbuild-module-target,make-external-module-target),$(word 1,$(subst :, ,$(p))),$$rpath,modules) || exit "$$?";  \
```

(Reproduces under any LineageOS build that uses
`TARGET_KERNEL_EXT_MODULES` against an external module whose Makefile
has `all: modules` + `%:` catch-all but no explicit `modules:` rule —
which is a common shape across Qualcomm vendor module trees.)

## Companion repos

- [`jm2/android_kernel_oneplus_sm8850`](https://github.com/jm2/android_kernel_oneplus_sm8850)
- [`jm2/android_kernel_oneplus_sm8850-modules`](https://github.com/jm2/android_kernel_oneplus_sm8850-modules)
- [`jm2/android_device_oneplus_sm8850-common`](https://github.com/jm2/android_device_oneplus_sm8850-common)
