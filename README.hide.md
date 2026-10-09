# Anti-Detection Patches (ko-hide-lsmod)

This branch adds a set of detection-hardening patches on top of upstream KernelSU.
Goal: remove the cheap, file-level fingerprints that commodity root-detection
SDKs scan for. It does **not** (and cannot) protect against timing side-channels,
hardware attestation (Play Integrity STRONG), or server-side fingerprinting.

Builds marked `CONFIG_KSU_DEBUG=y` intentionally keep everything visible.

## Patch inventory

### 1. Hide the module from `/proc/modules` and `/sys/module`

- **File:** `kernel/core/init.c`
- **What:** after successful init, `list_del_init(&THIS_MODULE->list)` removes the
  module from the kernel's module list (lsmod reads that list), complementing the
  upstream `kobject_del(&THIS_MODULE->mkobj.kobj)` which removes `/sys/module`.
- **Side effect:** `rmmod kernelsu` can no longer find the module — it cannot be
  unloaded by name. Reboot to unload. Expected for a hiding patch.
- **Verify:** `lsmod | grep kernel` is empty; `ls /sys/module | grep kernel` empty.

### 2. Symbol washing: random per-build prefix + `.BTF` strip

- **File:** `kernel/Makefile`
- **What:** after the ko is linked and `check_symbol` passes, every *defined*
  symbol is renamed via `objcopy --redefine-syms` to a random per-build prefix
  (`ks<hex>_`). Undefined symbols (resolved against vmlinux at load time, e.g.
  `kallsyms_lookup_name`, `selinux_state`) are left untouched. The `.BTF` section
  is removed because BTF carries original type/function names (`ksu_driver_context`,
  etc.). AArch64 mapping symbols (`$d`, `$x`) are skipped.
- **Why random:** two builds of the same source produce different symbol tables,
  so a detector cannot maintain a static `ksu_*` name list — it must match your
  specific build.
- **Side effects:**
  - Kernel stack traces / oops no longer show readable `ksu_*` names (they show
    the random prefix). Debugging crashes is harder; pair with matching `System.map`
    if you need it.
  - Requires `nm` and an objcopy that understands the target (the Makefile prefers
    `llvm-objcopy` next to `clang`; GNU objcopy in minimal containers often lacks
    aarch64 support).
- **Verify:** `cat /proc/kallsyms | grep ksu_` is empty; random-prefix symbols
  (`ks[0-9a-f]*_`) are present and root still works.

### 3. dmesg banner silenced

- **File:** `kernel/core/init.c`
- **What:** the `welcome to KernelSU version ... package name ...` banner is
  demoted from `pr_info` to `pr_debug`, so it no longer appears in dmesg.
- **Side effect:** none at runtime; with dynamic debug enabled
  (`echo 'file init.c +p' > /sys/kernel/debug/dynamic_debug/control`) it prints again.

### 4. ~~Neutral anon-inode names~~ — reverted

Neutralizing the supercall fd names (`[ksu_driver]` -> `[eventfd]`) was tried
and **reverted**: the name is a discovery contract with userspace — manager
(`manager/app/src/main/cpp/ksu.cc`) and ksud (`userspace/ksud/src/ksucalls.rs`)
find the driver fd by scanning `/proc/self/fd` for `[ksu_driver]`. Renaming it
breaks installation detection ("未安装"). It also leaks nothing to unprivileged
detectors: the fd only exists in processes that already hold it.

## Quick verification script (on device)

```sh
su
echo "== lsmod ==";          lsmod | grep kernel
echo "== /sys/module ==";    ls /sys/module | grep kernel
echo "== kallsyms ==";       grep ksu_ /proc/kallsyms
echo "== dmesg ==";          dmesg | grep -i kernelsu
```

Expected: all empty; `grep -E 'ks[0-9a-f]{8}_' /proc/kallsyms` shows your build's
random-prefix symbols.

## What this branch does NOT protect against

- **Timing side-channels:** syscall hooking adds latency; detectors measure it
  (see public APatch timing-detection research, Jan 2026).
- **Play Integrity STRONG:** hardware-backed attestation cannot be spoofed from
  the kernel side; use configuration-level tooling for that.
- **The published manager:** release APKs carry a public signing certificate and
  the stock package name (`me.weishu.kernelsu`). Anyone can build a
  "installed this package => rooted" rule. Rebuild the manager with your own
  `KSU_PACKAGE_NAME` and signing key if this matters to you.
