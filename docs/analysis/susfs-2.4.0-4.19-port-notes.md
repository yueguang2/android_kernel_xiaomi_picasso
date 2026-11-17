# SUSFS 2.4.0 → Linux 4.19 (picasso) Porting Notes

**Target:** Xiaomi Redmi K30 5G (`picasso`), Linux **4.19.248**, arm64, non-GKI, single boot partition, KernelSU 3.3.1.
**Upstream source analysed:** `/tmp/susfs4ksu-2.4.0`, branch `gki-android14-6.1`.
**Main patch:** `kernel_patches/50_add_susfs_in_gki-android14-6.1.patch` (3438 lines; written for 6.1 GKI).
**Baseline branch (B):** `feature/ksu3.3.1-on-73fe` — plain KernelSU 3.3.1, no SUSFS.
**Working 4.19 SUSFS reference (R):** `feature/ksu3.3.1-susfs-1.5.11-on-73fe` — current working branch, SUSFS v1.5.11 backport already booting on 4.19.
**Report scope:** every file touched by the 2.4.0 patch, plus `fs/susfs.c`, `include/linux/susfs.h`, `include/linux/susfs_def.h`.
**Constraint:** this report is analysis only; no kernel source has been modified.

Reference notation used below:

* `P:Lx–Ly` = line range in `50_add_susfs_in_gki-android14-6.1.patch`.
* `B:path:line` = exact line in the plain 4.19 KernelSU branch (the real 4.19 context).
* `R:path:line` = exact line in the current 1.5.11 4.19 SUSFS branch; this is the preferred starting point for adaptation because it already contains working 4.19 hook code.

---

## 1. Executive summary

1. **The 2.4.0 patch cannot be applied as a patch.** Every non-trivial hunk targets 6.1 VFS, fsnotify, procfs, or SELinux internals. A literal `patch -p1` will fail in `fs/namei.c`, `fs/namespace.c`, `fs/notify/fdinfo.c`, `fs/proc/*`, `fs/stat.c`, `fs/statfs.c`, `security/selinux/*`, and `kernel/sys.c`. The port must be a **manual re-implementation using the 2.4.0 source as the feature specification**, not a rebase of the patch file.

2. **The current R branch already solves several 4.19 infrastructure problems that 2.4.0 assumes:**
   * `struct nameidata` has an extra `state` field (`R:fs/namei.c`, added by the 1.5.11 backport). 2.4.0's `ND_STATE_LOOKUP_LAST`/`ND_STATE_OPEN_LAST` logic depends on it.
   * `struct vfsmount` has `susfs_mnt_id_backup` in the Android KABI reserve (`R:include/linux/mount.h`). 2.4.0's `fs/susfs.c` does not strictly need it, but R's namespace code and 1.5.11 helpers already use the pattern.
   * R's `fs/namei.c`, `fs/namespace.c`, `fs/readdir.c`, `fs/statfs.c`, `fs/notify/fdinfo.c`, `fs/proc_namespace.c` contain working 4.19 hook templates for SUS_PATH, SUS_MOUNT, SUS_KSTAT and OPEN_REDIRECT. These should be copied and then extended to 2.4.0 semantics; do **not** start from the 6.1 hunks.

3. **KernelSU 3.3.1 is the real blocker for full parity, not the 4.19 kernel APIs.** 2.4.0 assumes a much newer KernelSU hook layer:
   * 2.4.0 converts KernelSU flags to static keys: `DEFINE_STATIC_KEY_TRUE(ksu_su_compat_enabled)`, `ksu_is_input_hook_enabled`, `ksu_is_init_rc_hook_enabled`.
     The current submodule still has plain `bool ksu_su_compat_enabled` (`KernelSU/kernel/feature/sucompat.c:10`), `bool ksu_input_hook`/`bool ksu_vfs_read_hook` (`KernelSU/kernel/feature/ksud_integration.c:26–27`), and no `ksu_is_init_rc_hook_enabled`.
   * 2.4.0 uses `ksu_handle_post_execveat_sucompat()`, `ksu_handle_setresuid(ruid,euid,suid)`, `ksu_setup_preinit_file()`, `ksu_selinux_hide_running`, `fake_status`/`initialize_fake_status`, `susfs_ksu_sid`, `susfs_priv_app_sid`, and `susfs_is_current_ksu_domain()`. None of these exist in the current KernelSU submodule. The available primitives are `ksu_handle_execveat()`, `ksu_handle_execveat_sucompat()`, `ksu_handle_faccessat()`, `ksu_handle_stat()`, `ksu_handle_setresuid_cred()`, `ksu_selinux_hide_enabled` (static), and `is_ksu_domain()`.
   * Consequently, 2.4.0's exec/open/stat hooks that call `ksu_handle_*` with 2.4.0 signatures must either be dropped in Phase A or paired with a KernelSU patch. See §3.

4. **Recommended staging:**
   * **Phase A — minimal bootable port:** `CONFIG_KSU_SUSFS=y` (core `fs/susfs.c`), `CONFIG_KSU_SUSFS_ENABLE_LOG`, `CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS`, `CONFIG_KSU_SUSFS_SPOOF_UNAME`, `CONFIG_KSU_SUSFS_PRIORITY_UID_FILTER`, and `CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG` (cmdline only; no `/proc/bootconfig`). All VFS/mount/SELinux feature configs off. This exercises the core data structures, the KSU command interface, uname spoofing, and symbol hiding without touching the highest-risk code paths.
   * **Phase B — full parity, in this order:**
     1. `CONFIG_KSU_SUSFS_SUS_PATH` (namei + readdir + proc readlink); reuse R's 1.5.11 namei/readdir templates.
     2. `CONFIG_KSU_SUSFS_SUS_KSTAT` (stat/statfs/fdinfo/task_mmu); no kernel-namei risk, but many display paths.
     3. `CONFIG_KSU_SUSFS_SUS_MOUNT` (namespace + proc mounts + fdinfo); highest risk due to mount internals.
     4. `CONFIG_KSU_SUSFS_OPEN_REDIRECT` (namei + proc readlink + task_mmu); depends on SUS_PATH-style pathwalk plumbing.
     5. `CONFIG_KSU_SUSFS_SUS_MAP` (task_mmu + proc base); self-contained.
     6. SELinux policy hiding (`security/selinux/*`), only after the KernelSU side exports the required state (`ksu_selinux_hide_running` / fake status / `susfs_ksu_sid` / `susfs_priv_app_sid`). This is effectively a separate KernelSU + SELinux port, not a kernel-only task.

5. **Estimated rough effort (one experienced kernel porter):** Phase A 1–2 days; SUS_PATH 2–3 days; SUS_KSTAT 1–2 days; SUS_MOUNT 3–5 days; OPEN_REDIRECT 2–3 days; SUS_MAP 1 day; SELinux full parity 1–2 weeks including KernelSU changes. These are engineering estimates, not promises.

---

## 2. Quick matrix

| # | File touched by 2.4.0 | Patch lines | 4.19 status | Risk | Phase |
|---|------------------------|-------------|-------------|------|-------|
| 1 | `block/ioprio.c` | 2–48 | functions exist, signature-compatible | low | A (optional) |
| 2 | `drivers/input/input.c` | 51–78 | existing bool KSU hook, no `input_event_dispose` | low | skip |
| 3 | `fs/Makefile` | 79–91 | exists, different `obj-y` contents | low | A |
| 4 | `fs/exec.c` | 92–195 | `__do_execve_file`/`do_execveat_common` exist; no `bprm_execve` | high | B/skip |
| 5 | `fs/namei.c` | 196–637 | major pathwalk rewrite needed | high | B |
| 6 | `fs/namespace.c` | 638–972 | no `vfs_create_mount`, no `mnt_userns`; mount internals differ | high | B |
| 7 | `fs/notify/fdinfo.c` | 973–1093 | callback signature and group lock differ | medium | B |
| 8 | `fs/open.c` | 1094–1154 | existing KSU faccessat hook; 6.1 helpers absent | low | skip |
| 9 | `fs/proc/base.c` | 1155–1210 | `do_proc_readlink`/`proc_map_files_readdir` exist; loop rewrite | medium | B |
| 10 | `fs/proc/bootconfig.c` | 1211–1236 | **file absent in 4.19** | low | A via `cmdline.c` |
| 11 | `fs/proc/fd.c` | 1237–1335 | `seq_show` exists, output format differs | medium | B |
| 12 | `fs/proc/task_mmu.c` | 1336–1471 | `vma_lookup`/`mas_find`/`VMA_PAD_START` absent | medium | B |
| 13 | `fs/proc_namespace.c` | 1472–1690 | show functions exist, close to 4.19 | medium | B |
| 14 | `fs/read_write.c` | 1691–1713 | existing KSU vfs_read bool hook | low | skip |
| 15 | `fs/readdir.c` | 1714–1981 | callbacks rewritten (`filldir` bool→int, no `prev_reclen`) | medium-high | B |
| 16 | `fs/stat.c` | 1982–2113 | no `struct kstat.mnt_id`, no `statx` mnt-id API | medium-high | B |
| 17 | `fs/statfs.c` | 2114–2221 | `statfs_by_dentry`/`calculate_f_flags` exist | low-medium | B |
| 18 | `fs/super.c` | 2222–2270 | hook is simple, APIs exist | low | B |
| 19 | `kernel/kallsyms.c` | 2271–2321 | `s_show` exists, direct | low | A |
| 20 | `kernel/reboot.c` | 2322–2351 | existing KSU reboot hook | low | skip |
| 21 | `kernel/sys.c` | 2352–2431 | `newuname`/`setpriority`/`getpriority`/`setresuid` exist | low-medium | A |
| 22 | `mm/memory.c` | 2432–2457 | `__access_remote_vm` exists; simple hook | low | B |
| 23 | `security/selinux/avc.c` | 2458–2500 | `avc_dump_query` exists; globals need KSU | medium | B |
| 24 | `security/selinux/hooks.c` | 2501–2871 | LSM signatures differ; KSU selinux-hide state missing | high | B |
| 25 | `security/selinux/selinuxfs.c` | 2872–3076 | `fake_status`/KSU hooks missing in 4.19/KSU | high | B |
| 26 | `security/selinux/ss/services.c` | 3077–3389 | 6.1 policy object model vs 4.19 global `selinux_state` | high | B |
| 27 | `security/selinux/ss/status.c` | 3390–3438 | path differs (`ss/status.c`), functions exist | medium | B |
| — | `fs/susfs.c` | 2560 lines | mostly portable; see §4.28 | high | A core |
| — | `include/linux/susfs.h` | 261 lines | portable | low | A core |
| — | `include/linux/susfs_def.h` | 172 lines | portable; include and TIF notes | low-medium | A core |

---

## 3. KernelSU 3.3.1 hook-layer gap (read this before any per-file work)

The 2.4.0 kernel patch is only half of the 2.4.0 release. The other half is `kernel_patches/KernelSU/10_enable_susfs_for_ksu.patch` (4411 lines, 32 files). That patch *changes KernelSU itself*; it is not in the current submodule. The following mismatches determine what can work without touching KernelSU.

### 3.1 Symbols that exist in current KernelSU 3.3.1

Verified by grepping `KernelSU/kernel/`:

| API | Location | Signature / note |
|-----|----------|------------------|
| `ksu_handle_execveat` | `feature/sucompat.c` | `int ksu_handle_execveat(int *fd, struct filename **filename_ptr, void *argv, void *envp, int *flags)` — always returns 0 in this version; cannot be used as a success/failure result |
| `ksu_handle_execveat_sucompat` | `feature/sucompat.c:331/349` | `int ksu_handle_execveat_sucompat(...)`; only handles `su` path substitution |
| `ksu_handle_faccessat` | `feature/sucompat.c:225` | `int ksu_handle_faccessat(int *dfd, const char __user **filename_user, int *mode, int *flags)` |
| `ksu_handle_stat` | `feature/sucompat.c:235` | `int ksu_handle_stat(int *dfd, const char __user **filename_user, int *flags)` — note **`const char __user **`**, not `struct filename **` |
| `ksu_handle_setresuid_cred` | `hook/setuid_hook.c:1` | static inline, LSM `task_fix_setuid` style: `(struct cred *new, const struct cred *old)`; no syscall wrapper |
| `ksu_cred` | `kernel/ksu.c` | global `struct cred *ksu_cred` |
| `setup_selinux` | `KernelSU/kernel/selinux/selinux.c:50` | `void setup_selinux(const char *domain, struct cred *cred)` |
| `is_ksu_domain()` | `selinux/selinux.c:209` | `bool is_ksu_domain(void)` — the 4.19 equivalent of 2.4.0's `susfs_is_current_ksu_domain()` |
| `ksu_selinux_hide_enabled` | `KernelSU/kernel/feature/selinux_hide.c:34` | `bool`, static in that file; **no exported `ksu_selinux_hide_running`** |
| `ksu_su_compat_enabled` | `feature/sucompat.c:10` | plain `bool`, not a static key |
| `ksu_input_hook`, `ksu_vfs_read_hook` | `feature/ksud_integration.c:26–27` | plain `bool`, not static keys |

### 3.2 Symbols that 2.4.0 requires but KernelSU 3.3.1 does not provide

These are hard dependencies of the 2.4.0 kernel-side code:

* `ksu_su_compat_enabled` as `struct static_key_true` — used by `fs/namei.c`, `fs/open.c`, `fs/stat.c` in 2.4.0. Current KSU has `bool`.
* `ksu_is_input_hook_enabled` / `ksu_is_init_rc_hook_enabled` as static keys — used by `drivers/input/input.c`, `fs/stat.c`.
* `ksu_handle_post_execveat_sucompat()` — post-exec fixup; no equivalent.
* `ksu_handle_setresuid(uid_t, uid_t, uid_t)` — 2.4.0 `kernel/sys.c` calls it after `__sys_setresuid`; current KSU only has the LSM-cred hook.
* `ksu_handle_vfs_fstat(int fd, loff_t *kstat_size_ptr)` — current KSU has no such symbol.
* `ksu_setup_preinit_file()` — no equivalent.
* `ksu_selinux_hide_running` — current KSU has `ksu_selinux_hide_enabled` only.
* `fake_status` / `initialize_fake_status()` — current `selinux_hide.c` has internal fake-status machinery but does not export these names.
* `susfs_ksu_sid` / `susfs_priv_app_sid` — 2.4.0 `security/selinux/avc.c` externs; not defined in current KSU.
* `susfs_is_current_ksu_domain()` — add a tiny wrapper in `fs/susfs.c` calling `is_ksu_domain()`; no need to modify KSU for this one.
* `TIF_PROC_NO_SU` (bit 34) — 2.4.0's `susfs_is_current_proc_no_su()` is used by exec/open/stat hooks; current KSU never sets this flag. Either drop those uses or add a KSU setter.

**Conclusion:** Phase A must avoid every hook that calls these symbols. Phase B can either (a) patch KernelSU to 2.4.0's hook API, or (b) simplify the kernel-side calls to the current bool APIs. Option (b) is usually less invasive but loses some 2.4.0 behavior (notably `TIF_PROC_NO_SU` and post-exec handling).

### 3.3 Local KernelSU `susfs_cmd.c` bridge is written for the 1.5.11 ABI

The working tree contains `KernelSU/kernel/feature/susfs_cmd.c` (included from `KernelSU/kernel/ksu.c:139`), a prctl bridge for the SUSFS userspace tool. It currently targets the **1.5.11** interface:

* It calls `susfs_add_sus_path(struct st_susfs_sus_path __user *)`, `susfs_add_sus_path_loop(...)`, `susfs_add_sus_kstat(...)`, `susfs_update_sus_kstat(...)`, `susfs_set_uname(...)`, `susfs_enable_log(...)`, `susfs_set_cmdline_or_bootconfig(...)`, `susfs_add_open_redirect(...)` as **returning `int` and taking a typed user pointer**. 2.4.0 changed every one of these to `void fn(void __user **user_info)`: the argument is a pointer to the user's struct pointer, and the error is written into the struct's `err` field by `copy_to_user`. A direct compile against 2.4.0's `susfs.c` will fail with incompatible pointer types; even a cast would mis-handle errors.
* It lacks 2.4.0's new command cases: `CMD_SUSFS_SET_SUS_PATH_UID_FILTER` (`0x55554`), `CMD_SUSFS_HIDE_SUS_MNTS_FOR_NON_SU_PROCS` (`0x55561`), `CMD_SUSFS_ADD_SUS_MAP` (`0x60020`), and `CMD_SUSFS_SET_PRIORITY_UID_FILTER` (`0x60030`). Its existing `CMD_SUSFS_ADD_SUS_MOUNT`/`ADD_TRY_UMOUNT`/`SUS_SU` cases are deprecated in 2.4.0 and can stay as no-ops or be removed.
* It still references 1.5.11-only functions that 2.4.0's `fs/susfs.c` does not define: `susfs_add_sus_mount()`, `susfs_add_try_umount()`, `susfs_set_log()`, `susfs_sus_su()`, `susfs_get_sus_su_working_mode()`, `susfs_set_i_state_on_external_dir()`. Remove those cases or map them to 2.4.0 equivalents (`SUS_MOUNT` is now controlled by `susfs_set_hide_sus_mnts_for_non_su_procs`; `ENABLE_LOG` by `susfs_enable_log`).
* `KernelSU/kernel/ksu.c:282–285` also references `susfs_is_boot_completed_triggered`, which is a 1.5.11 global and **is not defined by 2.4.0's `fs/susfs.c`**. Either keep a compatibility definition (`bool susfs_is_boot_completed_triggered __read_mostly;`) in the ported `susfs.c` or delete those two KSU lines together with the 1.5.11 namespace code. Do not leave the extern unresolved.
* `susfs_init()` is already called from `KernelSU/kernel/ksu.c:282–284` in this tree; keep that call. `susfs_start_sdcard_monitor_fn()` is not called; add it in KSU init only when `CONFIG_KSU_SUSFS_SUS_MOUNT` is enabled (2.4.0's KSU patch calls it around `P:KernelSU/10...:4130`).

**Action:** rewrite `susfs_cmd.c` for the 2.4.0 `void __user **` ABI before enabling any SUSFS feature beyond core. This is a KernelSU-side file, not in the 27-file kernel patch list, but it is on the critical path for Phase A.

---

## 4. Per-file analysis

### 4.1 `block/ioprio.c`

**2.4.0 expects (`P:L2–L48`):**
* Include `<linux/susfs_def.h>` and declare `extern bool susfs_is_priority_uid_filtered(uid_t uid);`.
* In `SYSCALL_DEFINE3(ioprio_set, ...)`, inside `case IOPRIO_WHO_USER:`, add:
  ```c
  if (susfs_is_priority_uid_filtered(current_uid().val))
      break;
  ```
  before `uid = make_kuid(current_user_ns(), who);`.
* In `SYSCALL_DEFINE2(ioprio_get, ...)`, same check before `uid = make_kuid(...)`.

**4.19 reality (verified):**
* `int ioprio_check_cap(int ioprio)` at `B:block/ioprio.c:64`.
* `SYSCALL_DEFINE3(ioprio_set, int, which, int, who, int, ioprio)` at `B:block/ioprio.c:93`.
* `SYSCALL_DEFINE2(ioprio_get, int, which, int, who)` at `B:block/ioprio.c:185`.
* The `PRIO_USER` branch is the same shape as 6.1: `uid = make_kuid(current_user_ns(), who); if (!uid_valid(uid)) break; if (!who) user = current_user(); else user = find_user(uid);`.
* `current_uid().val` is `u32`; `susfs_is_priority_uid_filtered` takes `uid_t` (`u32` on arm64) — exact match.

**Adaptation:**
* Apply both checks literally; they are safe.
* In `ioprio_set` the `break` exits the `switch`, not the function; `user` remains uninitialised but the `free_uid(user)` path after the switch is only inside the case and is skipped by `break`, so this is correct. If this is considered too subtle, restructure with a `goto out;` after the switch.
* Add `#include <linux/susfs_def.h>` only if `susfs_is_priority_uid_filtered` is declared there; in 2.4.0 it is actually declared in `include/linux/susfs.h`. Include `<linux/susfs.h>` instead, or add a local `extern`.

**Risk:** low. **Config:** `CONFIG_KSU_SUSFS_PRIORITY_UID_FILTER`; can be enabled in Phase A (small, isolated).

---

### 4.2 `drivers/input/input.c`

**2.4.0 expects (`P:L51–L78`):**
* Add `<linux/susfs_def.h>`.
* Declare `extern struct static_key_true ksu_is_input_hook_enabled;`.
* In `input_event_dispose()` (a 6.1 helper that does not exist in 4.19), add a static-key branch calling `ksu_handle_input_handle_event(dev, type, code, value)`.
* In `input_handle_event()`, guard the existing call with `static_branch_unlikely(&ksu_is_input_hook_enabled)`.

**4.19 reality:**
* `static void input_handle_event(struct input_dev *dev, unsigned int type, unsigned int code, int value)` at `B:drivers/input/input.c:385`.
* The branch already contains the KernelSU hook:
  ```c
  #ifdef CONFIG_KSU
  extern bool ksu_input_hook __read_mostly;
  extern int ksu_handle_input_handle_event(struct input_dev *dev,
                                           unsigned int type,
                                           unsigned int code,
                                           int value);
  ...
  if (unlikely(ksu_input_hook))
      ksu_handle_input_handle_event(dev, type, code, value);
  #endif
  ```
  (declarations at `B:drivers/input/input.c:381–383`, call around `:391`).
* `input_event_dispose()` is absent; the 6.1 refactor split event delivery into that helper. The 4.19 hook location is already the right one.

**Adaptation:**
* **Do not port this hunk.** Keep the existing `bool ksu_input_hook` call.
* If a later KernelSU update changes `ksu_input_hook` to a static key, update this block in the same commit; it is unrelated to SUSFS feature correctness.

**Risk:** low. **Config:** none. **Phase:** skip.

---

### 4.3 `fs/Makefile`

**2.4.0 expects (`P:L79–L91`):**
* Add `obj-$(CONFIG_KSU_SUSFS) += susfs.o` after the `obj-y` list.

**4.19 reality:**
* `B:fs/Makefile:10–16` has a different `obj-y` list: it contains `open.o ... nsfs.o`, but no `fs_context.o`, `fs_parser.o`, `fsopen.o`, or `remap_range.o` (those are 5.x additions). The 2.4.0 hunk context will not match.

**Adaptation:**
* Manually add the line after `B:fs/Makefile:16` (after `stack.o fs_struct.o statfs.o fs_pin.o nsfs.o`).
* `fs/susfs.c` includes `"mount.h"` and `"fuse/fuse_i.h"`; building it as part of `fs/` works because the internal include path is available.
* If `CONFIG_KSU_SUSFS` is defined in KernelSU's `kernel/Kconfig`, the symbol is visible in `fs/Makefile` in a normal built-in configuration. If SUSFS is built as a module (`=m`), `fs/susfs.c` uses non-exported symbols (`real_mount`, `mntget`, etc.) and cannot link; keep it `bool`, not `tristate`.

**Risk:** low. **Phase:** A.

---

### 4.4 `fs/exec.c`

**2.4.0 expects (`P:L92–L195`):**
* Include `<linux/susfs_def.h>`.
* Declare externs: `ksu_su_compat_enabled`, `ksu_handle_execveat`, `ksu_handle_post_execveat_sucompat`, `ksu_setup_preinit_file`.
* In `do_execveat_common()`:
  * track `bool is_su_session = false;`
  * after `ksu_handle_execveat(...)`, set `is_su_session = !ret;`
  * before the final `return PTR_ERR(bprm);`/success path, if `is_su_session`, call `ksu_setup_preinit_file();`
* In `__do_execve_file()` (6.1 is split into `bprm_execve` + `__do_execve_file`):
  * after a successful `bprm_execve`, if `is_su_session`, call `susfs_set_current_proc_umounted()`-style flags and `ksu_handle_post_execveat_sucompat(...)`.

**4.19 reality:**
* `static int __do_execve_file(int fd, struct filename *filename, struct user_arg_ptr argv, struct user_arg_ptr envp, int flags, struct file *file)` at `B:fs/exec.c:1755`.
* `static int do_execveat_common(int fd, struct filename *filename, struct user_arg_ptr argv, struct user_arg_ptr envp, int flags)` at `B:fs/exec.c:1947`.
* There is **no `bprm_execve()`** in 4.19 (`git grep bprm_execve B -- fs/exec.c` returns nothing). The success path is the tail of `__do_execve_file()` after `bprm->point_of_no_return` handling.
* Existing KernelSU hooks are already in `do_execveat_common()`; they call `ksu_handle_execveat()` and `ksu_handle_execveat_sucompat()` with current signatures.
* `ksu_handle_execveat()` returns 0 unconditionally in KSU 3.3.1, so `is_su_session = !ret` would always be true. This is a logic bug if copied literally.
* No `ksu_setup_preinit_file()` / `ksu_handle_post_execveat_sucompat()`.

**Adaptation:**
* **Phase A:** do not apply this file at all. The existing KSU hooks stay.
* **Phase B:** implement only if you also extend KernelSU. Suggested shape:
  * In `do_execveat_common()`, after the existing `ksu_handle_execveat_sucompat()` call, derive `is_su_session` from a KSU helper that returns a real status (patch KSU to return 1/0), not from `ksu_handle_execveat()`'s return value.
  * In `__do_execve_file()`, after `bprm->point_of_no_return` block and before `return retval`, add the post-exec hook. Guard with `#ifdef CONFIG_KSU_SUSFS`.
  * `susfs_set_current_proc_umounted()` / `susfs_set_current_proc_no_su()` are static inlines in `susfs_def.h` and can be called here once the TIF bits are free (they are: 4.19 arm64 TIF max is well below 33).
* Simplest alternative: skip the 2.4.0 exec hooks entirely; they are not required for SUS_PATH/SUS_KSTAT/SUS_MOUNT correctness on a first port. The KSU sucompat path already has its own exec hook.

**Risk:** high. **Config:** no dedicated SUSFS option; needed only for 2.4.0 sucompat integration. **Phase:** B / defer.

---

### 4.5 `fs/namei.c` — the largest and riskiest file

**2.4.0 expects (`P:L196–L637`).** It contains approximately 15 hunks:
* include/externs: `P:L200–L220`.
* `lookup_dcache()` fake-qstr substitution: `P:L222–L236`.
* `lookup_one_qstr_excl()` (6.1 only) fake-qstr substitution: `P:L237–L274`.
* `lookup_fast()`: `P:L275–L371` (RCU and non-RCU branches, `nd->state` checks, `nd->flags |= ND_FLAGS_LOOKUP_LAST`).
* `walk_component()`: `P:L372–L383`.
* `link_path_walk()`: `P:L384–L399`.
* `lookup_last()`: `P:L400–L409`.
* `lookup_open()`: `P:L410–L450`.
* `open_last_lookups()` (6.1 only): `P:L451–L460`.
* `do_tmpfile()`: `P:L461–L494`.
* `do_o_path()`: `P:L495–L505`.
* `path_openat()`: `P:L506–L590`.
* `vfs_readlink()`: `P:L591–L637`.

**4.19 reality (all line numbers from `B:fs/namei.c`):**
* `lookup_dcache()` `:1599` — same signature `(const struct qstr *name, struct dentry *base, unsigned int flags)`.
* `__lookup_hash()` `:1623` — this is the 4.19 equivalent of `lookup_one_qstr_excl()`; signature `(const struct qstr *name, struct dentry *base, unsigned int flags)`. There is **no `lookup_one_qstr_excl()`** in 4.19.
* `lookup_fast()` `:1649` — signature `static int lookup_fast(struct nameidata *nd, struct path *path, struct inode **inode, unsigned *seqp)`. 6.1 has the same shape but different internals/`nd->state` use.
* `__lookup_slow()` `:1737` — signature `(const struct qstr *name, struct dentry *dir, unsigned int flags)`.
* `walk_component()` `:1884` — `static int walk_component(struct nameidata *nd, int flags)`.
* `link_path_walk()` `:2163` — `static int link_path_walk(const char *name, struct nameidata *nd)`.
* `lookup_last()` `:2362` — `static inline int lookup_last(struct nameidata *nd)`.
* `lookup_open()` `:3245` — `static int lookup_open(struct nameidata *nd, struct path *path, struct file *file, const struct open_flags *op, bool got_write)`.
* `do_last()` `:3375` — 4.19's equivalent of 6.1's `open_last_lookups()`; signature `static int do_last(struct nameidata *nd, struct file *file, const struct open_flags *op)`.
* `do_tmpfile()` `:3594` — `static int do_tmpfile(struct nameidata *nd, unsigned flags, const struct open_flags *op, struct file *file)` (same as 6.1).
* `do_o_path()` `:3626` — `static struct file *do_o_path(struct nameidata *nd, struct open_flags *op, int flags)` (same as 6.1).
* `path_openat()` `:3638` — `static struct file *path_openat(struct nameidata *nd, const struct open_flags *op, unsigned flags)`.
* `vfs_readlink()` `:4882` — `int vfs_readlink(struct dentry *dentry, char __user *buffer, int buflen)` (same as 6.1).
* `struct nameidata` `:503` — has `flags`, `last_type`, `depth`, `root`, `path`, `inode`, `seq`, `m_seq`, but **no `state` field** in B. R already added `unsigned int state;` at `R:fs/namei.c:522` (inside `#ifdef CONFIG_KSU_SUSFS_SUS_PATH`). Use that.
* 4.19 `lookup_one_len()` / `lookup_one_len2()` exist; `lookup_one_qstr_excl()` does not.

**Adaptation:**
1. **Add `state` to `struct nameidata` and initialize it.** Copy R's change (`R:fs/namei.c:522`, `set_nameidata()` around `:541–552`). Without this, `ND_STATE_LOOKUP_LAST`/`ND_STATE_OPEN_LAST` have nowhere to live. Alternative: reuse two high bits in `nd->flags`; not recommended because `nd->flags` bits are consumed by VFS (`LOOKUP_*`, `ND_FLAGS_LOOKUP_LAST` already uses 0x02000000, which is safe in 4.19).
2. **`lookup_dcache`:** apply the fake-qstr check directly; no API change.
3. **`__lookup_hash`:** replace the 6.1 `lookup_one_qstr_excl()` hunk with R's `__lookup_hash` code (`R:fs/namei.c:1641–1700` in the 1.5.11 diff). It already implements the same `found_sus_path`/`susfs_fake_qstr_name` retry logic using 4.19 `lookup_dcache`/`d_alloc`.
4. **`lookup_fast`:** rewrite from R's 1.5.11 code, not from 6.1. Key points:
   * `is_nd_state_lookup_last_and_open_last = (nd->state & ND_STATE_LOOKUP_LAST || nd->state & ND_STATE_OPEN_LAST);`
   * RCU branch: `__d_lookup_rcu(parent, &susfs_fake_qstr_name, &seq)` exists in 4.19.
   * non-RCU branch: `__d_lookup(parent, &susfs_fake_qstr_name)` exists.
   * 4.19's `lookup_fast` has `status = d_revalidate(...)`; preserve it.
5. **`__lookup_slow`:** use `ND_FLAGS_LOOKUP_LAST` (define `0x2000000`; 4.19 `nd->flags` does not use it), `d_alloc_parallel` exists, `DECLARE_WAIT_QUEUE_HEAD_ONSTACK` exists. R already has this exact adaptation.
6. **`walk_component`:** insert `if (nd->state & ND_STATE_LOOKUP_LAST) nd->flags |= ND_FLAGS_LOOKUP_LAST;` at the correct branch; R has it.
7. **`link_path_walk`:** add `if (dentry->d_inode && susfs_is_inode_sus_path(dentry->d_inode)) return -ENOENT;` after `may_lookup()`. The 6.1 patch also has a `nd->state & ND_STATE_LAST_SDCARD_SUS_PATH` check; 2.4.0's `susfs_def.h` does not define that state, so either add it or omit it and rely on `link_path_walk`'s inode check. R's 1.5.11 uses `ND_STATE_LAST_SDCARD_SUS_PATH 128`; if you want full 2.4.0 behavior, add it to `susfs_def.h`.
8. **`lookup_last`:** set `nd->state |= ND_STATE_LOOKUP_LAST;`.
9. **`lookup_open`:** apply R's 1.5.11 adaptation; 4.19 signature already matches. Set `nd->state |= ND_STATE_OPEN_LAST` in `do_last()` (4.19 line 3375), not in `open_last_lookups()` (absent).
10. **`do_tmpfile` / `do_o_path`:** 4.19 signatures match the 6.1 patch; apply the open-redirect block. Critical differences:
    * 4.19 `set_nameidata()` takes **3 arguments** `(struct nameidata *, int dfd, struct filename *)`; 6.1 takes 4 (with `struct filename *` and `NULL`). Drop the last argument.
    * 4.19 `nd->flags & LOOKUP_ROOT` replaces 6.1 `nd->state & ND_ROOT_PRESET`.
    * 4.19 `do_o_path` calls `path_openat` with `LOOKUP_RCU`; the 6.1 block uses `nd->state & ND_STATE_OPEN_LAST`; preserve 4.19 semantics.
11. **`path_openat`:** the 6.1 patch's redirect block is written around `open_last_lookups`; in 4.19 the simplest working pattern is R's `do_filp_open()` hook (`R:fs/namei.c` diff around `do_filp_open`), which closes the original file and re-opens `susfs_get_redirected_path(inode->i_ino)`. Reuse it and adapt the helper name: 2.4.0 provides `susfs_open_redirect_spoof_do_sys_openat(struct inode *inode)` returning `struct filename *`; 1.5.11's `susfs_get_redirected_path(unsigned long ino)` does the same by inode number. Either helper can be used.
12. **`vfs_readlink`:** 4.19 line 4882. Add before `readlink_copy`:
    ```c
    #ifdef CONFIG_KSU_SUSFS_OPEN_REDIRECT
    if (SUSFS_IS_INODE_OPEN_REDIRECT(dentry->d_inode)) {
        int err = susfs_open_redirect_spoof_vfs_readlink(dentry->d_inode, buffer, buflen);
        if (!err)
            return 0;
    }
    #endif
    ```
    `dentry->d_inode` is valid; `vfs_readlink` already handles `get_link`/`readlink_copy`. The 2.4.0 patch checks `SUSFS_IS_INODE_OPEN_REDIRECT` only when `inode->i_op->get_link` exists; in 4.19 put the check at the top of `vfs_readlink`, before the `if (!inode->i_op->get_link)` shortcut, so symlinks with no `get_link` still get spoofed.

**Risk:** high. **Config:** `CONFIG_KSU_SUSFS_SUS_PATH`, `CONFIG_KSU_SUSFS_OPEN_REDIRECT`. **Phase:** B (SUS_PATH first, OPEN_REDIRECT after).

**Do not copy the 6.1 patch literally.** R's `fs/namei.c` is the single most valuable reference in the whole task. It compiles and boots on this kernel.

---

### 4.6 `fs/namespace.c`

**2.4.0 expects (`P:L638–L972`):**
* Include `<linux/susfs_def.h>`; extern `susfs_is_current_ksu_domain`, `susfs_is_sdcard_android_data_not_decrypted`.
* Define `CL_COPY_MNT_NS BIT(25)` local flag.
* `mnt_free_id()`: skip `ida_free` when `mnt->mnt.mnt_flags & VFSMOUNT_MNT_FLAGS_KSU_UNSHARED_MNT`.
* `mnt_alloc_group_id()`: for `susfs_is_current_ksu_domain()`, allocate from `mnt_group_ida` with `DEFAULT_KSU_MNT_GROUP_ID`; otherwise use the normal minimum 1. No separate IDA is used in 2.4.0 (unlike the 1.5.11 branch).
* Add `susfs_alloc_unshare_ksu_vfsmnt()` and `susfs_alloc_non_unshare_ksu_vfsmnt()` — copies of `alloc_vfsmnt()` with explicit/fake ID allocation.
* Hook `__lookup_mnt()`: if `susfs_is_current_proc_umounted()`, return only a non-sus mount whose id < `DEFAULT_KSU_MNT_ID`.
* Hook `vfs_create_mount()`: for KSU domain, allocate fake mount.
* Hook `clone_mnt()`: for KSU/`CL_COPY_MNT_NS`, allocate fake/reuse mount id; propagate `mnt.mnt_flags` and set `VFSMOUNT_MNT_FLAGS_KSU_UNSHARED_MNT` for the unshared case. 2.4.0 does not use `susfs_mnt_id_backup`; only R's 1.5.11 code does.
* `copy_mnt_ns()`: add `copy_flags |= CL_COPY_MNT_NS`.
* Add `susfs_get_non_sus_mnt_id_from_mnt()` / `susfs_get_non_sus_vfsmnt_from_vfsmnt()` at end of file.

**4.19 reality:**
* `mnt_alloc_id()` `B:fs/namespace.c:99`; `mnt_free_id()` `:109`; `mnt_alloc_group_id()` `:117`; `alloc_vfsmnt()` `:177`; `__lookup_mnt()` `:615`; `vfs_kern_mount()` `:948`; `clone_mnt()` `:1004`; `copy_mnt_ns()` `:2928`.
* **No `vfs_create_mount()`** in 4.19 (`git grep vfs_create_mount B -- fs` returns nothing). The equivalent is `struct vfsmount *vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data)`.
* `struct mount` in 4.19 has `mnt_id`, `mnt_group_id`, `mnt_devname`, `mnt_parent`, `mnt_mountpoint`, `mnt.mnt_flags`, `mnt.mnt_root`, `mnt.mnt_sb`, `mnt.data`, `mnt_umount` fs_pin. It does **not** have `mnt_userns` or `mnt_stuck_children`.
* `alloc_vfsmnt()` `B:fs/namespace.c:177` initializes `mnt->mnt.data = NULL;` and `init_fs_pin(&mnt->mnt_umount, drop_mountpoint);`; uses `kstrdup_const(name, GFP_KERNEL)` for `mnt_devname`; uses `ida_alloc` for id.
* `clone_mnt()` 4.19 uses `sb->s_op->clone_mnt_data` and `mnt->mnt.mnt_flags = old->mnt.mnt_flags;`.
* `copy_mnt_ns()` 4.19 sets `copy_flags = CL_COPY_UNBINDABLE | CL_EXPIRE;` and optionally `CL_SHARED_TO_SLAVE | CL_UNPRIVILEGED`.
* `struct vfsmount` in B has `int mnt_flags`; R adds `u64 susfs_mnt_id_backup` in KABI slot 4.

**Adaptation:**
* Add the local `CL_COPY_MNT_NS` definition. `fs/pnode.h` in 4.19 defines `CL_COPY_UNBINDABLE 0x04`, `CL_SHARED_TO_SLAVE 0x20`, `CL_COPY_MNT_NS_FILE 0x80`; `BIT(25)` is free (check `fs/pnode.h` before finalizing).
* `mnt_free_id`: the 2.4.0 check using `mnt->mnt.mnt_flags` works in 4.19, but note `mnt_flags` is `int` and `0x80000000` is a signed constant. Write `(mnt->mnt.mnt_flags & (int)VFSMOUNT_MNT_FLAGS_KSU_UNSHARED_MNT)` or change the macro to `0x40000000` if no other user. Verify `MNT_*` flags do not use bit 31.
* `susfs_alloc_*`: copy 4.19 `alloc_vfsmnt()` body and **do not copy** `mnt->mnt.mnt_userns = &init_user_ns;` or `mnt->mnt_stuck_children` (both absent). Keep:
  ```c
  mnt->mnt.data = NULL;
  init_fs_pin(&mnt->mnt_umount, drop_mountpoint);
  ```
  `ida_alloc_min(&mnt_id_ida, DEFAULT_KSU_MNT_ID, GFP_KERNEL)` exists in 4.19 (`include/linux/idr.h`).
* `__lookup_mnt`: direct adaptation; `p->mnt_id` exists. Note the 4.19 function is under `CONFIG_SMP`? No, it is normal. `hlist_for_each_entry_rcu` works.
* `vfs_create_mount` → `vfs_kern_mount`: insert the hook after `if (!type) return ERR_PTR(-ENODEV);` and before `mnt = alloc_vfsmnt(name);`. Use `name` as the devname; 4.19 has no `fc->source`. Add `if (flags & SB_KERNMOUNT) goto orig_flow;` to avoid spoofing internal mounts. If `type->alloc_mnt_data` is set, the custom alloc must call it or leave `mnt->mnt.data = NULL`; check every filesystem used on picasso (f2fs, ext4, fuse, tmpfs) — tmpfs uses `alloc_mnt_data`? `shmem` does not in 4.19. Safer: after allocating, if `type->alloc_mnt_data` exists, call it and handle failure.
* `clone_mnt`: 4.19 has no `mnt_user_ns`/`mnt_userns` lines; drop them. Keep the `sb->s_op->clone_mnt_data` block. Add `if (unlikely(is_mnt_ksu_unshared)) mnt->mnt.mnt_flags |= ...;` after the existing `mnt->mnt.mnt_flags = old->mnt.mnt_flags;` and before `atomic_inc(&sb->s_active);`.
* `copy_mnt_ns`: 4.19 has `CL_SHARED_TO_SLAVE | CL_UNPRIVILEGED`; add `CL_COPY_MNT_NS` after them.
* Helpers at end: 4.19 `struct mount` has `mnt.mnt_root`, `mnt_parent`, `mnt_id`; functions compile. `lock_mount_hash()`/`unlock_mount_hash()` exist in 4.19? Check `fs/namespace.c`; if they are `static inline` in `fs/mount.h`, include it (already included). Yes.
* Important semantic difference: 4.19 `vfs_kern_mount()` is used by `kern_mount()` internally with `SB_KERNMOUNT`; without the guard, KSU-domain kernel mounts can get fake IDs. Add the guard.

**Risk:** high. **Config:** `CONFIG_KSU_SUSFS_SUS_MOUNT`. **Phase:** B last among the mount-related features.

---

### 4.7 `fs/notify/fdinfo.c`

**2.4.0 expects (`P:L973–L1093`):**
* Include `<linux/susfs_def.h>`.
* Change `show_fdinfo()` callback to pass `struct file *` to `show`, because `inotify_fdinfo` needs `file->f_path.mnt`.
* Use `fsnotify_group_lock(group)`.
* `inotify_fdinfo()` gains `struct file *file` and gains SUS_KSTAT/SUS_MOUNT branches that replace `ino`/`sdev` with spoofed values and use `d_path`/`kern_path` for the real inode.

**4.19 reality:**
* `static void show_fdinfo(struct seq_file *m, struct file *f, void (*show)(struct seq_file *m, struct fsnotify_mark *mark))` at `B:fs/notify/fdinfo.c:24`.
* `static void inotify_fdinfo(struct seq_file *m, struct fsnotify_mark *mark)` at `:75`.
* There is **no `fsnotify_group_lock()`** in 4.19; the group has `mark_mutex`. `fsnotify_group_lock` is a 5.x/6.x helper. Use `mutex_lock(&group->mark_mutex)` / `mutex_unlock`.
* `fsnotify_conn_inode(mark->connector)` exists in 4.19; `inotify_mark_user_mask(mark)` exists.
* `inotify_mark->wd` exists.

**Adaptation:**
* Change callback signature to `void (*show)(struct seq_file *, struct fsnotify_mark *, struct file *)` and pass `f`.
* Replace `fsnotify_group_lock` with `mutex_lock(&group->mark_mutex)`.
* Apply the SUS_KSTAT branch: `susfs_is_current_app_uid()`, `susfs_is_inode_sus_kstat(inode,&is_fuse)`, `susfs_sus_kstat_spoof_inotify_fdinfo(&ino,&dev)`, print `ino:%lx sdev:%x`. 4.19 `seq_printf` format `%lx` for `unsigned long`; `dev_t` is `u32`, `%x` expects unsigned int; cast.
* Apply the SUS_MOUNT branch: `real_mount(file->f_path.mnt)`, check `mnt_id >= DEFAULT_KSU_MNT_ID`, `d_path`, `kern_path`, `susfs_get_non_sus_mnt_id_from_mnt(mnt)`, print `ino`/`sdev` from the resolved dentry.
* The 6.1 patch uses `kmalloc(PAGE_SIZE)` and `if (!dpath)`; in 4.19 `d_path()` returns `char *` or ERR_PTR; check `IS_ERR(dpath)` as well as NULL.
* `show_mark_fhandle(m, inode)` is declared before `inotify_fdinfo` in 4.19? It is at line 71 in 4.19; yes, forward declaration exists above.

**Risk:** medium. **Config:** `CONFIG_KSU_SUSFS_SUS_KSTAT` / `SUS_MOUNT`. **Phase:** B.

---

### 4.8 `fs/open.c`

**2.4.0 expects (`P:L1094–L1154`):**
* Include `<linux/susfs_def.h>`.
* Extern `ksu_su_compat_enabled`, `ksu_handle_faccessat`.
* Rewrite `do_faccessat()` to use `getname_flags()` + `filename_lookup()` (6.1 style) because it needs to intercept the filename before lookup.
* Add `susfs_is_current_proc_no_su()` fast path and only call `ksu_handle_faccessat()` when UID is allowed.

**4.19 reality:**
* `long do_faccessat(int dfd, const char __user *filename, int mode)` at `B:fs/open.c:360`.
* It already has a KernelSU hook:
  ```c
  #ifdef CONFIG_KSU
  extern int ksu_handle_faccessat(int *dfd, const char __user **filename_user, int *mode, int *flags);
  ...
  ksu_handle_faccessat(&dfd, &filename, &mode, NULL);
  #endif
  ```
  around `B:fs/open.c:366–371`.
* No `access_override_creds()` helper in 4.19 (6.1 refactor); no `getname_flags()` conversion needed for KSU hook.

**Adaptation:**
* **Do not port the 6.1 rewrite.** Leave the existing hook in place.
* If 2.4.0's `TIF_PROC_NO_SU` fast path is desired, add `if (susfs_is_current_proc_no_su()) goto orig_flow;` around the hook; but current KSU never sets `TIF_PROC_NO_SU`, so it is dead code until the KSU side is patched.
* The 2.4.0 hunk's `ksu_handle_faccessat(&dfd, &filename, &mode, NULL)` matches the current signature.

**Risk:** low. **Config:** none. **Phase:** skip.

---

### 4.9 `fs/proc/base.c`

**2.4.0 expects (`P:L1155–L1210`):**
* Include `<linux/susfs_def.h>` for `SUS_MAP`/`OPEN_REDIRECT`.
* In `do_proc_readlink()`, before `d_path()`, if `SUSFS_IS_INODE_OPEN_REDIRECT(path->dentry->d_inode)`, call `susfs_open_redirect_spoof_do_proc_readlink()` and copy the fake path to user.
* In `proc_map_files_readdir()`, inside `for_each_vma(vmi, vma)`, skip `SUSFS_IS_INODE_SUS_MAP(file_inode(vma->vm_file))`.

**4.19 reality:**
* `static int do_proc_readlink(const struct path *path, char __user *buffer, int buflen)` at `B:fs/proc/base.c:1640`.
* It uses `char *tmp = (char *)__get_free_page(GFP_KERNEL);` and `d_path(path, tmp, PAGE_SIZE)`; frees with `free_page((unsigned long)tmp)`. The 2.4.0 patch's snippet is compatible if you keep the page allocation and `PAGE_SIZE` limit.
* **Adaptation:** apply the `do_proc_readlink` block; 4.19's `tmp` is one page (4096), so cap the fake path at `PAGE_SIZE - 1` and copy with `strlen(tmp)`; keep the `free_page` on every path. If `susfs_open_redirect_spoof_do_proc_readlink()` returns `-ENAMETOOLONG`/`-ENOENT`, fall through to the original `d_path`.
* `proc_map_files_readdir(struct file *file, struct dir_context *ctx)` at `:2168`. 4.19 uses `struct mm_struct *mm = file->private_data; struct vm_area_struct *vma;` and **two passes** over `mm->mmap`/`vma->vm_next`; there is no `for_each_vma()` / maple tree.
* `SUSFS_IS_INODE_SUS_MAP` uses `susfs_is_current_proc_umounted_app()`, which is valid.

**Adaptation:**
* Apply the `do_proc_readlink` block as described above; 4.19's `buflen` is `int` and the temporary buffer is one page, so do not use `PATH_MAX` as the allocation size. Ensure `free_page()` runs on all paths.
* For `proc_map_files_readdir`, add the SUS_MAP skip in **both** loops:
  * counting pass: before `if (++pos <= ctx->pos) continue;`
  * emitting pass: before `if (++pos <= ctx->pos) continue;` (or before `if (pos < ctx->pos)` depending on exact loop).
  Use `if (!vma->vm_file) continue;` first; then `if (SUSFS_IS_INODE_SUS_MAP(file_inode(vma->vm_file))) continue;`.
* `OPEN_REDIRECT` block uses `path->dentry->d_inode`; in 4.19 `dentry->d_inode` is valid.

**Risk:** medium. **Config:** `CONFIG_KSU_SUSFS_OPEN_REDIRECT`, `CONFIG_KSU_SUSFS_SUS_MAP`. **Phase:** B.

---

### 4.10 `fs/proc/bootconfig.c`

**2.4.0 expects (`P:L1211–L1236`):**
* Add extern `susfs_is_fake_cmdline_or_bootconfig_buffer_set` and `susfs_spoof_cmdline_or_bootconfig()`.
* In `boot_config_proc_show()`, if the static key is set and `saved_boot_config` is non-NULL, output the fake data.

**4.19 reality:**
* **`fs/proc/bootconfig.c` does not exist in 4.19.** Bootconfig was added in Linux 5.5; 4.19 has only `fs/proc/cmdline.c`.
* `B:fs/proc/cmdline.c:8` has `static int cmdline_proc_show(struct seq_file *m, void *v)` and prints `saved_command_line`.

**Adaptation:**
* Do not create `fs/proc/bootconfig.c`; it would add dead code.
* In `fs/proc/cmdline.c`, include `<linux/susfs_def.h>` and add:
  ```c
  #ifdef CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG
  if (static_branch_likely(&susfs_is_fake_cmdline_or_bootconfig_buffer_set)) {
      susfs_spoof_cmdline_or_bootconfig(m);
      return 0;
  }
  #endif
  seq_puts(m, saved_command_line);
  ```
  This is the non-GKI path; it matches the 2.4.0 Kconfig help ("`/proc/bootconfig` (gki) or `/proc/cmdline` (non-gki)"). `susfs_spoof_cmdline_or_bootconfig()` writes to a `seq_file`, so it works for cmdline too.
* Keep the config name for compatibility with userspace; document that on picasso it only affects `/proc/cmdline`.

**Risk:** low. **Config:** `CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG`. **Phase:** A (easy win).

---

### 4.11 `fs/proc/fd.c`

**2.4.0 expects (`P:L1237–L1335`):**
* Include `<linux/susfs_def.h>`.
* Extern `susfs_get_non_sus_mnt_id_from_mnt`, `susfs_is_inode_sus_kstat`, `susfs_sus_kstat_spoof_proc_fd_seq_show`.
* In `seq_show()`:
  * SUS_KSTAT: if app UID and inode is sus-kstat, print `pos/flags/mnt_id/ino` with spoofed `mnt_id` and `ino`, then `goto bypass_orig_flow`.
  * SUS_MOUNT: if `susfs_is_current_proc_umounted()` and `mnt_id >= DEFAULT_KSU_MNT_ID`, `d_path`, `kern_path`, print non-sus `mnt_id` and real inode, then jump.
* The normal print in 6.1 includes `ino:`; 4.19 normal print does **not**.

**4.19 reality:**
* `static int seq_show(struct seq_file *m, void *v)` at `B:fs/proc/fd.c:20`.
* Normal output:
  ```c
  seq_printf(m, "pos:\t%lli\nflags:\t0%o\nmnt_id:\t%i\n",
             (long long)file->f_pos, f_flags,
             real_mount(file->f_path.mnt)->mnt_id);
  ```
  no `ino`. Add `ino:\t%lu\n` for spoofed branches and for the normal branch if you want consistent output (the 2.4.0 behavior prints `ino` in both).
* `file_inode(file)` is available.
* `show_fd_locks()` and `file->f_op->show_fdinfo` follow; jumps must land before `show_fd_locks` to preserve them, or skip them deliberately. 2.4.0's `bypass_orig_flow` label is before `show_fd_locks` (the block prints then falls through), so keep the label just before `show_fd_locks`.

**Adaptation:**
* Add both blocks. Use `file_inode(file)` not `file->f_inode` (both exist; `file_inode` is cleaner).
* For SUS_MOUNT, `d_path(&file->f_path, pathname, PAGE_SIZE)` returns `char *` or ERR_PTR; check `IS_ERR`.
* `kern_path(dpath, 0, &path)` uses the 4.19 signature; `path_put` on all success paths.
* Add `ino` to the normal print if 2.4.0 userspace expects it; otherwise leave normal output unchanged to minimize ABI changes.

**Risk:** medium. **Config:** `CONFIG_KSU_SUSFS_SUS_MOUNT` / `SUS_KSTAT`. **Phase:** B.

---

### 4.12 `fs/proc/task_mmu.c`

**2.4.0 expects (`P:L1336–L1471`):**
* Include `<linux/susfs_def.h>` for `SUS_KSTAT`/`SUS_MAP`/`OPEN_REDIRECT`.
* `show_map_vma()`: before assigning `dev`/`ino`, if inode is sus-kstat, spoof `dev`/`ino`; if inode is sus-map, skip map printing; if open-redirect, print the redirected name/dev/ino using SRCU.
* `show_smap()`: `if (SUSFS_IS_INODE_SUS_MAP(file_inode(vma->vm_file))) return 0;`.
* `show_smaps_rollup()`: skip sus-map VMAs, using `mas_find()` and `VMA_PAD_START`.
* `pagemap_read()`: skip `vma_lookup()` results that are sus-map.

**4.19 reality:**
* `show_map_vma()` `B:fs/proc/task_mmu.c:350`.
* `show_smap()` `:849`.
* `show_smaps_rollup()` `:883` — 4.19 uses `for (vma = mm->mmap; vma; vma = vma->vm_next)` and `smap_gather_stats(vma, &mss)`; no `mas_find()`, no `VMA_PAD_START`.
* `pagemap_read()` `:1558` — uses `down_read_killable(&mm->mmap_sem)`, `walk_page_range(start_vaddr, end_vaddr, &walk)` (5-arg 4.19 signature), and `find_vma(mm, start_vaddr)` (no `vma_lookup`).
* `VMA_PAD_START()` is a 5.8+ helper; absent in 4.19.
* `mmap_read_lock_killable()` is a 5.8+ helper; 4.19 uses `down_read_killable(&mm->mmap_sem)`.

**Adaptation:**
* `show_map_vma`: add the three feature blocks before `dev = inode->i_sb->s_dev;`. For OPEN_REDIRECT, the 6.1 patch uses `show_vma_header_prefix(m, start, VMA_PAD_START(vma), ...)`. In 4.19 replace `VMA_PAD_START(vma)` with `vma->vm_end`; `vma->vm_end` is the correct end. Also `vma->vm_pgoff` and `dev/ino` are available.
* `show_smap`: add `if (vma->vm_file && SUSFS_IS_INODE_SUS_MAP(file_inode(vma->vm_file))) return 0;` after `if (!vma->vm_file) return 0;` (or before).
* `show_smaps_rollup`: rewrite the loop:
  ```c
  for (vma = mm->mmap; vma; vma = vma->vm_next) {
      if (vma->vm_file && SUSFS_IS_INODE_SUS_MAP(file_inode(vma->vm_file)))
          continue;
      ...
      smap_gather_stats(vma, &mss);
      last_vma_end = vma->vm_end;
  }
  ```
  Keep the 4.19 `last_vma_end`/`last_vma_start` logic; do not port `mas_find`.
* `pagemap_read`: replace `vma = vma_lookup(mm, start_vaddr);` with `vma = find_vma(mm, start_vaddr);` and check `vma->vm_start <= start_vaddr` before using it. Replace `mmap_read_lock_killable` with `down_read_killable(&mm->mmap_sem)`; the rest of the 4.19 function is unchanged. Add a `bypass` label before the original `get_user_pages_remote` call if you choose to skip the VMA.
* `SUSFS_IS_INODE_SUS_MAP` macro uses `inode->i_mapping`; for `vma->vm_file` always non-NULL in these branches after the `if (!vma->vm_file)` check.

**Risk:** medium. **Config:** `CONFIG_KSU_SUSFS_SUS_MAP` / `SUS_KSTAT` / `OPEN_REDIRECT`. **Phase:** B.

---

### 4.13 `fs/proc_namespace.c`

**2.4.0 expects (`P:L1472–L1690`):**
* Include `<linux/susfs_def.h>`.
* Add extern `struct static_key_false susfs_is_hide_sus_mnts_for_non_su_procs_enabled;` and `susfs_is_current_ksu_domain()`.
* Duplicate `show_vfsmnt()`, `show_mountinfo()`, `show_vfsstat()` as `susfs_show_*()`; each begins with `if (r->mnt_id >= DEFAULT_KSU_MNT_ID) return 0;`.
* In `mounts_open()` / `mountinfo_open()` / `mountstats_open()`, select the `susfs_show_*()` callback when the static key is set and the current process is **not** KSU domain.

**4.19 reality:**
* `show_vfsmnt()` `B:fs/proc_namespace.c:97`; `show_mountinfo()` `:133`; `show_vfsstat()` `:197`; `mounts_open_common()` `:242`.
* 4.19 `show_vfsmnt()` includes:
  ```c
  if (sb->s_op->show_devname) { ... }
  else seq_puts(m, mnt->mnt_devname ? ... : "none");
  seq_putc(m, ' ');
  ...
  if (sb->s_op->show_options2)
      err = sb->s_op->show_options2(m, mnt_path.dentry, mnt);
  else if (sb->s_op->show_options)
      err = show_sb_opts(m, sb);
  ```
  The 6.1 patch may lack `show_options2` (Xiaomi-specific in 4.19); preserve the 4.19 branch.
* `struct proc_mounts` has `struct mnt_namespace *ns`, `struct path root`, `struct seq_file m`; `r->mnt_id` is available via `real_mount(mnt)`.
* `seq_path_root()` exists in 4.19.

**Adaptation:**
* Copy R's 1.5.11 `fs/proc_namespace.c` changes, which already implement SUS_MOUNT hiding for 4.19. Then adapt the condition to 2.4.0's `susfs_is_hide_sus_mnts_for_non_su_procs_enabled` static key (R's variable name may differ; align with `fs/susfs.c`).
* In `susfs_show_vfsmnt`, keep the `show_options2` branch; otherwise the output of `/proc/self/mounts` will differ from stock for filesystems that implement it.
* `mounts_open_common()` in 4.19 takes a `show` callback; add the SUS_MOUNT selection in `mounts_open()`, `mountinfo_open()` and `mountstats_open()` rather than inside `mounts_open_common()`.

**Risk:** medium-low. **Config:** `CONFIG_KSU_SUSFS_SUS_MOUNT`. **Phase:** B.

---

### 4.14 `fs/read_write.c`

**2.4.0 expects (`P:L1691–L1713`):**
* In `ksys_read()`, call `ksu_handle_vfs_read(&file, &buf, &count, &pos)` behind `static_branch_likely(&ksu_is_init_rc_hook_enabled)`.

**4.19 reality:**
* `ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)` at `B:fs/read_write.c:583`.
* The file already has a KernelSU vfs-read hook at `B:fs/read_write.c:438–448`:
  ```c
  extern bool ksu_vfs_read_hook __read_mostly;
  extern int ksu_handle_vfs_read(struct file **file_ptr, char __user **buf_ptr,
                                 size_t *count_ptr, loff_t **pos);
  ...
  if (unlikely(ksu_vfs_read_hook))
      ksu_handle_vfs_read(&file, &buf, &count, &pos);
  ```
  It is in `vfs_read()` rather than `ksys_read()` and uses a bool, not a static key.

**Adaptation:**
* Do not port. The existing hook covers the same path. If 2.4.0's `ksu_is_init_rc_hook_enabled` behavior is needed later, patch KernelSU to a static key and move/duplicate the hook; not required for SUSFS.

**Risk:** low. **Config:** none. **Phase:** skip.

---

### 4.15 `fs/readdir.c`

**2.4.0 expects (`P:L1714–L1981`):**
* Include `<linux/susfs_def.h>`; extern `susfs_is_inode_sus_path`.
* Add `struct super_block *sb` to `readdir_callback`, `getdents_callback`, `getdents_callback64`.
* In `fillonedir`, `filldir`, `filldir64`, call `ilookup(buf->sb, ino)` and skip if `susfs_is_inode_sus_path(inode)`.
* Set `buf.sb = f.file->f_inode->i_sb` in `old_readdir`, `getdents`, `getdents64` syscalls.
* 6.1 callbacks have `prev_reclen`; 4.19 callbacks do not.

**4.19 reality:**
* `fillonedir()` `B:fs/readdir.c:125` — returns `int`, signature `(struct dir_context *ctx, const char *name, int namlen, loff_t offset, u64 ino, unsigned int d_type)`, uses `struct linux_dirent __user *dirent`, `buf->result`, `buf->error`.
* `filldir()` `:203` — returns `int`; `struct getdents_callback` has `previous` pointer, `count`, `error`; **no `prev_reclen`**.
* `filldir64()` `:293` — same, no `prev_reclen`.
* `ksys_getdents64()` `:338`.
* `old_readdir()` uses `struct readdir_callback` with `dirent`, `result`; no `sb`.
* 4.19 uses `access_ok(VERIFY_WRITE, ...)` and `__put_user`; `user_write_access_begin()` is a 5.x helper and is absent.

**Adaptation:**
* Add the `sb` fields and set them in the three syscall entry points (`old_readdir`, `getdents`, `getdents64`). Also `compat_filldir`/`compat_fillonedir` if you want 32-bit compat coverage; 2.4.0's 6.1 patch covers only the native callbacks because 6.1 compat uses the same `filldir64`? Actually 6.1 has separate `compat_filldir`; the 2.4.0 patch does not add SUS_PATH checks to compat callbacks. For picasso arm64 with 32-bit userspace, add them to `compat_filldir`/`compat_fillonedir` as well or document the gap.
* In `fillonedir`, add after the `EOVERFLOW` check:
  ```c
  inode = ilookup(buf->sb, ino);
  if (inode) {
      if (susfs_is_inode_sus_path(inode)) { iput(inode); return 0; }
      iput(inode);
  }
  ```
  `ino` is `u64`; `ilookup` takes `unsigned long`; cast `(unsigned long)ino`.
* In `filldir`/`filldir64`, same before `buf->error`/`prev_reclen` handling. 4.19 has `buf->previous` instead of `prev_reclen`; the skip must happen **before** `buf->previous` offset update. If skipping, return 0 and do not touch `buf->previous`.
* If `buf->sb` is NULL (some call paths), skip the `ilookup` check rather than crashing.
* `ilookup` can be expensive on every dirent; 2.4.0 accepts this. For performance, consider a per-directory flag or only check when `susfs_is_current_proc_umounted()`, but match 2.4.0 behavior first.

**Risk:** medium-high (all getdents paths, hot path). **Config:** `CONFIG_KSU_SUSFS_SUS_PATH`. **Phase:** B.

---

### 4.16 `fs/stat.c`

**2.4.0 expects (`P:L1982–L2113`):**
* Include `<linux/susfs_def.h>`.
* Extern `ksu_is_init_rc_hook_enabled`, `ksu_handle_vfs_fstat`, `ksu_su_compat_enabled`, `ksu_handle_stat`; plus SUS_KSTAT helpers.
* `vfs_getattr_nosec()`: if app UID and inode is sus-kstat, set `stat->mnt_id` and `stat->result_mask |= STATX_SUS_KSTAT` / `STATX_SUS_KSTAT_FUSE`; after `getattr`/`generic_fillattr`, call `susfs_sus_kstat_spoof_generic_fillattr()`.
* `vfs_fstat()`: call `ksu_handle_vfs_fstat(fd, &stat->size)`.
* `vfs_statx()`: `susfs_is_current_proc_no_su()` fast path, `ksu_su_compat_enabled` check, `ksu_handle_stat`; after `vfs_getattr`, clear `STATX_SUS_KSTAT*` and skip `stat->mnt_id`.

**4.19 reality:**
* `int vfs_getattr_nosec(const struct path *path, struct kstat *stat, u32 request_mask, unsigned int query_flags)` at `B:fs/stat.c:69`.
  * 6.1 has `struct user_namespace *mnt_userns = mnt_user_ns(path->mnt);` and `inode->i_op->getattr(mnt_userns, path, stat, request_mask, query_flags)`. 4.19 has **no `mnt_userns`**: `inode->i_op->getattr(path, stat, request_mask, query_flags)`; `generic_fillattr(inode, stat)` takes two args.
  * `struct kstat` in 4.19 (`B:include/linux/stat.h:24`) has **no `mnt_id` field**; `STATX_MNT_ID` is not defined.
* `vfs_fstat()` is an inline in `B:include/linux/fs.h:3316`:
  ```c
  static inline int vfs_fstat(unsigned int fd, struct kstat *stat)
  {
      return vfs_statx_fd(fd, stat, STATX_BASIC_STATS, 0);
  }
  ```
  There is no `int vfs_fstat(...)` function in `fs/stat.c`; the 6.1 patch hunk context (`@@ -181,6 +237,10 @@ int vfs_fstat`) will not apply.
* `int vfs_statx(int dfd, const char __user *filename, int flags, struct kstat *stat, u32 request_mask)` at `B:fs/stat.c:171`; `vfs_statx_fd()` at `:121`. It already calls `ksu_handle_stat(&dfd, &filename, &flags)` at the top under `CONFIG_KSU`.
* `struct kstat` has `result_mask` but no `mnt_id`.

**Adaptation:**
* `vfs_getattr_nosec`: rewrite the 6.1 block:
  ```c
  #ifdef CONFIG_KSU_SUSFS_SUS_KSTAT
  if (susfs_is_current_app_uid()) {
      bool is_fuse = false;
      if (susfs_is_inode_sus_kstat(d_backing_inode(path->dentry), &is_fuse)) {
          stat->result_mask |= is_fuse ? STATX_SUS_KSTAT_FUSE : STATX_SUS_KSTAT;
      }
  }
  #endif
  if (inode->i_op->getattr) {
      int err = inode->i_op->getattr(path, stat, request_mask, query_flags);
      if (!err && (stat->result_mask & (STATX_SUS_KSTAT|STATX_SUS_KSTAT_FUSE)))
          susfs_sus_kstat_spoof_generic_fillattr(inode, stat,
              stat->result_mask & STATX_SUS_KSTAT_FUSE ? STATX_SUS_KSTAT_FUSE : STATX_SUS_KSTAT);
      return err;
  }
  #ifdef CONFIG_KSU_SUSFS_SUS_KSTAT
  if (stat->result_mask & (STATX_SUS_KSTAT|STATX_SUS_KSTAT_FUSE)) {
      generic_fillattr(inode, stat);
      susfs_sus_kstat_spoof_generic_fillattr(inode, stat,
          stat->result_mask & STATX_SUS_KSTAT_FUSE ? STATX_SUS_KSTAT_FUSE : STATX_SUS_KSTAT);
      return 0;
  }
  #endif
  generic_fillattr(inode, stat);
  return 0;
  ```
  Do **not** assign `stat->mnt_id`; 4.19 has no such field. `susfs_sus_kstat_spoof_generic_fillattr()` itself guards `stat->mnt_id` with `LINUX_VERSION_CODE >= 5.10`, so it compiles unchanged.
* `vfs_fstat`: do not add a `vfs_fstat` function. If `ksu_handle_vfs_fstat` is needed, add it inside `vfs_statx_fd()` after `vfs_getattr` (line 121), guarded by `#ifdef CONFIG_KSU_SUSFS`; but current KSU does not export it, so drop this hook in Phase A.
* `vfs_statx`: 4.19's `vfs_statx` takes `const char __user *filename`, not `struct filename *`. Keep the existing `ksu_handle_stat(&dfd, &filename, &flags)` call; if you add the 2.4.0 `susfs_is_current_proc_no_su()` guard, note current KSU never sets `TIF_PROC_NO_SU`. After `error = vfs_getattr(&path, stat, request_mask, flags);`, add:
  ```c
  #ifdef CONFIG_KSU_SUSFS_SUS_KSTAT
  if (stat->result_mask & (STATX_SUS_KSTAT|STATX_SUS_KSTAT_FUSE)) {
      stat->result_mask &= ~(STATX_SUS_KSTAT|STATX_SUS_KSTAT_FUSE);
      return error;
  }
  #endif
  ```
  Remove all `stat->mnt_id = ...` and `STATX_MNT_ID` lines; they do not exist in 4.19. If userspace needs `stx_mnt_id`, 4.19's `struct statx` does not have it either (`B:include/uapi/linux/stat.h`).
* Alternative: R's 1.5.11 approach hooks `generic_fillattr()` directly; it is simpler for the common case but misses filesystems with custom `getattr`. 2.4.0's `vfs_getattr_nosec` approach is more complete and should be used.

**Risk:** medium-high due to stat being ubiquitous and 32/64-bit compat. **Config:** `CONFIG_KSU_SUSFS_SUS_KSTAT`. **Phase:** B.

---

### 4.17 `fs/statfs.c`

**2.4.0 expects (`P:L2114–L2221`):**
* Include `<linux/susfs_def.h>`, `"mount.h"`, `"internal.h"`.
* Add `statfs_by_dentry_wrapper()` and `calculate_f_flags_wrapper()` non-static functions for `susfs.c`.
* `vfs_statfs()`: if `susfs_is_current_proc_umounted()`, walk `mnt->mnt_parent` while `mnt_id >= DEFAULT_KSU_MNT_ID`, use the non-sus mnt root and flags.
* `vfs_ustat()`: if `s_root->d_inode->i_mapping->flags & AS_FLAGS_SUS_MOUNT`, return `-EINVAL`.

**4.19 reality:**
* `static int calculate_f_flags(struct vfsmount *mnt)` at `B:fs/statfs.c:47`.
* `static int statfs_by_dentry(struct dentry *dentry, struct kstatfs *buf)` at `:53`.
* `int vfs_statfs(const struct path *path, struct kstatfs *buf)` at `:70`; calls `statfs_by_dentry(path->dentry, buf)` and `calculate_f_flags(path->mnt)`.
* `vfs_ustat()` exists around `:224` in R's diff; 4.19 has `struct super_block *s = user_get_super(dev);`.
* `statfs_by_dentry` and `calculate_f_flags` are static; susfs.c cannot call them directly (different translation unit), hence the wrappers.

**Adaptation:**
* Add non-static wrappers:
  ```c
  int statfs_by_dentry_wrapper(struct dentry *dentry, struct kstatfs *buf)
  {
      return statfs_by_dentry(dentry, buf);
  }
  int calculate_f_flags_wrapper(struct vfsmount *mnt)
  {
      return calculate_f_flags(mnt);
  }
  ```
  Declare them in `fs/susfs.c` (extern) or in `susfs.h`. No `EXPORT_SYMBOL` needed for built-in.
* `vfs_statfs` 4.19 adaptation:
  ```c
  struct mount *mnt = real_mount(path->mnt);
  if (likely(susfs_is_current_proc_umounted())) {
      for (; mnt->mnt_id >= DEFAULT_KSU_MNT_ID && mnt->mnt_parent != mnt;
           mnt = mnt->mnt_parent)
          ;
  }
  error = statfs_by_dentry(mnt->mnt.mnt_root, buf);
  if (!error)
      buf->f_flags = calculate_f_flags(&mnt->mnt);
  ```
  Guard against `mnt->mnt_parent == mnt` to avoid infinite loop; 4.19 `real_mount` works.
* `vfs_ustat`: 4.19 `s->s_root->d_inode->i_mapping` exists; use `SUSFS_IS_INODE_SUS_MAP`? No, 2.4.0 uses `AS_FLAGS_SUS_MOUNT`; the flag is set in `susfs.c`. Use the raw bit like R's 1.5.11 code or define a helper.
* `struct kstatfs` fields same.

**Risk:** low-medium. **Config:** `CONFIG_KSU_SUSFS_SUS_MOUNT` / `SUS_KSTAT`. **Phase:** B.

---

### 4.18 `fs/super.c`

**2.4.0 expects (`P:L2222–L2270`):**
* Include `<linux/susfs_def.h>`.
* In `get_anon_bdev()`, if the sdcard-not-decrypted static key is set and current is KSU domain, allocate a minor from `DEFAULT_KSU_MNT_MINOR_DEV` (`1 << 12`) instead of the normal range, to avoid anon-device minor gaps/collisions.

**4.19 reality:**
* `int get_anon_bdev(dev_t *p)` at `B:fs/super.c:1014`.
* It uses `ida_alloc_range(&unnamed_dev_ida, 1, (1 << MINORBITS) - 1, GFP_ATOMIC)` at `:1022`; `MINORBITS` is 20; `DEFAULT_KSU_MNT_MINOR_DEV` 4096 fits.
* `ida_alloc_range()` exists in 4.19 (`lib/idr.c`).

**Adaptation:**
* Apply literally. This is one of the few 2.4.0 hunks that is nearly copy-paste; only check `static_branch_unlikely` macro spelling.
* `get_anon_bdev()` is called for anonymous filesystems (tmpfs, etc.) in KSU context; the special range avoids normal minor allocations. Keep the key check.

**Risk:** low. **Config:** `CONFIG_KSU_SUSFS_SUS_MOUNT`. **Phase:** B (SUS_MOUNT).

---

### 4.19 `kernel/kallsyms.c`

**2.4.0 expects (`P:L2271–L2321`):**
* Include `<linux/susfs_def.h>`.
* In `s_show()`, when `CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS`, suppress symbols whose names start with `ksu_`, `__ksu_`, `susfs_`, `ksud`, `is_ksu_`, `is_manager_`, `escape_to_`, `setup_selinux`, `track_throne`, `on_post_fs_data`, `try_umount`, `kernelsu`, `__initcall__kmod_kernelsu`, `apply_kernelsu`, `handle_sepolicy`, `getenforce`, `setenforce`, `is_zygote`.

**4.19 reality:**
* `static int s_show(struct seq_file *m, void *p)` at `B:kernel/kallsyms.c:637`.
* The function prints `seq_printf(m, "%px %c %s\n", value, iter->type, iter->name);`.
* `susfs_starts_with()` is a static inline in `susfs_def.h`; no external dependency.

**Adaptation:**
* Apply the `#else` block; 4.19's `s_show` has the same `iter->name`/`iter->type`.
* Warning: hiding symbols by prefix is fragile with LTO/CFI; test with `grep -i ksu /proc/kallsyms` after boot. This feature only hides from `/proc/kallsyms`, not from `/sys/module` sections or `kallsyms_lookup_name`; it is an anti-detection aid, not a security boundary.

**Risk:** low. **Config:** `CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS`. **Phase:** A.

---

### 4.20 `kernel/reboot.c`

**2.4.0 expects (`P:L2322–L2351`):**
* Extern `ksu_handle_sys_reboot(int magic1, int magic2, unsigned int cmd, void __user **arg)`.
* In `SYSCALL_DEFINE4(reboot, ...)`, call it before the normal reboot handling.

**4.19 reality:**
* `SYSCALL_DEFINE4(reboot, int, magic1, int magic2, unsigned int, cmd, void __user *, arg)` exists around `B:kernel/reboot.c:309`.
* The baseline already contains the KernelSU reboot hook (`ksu_handle_sys_reboot`) — search around `B:kernel/reboot.c:300–330`; the same call is present in the KernelSU branch.

**Adaptation:**
* Do not duplicate. Keep the existing KSU hook. 2.4.0's hunk is essentially a no-op relative to the current KernelSU tree.

**Risk:** low. **Config:** none. **Phase:** skip.

---

### 4.21 `kernel/sys.c`

**2.4.0 expects (`P:L2352–L2431`):**
* `newuname()`: if `susfs_is_uname_spoof_buffer_set`, call `susfs_spoof_uname(&tmp)` before copying to user. Include `<linux/susfs_def.h>` and declare the static key + helper.
* `setpriority()`/`getpriority()`: if `susfs_is_priority_uid_filtered(current_uid().val)`, skip/hide the target process.
* `__sys_setresuid()`: call `ksu_handle_setresuid(ruid, euid, suid)` (KSU 2.4.0 API) after changing UIDs; also `susfs_set_current_proc_umounted` work.

**4.19 reality:**
* `SYSCALL_DEFINE1(newuname, struct new_utsname __user *, name)` at `B:kernel/sys.c:1243`. It does `down_read(&uts_sem); memcpy(&tmp, utsname(), sizeof(tmp)); up_read(&uts_sem);` then `copy_to_user`.
* `SYSCALL_DEFINE3(setpriority, int, which, int, who, int, niceval)` at `B:kernel/sys.c:201`; `getpriority` at `:271`.
* `long __sys_setresuid(uid_t ruid, uid_t euid, uid_t suid)` at `B:kernel/sys.c:626`; it already calls `ksu_handle_setresuid_cred(new, old)` indirectly via LSM (`security_task_fix_setuid`), not a direct `ksu_handle_setresuid()`.
* No `ksu_handle_setresuid` symbol in current KSU.
* `utsname()` returns `struct new_utsname *`; `struct new_utsname` has `sysname/nodename/release/version/machine/domainname` — same in 4.19.

**Adaptation:**
* **`newuname`:** apply the spoof hook. It is easy and independent of KSU internals:
  ```c
  #ifdef CONFIG_KSU_SUSFS_SPOOF_UNAME
  if (static_branch_likely(&susfs_is_uname_spoof_buffer_set))
      susfs_spoof_uname(&tmp);
  #endif
  ```
  Note `susfs_spoof_uname()` expects `struct new_utsname *tmp`; it is called after `memcpy` while `uts_sem` is held; the function uses a seqlock, not sleeping. Good.
* **`setpriority`/`getpriority`:** apply the 2.4.0 checks. In 4.19 `current_uid().val` is `u32`; `susfs_is_priority_uid_filtered` takes `uid_t`. In `setpriority`, return `-EINVAL`/`-ESRCH` semantics? 2.4.0's patch typically does `if (susfs_is_priority_uid_filtered(current_uid().val)) return -EINVAL;` — match the patch exactly; check `P:L2380–L2410`.
* **`__sys_setresuid`:** **do not call `ksu_handle_setresuid()`** in Phase A; it does not exist. The current KSU LSM hook already handles UID changes. If 2.4.0's `TIF_PROC_UMOUNTED` flag needs to be set after `setresuid`, add that in a KSU patch or in the LSM hook; do not invent a direct call. Mark this sub-hunk B/deferred.

**Risk:** low for `newuname`/priority; medium for setresuid. **Config:** `SPOOF_UNAME`, `PRIORITY_UID_FILTER`. **Phase:** A for uname/priority; B for setresuid.

---

### 4.22 `mm/memory.c`

**2.4.0 expects (`P:L2432–L2457`):**
* In `__access_remote_vm()`, after `get_user_pages_remote()` returns a `vma`, if `vma->vm_file` and `SUSFS_IS_INODE_SUS_MAP(file_inode(vma->vm_file))`, break/return 0 so `/proc/pid/mem` cannot read the hidden mapping.

**4.19 reality:**
* `int __access_remote_vm(struct task_struct *tsk, struct mm_struct *mm, unsigned long addr, void *buf, int len, unsigned int gup_flags)` at `B:mm/memory.c:4472`.
* 4.19's version uses `get_user_pages_remote(tsk, mm, addr, 1, gup_flags, &page, &vma, NULL)` (7 args) and then `copy_from_user_page`/`copy_to_user_page`. The 6.1 patch uses the same `get_user_pages_remote` shape but `vma` handling is a bit different.
* `SUSFS_IS_INODE_SUS_MAP` is a macro; no extra include unless `susfs_def.h`.

**Adaptation:**
* Add after the `get_user_pages_remote` success/failure handling:
  ```c
  #ifdef CONFIG_KSU_SUSFS_SUS_MAP
  if (vma && vma->vm_file && SUSFS_IS_INODE_SUS_MAP(file_inode(vma->vm_file)))
      break;
  #endif
  ```
  If `vma` is NULL on failure, guard. Place it before copying to/from the page; `break` exits the `while (len > 0)` loop.
* Include `<linux/susfs_def.h>` at the top under `#ifdef CONFIG_KSU_SUSFS_SUS_MAP`.

**Risk:** low. **Config:** `CONFIG_KSU_SUSFS_SUS_MAP`. **Phase:** B.

---

### 4.23 `security/selinux/avc.c`

**2.4.0 expects (`P:L2458–L2500`):**
* Extern `susfs_ksu_sid`, `susfs_priv_app_sid`, `susfs_is_avc_log_spoofing_enabled`.
* In `avc_audit_post_callback()` / `avc_dump_query()`: if AVC log spoofing is enabled and current is app UID, replace `scontext`/`tcontext` SIDs before `security_sid_to_context()`.

**4.19 reality:**
* `static void avc_audit_post_callback(struct audit_buffer *ab, void *a)` at `B:security/selinux/avc.c:751` (comment at `:746`).
* `static void avc_dump_query(struct audit_buffer *ab, u32 ssid, u32 tsid, u16 tclass)` at `B:security/selinux/avc.c:174`.
* `security_sid_to_context(state, sid, &scontext, &scontext_len)` exists in 4.19; `avc_dump_query` uses it.
* `susfs_ksu_sid`/`susfs_priv_app_sid` are **not defined** in 4.19 KSU. They are supplied by 2.4.0's KernelSU patch. Without them, the AVC spoofing can only work for `susfs_priv_app_sid` if you define a reasonable SID by looking up `u:r:priv_app:s0`, and `susfs_ksu_sid` by looking up `u:r:ksu:s0` at boot. That is possible but requires a SELinux policy that has those types.

**Adaptation:**
* Phase A: do not apply; no AVC spoofing.
* Phase B: port the *logic* but implement the two SID globals in `fs/susfs.c` or KSU:
  * At `susfs_init` or first policy load, resolve `u:r:ksu:s0` via `security_context_to_sid()` and store; resolve `u:r:priv_app:s0` similarly.
  * `susfs_is_avc_log_spoofing_enabled` is a static key in `susfs.c`; declare extern.
* `avc_dump_query` in 4.19 has `u32 ssid, u32 tsid, u16 tclass`; replace arguments before `security_sid_to_context`. No `state->policy` indirection.

**Risk:** medium (policy-dependent). **Config:** always part of `CONFIG_KSU_SUSFS` but can be left inert if `susfs_set_avc_log_spoofing` is never called. **Phase:** B.

---

### 4.24 `security/selinux/hooks.c`

**2.4.0 expects (`P:L2501–L2871`):**
* Extern `ksu_selinux_hide_running`, `susfs_security_context_to_sid`, `susfs_security_context_to_sid_force`.
* Add `susfs_selinux_inode_setxattr()` and `susfs_selinux_inode_post_setxattr()` — copies of the SELinux functions with `susfs_security_context_to_sid*()` instead of `security_context_to_sid*()`; wrapper `handle_*` selects them when `current_uid().val >= 10000 && ksu_selinux_hide_running`.
* Add `susfs_selinux_setprocattr()` and wrapper; replace hook table entries `selinux_inode_setxattr`/`selinux_inode_post_setxattr`/`selinux_setprocattr` with the `handle_*` versions.
* `selinux_setprocattr()` logic uses `security_context_to_sid()` and `security_bounded_transition()`.

**4.19 reality:**
* `selinux_inode_setxattr(struct dentry *dentry, const char *name, const void *value, size_t size, int flags)` at `B:security/selinux/hooks.c:3237`; **no `mnt_userns` argument** (6.1 has `struct user_namespace *mnt_userns`).
* `selinux_inode_post_setxattr()` at `:3329`; same signature minus mnt_userns.
* `selinux_setprocattr(const char *name, void *value, size_t size)` at `:6407`.
* `selinux_hooks[]` table around `:6969`.
* `selinux_superblock(inode->i_sb)` is a 6.1 helper; 4.19 uses `inode->i_sb->s_security` cast to `struct superblock_security_struct *`.
* `inode_owner_or_capable(mnt_userns, inode)` in 6.1; 4.19 is `inode_owner_or_capable(inode)`.
* `security_validate_transition(state, oldsid, newsid, tasksid, class)` exists in 4.19 (`security/selinux/ss/services.c`), same name.
* `avc_has_perm(&selinux_state, ...)` exists; 4.19 `selinux_state` is a global, not a per-policy pointer.
* `ksu_selinux_hide_running` does not exist; current KSU has `ksu_selinux_hide_enabled` static in `selinux_hide.c`.
* `security_context_to_sid()` 4.19 signature: `int security_context_to_sid(struct selinux_state *state, const char *scontext, u32 scontext_len, u32 *sid, gfp_t gfp)`; `security_context_to_sid_force()` same minus gfp. 2.4.0's `susfs_security_context_to_sid*()` wrappers are designed for this.

**Adaptation:**
* This file is a *copy-and-adapt* job, not a hunk apply. Use 4.19's existing functions as the base and change only the context-to-SID calls and permission checks.
* Change all `selinux_inode_setxattr(struct user_namespace *mnt_userns, ...)` to 4.19 signature and drop `mnt_userns` arguments:
  * `inode_owner_or_capable(inode)`
  * `selinux_superblock(inode->i_sb)` → `(struct superblock_security_struct *)inode->i_sb->s_security`
* Define `susfs_security_context_to_sid()` / `_force()` in `security/selinux/ss/services.c` (see §4.26) and declare them in a header or `extern` in hooks.c.
* `handle_selinux_inode_setxattr()` condition: replace `ksu_selinux_hide_running` with `is_ksu_domain()` or a new exported `ksu_selinux_hide_enabled()` getter. **Do not silently use `ksu_selinux_hide_enabled` if it is static**; add an exported getter in KSU or make it non-static.
* Hook table: 4.19's `selinux_hooks[]` entries must match the wrapper signatures. Search for `LSM_HOOK_INIT(inode_setxattr, selinux_inode_setxattr)` and replace with `handle_selinux_inode_setxattr`; same for post_setxattr and setprocattr.
* `susfs_selinux_setprocattr`: 4.19's `selinux_setprocattr` uses `security_context_to_sid()` and `security_bounded_transition()`; the 2.4.0 copy is large. Apply the same substitution and keep 4.19 error paths. The `fake_status` pieces belong in `selinuxfs.c`, not here.
* If `CONFIG_KSU_SUSFS` is enabled but SELinux hiding is not ready, compile the wrappers as pass-through to the original functions (no behavior change), so the hook table can stay untouched.

**Risk:** high. **Config:** `CONFIG_KSU_SUSFS` (no separate option). **Phase:** B, only with KernelSU selinux-hide changes.

---

### 4.25 `security/selinux/selinuxfs.c`

**2.4.0 expects (`P:L2872–L3076`):**
* Extern `ksu_selinux_hide_running`, `fake_status`, `initialize_fake_status`.
* `sel_open_handle_status()`: if `ksu_selinux_hide_running`, use `fake_status` instead of `selinux_kernel_status_page()`.
* `sel_write_context()` / `sel_write_access()`: use `susfs_security_context_to_sid()` and `susfs_security_compute_av_user()` to keep sus types hidden.
* `sel_write_create()` etc. may be touched.

**4.19 reality:**
* `sel_open_handle_status()` at `B:security/selinux/selinuxfs.c:220` — uses `selinux_kernel_status_page(fsi->state)`.
* `sel_write_context()` at `:599`; `sel_write_access()` at `:819` (forward declaration at 770).
* `sel_write_access()` uses `security_context_str_to_sid()` and `security_compute_av_user()`; both exist in 4.19 (`ss/services.c`).
* `fake_status` / `initialize_fake_status` are **not exported** by current KSU's `selinux_hide.c`; its fake status is set up internally and its `sel_open_handle_status` hook is installed by `kernelsu`'s own `selinux_hide` init, not through these globals.

**Adaptation:**
* **Avoid double-patching.** Current KSU's `selinux_hide.c` already hooks `sel_open_handle_status` via `kernelsu` internals. Adding 2.4.0's `sel_open_handle_status` change on top will create two fake-status paths and can break `getenforce`/`selinuxfs`.
* Recommended: skip `sel_open_handle_status` entirely; let KSU's existing selinux_hide own it. Port only `sel_write_context`/`sel_write_access` if you need sus-type hiding in `selinuxfs`; these are independent and use `susfs_security_context_to_sid()`/`susfs_security_compute_av_user()`.
* If you must port the fake status: add an exported getter/`struct page *` from KSU and guard with `is_ksu_domain()`; ensure it is initialized before any `selinuxfs` open. Do not define your own `fake_status` in the kernel tree, because KSU already has one.
* `sel_write_access` 4.19 signature `static ssize_t sel_write_access(struct file *file, char *buf, size_t size)`; same shape as 6.1.

**Risk:** high (can break SELinux userspace). **Config:** `CONFIG_KSU_SUSFS` + KSU selinux hide. **Phase:** B last; may be omitted if KSU selinux_hide already satisfies the use case.

---

### 4.26 `security/selinux/ss/services.c`

**2.4.0 expects (`P:L3077–L3389`):**
* Extern `susfs_is_current_ksu_domain`, `susfs_is_sus_type_rcu`, `susfs_spoof_avd_allowed_rcu`.
* Hook `selinux_policy_commit()`: if KSU domain and oldpolicy, preserve `latest_granting` (no seqno bump).
* Hook `security_set_bools()`: same preserve.
* Add `susfs_string_to_context_struct()`, `susfs_security_context_to_sid_core()`, `susfs_security_context_to_sid()`, `susfs_security_context_to_sid_force()`, `susfs_security_compute_av_user()`.

**4.19 reality:**
* **No `selinux_policy_commit()`** in 4.19; policy commit is inline in `security_load_policy()` at `B:security/selinux/ss/services.c:2154`. The seqno increment is at ~`:2208` (`seqno = ++state->ss->latest_granting;`) and ~`:2282`.
* `security_set_bools()` at `:2923`; it has `newpolicy->latest_granting = oldpolicy->latest_granting + 1;` around `:2962`.
* `security_context_to_sid_core()` at `:1495` — static; 4.19 uses `state->ss->policydb` and `state->ss->sidtab`, not `state->policy`.
* `security_compute_av_user()` at `:1163` — `void security_compute_av_user(struct selinux_state *state, u32 ssid, u32 tsid, u16 tclass, struct av_decision *avd)` (same shape).
* `avd_init()` at `:945` — signature `void avd_init(struct selinux_state *state, struct av_decision *avd)` (4.19 state-first; 6.1 `avd_init(policy, avd)`).
* `policydb` symbol tables in 4.19 are `struct hashtab`; use `hashtab_search(pol->p_users.table, scontextp)` instead of `symtab_search(&pol->p_users, ...)`.
* `typdatum->attribute` exists; `policydb->allow_unknown` exists; `context_struct_compute_av()` signature 4.19: `(struct policydb *policydb, struct context *scontext, struct context *tcontext, u16 tclass, struct av_decision *avd, struct extended_perms *xperms)` — same as 6.1.
* `sidtab_context_to_sid()` 4.19 returns int; 6.1 may return `-ESTALE` under RCU policy; 4.19 has no such retry.
* `security_context_to_sid_core` in 4.19:
  ```c
  static int security_context_to_sid_core(struct selinux_state *state,
                                          const char *scontext, u32 scontext_len,
                                          u32 *sid, u32 def_sid, gfp_t gfp_flags,
                                          int force)
  ```
  It takes `read_lock(&state->ss->policy_rwlock)` and uses `state->ss->policydb`/`state->ss->sidtab`.
* `security_load_policy()` 4.19 does not have a separate `selinux_policy` object; `oldpolicy` is `state->ss->policydb`, `newpolicy` is a `struct policydb *`.

**Adaptation:**
* **Policy commit hooks:** replace `selinux_policy_commit()` with inline guards:
  * In `security_load_policy()`, at the point where `seqno = ++state->ss->latest_granting;` on reload, change to:
    ```c
    if (susfs_is_current_ksu_domain())
        seqno = state->ss->latest_granting;
    else
        seqno = ++state->ss->latest_granting;
    ```
    Only apply on the reload path, not the initial policy path (`!selinux_initialized(state)`); the initial policy must still set 1.
  * In `security_set_bools()`, similar guard before `newpolicy->latest_granting = oldpolicy->latest_granting + 1;`.
  * Use `is_ksu_domain()`/a wrapper instead of `susfs_is_current_ksu_domain()`.
* **Context parsing:** port `susfs_string_to_context_struct()` but replace every `symtab_search(&pol->p_xxx, ...)` with `hashtab_search(pol->p_xxx.table, ...)`. `context_init`, `context_destroy`, `mls_context_to_sid`, `policydb_context_isvalid` all exist in 4.19 with the same names (verify `policydb_context_isvalid` signature: `int policydb_context_isvalid(struct policydb *p, struct context *c)`).
* **`susfs_security_context_to_sid_core`:** write a 4.19 version:
  ```c
  static int susfs_security_context_to_sid_core(struct selinux_state *state,
      const char *scontext, u32 scontext_len, u32 *sid, u32 def_sid,
      gfp_t gfp_flags, int force)
  {
      char *scontext2;
      struct context context;
      int rc = -EINVAL;
      if (!selinux_initialized(state)) { *sid = SECINITSID_UNLABELED; return 0; }
      if (!scontext_len) return -EINVAL;
      scontext2 = kmemdup_nul(scontext, scontext_len, gfp_flags);
      ...
      read_lock(&state->ss->policy_rwlock);
      rc = susfs_string_to_context_struct(state->ss->policydb, state->ss->sidtab,
                                          scontext2, &context, def_sid);
      if (rc == -EINVAL && force) { context.type = ...; rc = 0; }
      if (!rc) rc = sidtab_context_to_sid(state->ss->sidtab, &context, sid);
      read_unlock(&state->ss->policy_rwlock);
      context_destroy(&context);
      kfree(scontext2);
      return rc;
  }
  ```
  Match the 4.19 original's `def_sid` handling and `SECINITSID_UNLABELED` early return.
* **`susfs_security_context_to_sid()`** = call core with `force=0`; **`_force()`** = `force=1`.
* **`susfs_security_compute_av_user()`:** easiest is:
  ```c
  void susfs_security_compute_av_user(struct selinux_state *state, u32 ssid, u32 tsid,
                                      u16 tclass, struct av_decision *avd)
  {
      security_compute_av_user(state, ssid, tsid, tclass, avd);
      susfs_spoof_avd_allowed_rcu(...); // needs scontext/tcontext types
  }
  ```
  But 2.4.0's copy computes the types itself to call `susfs_spoof_avd_allowed_rcu(scontext->type, tcontext->type, tclass, &avd->allowed)`. In 4.19, `security_compute_av_user()` already has `scontext`/`tcontext` internally; either duplicate its body (as 2.4.0 does) or extend `security_compute_av_user` with a hook. The duplicate-body approach is safer for a first port but must copy 4.19's `avd_init(state, avd)` and `rcu_read_lock()` semantics.
* **Do not use `state->policy`** anywhere; 4.19's `struct selinux_state` has `struct selinux_ss *ss` with `policydb`, `sidtab`, `policy_rwlock`, `latest_granting`, `status_lock`.
* **`security_context_to_sid_force`** in 4.19 is a separate function at ~`:1585`; its signature `(state, scontext, scontext_len, sid)`; call the core with `force=1`.

**Risk:** high. **Config:** `CONFIG_KSU_SUSFS` + SELinux hide. **Phase:** B last.

---

### 4.27 `security/selinux/status.c` (4.19: `security/selinux/ss/status.c`)

**2.4.0 expects (`P:L3390–L3438`):**
* Add `#include <linux/susfs.h>` / extern `susfs_is_current_ksu_domain`.
* In `selinux_status_update_setenforce()` and `selinux_status_update_policyload()`, take `status_lock`, update sequence, and bypass the sequence bump when current is KSU domain.

**4.19 reality:**
* **There is no `security/selinux/status.c`**; the file is `security/selinux/ss/status.c`.
* Functions: `selinux_kernel_status_page()` `:45`, `selinux_status_update_setenforce()` `:82`, `selinux_status_update_policyload()` `:108`.
* Signatures:
  ```c
  void selinux_status_update_setenforce(struct selinux_state *state, int enforcing);
  void selinux_status_update_policyload(struct selinux_state *state, u32 seqno);
  ```
* `state->status_lock` and `state->ss->status` exist.

**Adaptation:**
* Apply the same logic to `security/selinux/ss/status.c`; no path rename needed in Kconfig/Makefile because the file is already built.
* Replace `susfs_is_current_ksu_domain()` with `is_ksu_domain()` (or wrapper).
* Keep the `mutex_lock(&state->status_lock)` / `mutex_unlock` pairs; 4.19 uses `struct mutex status_lock` in `selinux_state`.
* If the status page is mapped read-only by userspace, the sequence update still works through the kernel mapping; same as 6.1.

**Risk:** medium. **Config:** `CONFIG_KSU_SUSFS` + SELinux hide. **Phase:** B.

---

## 5. Core source audit: `fs/susfs.c`, `include/linux/susfs.h`, `include/linux/susfs_def.h`

### 5.1 `fs/susfs.c` (2560 lines)

**Overall:** the file is surprisingly portable. It is mostly self-contained data structures, RCU/hash tables, seq locks, string parsing, and `kern_path`/`filp_open`/`kernel_read`. It does **not** include `fs/namei.c` internals or `struct selinux_policy`. Most of the 6.1-specific code is in the patch hunks that call into it, not in `susfs.c` itself.

**Verified present in 4.19 (grep evidence):**
* `strscpy` — `include/linux/string.h:34`, `lib/string.c:178`. **Present**; no replacement needed.
* `kstrdup_const` / `kfree_const` — present.
* `kmemdup_nul` — present.
* `kvmalloc` / `kvfree` — present.
* `ida_alloc_min` / `ida_alloc_range` / `ida_free` — present (`include/linux/idr.h`, `lib/idr.c`).
* `DEFINE_SRCU` / `DEFINE_STATIC_SRCU` — present in `include/linux/srcutree.h:147–148` (via `srcu.h` with `CONFIG_TREE_SRCU`).
* `srcu_read_lock` / `srcu_read_unlock` / `synchronize_srcu` — present.
* `DEFINE_STATIC_KEY_TRUE/FALSE`, `static_branch_likely/unlikely`, `static_branch_enable/disable`, `static_key_enabled` — present (`include/linux/jump_label.h:164–167`, `:436–439`).
* `DEFINE_SEQLOCK`, `write_seqlock`, `read_seqbegin`, `read_seqretry` — present.
* `DEFINE_HASHTABLE`, `hash_add_rcu`, `hash_del_rcu`, `hash_for_each_possible_rcu`, `hash_for_each_possible_safe` — present.
* `struct address_space.flags` is `unsigned long`; `set_bit(AS_FLAGS_*, &inode->i_mapping->flags)` works. `AS_FLAGS_*` 33–39 are above the 4.19 `AS_*` enum (0–5), so no conflict.
* `struct fuse_inode` has `struct inode inode; u64 nodeid;` (`fs/fuse/fuse_i.h:63–69`), so `get_fuse_inode(inode)->inode.i_mapping` and `fi->nodeid` work.
* `struct kstatfs` has `f_flags`, `f_type`, etc.
* `struct new_utsname` and `__NEW_UTS_LEN` exist.
* `huge_decode_dev` / `old_decode_dev` / `new_decode_dev` exist (`include/linux/kdev_t.h:58`); arm64 defines `__ARCH_WANT_COMPAT_STAT64`, so the `#if defined(__ARCH_WANT_STAT64) || defined(__ARCH_WANT_COMPAT_STAT64)` branch in `susfs_add_sus_kstat()` is active.
* `GFP_KERNEL_ACCOUNT` exists (`include/linux/gfp.h:298`).
* `strsep` exists (`include/linux/string.h:98`).
* `pr_info_ratelimited` / `pr_err_ratelimited` exist.
* `system_unbound_wq` / `queue_delayed_work` / `INIT_DELAYED_WORK` exist.

**Genuine 4.19 issues in `susfs.c`:**

1. **`fsnotify` API is 4.19, not 6.1.** The sdcard monitor (`susfs_sdcard_monitor_fn`, `susfs_handle_sdcard_inode_event`, `add_mark_on_inode`) uses:
   * `struct fsnotify_ops` with `.handle_inode_event` — **absent in 4.19**; 4.19's `fsnotify_ops` has `.handle_event(group, to_tell, mask, data, data_is, name, cookie, iter_info)` (`fs/notify/fsnotify.c:244`).
   * `fsnotify_alloc_group(&fsnotify_ops, 0)` — 2.4.0 already guards with `#if LINUX_VERSION_CODE >= 6.0`, so 4.19 takes the one-argument branch; that part is fine.
   * `fsnotify_group_lock` — not used in `susfs.c`; only in `fdinfo.c`.
   * `fsnotify_init_mark`, `fsnotify_add_inode_mark`, `fsnotify_destroy_group`, `fsnotify_put_mark` — all present in 4.19.
   **Fix:** rewrite the callback as:
   ```c
   static int susfs_handle_sdcard_inode_event(struct fsnotify_group *group,
       struct inode *to_tell, u32 mask, const void *data, int data_is,
       const unsigned char *name, u32 cookie, struct fsnotify_iter_info *iter_info)
   {
       if (!name || strlen(name) != 7 || memcmp(name, "Android", 7))
           return 0;
       ...
   }
   static const struct fsnotify_ops fsnotify_ops = {
       .handle_event = susfs_handle_sdcard_inode_event,
   };
   ```
   `FS_ISDIR`, `FS_EVENT_ON_CHILD`, `FS_OPEN_PERM` exist in 4.19 (`include/linux/fsnotify_backend.h:52` etc.).
2. **`susfs_is_current_ksu_domain()`** is called in `susfs_switch_to_ksu_domain()` and `susfs_sdcard_monitor_fn()`. Define it in `susfs.c` as:
   ```c
   bool susfs_is_current_ksu_domain(void) { return is_ksu_domain(); }
   ```
   or replace calls with `is_ksu_domain()` and declare `extern bool is_ksu_domain(void);`.
3. **`setup_selinux("u:r:ksu:s0", cred)`** — current KSU signature is `void setup_selinux(const char *domain, struct cred *cred)` (`KernelSU/kernel/selinux/selinux.c:50`), same call shape. Good.
4. **`ksu_cred`** — current KSU global exists. Good.
5. **`susfs_add_sus_type()` / `susfs_spoof_avd_allowed_rcu()`** — these are only used by SELinux code; `susfs.c` itself compiles them unconditionally under `CONFIG_KSU_SUSFS`. No 4.19 API issue inside `susfs.c`.
6. **`susfs_get_non_sus_mnt_id_from_mnt` / `susfs_get_non_sus_vfsmnt_from_vfsmnt`** — externs at `susfs.c:36–37`; definitions are in `fs/namespace.c` patch (§4.6). If `CONFIG_KSU_SUSFS_SUS_MOUNT` is off, these are still declared but not referenced (references are inside `#ifdef CONFIG_KSU_SUSFS_SUS_KSTAT` at `susfs.c:902/923`); link is fine because the functions are compiled only under `CONFIG_KSU_SUSFS` in namespace.c. If you compile `susfs.c` without namespace.c helpers, linker errors will occur; ensure `CONFIG_KSU_SUSFS_SUS_MOUNT` or stub the functions.
7. **`susfs_sus_kstat_spoof_generic_fillattr()`** has `#if LINUX_VERSION_CODE >= 5.10` around `stat->mnt_id`; 4.19 skips it, so no `struct kstat.mnt_id` compile error. Good.
8. **`susfs_sus_kstat_spoof_vfs_statfs()`** uses `memcpy(buf, &entry->spoofed_kstatfs, sizeof(struct kstatfs))`; `struct kstatfs` layout is stable enough. Good.
9. **`susfs_open_redirect_spoof_do_proc_readlink()`** uses `strscpy(tmp_buf, ..., SUSFS_MAX_LEN_PATHNAME - 1)`; `tmp_buf` is `PATH_MAX` (4096) so 255 is conservative. Good.
10. **`susfs_add_open_redirect()`** calls `vfs_statfs(&target_path, &new_entry_target->spoofed_kstatfs)`; `vfs_statfs` signature in 4.19 is `int vfs_statfs(const struct path *, struct kstatfs *)`; same. Good.
11. **`susfs_add_sus_kstat()`** uses `statfs_by_dentry_wrapper`/`calculate_f_flags_wrapper`; these must be added to `fs/statfs.c` (§4.17) or the file will not link. This is a **hard link dependency** when `CONFIG_KSU_SUSFS_SUS_KSTAT` is enabled.
12. **`susfs_open_redirect_spoof_do_sys_openat()`** returns `getname_kernel(redirected_pathname)`; `getname_kernel` exists in 4.19. Good. It is used by the `fs/namei.c` redirect hook, not by `susfs.c` internals.

**Core compile recommendation:** build `fs/susfs.c` first with only `CONFIG_KSU_SUSFS=y` and all subfeatures off. The only non-portable block that is unconditionally compiled is the sdcard fsnotify monitor; fix that and `susfs_is_current_ksu_domain()` and the file should compile on 4.19. Then enable features one at a time.

### 5.2 `include/linux/susfs.h`

**Portability:** high. It includes `<linux/version.h>`, `<linux/types.h>`, `<linux/utsname.h>`, `<linux/hashtable.h>`, `<linux/path.h>`, `<linux/susfs_def.h>`, `<linux/statfs.h>`. All exist in 4.19 except `<linux/version.h>`, which is generated at build time (normal).
* `struct st_susfs_sus_path` etc. use only fixed-width types, `dev_t`, `struct kstatfs`, `struct new_utsname`, `bool`. All present.
* `SUSFS_APP_STR_BUF_SIZE 65535` and `SUSFS_ENABLED_FEATURES_SIZE 8192` are large message-buffer sizes. **2.4.0 already allocates these structures on the heap:** `susfs_set_priority_uid_filter()` uses `kvmalloc(sizeof(*info), GFP_KERNEL | __GFP_ZERO)`, `susfs_set_cmdline_or_bootconfig()` uses `kzalloc`, and `susfs_get_enabled_features()` uses `kzalloc`. This is the correct pattern for a 16 KB arm64 kernel stack; do not "simplify" these to local stack variables during the port. Any new helper added while adapting should keep the heap allocation and `copy_from_user` on the heap pointer.
  * `struct st_susfs_priority_uid_filter` contains `char app_strings[SUSFS_APP_STR_BUF_SIZE]` = 65535 bytes.
  * `struct st_susfs_enabled_features` = 8192 bytes; `struct st_susfs_spoof_cmdline_or_bootconfig` = 8192 bytes.
  * This is independent of 6.1 vs 4.19 but critical for picasso: a 64 KB `struct st_susfs_priority_uid_filter` on the stack would overflow.
* `susfs.h` prototypes are fine.

### 5.3 `include/linux/susfs_def.h`

**Portability:** high, with three notes:
1. It uses `test_thread_flag()`, `set_thread_flag()`, `clear_thread_flag()` and `current_uid()` but includes only `<linux/bits.h>`, `<linux/bitops.h>`, `<linux/bitmap.h>`, `<linux/string.h>`. In 4.19, `test_thread_flag` is in `<linux/thread_info.h>`, `current_uid` in `<linux/cred.h>`, `current` in `<linux/sched.h>`. Most includers already include those, but add explicit:
   ```c
   #include <linux/sched.h>
   #include <linux/cred.h>
   #include <linux/thread_info.h>
   ```
   to make the header self-contained. `R:include/linux/susfs_def.h` uses `test_ti_thread_flag(&current->thread_info, ...)`, which is also fine; either form works in 4.19 with `CONFIG_THREAD_INFO_IN_TASK` (arm64 selects it).
2. `TIF_PROC_UMOUNTED 33`, `TIF_PROC_NO_SU 34`, `TIF_PROC_UMOUNTED_FOR_ZYGOTE_NEXT 35`: 4.19 arm64 `TIF_*` values are well below 33 (check `arch/arm64/include/asm/thread_info.h`; `TIF_*` are 0–22ish). These are safe. Do **not** use `TIF_PROC_NO_SU` unless KSU sets it.
3. `AS_FLAGS_*` 33–39: 4.19 `address_space.flags` is `unsigned long` (64-bit on arm64), so bits 33–39 are safe. `AS_*` enum in 4.19 only uses 0–5. `VFSMOUNT_MNT_FLAGS_KSU_UNSHARED_MNT 0x80000000` used in `int mnt_flags` needs a cast (see §4.6).

### 5.4 Missing 4.19 API/type checklist (grep evidence)

| Symbol / type used by 2.4.0 | Present in 4.19? | Evidence / replacement |
|---|---|---|
| `struct kstat.mnt_id` | **No** | `B:include/linux/stat.h:24` has no `mnt_id`; remove assignments; `susfs_sus_kstat_spoof_generic_fillattr` has a `>=5.10` guard |
| `STATX_MNT_ID` / `stx_mnt_id` | **No** | 4.19 `include/uapi/linux/stat.h` lacks them; do not add `stat->result_mask |= STATX_MNT_ID` |
| `mnt_user_ns()` / `struct mount.mnt_userns` | **No** | `git grep mnt_user_ns B -- fs include` returns nothing; drop all `mnt_userns` args |
| `vfs_create_mount()` | **No** | use `vfs_kern_mount()` (`B:fs/namespace.c:948`) |
| `lookup_one_qstr_excl()` | **No** | use `__lookup_hash()` (`B:fs/namei.c:1623`) |
| `open_last_lookups()` | **No** | use `do_last()` (`B:fs/namei.c:3375`) |
| `struct nameidata.state` | **No in B**; yes in R | add as R does (`R:fs/namei.c:522`) |
| `fsnotify_ops.handle_inode_event` | **No** | use `.handle_event` with 8-arg callback (`B:fs/notify/fsnotify.c:244`) |
| `fsnotify_group_lock()` | **No** | `mutex_lock(&group->mark_mutex)` |
| `vma_lookup()` | **No** | `find_vma(mm, addr)` |
| `mas_find()` | **No** | `for (vma = mm->mmap; vma; vma = vma->vm_next)` |
| `VMA_PAD_START()` | **No** | `vma->vm_end` |
| `mmap_read_lock_killable()` | **No** | `down_read_killable(&mm->mmap_sem)` |
| `for_each_vma()` | **No** | `for (vma = mm->mmap; vma; vma = vma->vm_next)` |
| `user_write_access_begin()` | **No** | `access_ok(VERIFY_WRITE, ...)` + `__put_user` (4.19 `fs/readdir.c`) |
| `getdents_callback.prev_reclen` | **No** | 4.19 has `struct linux_dirent __user *previous` and `int count` |
| `filldir` return `bool` | **No** | 4.19 returns `int` (`-EINVAL`/`-EFAULT`); return `0` to skip |
| `input_event_dispose()` | **No** | keep existing `input_handle_event()` hook |
| `strscpy` | **Yes** | `B:include/linux/string.h:34` |
| `static_key_disable(&key.key)` style | N/A | 2.4.0 uses `static_branch_disable(&key)` / `static_key_enabled(&key)`, all present |
| `DEFINE_SRCU` / `DEFINE_STATIC_SRCU` | **Yes** | `include/linux/srcutree.h:147–148` |
| `kvmalloc` / `kvfree` | **Yes** | `include/linux/mm.h` |
| `ida_alloc_min` / `ida_alloc_range` | **Yes** | `include/linux/idr.h` |
| `huge_decode_dev` / `old_decode_dev` | **Yes** | `include/linux/kdev_t.h:58` |
| `GFP_KERNEL_ACCOUNT` | **Yes** | `include/linux/gfp.h:298` |
| `struct fuse_inode.inode/nodeid` | **Yes** | `fs/fuse/fuse_i.h:63–69` |
| `security_context_to_sid_force` | **Yes** | `security/selinux/ss/services.c` (4.19) |
| `security_compute_av_user` | **Yes** | `security/selinux/ss/services.c:1163` |
| `security_validate_transition` | **Yes** | `security/selinux/ss/services.c` |
| `selinux_superblock()` | **No** | `(struct superblock_security_struct *)inode->i_sb->s_security` |
| `selinux_policy_commit()` | **No** | inline seqno logic in `security_load_policy()` |

---

## 6. Prioritized porting plan

### Phase A — minimal bootable port (target: boots, no SUSFS VFS hooks)

**A1. Kernel config prerequisites**
* Add the 2.4.0 `KSU_SUSFS*` Kconfig block (from `P:KernelSU/10...` patch lines 37–150) to `KernelSU/kernel/Kconfig` or the kernel's `Kconfig`; `CONFIG_KSU_SUSFS` must be `bool` (not tristate).
* `depends on KSU && THREAD_INFO_IN_TASK`; arm64 4.19 selects `THREAD_INFO_IN_TASK` (`arch/arm64/Kconfig:182`), so fine.
* Start with:
  ```text
  CONFIG_KSU_SUSFS=y
  CONFIG_KSU_SUSFS_ENABLE_LOG=y
  CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS=y
  CONFIG_KSU_SUSFS_SPOOF_UNAME=y
  CONFIG_KSU_SUSFS_PRIORITY_UID_FILTER=y
  CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG=y
  # all others = n
  ```

**A2. Files to change**
1. `fs/Makefile`: add `obj-$(CONFIG_KSU_SUSFS) += susfs.o`.
2. Add `fs/susfs.c` (2.4.0), `include/linux/susfs.h`, `include/linux/susfs_def.h`.
3. `fs/susfs.c`: fix fsnotify callback, `susfs_is_current_ksu_domain()` wrapper, and confirm the large 2.4.0 structures stay heap-allocated (`kvmalloc`/`kzalloc`, not stack copies).
4. `kernel/kallsyms.c`: symbol hiding (`P:L2271–L2321`).
5. `kernel/sys.c`: `newuname` spoof (`P:L2352–L2360` only), `setpriority`/`getpriority` filters (`P:L2370–L2410`).
6. `block/ioprio.c`: priority filter (`P:L2–L48`).
7. `fs/proc/cmdline.c`: cmdline spoof (instead of `fs/proc/bootconfig.c`).
8. `KernelSU/kernel/feature/susfs_cmd.c`: rewrite the command bridge for 2.4.0's `void __user **` ABI and add the new command IDs (see §3.3).
9. `security/selinux/ss/status.c` maybe optional; if `CONFIG_KSU_SUSFS` compiles `susfs.c` without SELinux hooks, no need.

**A3. Do not touch**
`fs/namei.c`, `fs/namespace.c`, `fs/exec.c`, `fs/open.c`, `fs/read_write.c`, `fs/stat.c`, `fs/statfs.c`, `fs/readdir.c`, `fs/proc/{base,fd,task_mmu}.c`, `fs/notify/fdinfo.c`, `security/selinux/*`, `kernel/reboot.c`, `drivers/input/input.c`, `mm/memory.c`.

**A4. Acceptance criteria**
* Kernel compiles with `CONFIG_KSU_SUSFS=y` and all subfeatures off.
* Boots to Android UI; no crash in `susfs_init` (watch `dmesg | grep susfs`).
* `ksu_susfs show version` / `show enabled_features` works (if userspace tool matches 2.4.0 command set).
* `uname -r` spoof works if configured.
* `/proc/kallsyms` no longer lists `ksu_`/`susfs_` symbols.
* No regression in normal mount/stat/getdents paths because none are patched yet.

**A5. Why not enable SUS_PATH in Phase A?**
It is the highest-value feature but requires the whole `fs/namei.c` rewrite. Keeping it off gives a clean boot baseline and a bisectable point. Once Phase A boots, enable SUS_PATH as the first Phase B feature.

### Phase B — full parity, ordered

1. **B1 `SUS_PATH`** (`fs/namei.c`, `fs/readdir.c`, `fs/proc/base.c` for readlink if open redirect later):
   * Reuse R's 1.5.11 namei/readdir code; adapt 2.4.0's `susfs_is_inode_sus_path` signature. 2.4.0's function may take `(struct inode *)` or `(struct mnt_idmap *, struct inode *)` depending on the 6.1 branch; 4.19 uses `(struct inode *)` (R's `susfs.c:438` has the 4.19 version).
   * Test: hide a test file under `/data/local/tmp`, verify `ls`, `stat`, `open`, `getdents` all hide it from an app UID; verify `su`/KSU domain still sees it.
2. **B2 `SUS_KSTAT`** (`fs/stat.c`, `fs/statfs.c`, `fs/notify/fdinfo.c`, `fs/proc/fd.c`, `fs/proc/task_mmu.c`, `fs/susfs.c`):
   * Add `statfs_by_dentry_wrapper`/`calculate_f_flags_wrapper`.
   * Remove all `mnt_id` assumptions; use 4.19 `kstat`.
   * Test `stat`, `fstat`, `statfs`, `/proc/self/fdinfo`, `/proc/self/maps`, `/proc/self/smaps`.
3. **B3 `SUS_MOUNT`** (`fs/namespace.c`, `fs/super.c`, `fs/proc_namespace.c`, `fs/proc/fd.c`, `fs/notify/fdinfo.c`, `fs/statfs.c`):
   * Highest risk. Port from R's 1.5.11 namespace code first; then add 2.4.0's fake-ID allocator.
   * Test mount/umount, `mountinfo`, `statfs`, `fdinfo`, `copy_mnt_ns` (zygote), and `setns`.
4. **B4 `OPEN_REDIRECT`** (`fs/namei.c`, `fs/proc/base.c`, `fs/proc/task_mmu.c`):
   * Reuse R's `do_filp_open` hook; add `susfs_open_redirect_spoof_do_sys_openat()`/`vfs_readlink`/`do_proc_readlink`.
   * Test `open`, `readlink`, `/proc/self/maps`, `/proc/self/fd`.
5. **B5 `SUS_MAP`** (`fs/proc/task_mmu.c`, `fs/proc/base.c`, `mm/memory.c`):
   * Self-contained; test `maps`, `smaps`, `smaps_rollup`, `map_files`, `mem`, `pagemap`.
6. **B6 SELinux policy hiding** (`security/selinux/*` + KernelSU `selinux_hide.c`):
   * Only after KernelSU exports `ksu_selinux_hide_running`/fake status or you decide to use `ksu_selinux_hide_enabled`.
   * Port `susfs_security_context_to_sid*()` to 4.19 `state->ss` first; then hooks; then `selinuxfs`; then `ss/status.c`.
   * Test `getenforce`, `setenforce`, `getprop ro.boot.selinux`, `dmesg` AVC, `ls -Z`.

### Phase C — cleanup / hardening (optional)

* Replace all `strscpy` with 4.19-native calls only if needed (it is present; no change).
* Keep the large `SUSFS_APP_STR_BUF_SIZE` / `SUSFS_ENABLED_FEATURES_SIZE` structures heap-allocated (2.4.0 already does this with `kvmalloc`/`kzalloc`); never introduce stack copies while adapting.
* Add `Kconfig` help text explaining `CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG` only affects `/proc/cmdline` on picasso.
* Add a build-time `#if LINUX_VERSION_CODE < KERNEL_VERSION(6,0,0)` compatibility header to avoid future 6.1-only code creeping in.

---

## 7. File-by-file risk summary and go/no-go

| Feature | Files | Risk | Go/No-Go |
|---------|-------|------|-----------|
| Core `CONFIG_KSU_SUSFS` | `fs/susfs.c`, headers, Makefile | medium (fsnotify) | **Go Phase A** after fsnotify fix |
| `SPOOF_UNAME` | `kernel/sys.c` | low | **Go Phase A** |
| `PRIORITY_UID_FILTER` | `block/ioprio.c`, `kernel/sys.c` | low | **Go Phase A** |
| `HIDE_KSU_SUSFS_SYMBOLS` | `kernel/kallsyms.c` | low | **Go Phase A** |
| `SPOOF_CMDLINE_OR_BOOTCONFIG` | `fs/proc/cmdline.c` | low | **Go Phase A** |
| `SUS_PATH` | `fs/namei.c`, `fs/readdir.c` | high | Phase B1, reuse R |
| `SUS_KSTAT` | `fs/stat.c`, `fs/statfs.c`, `fs/proc/*` | medium-high | Phase B2 |
| `SUS_MOUNT` | `fs/namespace.c`, `fs/super.c`, `fs/proc_namespace.c` | high | Phase B3 |
| `OPEN_REDIRECT` | `fs/namei.c`, `fs/proc/base.c`, `fs/proc/task_mmu.c` | high | Phase B4 |
| `SUS_MAP` | `fs/proc/task_mmu.c`, `fs/proc/base.c`, `mm/memory.c` | medium | Phase B5 |
| SELinux hiding | `security/selinux/*` + KSU | very high | Phase B6 / optional |

---

## 8. Concrete evidence appendix (commands used)

All line numbers above came from the 4.19 tree, not from memory. Representative commands:

```bash
# baseline 4.19 context
git show feature/ksu3.3.1-on-73fe:fs/namei.c | grep -n 'lookup_dcache\|__lookup_hash\|lookup_fast\|lookup_open\|do_last\|do_tmpfile\|do_o_path\|path_openat\|vfs_readlink'
git show feature/ksu3.3.1-on-73fe:fs/namespace.c | grep -n 'mnt_alloc_id\|mnt_free_id\|mnt_alloc_group_id\|alloc_vfsmnt\|__lookup_mnt\|vfs_kern_mount\|clone_mnt\|copy_mnt_ns'
git show feature/ksu3.3.1-on-73fe:fs/stat.c | grep -n 'vfs_getattr_nosec\|vfs_statx\|vfs_statx_fd'
git show feature/ksu3.3.1-on-73fe:fs/readdir.c | grep -n 'fillonedir\|filldir\|filldir64'
git show feature/ksu3.3.1-on-73fe:fs/proc/task_mmu.c | grep -n 'show_map_vma\|show_smap\|show_smaps_rollup\|pagemap_read'
git show feature/ksu3.3.1-on-73fe:security/selinux/ss/services.c | grep -n 'security_context_to_sid_core\|security_load_policy\|security_set_bools'
git show feature/ksu3.3.1-on-73fe:security/selinux/ss/status.c | grep -n 'selinux_status_update'
```

```bash
# 4.19 API presence checks
git grep -n 'strscpy' feature/ksu3.3.1-on-73fe -- include/linux/string.h lib/string.c
git grep -n 'define DEFINE_SRCU\|define DEFINE_STATIC_SRCU' feature/ksu3.3.1-on-73fe -- include/linux
git grep -n 'static_key_enable\|static_branch_enable' feature/ksu3.3.1-on-73fe -- include/linux/jump_label.h
git grep -n 'vfs_create_mount\|lookup_one_qstr_excl\|open_last_lookups\|vma_lookup\|mas_find\|VMA_PAD_START\|mmap_read_lock_killable\|for_each_vma' feature/ksu3.3.1-on-73fe -- fs include
git grep -n 'input_event_dispose\|bprm_execve\|fsnotify_group_lock' feature/ksu3.3.1-on-73fe -- fs include kernel
```

```bash
# 1.5.11 working 4.19 templates
git diff feature/ksu3.3.1-on-73fe..feature/ksu3.3.1-susfs-1.5.11-on-73fe -- fs/namei.c fs/namespace.c fs/readdir.c fs/statfs.c fs/notify/fdinfo.c fs/proc_namespace.c include/linux/mount.h
```

---

## 9. Final recommendation

* **Start from R, not from B and not from the 6.1 patch.** R already has working 4.19 `namei`/`namespace`/`readdir`/`statfs` hooks; carry them forward and extend to 2.4.0.
* **Do not attempt a full 2.4.0 feature port in one step.** The kernel-side API differences are manageable; the KernelSU-side hook API differences are the real risk. Phase A gives a bootable SUSFS core; Phase B adds features one at a time with clear bisection points.
* **Keep `CONFIG_KSU_SUSFS_SUS_PATH`/`SUS_MOUNT`/`SELINUX` off until each has its own boot + functionality test.** In particular, `SUS_MOUNT` touches `vfs_kern_mount`/`clone_mnt`, which can break boot if the fake-ID allocation is wrong; test on a recoverable device with a known-good boot image.
* **Do not port the 2.4.0 `fs/exec.c` and `fs/stat.c` KSU hook changes until the KernelSU side is updated.** They call symbols that simply do not exist in the current submodule; compiling them will fail, and stubbing them changes behavior silently.

---

*End of report.*
