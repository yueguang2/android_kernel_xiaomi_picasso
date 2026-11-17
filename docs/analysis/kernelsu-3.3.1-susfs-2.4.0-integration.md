# KernelSU 3.3.1+ / SUSFS 2.4.0 KernelSU patch integration matrix

Status: analysis only. No source in this repository was modified for this report; the only write is this file.

- Repository: `/home/yueguang/DSH/安卓/repos/picasso-kernel-ksu-susfs`
- Our KernelSU submodule: `KernelSU/`, HEAD `3b60f2af951d1dd9540421ac9b57156912ddedaf` (`KernelSU v3.3.1+`, `origin=https://github.com/rsuntk/KernelSU.git`)
- Our current integration: SUSFS **1.5.11** kernel side + uncommitted KernelSU-side 1.5.11 glue (`git -C KernelSU diff` = 233 lines over 6 tracked files, plus untracked `kernel/feature/susfs_cmd.c`)
- Upstream SUSFS: **2.4.0**, `/tmp/susfs4ksu-2.4.0`
- Upstream KernelSU patch: `kernel_patches/KernelSU/10_enable_susfs_for_ksu.patch` (150,850 bytes, 4,411 lines, 106 hunks, 32 files, +939/-2,281)
- Reference baseline discovered during the analysis: official KernelSU **v3.3.1-51** = commit `f49be695` (the commit immediately before official `main` HEAD `f4603d14`). **Every one of the 32 patch pre-image blob hashes matches official KernelSU `f49be695` exactly**, and the patch applies 100% clean to that tree. This is important: the 2.4.0 patch is *not* stale upstream; the conflicts below are caused by the rsuntk fork's tree reorganization and by our 4.19 kernel API, not by the patch being old.

## 0. Executive summary

1. **The 2.4.0 KernelSU patch cannot be applied to our tree with fuzz and fixed locally.** On a temp copy of the current working tree, `git apply --check` succeeds for **1 of 32 files** (`kernel/selinux/selinux.h`). GNU `patch -p1 --dry-run` (default fuzz 2) accepts only **3 files**: `selinux.h` (clean), `kernel/Makefile` (fuzz 2, cosmetic SUSFS version print) and `kernel/selinux/selinux.c` (fuzz 2, append-at-EOF). That is **4 of 106 hunks**; the other 102 hunks reject. **29 of 32 files reject outright** (3 paths do not exist at all: `kernel/Kbuild`, `kernel/core/init.c`, `kernel/hook/setuid_hook.h`).
2. The root cause is architectural. Our rsuntk fork is a **single-translation-unit build**: `kernel/Makefile` builds only `ksu.o`, and `kernel/ksu.c` `#include`s every `kernel/**/*.c`. Official KernelSU v3.3.1-51 builds one object per file through `kernel/Kbuild` and has `kernel/core/init.c`. Consequently the patch's build/init changes have no direct target in our tree.
3. 2.4.0 changes the **SUSFS command transport from prctl to `sys_reboot` + `SUSFS_MAGIC`**. Our current 1.5.11 glue uses `prctl(KERNEL_SU_OPTION, CMD_SUSFS_*, ...)` through a kprobe in `kernel/feature/susfs_cmd.c`. The 2.4.0 `ksu_handle_sys_reboot()` dispatch is **not** a new `KSU_IOCTL` interface; the manager ioctl-fd API (uapi 6, 25 `KSU_IOCTL_*` IDs) is unchanged.
4. 2.4.0 also removes the KernelSU syscall-hook/kprobe/LSM machinery and expects the companion **`50_add_susfs_in_gki-android14-6.1.patch`** to call KernelSU hooks directly from `fs/exec.c`, `fs/open.c`, `fs/stat.c`, `fs/read_write.c`, `drivers/input/input.c`, `kernel/sys.c` and `kernel/reboot.c`. Our 4.19 tree already has a subset of those direct call sites from SUSFS 1.5.11 (`fs/exec.c`, `fs/open.c`, `fs/stat.c`, `fs/read_write.c`, `drivers/input/input.c`), but the handler signatures differ because 4.19's VFS passes `const char __user *filename`, not `struct filename *`.
5. **Recommendation: cherry-pick equivalent changes manually; do not use `patch --fuzz`.** Phase A should port only the command transport, the setuid/SID/umount glue, the sucompat initialization/marking semantics and the `susfs` filter refresh hooks. `selinux_hide`, `adb_root`, `preinit`, `sulog` and `priority_uid_filter` should be staged separately (see §5).
6. **ABI warning:** 2.4.0 `struct st_susfs_*` layouts and the `susfs_*()` call signatures are incompatible with 1.5.11. Do not mix a 2.4.0 `ksu_susfs` binary with a 1.5.11 `fs/susfs.c`, or vice versa. See §6.

---

## 1. Per-file integration matrix

Legend for **Result**:

- **clean** — `git apply --check` passes and `patch -p1 --dry-run` reports no fuzz.
- **fuzz N** — `git apply` (strict) rejects; GNU `patch` applies with fuzz N and/or a large offset.
- **reject** — both `git apply --check` and `patch -p1 --dry-run` fail; the file would be left with `.rej` or untouched.

Method: a copy of the **current working tree** (including the uncommitted 1.5.11 glue and untracked `kernel/feature/susfs_cmd.c`) was made under `/tmp/ksu-check`; each split hunk of the upstream patch was tested with `git apply --check --verbose` and `patch -p1 --dry-run --forward --batch`. All 32 pre-image hashes match official KernelSU `f49be695` (see §0), so “ours” below means rsuntk `3b60f2af` + 1.5.11 glue.

| # | Patch path | Ours? | What 2.4.0 changes | Exact conflict against our tree | Result |
|---:|---|---|---|---|---|
| 1 | `kernel/Kbuild` | **No** | Deletes `hook/lsm_hook.o`, `hook/syscall_event_bridge.o`, `hook/syscall_hook_manager.o`, `hook/tp_marker.o` and all `hook/{arm64,x86_64,riscv64}/patch_memory.o` + `syscall_hook.o`; removes `CONFIG_KSU_X86_PATCH_SYSCALL_DISPATCHER` cflags. | Our build has no `kernel/Kbuild`; `kernel/Makefile` is a unity build (`obj-$(CONFIG_KSU) := kernelsu.o`, `kernelsu-y := ksu.o`) and `kernel/ksu.c` includes every `.c`. The object list has no equivalent. | **reject** (path missing) |
| 2 | `kernel/Kconfig` | Yes (244 lines) | Hunk 1 removes `depends on KPROBES && EXT4_FS` from `config KSU`. Hunk 2 removes `KSU_X86_PATCH_SYSCALL_DISPATCHER` and inserts an 11-option `menu "KernelSU - SUSFS"` (all `default y`: `KSU_SUSFS`, `SUS_PATH`, `SUS_MOUNT`, `SUS_KSTAT`, `SPOOF_UNAME`, `ENABLE_LOG`, `HIDE_KSU_SUSFS_SYMBOLS`, `SPOOF_CMDLINE_OR_BOOTCONFIG`, `OPEN_REDIRECT`, `SUS_MAP`, `PRIORITY_UID_FILTER`). | Our `KSU` is `depends on !CPU_BIG_ENDIAN`, `depends on SECURITY_SELINUX`, `select SECCOMP`; `KSU_X86_PATCH_SYSCALL_DISPATCHER` does not exist; our file already has a `menu "KernelSU - SUSFS"` at line 146 containing the 1.5.11 option set (includes `KSU_SUSFS_TRY_UMOUNT`, lacks `KSU_SUSFS_PRIORITY_UID_FILTER`). Hunk 2's insertion point/context is absent. | **reject** (2/2 hunks) |
| 3 | `kernel/Makefile` | Yes (155 lines) | Appends a 10-line block that prints `SUSFS_VERSION` from `$(srctree)/include/linux/susfs.h`, or “not integrated”, at the end of the file. | The block is cosmetic. Our file already ends with `# Keep a new line here!!...`; `patch` matches it with fuzz 2 at offset +116. No functional effect. | **fuzz 2** (only accepted file; optional) |
| 4 | `kernel/core/init.c` | **No** | Removes `hook/syscall_hook_manager.h`, `hook/lsm_hook.h`, `hook/syscall_hook.h` includes; removes `ksu_syscall_hook_init()` and `ksu_lsm_hook_init()`; calls `susfs_init()` early; keeps `ksu_sucompat_init()`, adds `ksu_setuid_hook_init()` ordering; removes `ksu_late_loaded`/`ksu_bundled` branches. | No such file. Our equivalent is `kernel/ksu.c:256 kernelsu_init()` (and `kernelsu_lkm_init()` at `:321`); our sequence is already different and still uses `ksu_lsm_hook_init()` at `:310`, which our setuid/ksud/selinux-hide paths need. | **reject** (path missing) |
| 5 | `kernel/feature/adb_root.c` | Yes (313 lines) | Replaces `is_exec_adbd(const char __user *)` with `is_exec_adbd(const char *)` using `susfs_starts_with("/apex/")` + `susfs_ends_with("/adbd")`; replaces `setup_ld_preload(struct pt_regs *, unsigned long *)` with `setup_ld_preload(void ***)`; replaces `do_ksu_adb_root_handle_execve()` + `ksu_adb_root_handle_execve/execveat(struct pt_regs *)` with `do_ksu_adb_root_handle_execveat(const char *, void ***)` + `ksu_adb_root_handle_execveat(const char *, void __user ***)`; adds `escape_with_root_profile()` call. | Our file already has the newer `is_exec_adbd(const char __user **)`, `setup_ld_preload(void ***)`, `do_ksu_adb_root_execve_user()`, `do_ksu_adb_root_execve_kernel()`, `ksu_adb_root_execve_user/kernel()` (lines 5, 43, 165, 192, 240-265). None of the target's `ksu_adb_root_handle_execveat` names exist; all 7 hunks fail on signature and include context. | **reject** (7/7 hunks) |
| 6 | `kernel/feature/adb_root.h` | Yes (14 lines) | Declares `long ksu_adb_root_handle_execveat(const char *filename, void __user ***envp_user_ptr);`, removes the two `struct pt_regs` prototypes. | Our header has only `ksu_adb_root_init/exit` plus `ksu_adb_root_execve_user/kernel` macros. Hunk fails. | **reject** (1/1) |
| 7 | `kernel/feature/kernel_umount.c` | Yes (138 lines) | Deletes the 6-scenario validation block from `ksu_handle_umount()`: `is_appuid(new_uid) && new_uid != WEBVIEW_ZYGOTE_UID && !is_isolated_process(new_uid)` early-return, `ksu_uid_should_umount(new_uid)` check, and `is_zygote(current_cred())` check. | Our function is `static inline int ksu_handle_umount(struct cred *new, const struct cred *old)` (line 73), not the target's `ksu_handle_umount(uid_t old_uid, uid_t new_uid)`; the same block exists at lines 92-108 but the hunk header/surrounding context differs. A manual deletion is possible, but our setuid path currently relies on these checks. | **reject** (1/1) |
| 8 | `kernel/feature/selinux_hide.c` | Yes (377 lines) | Replaces a 1,563-line upstream implementation with a 174-line “fake status page” implementation: removes kallsyms/patch_memory/LSM-hook plus `my_write_context`, `my_write_access`, `security_context_to_sid_with_policy`, `security_compute_av_user_with_policy`, `context_struct_compute_av`, etc.; keeps `initialize_fake_status`, `ksu_selinux_hide_handle_second_stage`, `ksu_selinux_hide_handle_post_fs_data`. | Our file is a different 377-line self-contained implementation (`ksu_prepare_fake_status_page`, `ksu_init_hook_sel_handle_status_ops_open`, `ksu_init_hook_transaction_ops_write`, `ksu_init_slow_avc_audit_hook`), adapted to 4.19's `selinux_kernel_status_page()` (4.19 has no `selinux_state.status_page` field). None of the target's 10 hunk contexts match; all 10 fail. | **reject** (10/10) |
| 9 | `kernel/feature/sucompat.c` | Yes (437 lines) | Rewrites pt_regs-era handlers into `struct filename` handlers: removes `ksu_handle_faccessat_sucompat/stat_sucompat/execve/execveat(..., struct pt_regs *)`; adds `ksu_handle_execveat_init()`, `ksu_handle_execveat_sucompat(int *fd, struct filename **, void *, void *, int *)`, `ksu_handle_post_execveat_sucompat(...)`, `ksu_handle_execveat()`, `ksu_handle_faccessat(int *, struct filename **, int *, int *)`, `ksu_handle_stat(int *, struct filename **, int *)`; makes `ksu_su_compat_enabled` a `DEFINE_STATIC_KEY_TRUE`; adds `susfs_set_current_proc_no_su()`, `current_chrooted()`, `ksu_adb_root_handle_execveat()` calls. | Our file has 4.19-adapted `ksu_handle_faccessat/stat(int *, const char __user **, ...)` (lines 225/235), `ksu_handle_sys_execve/execveat(const char __user **, ...)` (245/258), `ksu_handle_execveat(int *, struct filename **, ...)` (331), `ksu_execveat_hook` bool (347) and wrapper `ksu_handle_execveat_sucompat` (349). `ksu_su_compat_enabled` is `static bool` (line 10), not a static key. All 4 hunks fail. | **reject** (4/4) |
| 10 | `kernel/feature/sucompat.h` | Yes (7 lines) | Adds `extern struct static_key_true ksu_su_compat_enabled;` and declarations for `ksu_handle_faccessat`, `ksu_handle_stat`, `ksu_handle_execveat_sucompat`; removes old pt_regs prototypes. | Our header only declares `ksu_sucompat_init/exit`. Hunk fails at line 3. Manual header rewrite is trivial. | **reject** (1/1) |
| 11 | `kernel/hook/setuid_hook.c` | Yes (36 lines) | Replaces `int ksu_handle_setresuid(uid_t old_uid, uid_t new_uid)` with `int ksu_handle_setresuid(uid_t ruid, uid_t euid, uid_t suid)`, plus `handle_zygote_setresuid()`/`handle_zygote_next_setresuid()`, `susfs_zygote_sid`/`susfs_zygote_next_sid`, `susfs_set_current_proc_no_su/umounted`, `susfs_extra_works`, `disable_seccomp()` and `ksu_install_fd()`. | Our file is a 36-line LSM callback `ksu_handle_setresuid_cred(struct cred *new, const struct cred *old)` (line 1), called from `task_fix_setuid` in all four LSM variants; it does not define `ksu_handle_setresuid` and has no zygote-SID logic. Both hunks fail. | **reject** (2/2) |
| 12 | `kernel/hook/setuid_hook.h` | **No** | Declares `ksu_setuid_hook_init/exit` and removes `ksu_handle_setresuid` pt_regs prototype. | Our tree has no `hook/setuid_hook.h` (the function is local to `hook/setuid_hook.c` in the unity build). | **reject** (path missing) |
| 13 | `kernel/include/ksu.h` | Yes (31 lines) | Removes `extern bool ksu_late_loaded;` and `#ifdef MODULE extern bool ksu_bundled;`. | Our header has `static bool ksu_late_loaded = false;` (line 7) used by `kernel/supercall/dispatch.c:26` and set by `kernel/ksu.c:339`; we also have no `ksu_bundled`. Removing `ksu_late_loaded` breaks our LKM late-load flow. Hunk fails. | **reject** (1/1) |
| 14 | `kernel/manager/pkg_observer.c` | Yes (110 lines) | Adds `#include <linux/susfs.h>` and, inside `ksu_track_throne_tw_func()`, calls `susfs_parse_priority_app_strings()` (`CONFIG_KSU_SUSFS_PRIORITY_UID_FILTER`) and `susfs_parse_sus_path_app_strings()` (`CONFIG_KSU_SUSFS_SUS_PATH`). | Our `pkg_observer.c` has no `ksu_track_throne_tw_func`; it uses `ksu_rename_observer_slow()` (line 31) and direct `track_throne(false)` (lines 55/104). Both hunks fail. Port the two calls into our `ksu_rename_observer_slow()`/track_throne path. | **reject** (2/2) |
| 15 | `kernel/policy/allowlist.c` | Yes (557 lines) | Deletes the manager-app check from `ksu_uid_should_umount()`: `if (ksu_is_manager_appid_valid() && ksu_get_manager_appid() == uid % PER_USER_RANGE) return false;`. | Our `ksu_uid_should_umount()` at line 249 contains that exact block at lines 253-257, but the hunk header and following context differ. Manual deletion is easy but not required if our setuid hook keeps its `is_uid_manager()` check. | **reject** (1/1) |
| 16 | `kernel/policy/app_profile.c` | Yes (187 lines) | Makes `disable_seccomp()` non-static; removes `cred->euid.val == 0` early-exit; adds `CAP_DAC_READ_SEARCH` to `cap_effective`; changes `#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 11, 0)` to 6.6; calls `disable_seccomp()` only when `TIF_SECCOMP`; removes `ksu_set_task_tracepoint_flag()` loop; changes `escape_to_root_for_init()` from `void` to `int`; removes `ksu_app_profile_init()`. | Our file has `static void disable_seccomp()` (line 59), no `escape_to_root_for_init` (internal `escape_to_root(bool)` at line 74), `escape_to_root_forced()` (line 180), `ksu_app_profile_init()` (line 187), different seccomp backport guards. All 9 hunks fail. Only the `escape_to_root_for_init()` wrapper is materially needed by a 2.4.0-style sucompat; add it manually. | **reject** (9/9) |
| 17 | `kernel/policy/app_profile.h` | Yes (23 lines) | Replaces `void escape_to_root_for_init(void);` and `void __init ksu_app_profile_init(void);` with `int escape_to_root_for_init(void);`. | Our header has `escape_with_root_profile`, `escape_to_root_forced`, `ksu_app_profile_init`; no `escape_to_root_for_init`. Hunk fails. Add the declaration manually. | **reject** (1/1) |
| 18 | `kernel/policy/feature.h` | Yes (35 lines) | Only removes `// IWYU pragma: keep` from `#include "uapi/feature.h"`. | Our header has the same include with a comment; context at line 2 differs. No functional value. | **reject** (1/1) |
| 19 | `kernel/preinit/preinit.c` | Yes (283 lines) | Replaces syscall-table-based `ksu_preinit_handle_execve()` with `ksu_setup_preinit_file(struct filename **filename_ptr, struct user_arg_ptr *argv_user)` + `ksu_preinit_handle_post_execveat(..., int *preinit_fd)` and removes `ksu_syscall_table[orig_nr](regs)` use. | Our preinit is a 4.19-specific implementation: `is_selinux_setup(const char __user *const __user *)`, `ksu_stage_preinit_binary(void ***u_argv, void ***u_envp)`, `ksu_preinit_handle_kexecve()`, `ksu_preinit_handle_uexecve()`, `ksu_preinit_kernel_execve_hook/user_execve_hook`. No target functions exist; both hunks fail. | **reject** (2/2) |
| 20 | `kernel/preinit/preinit.h` | Yes (44 lines) | Adds `struct filename`/`struct user_arg_ptr`-based prototypes and `KSU_NO_PREINIT` stub messages. | Our header defines `DEFINE_STATIC_KEY_FALSE(ksu_preinit_key)` and `ksu_preinit_kernel/user_execve_hook`; no target prototypes. Both hunks fail. | **reject** (2/2) |
| 21 | `kernel/runtime/boot_event.c` | Yes (60 lines) | Replaces `ksu_stop_input_hook_runtime()` with `extern struct static_key_true ksu_is_input_hook_enabled;` and `static_branch_disable()`/`static_key_enabled()` checks in `on_post_fs_data()`; adds `ksu_selinux_hide_handle_post_fs_data()` call. | Our `on_post_fs_data()` calls `ksu_stop_input_hook_runtime()` (line 17) and has no `ksu_is_input_hook_enabled` static key; our input hook is an `input_handler`, not a static key. Both hunks fail. | **reject** (2/2) |
| 22 | `kernel/runtime/ksud.h` | Yes (14 lines) | Adds `MAX_ARG_STRINGS`, `struct user_arg_ptr`, `get_user_arg_ptr()` and `ksu_handle_execveat_ksud(int *fd, struct filename **, struct user_arg_ptr *, struct user_arg_ptr *, int *flags)` to the header. | Our `struct user_arg_ptr` is already in `kernel/kernel_compat.h:306`; our `ksud.h` has only `ksu_stop_input_hook_runtime()`, `ksu_vfs_read_hook`, `ksu_input_hook`. Adding the target header verbatim would redefine `struct user_arg_ptr` in the unity build. Hunk fails; port only `get_user_arg_ptr` as a uniquely named helper. | **reject** (1/1) |
| 23 | `kernel/runtime/ksud_integration.c` | Yes (703 lines) | Removes kprobe/input/syscall hooks: `#include <linux/kprobes.h>`, `hook/syscall_hook.h`, `hook/syscall_event_bridge.h`; removes `struct user_arg_ptr`/`get_user_arg_ptr`, `ksu_handle_execveat_ksud(const char *)`, `ksu_execve_hook_ksud`, `ksu_execveat_hook_ksud`, `orig_sys_read`, `orig_sys_fstat`, `input_event_kp`, `stop_init_rc_hook`, `ksu_stop_input_hook_runtime`. Adds `DEFINE_STATIC_KEY_TRUE(ksu_is_init_rc_hook_enabled/ksu_is_input_hook_enabled/is_init_second_stage_not_executed/is_first_zygote)`, `ksu_handle_execveat_ksud(int *fd, struct filename **, struct user_arg_ptr *, struct user_arg_ptr *, int *)`, `ksu_handle_sys_read(unsigned int fd, char __user **, size_t *)`, `ksu_handle_vfs_fstat(int fd, loff_t *)`, and rewrites `ksu_handle_input_handle_event`. | Our file uses a modern `input_handler` (`vol_detector_event/connect`, lines 529-646), an `f_op` proxy for init.rc (`read_proxy`/`read_iter_proxy`), a watchdog kthread, and explicit stubs `ksu_handle_input_handle_event` (line 29) / `ksu_handle_vfs_read` (line 34) for the 1.5.11 kernel hooks. It has no `ksu_handle_execveat_ksud`, no static keys. All 11 hunks fail. | **reject** (11/11) |
| 24 | `kernel/selinux/rules.c` | Yes (738 lines) | Removes `pol->latest_granting = old_pol->latest_granting + 1` and `ksu_set_original_types()/ksu_advance_original_types()` calls; changes `reset_avc_cache(u32 seqno)` to `reset_avc_cache()` using 0; calls `susfs_set_batch_sid()` after `reset_avc_cache()`. | Our `rules.c` has `get_latest_granting_inc()` compat helpers and `pol->latest_granting` logic (lines 195/591), and does not have `ksu_set_original_types`/`ksu_advance_original_types`. Target hunk contexts do not match; 4 hunks fail. Only `susfs_set_batch_sid()` addition is worth porting manually. | **reject** (4/4) |
| 25 | `kernel/selinux/selinux.c` | Yes (243 lines) | Appends 93 lines: `#define KERNEL_INIT_DOMAIN/ZYGOTE_DOMAIN/ZYGOTE_NEXT_DOMAIN/PRIV_APP_DOMAIN`, `u32 susfs_ksu_sid/init_sid/zygote_sid/zygote_next_sid/priv_app_sid`, `susfs_set_sid()`, `susfs_is_sid_equal()`, `susfs_get_sid_from_name()`, `susfs_get_current_sid()`, `susfs_is_current_zygote/zygote_next/ksu/init_domain()`, `susfs_set_batch_sid()`. | Our file ends with the same `escape_to_root_for_adb_root()` function; the append hunk succeeds with **fuzz 2** (offset +7) even though `git apply` rejects due to tab/space context. Content compiles on 4.19 because `current_sid()` and `selinux_cred()` are available (`security/selinux/include/objsec.h:168`, `kernel_compat.h:442`). This is a real, usable addition (whitespace cleanup needed). | **fuzz 2** (1/1; usable) |
| 26 | `kernel/selinux/selinux.h` | Yes (45 lines) | Appends declarations: `susfs_is_sid_equal`, `susfs_get_sid_from_name`, `susfs_get_current_sid`, `susfs_set_batch_sid`, `susfs_is_current_zygote_domain`, `susfs_is_current_zygote_next_domain`, `susfs_is_current_ksu_domain`, `susfs_is_current_init_domain`. | Our header ends with `extern u32 ksu_file_sid;`; the hunk applies cleanly at line 42 with offset -26. | **clean** (1/1) |
| 27 | `kernel/selinux/sepolicy.c` | Yes (1,320 lines) | Adds `#ifdef CONFIG_KSU_SUSFS extern int susfs_add_sus_avd_allowed_by_type(u32 s_type, u32 t_type, u16 type, u32 orig_avd_allowed); extern int susfs_add_sus_type(const char *secctx, u32 type_value);`; calls them inside `add_rule_raw()` and `add_type()`. | Our `add_rule_raw()` starts at line 280, `add_type()` at 729; target hunk context at 311/728 is close but our function bodies/comments differ; 3 hunks fail. Manual insertion is straightforward and required if the 2.4.0 `fs/susfs.c` policy tables are used. | **reject** (3/3) |
| 28 | `kernel/sulog/event.c` | Yes (264 lines) | Replaces the local `struct user_arg_ptr`/`ksu_sulog_get_user_arg_ptr` with `runtime/ksud.h`; changes `ksu_sulog_capture()` to take `const char *filename` + `struct user_arg_ptr *argv`; adds `ksu_sulog_copy_filename_kernel()`; changes `ksu_sulog_capture_sucompat()` to `(const char *, struct user_arg_ptr *)`; removes `ksu_sulog_capture_root_execve()`. | Our `sulog/event.c` has a different newer API: `ksu_sulog_capture(__u16, const char *bprm_argv, size_t, gfp_t)`, `ksu_sulog_capture_grant_root()`, `ksu_sulog_emit_bprm()`; no `ksu_sulog_capture_sucompat` or `ksu_sulog_capture_root_execve`. All 10 hunks fail. | **reject** (10/10) |
| 29 | `kernel/sulog/event.h` | Yes (42 lines) | Declares `ksu_sulog_capture_sucompat(const char *filename, struct user_arg_ptr *argv_user, gfp_t)` and removes `ksu_sulog_capture_root_execve`; includes `runtime/ksud.h`. | Our header declares `ksu_sulog_emit_bprm`, `ksu_sulog_capture_grant_root`, etc.; no target prototypes. Both hunks fail. | **reject** (2/2) |
| 30 | `kernel/supercall/dispatch.c` | Yes (860 lines) | Adds `int ksu_handle_sys_reboot(int magic1, int magic2, unsigned int cmd, void __user **arg)` with the full `SUSFS_MAGIC` switch; changes `do_get_info`/`do_get_info_legacy` flags (`ksu_bundled`, `ksu_late_loaded`); skips late-load post-fs/boot; removes `ksu_mark_running_process()` from `do_set_app_profile`; rewrites `do_manage_mark` around `susfs_is_current_proc_no_su()`; calls `susfs_start_sdcard_monitor_fn()` on `EVENT_BOOT_COMPLETED`. | Our dispatch has the same 25-command ioctl table but different `do_get_info` (keeps `KSU_GET_INFO_FLAG_LATE_LOAD`), no `ksu_bundled`, `do_manage_mark` only implements `KSU_MARK_GET` with `ksu_get_task_mark()` (seccomp proxy), `do_set_app_profile` already has no mark call. `ksu_handle_sys_reboot` already exists in `supercall.c`; adding the target function here creates a duplicate symbol. 8 hunks fail; **hunk #9 (the 95-line `ksu_handle_sys_reboot` insertion) is applied by `patch` with fuzz 1** at line 476, creating an uncompilable duplicate — a trap. | **reject** (8/9 hunks fail; 1 dangerous fuzz-1 insertion) |
| 31 | `kernel/supercall/supercall.c` | Yes (118 lines) | Changes fd-install log messages to include `name`; removes `reboot_handler_pre()` kprobe and `reboot_kp`, adds `int ksu_supercall_reboot_handler(void __user **arg)` that allocates `struct ksu_install_fd_tw`, sets `tw->outp = (int __user *)(*arg)`, `tw->cb.func = ksu_install_fd_tw_func`, `task_work_add(current, &tw->cb, TWA_RESUME)`; removes `register_kprobe` from `ksu_supercalls_init`. | Our `supercall.c` has no reboot kprobe at all; it already installs fd synchronously in `ksu_handle_sys_reboot()` (line 81) with `copy_to_user`, and `ksu_supercalls_init()` only dumps commands (line 113). The log-message hunks are trivial; the kprobe-removal context is absent. Both hunks fail. | **reject** (2/2) |
| 32 | `kernel/supercall/supercall.h` | Yes (26 lines) | Adds `int ksu_supercall_reboot_handler(void __user **arg);`. | Our header has `ksu_install_fd`, `ksu_install_su_fd`, `ksu_is_su_session_fd`, `ksu_supercalls_init/exit`; no reboot handler. Hunk fails at line 26; trivial manual add. | **reject** (1/1) |

### 1.1 Patch-application statistics

| Metric | Result |
|---|---|
| Files in patch | 32 |
| Patch lines | 4,411 |
| Additions / deletions | +939 / −2,281 |
| Hunks | 106 |
| `git apply --check` fully-clean files | **1** (`kernel/selinux/selinux.h`) |
| GNU `patch --dry-run` accepted files | **3** (`selinux.h` clean; `kernel/Makefile` fuzz 2; `kernel/selinux/selinux.c` fuzz 2) |
| Hunks applied by GNU `patch` | **4 / 106** (3 of them with fuzz; 1 dangerous dispatch insertion) |
| Files rejecting outright | **29 / 32** |
| Missing paths | **3** (`kernel/Kbuild`, `kernel/core/init.c`, `kernel/hook/setuid_hook.h`) |

Reproduction:

```bash
cd KernelSU
git apply --check --verbose /tmp/susfs4ksu-2.4.0/kernel_patches/KernelSU/10_enable_susfs_for_ksu.patch
# or, per split file:
git apply --check --verbose kernel__feature__sucompat.c.diff
patch -p1 --dry-run --forward --batch --no-backup-if-mismatch < kernel__feature__sucompat.c.diff
```

On official KernelSU `f49be695` the same patch is **100% clean** (`git apply --check` exit 0, no fuzz), confirming that all rejects are specific to our fork/4.19 layout.

---

## 2. Supercall interface expected by SUSFS 2.4.0

### 2.1 The two transports in KernelSU v3.3.1-51

KernelSU v3.3.1-51 has two distinct “supercall” mechanisms; 2.4.0 touches only the first, and leaves the second intact.

**A. Reboot-magic transport (`sys_reboot`) — used for fd installation and SUSFS commands**

- `KSU_INSTALL_MAGIC1 = 0xDEADBEEF`, `KSU_INSTALL_MAGIC2 = 0xCAFEBABE` (`kernel/include/uapi/supercall.h`).
- In stock v3.3.1-51, `kernel/supercall/supercall.c` registers a kprobe on `__arm64_sys_reboot` / `__x64_sys_reboot` (`reboot_handler_pre`); when `magic1 == KSU_INSTALL_MAGIC1 && magic2 == KSU_INSTALL_MAGIC2`, it queues `ksu_install_fd_tw_func` on `task_work_add()` so the fd is installed after the syscall returns.
- 2.4.0 removes that kprobe entirely. `kernel/supercall/supercall.c` keeps `ksu_install_fd_tw_func` but exposes:

```c
int ksu_supercall_reboot_handler(void __user **arg)
{
    struct ksu_install_fd_tw *tw = kzalloc(sizeof(*tw), GFP_KERNEL);
    if (!tw) return 0;
    tw->outp = (int __user *)(*arg);
    tw->cb.func = ksu_install_fd_tw_func;
    if (task_work_add(current, &tw->cb, TWA_RESUME)) { kfree(tw); pr_warn(...); }
    return 0;
}
```

- `kernel/supercall/dispatch.c` gains a dispatcher that the companion kernel patch calls from `SYSCALL_DEFINE4(reboot)`:

```c
int ksu_handle_sys_reboot(int magic1, int magic2, unsigned int cmd, void __user **arg)
{
    if (magic1 != KSU_INSTALL_MAGIC1) return -EINVAL;

    if (magic2 == SUSFS_MAGIC && current_uid().val == 0) {
        switch (cmd) {
        case CMD_SUSFS_ADD_SUS_PATH:            susfs_add_sus_path(arg); return 0;
        case CMD_SUSFS_ADD_SUS_PATH_LOOP:       susfs_add_sus_path_loop(arg); return 0;
        case CMD_SUSFS_SET_SUS_PATH_UID_FILTER: susfs_set_sus_path_uid_filter(arg); return 0;
        /* ... all CMD_SUSFS_* below ... */
        default: return -EINVAL;
        }
    }
    if (magic2 == KSU_INSTALL_MAGIC2)
        return ksu_supercall_reboot_handler(arg);
    return -EINVAL;
}
```

- Companion `50_add_susfs_in_gki-android14-6.1.patch` changes `kernel/reboot.c` to honour the return value:

```c
# ifdef CONFIG_KSU_SUSFS
    ret = ksu_handle_sys_reboot(magic1, magic2, cmd, &arg);
    if (ret) goto orig_flow;
    return ret;
orig_flow:
# endif
```

Thus “handled” is signalled by returning **0** and bypassing the normal reboot validation; any non-zero return falls through to stock `sys_reboot()` (which will fail with `-EPERM`/`-EINVAL` if not a real reboot). Our current `kernel/reboot.c:308-320` calls `ksu_handle_sys_reboot(...)` but **discards the return value**; this must be changed exactly as above for 2.4.0 command dispatch to work.

**B. IOCTL-fd transport — used by the manager app and `ksud` — unchanged by 2.4.0**

- `sys_reboot(KSU_INSTALL_MAGIC1, KSU_INSTALL_MAGIC2, 0, &fd)` installs an anonymous inode fd (`ksu_install_fd()`/`ksu_install_su_fd()`).
- `anon_ksu_ioctl()` calls `ksu_supercall_handle_ioctl(filp, cmd, argp)`.
- The command table in `kernel/supercall/dispatch.c` (`ksu_ioctl_handlers[]`) maps 25 commands to `handler` + `perm_check` (+ optional `allow_su_session`). Handler registration is static: there is **no per-command registration API**; adding a command means editing this table.

### 2.2 KSU_IOCTL IDs in our tree and v3.3.1-51 (identical, uapi 6)

| IOCTL | Encoding (dir, 'K', nr, size) | Handler | Permission |
|---|---|---|---|
| `KSU_IOCTL_GRANT_ROOT` | `_IOC(_IOC_NONE,'K',1,0)` | `do_grant_root` | `allowed_for_su` |
| `KSU_IOCTL_GET_INFO` | `_IOR('K',2,struct ksu_get_info_cmd)` | `do_get_info` | `always_allow` |
| `KSU_IOCTL_GET_INFO_LEGACY` | `_IOC(_IOC_READ,'K',2,0)` | `do_get_info_legacy` | `always_allow` |
| `KSU_IOCTL_REPORT_EVENT` | `_IOC(_IOC_WRITE,'K',3,0)` | `do_report_event` | `only_root` |
| `KSU_IOCTL_SET_SEPOLICY` | `_IOC(_IOC_READ|_IOC_WRITE,'K',4,0)` | `do_set_sepolicy` | `only_root` |
| `KSU_IOCTL_CHECK_SAFEMODE` | `_IOC(_IOC_READ,'K',5,0)` | `do_check_safemode` | `always_allow` |
| `KSU_IOCTL_GET_ALLOW_LIST` (deprecated) | `_IOC(_IOC_READ|_IOC_WRITE,'K',6,0)` | `do_get_allow_list` | `manager_or_root` |
| `KSU_IOCTL_GET_DENY_LIST` (deprecated) | `_IOC(_IOC_READ|_IOC_WRITE,'K',7,0)` | `do_get_deny_list` | `manager_or_root` |
| `KSU_IOCTL_NEW_GET_ALLOW_LIST` | `_IOWR('K',6,struct ksu_new_get_allow_list_cmd)` | `do_new_get_allow_list` | `manager_or_root` |
| `KSU_IOCTL_NEW_GET_DENY_LIST` | `_IOWR('K',7,struct ksu_new_get_allow_list_cmd)` | `do_new_get_deny_list` | `manager_or_root` |
| `KSU_IOCTL_UID_GRANTED_ROOT` | `_IOC(_IOC_READ|_IOC_WRITE,'K',8,0)` | `do_uid_granted_root` | `manager_or_root` |
| `KSU_IOCTL_UID_SHOULD_UMOUNT` | `_IOC(_IOC_READ|_IOC_WRITE,'K',9,0)` | `do_uid_should_umount` | `manager_or_root` |
| `KSU_IOCTL_GET_MANAGER_APPID` | `_IOC(_IOC_READ,'K',10,0)` | `do_get_manager_appid` | `manager_or_root` |
| `KSU_IOCTL_GET_APP_PROFILE` | `_IOC(_IOC_READ|_IOC_WRITE,'K',11,0)` | `do_get_app_profile` | `only_manager` |
| `KSU_IOCTL_SET_APP_PROFILE` | `_IOC(_IOC_WRITE,'K',12,0)` | `do_set_app_profile` | `only_manager` |
| `KSU_IOCTL_GET_FEATURE` | `_IOC(_IOC_READ|_IOC_WRITE,'K',13,0)` | `do_get_feature` | `manager_or_root` |
| `KSU_IOCTL_SET_FEATURE` | `_IOC(_IOC_WRITE,'K',14,0)` | `do_set_feature` | `manager_or_root` |
| `KSU_IOCTL_GET_WRAPPER_FD` | `_IOC(_IOC_WRITE,'K',15,0)` | `do_get_wrapper_fd` | `manager_or_root` + `allow_su_session=true` |
| `KSU_IOCTL_MANAGE_MARK` | `_IOC(_IOC_READ|_IOC_WRITE,'K',16,0)` | `do_manage_mark` | `manager_or_root` |
| `KSU_IOCTL_NUKE_EXT4_SYSFS` | `_IOC(_IOC_WRITE,'K',17,0)` | `do_nuke_ext4_sysfs` | `manager_or_root` |
| `KSU_IOCTL_ADD_TRY_UMOUNT` | `_IOC(_IOC_WRITE,'K',18,0)` | `add_try_umount` | `manager_or_root` |
| `KSU_IOCTL_SET_INIT_PGRP` | `_IO('K',19)` | `do_set_init_pgrp` | `only_root` |
| `KSU_IOCTL_GET_SULOG_FD` | `_IOW('K',20,struct ksu_get_sulog_fd_cmd)` | `do_get_sulog_fd` | `only_root` |
| `KSU_IOCTL_DISABLE_ESCAPE_TO_ROOT` | `_IO('K',21)` | `do_disable_escape_to_root` | `only_root` + `allow_su_session=true` |
| `KSU_IOCTL_LOOKUP_SYMBOLS` | `_IOW('K',22,struct ksu_lookup_symbols_cmd)` | `do_lookup_symbols` | `only_root` |

2.4.0 adds **zero** `KSU_IOCTL_*` IDs and **zero** `struct ksu_*` uapi layouts. The only new protocol is `SYS_reboot` + `SUSFS_MAGIC`.

### 2.3 SUSFS 2.4.0 command dispatch table (reboot transport)

`SUSFS_MAGIC = 0xFAFAFAFA`; `cmd` is the `reboot()` third argument; `arg` is `void __user **` (so the kernel-side `susfs_*` function dereferences `*arg` to the user struct; **not** a direct `void __user *`).

| cmd | ID | Target function | Config guard | User struct (LP64 size) |
|---|---|---|---:|---|
| `CMD_SUSFS_ADD_SUS_PATH` | 0x55550 | `susfs_add_sus_path(arg)` | `CONFIG_KSU_SUSFS_SUS_PATH` | `st_susfs_sus_path` (260) |
| `CMD_SUSFS_ADD_SUS_PATH_LOOP` | 0x55553 | `susfs_add_sus_path_loop(arg)` | `CONFIG_KSU_SUSFS_SUS_PATH` | `st_susfs_sus_path` (260) |
| `CMD_SUSFS_SET_SUS_PATH_UID_FILTER` | 0x55554 | `susfs_set_sus_path_uid_filter(arg)` | `CONFIG_KSU_SUSFS_SUS_PATH` | `st_susfs_sus_path_uid_filter` (65,540) |
| `CMD_SUSFS_HIDE_SUS_MNTS_FOR_NON_SU_PROCS` | 0x55561 | `susfs_set_hide_sus_mnts_for_non_su_procs(arg)` | `CONFIG_KSU_SUSFS_SUS_MOUNT` | `st_susfs_hide_sus_mnts_for_non_su_procs` (8) |
| `CMD_SUSFS_ADD_SUS_KSTAT` | 0x55570 | `susfs_add_sus_kstat(arg)` | `CONFIG_KSU_SUSFS_SUS_KSTAT` | `st_susfs_sus_kstat` (376) |
| `CMD_SUSFS_UPDATE_SUS_KSTAT` | 0x55571 | `susfs_update_sus_kstat(arg)` | `CONFIG_KSU_SUSFS_SUS_KSTAT` | `st_susfs_sus_kstat` (376) |
| `CMD_SUSFS_ADD_SUS_KSTAT_STATICALLY` | 0x55572 | `susfs_add_sus_kstat(arg)` (same function) | `CONFIG_KSU_SUSFS_SUS_KSTAT` | `st_susfs_sus_kstat` (376) |
| `CMD_SUSFS_SET_UNAME` | 0x55590 | `susfs_set_uname(arg)` | `CONFIG_KSU_SUSFS_SPOOF_UNAME` | `st_susfs_uname` (136) |
| `CMD_SUSFS_ENABLE_LOG` | 0x555a0 | `susfs_enable_log(arg)` | `CONFIG_KSU_SUSFS_ENABLE_LOG` | `st_susfs_log` (8) |
| `CMD_SUSFS_SET_CMDLINE_OR_BOOTCONFIG` | 0x555b0 | `susfs_set_cmdline_or_bootconfig(arg)` | `CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG` | `st_susfs_spoof_cmdline_or_bootconfig` (8,196) |
| `CMD_SUSFS_ADD_OPEN_REDIRECT` | 0x555c0 | `susfs_add_open_redirect(arg)` | `CONFIG_KSU_SUSFS_OPEN_REDIRECT` | `st_susfs_open_redirect` (520) |
| `CMD_SUSFS_ADD_SUS_MAP` | 0x60020 | `susfs_add_sus_map(arg)` | `CONFIG_KSU_SUSFS_SUS_MAP` | `st_susfs_sus_map` (260) |
| `CMD_SUSFS_SET_PRIORITY_UID_FILTER` | 0x60030 | `susfs_set_priority_uid_filter(arg)` | `CONFIG_KSU_SUSFS_PRIORITY_UID_FILTER` | `st_susfs_priority_uid_filter` (65,540) |
| `CMD_SUSFS_ENABLE_AVC_LOG_SPOOFING` | 0x60010 | `susfs_set_avc_log_spoofing(arg)` | unconditional | `st_susfs_avc_log_spoofing` (8) |
| `CMD_SUSFS_SHOW_ENABLED_FEATURES` | 0x555e2 | `susfs_get_enabled_features(arg)` | unconditional | `st_susfs_enabled_features` (8,196) |
| `CMD_SUSFS_SHOW_VARIANT` | 0x555e3 | `susfs_show_variant(arg)` | unconditional | `st_susfs_variant` (20) |
| `CMD_SUSFS_SHOW_VERSION` | 0x555e1 | `susfs_show_version(arg)` | unconditional | `st_susfs_version` (20) |

Deprecated and *not dispatched* by 2.4.0: `CMD_SUSFS_SET_ANDROID_DATA_ROOT_PATH` (0x55551), `CMD_SUSFS_SET_SDCARD_ROOT_PATH` (0x55552), `CMD_SUSFS_ADD_SUS_MOUNT` (0x55560), `CMD_SUSFS_UMOUNT_FOR_ZYGOTE_ISO_SERVICE` (0x55562), `CMD_SUSFS_ADD_TRY_UMOUNT` (0x55580), `CMD_SUSFS_SHOW_SUS_SU_WORKING_MODE` (0x555e4), `CMD_SUSFS_IS_SUS_SU_READY` (0x555f0), `CMD_SUSFS_SUS_SU` (0x60000).

Representative layouts (LP64; kernel `bool`=1, `unsigned long`/`long`=8, natural alignment):

```c
/* 2.4.0 */
struct st_susfs_sus_path {                 /* 260 bytes */
    char target_pathname[256];
    int  err;
};
struct st_susfs_sus_kstat {                /* 376 bytes */
    bool is_statically;
    unsigned long target_ino;
    char target_pathname[256];
    unsigned long spoofed_ino, spoofed_dev;
    unsigned int spoofed_nlink;
    long long spoofed_size;
    long spoofed_atime_tv_sec;  unsigned long spoofed_atime_tv_nsec;
    long spoofed_mtime_tv_sec;  unsigned long spoofed_mtime_tv_nsec;
    long spoofed_ctime_tv_sec;  unsigned long spoofed_ctime_tv_nsec;
    long long spoofed_blocks;
    long spoofed_blksize;
    int flags, err;
};
struct st_susfs_uname { char release[65]; char version[65]; int err; }; /* 136 */
struct st_susfs_open_redirect { char target[256]; char redirected[256]; int uid_scheme, err; }; /* 520 */
struct st_susfs_enabled_features { char enabled_features[8192]; int err; }; /* 8196 */
```

### 2.4 Compare with our `dispatch.c` / `supercall.c`

| Aspect | 2.4.0 expectation | Our current implementation | Needed shim? |
|---|---|---|---|
| Reboot fd install | `ksu_supercall_reboot_handler(void __user **arg)` with `task_work_add(current, TWA_RESUME)` | `ksu_handle_sys_reboot()` in `kernel/supercall/supercall.c:81` installs fd synchronously with `get_unused_fd_flags()` + `copy_to_user()`; no task_work | No functional shim needed; keep ours. Optionally add target wrapper for source compatibility. |
| SUSFS command dispatch | `ksu_handle_sys_reboot()` in `dispatch.c`; `SUSFS_MAGIC` branch | No `SUSFS_MAGIC`, no `susfs_*` calls; only `KSU_INSTALL_MAGIC2` fd install and `toolkit_handle_sys_reboot()` downstream commands | **Yes — this is the core Phase A shim.** |
| Return protocol | non-zero = unhandled → `orig_flow`; 0 = handled, skip real reboot | `ksu_handle_sys_reboot()` returns 0 for every `magic1 == KSU_INSTALL_MAGIC1`, including unknown `magic2`; `kernel/reboot.c` ignores result | **Must fix both function and `kernel/reboot.c`.** |
| ioctl table | 25 entries, unchanged | 25 entries, identical IDs/handlers/perms | No |
| `ksu_supercalls_init` | only `ksu_supercall_dump_commands()` | same (our line 113) | No |
| `do_manage_mark` | `KSU_MARK_GET` returns `!susfs_is_current_proc_no_su()`; MARK/UNMARK/REFRESH no-op | `KSU_MARK_GET` returns `ksu_get_task_mark(pid)` = `!task->seccomp.mode`; other ops return -EINVAL | Semantic shim optional (see §6) |
| `do_get_info` | removes `KSU_GET_INFO_FLAG_LATE_LOAD`/`BUNDLED`; PREINIT if `ksu_preinit_enabled_param()` unconditionally | retains LATE_LOAD and PREINIT; has `ksu_late_loaded` | Keep ours; do not port |
| permission model | SUSFS branch: **`current_uid().val == 0` only**; ioctl table: per-handler `perm_check` + `allow_su_session` | exactly the same ioctl permission model | No new shim |

Conclusion for §2: a compatibility shim is **required**, but it is small and localized to `ksu_handle_sys_reboot()` + `kernel/reboot.c`. The ioctl/UAPI layer itself is already compatible (same uapi 6 and same 25 IDs), so no new ioctl ID or struct is needed.

---

## 3. Sucompat changes: 2.4.0 vs our 1.5.11 integration

### 3.1 What 2.4.0 removes (kprobes/manual hooks)

The patch deliberately deletes all KernelSU-side hook machinery; the companion `50_add_susfs_in_gki-android14-6.1.patch` replaces it with direct function calls in kernel source. Removed exactly:

- `kernel/Kbuild`: `hook/lsm_hook.o`, `hook/syscall_event_bridge.o`, `hook/syscall_hook_manager.o`, `hook/tp_marker.o`, `hook/{arm64,x86_64,riscv64}/patch_memory.o`, `hook/{arm64,x86_64,riscv64}/syscall_hook.o`.
- `kernel/core/init.c`: includes `hook/syscall_hook_manager.h`, `hook/lsm_hook.h`, `hook/syscall_hook.h`; calls `ksu_syscall_hook_init()` and `ksu_lsm_hook_init()`; LKM late-load branches; `ksu_bundled`/`ksu_late_loaded`.
- `kernel/runtime/ksud_integration.c`: `register_kprobe(&input_event_kp)`, `ksu_syscall_table_hook(__NR_read, ksu_sys_read, &orig_sys_read)`, `ksu_syscall_table_hook(__NR_fstat, ksu_sys_fstat, &orig_sys_fstat)`, `orig_sys_read`/`orig_sys_fstat`, `ksu_execve_hook_ksud`/`ksu_execveat_hook_ksud`, `stop_init_rc_hook`, `ksu_stop_input_hook_runtime`.
- `kernel/feature/sucompat.c`: `#include "hook/syscall_hook.h"`, all `ksu_syscall_table[orig_nr](regs)` calls, all `struct pt_regs` handler prototypes, `ksu_handle_faccessat_sucompat/stat_sucompat/execve_sucompat/execveat_sucompat`.
- `kernel/hook/setuid_hook.c`: `#include "hook/tp_marker.h"`, `ksu_set_task_tracepoint_flag()` loop; setuid is now a direct `ksu_handle_setresuid(ruid, euid, suid)` call from `__sys_setresuid` in `kernel/sys.c`.
- `kernel/preinit/preinit.c`: `#include "hook/syscall_hook.h"` and `ksu_syscall_table[orig_nr](regs)`.

### 3.2 Exact call sites 2.4.0 expects (from the 50 patch)

| Kernel file / function | Exact 2.4.0 call site | Extern symbol(s) |
|---|---|---|
| `fs/exec.c: do_execveat_common()` before `bprm_execve()` | `if (susfs_is_current_proc_no_su()) goto orig_flow;` then if `ksu_su_compat_enabled`: `ksu_handle_execveat(&fd, &filename, &argv, &envp, &flags)` when `susfs_is_sdcard_android_data_not_decrypted`, else `ksu_handle_execveat_sucompat(...)`; for PID 1: `preinit_fd = ksu_setup_preinit_file(&filename, &argv)` | `ksu_handle_execveat`, `ksu_handle_execveat_sucompat`, `ksu_setup_preinit_file`, `susfs_is_current_proc_no_su`, `susfs_is_sdcard_android_data_not_decrypted` |
| `fs/exec.c: do_execveat_common()` after `bprm_execve()` | `ksu_handle_post_execveat_sucompat(&fd, &filename, &argv, &envp, &flags, &retval)`; `ksu_preinit_handle_post_execveat(..., &preinit_fd)` | `ksu_handle_post_execveat_sucompat`, `ksu_preinit_handle_post_execveat` |
| `fs/open.c: do_faccessat()` after `getname_flags()` | `if (susfs_is_current_proc_no_su()) goto orig_flow; if (ksu_su_compat_enabled && __ksu_is_allow_uid_for_current(current_uid().val)) ksu_handle_faccessat(&dfd, &fname, &mode, NULL);` | `ksu_handle_faccessat(int *, struct filename **, int *, int *)`, `__ksu_is_allow_uid_for_current` |
| `fs/stat.c: vfs_statx()` before lookup | `if (susfs_is_current_proc_no_su()) goto orig_flow; if (ksu_su_compat_enabled && __ksu_is_allow_uid_for_current(...)) ksu_handle_stat(&dfd, &filename, &flags);` | `ksu_handle_stat(int *, struct filename **, int *)` |
| `fs/stat.c: vfs_fstat()` after `vfs_getattr()` | `if (static_branch_unlikely(&ksu_is_init_rc_hook_enabled)) ksu_handle_vfs_fstat(fd, &stat->size);` | `ksu_handle_vfs_fstat(int, loff_t *)`, `ksu_is_init_rc_hook_enabled` |
| `fs/read_write.c: SYSCALL_DEFINE3(read)` | `if (static_branch_unlikely(&ksu_is_init_rc_hook_enabled)) ksu_handle_sys_read(fd);` | `ksu_handle_sys_read(unsigned int fd)` (50 patch declares 1 arg; 2.4.0 KSU defines 3 args `fd, char __user **, size_t *`; the callee only uses `fd`, but this is an ABI sloppiness to fix in our backport) |
| `drivers/input/input.c: input_handle_event()` | `if (static_branch_unlikely(&ksu_is_input_hook_enabled)) ksu_handle_input_handle_event(&type, &code, &value);` | `ksu_handle_input_handle_event(unsigned int *, unsigned int *, int *)`, `ksu_is_input_hook_enabled` |
| `kernel/sys.c: __sys_setresuid()` | `(void)ksu_handle_setresuid(ruid, euid, suid);` | `ksu_handle_setresuid(uid_t, uid_t, uid_t)` |
| `kernel/reboot.c: SYSCALL_DEFINE4(reboot)` | `ret = ksu_handle_sys_reboot(magic1, magic2, cmd, &arg); if (ret) goto orig_flow; return ret;` | `ksu_handle_sys_reboot(int, int, unsigned int, void __user **)` |

### 3.3 What our 1.5.11 integration currently does

| Kernel file | Our current call site | Our handler signature / behavior |
|---|---|---|
| `fs/exec.c:1960-1974` | `if (ksu_execveat_hook) ksu_handle_execveat(&fd,&filename,&argv,&envp,&flags); else ksu_handle_execveat_sucompat(...)`; under `CONFIG_KSU_SUSFS_SUS_SU`: skip if `susfs_is_current_proc_umounted()`, else if `susfs_is_sus_su_hooks_enabled && __ksu_is_allow_uid` call sucompat. | `ksu_handle_execveat(int *, struct filename **, void *, void *, int *)` exists (sucompat.c:331); `ksu_handle_execveat_sucompat` wrapper exists (line 349, added by `kernelsu-3.3.1-stale-hooks-compat.patch`); **no post-exec hook**, and su fd is installed inside the common escape path before exec succeeds. |
| `fs/open.c:354-388` | `if (CONFIG_KSU) ksu_handle_faccessat(&dfd,&filename,&mode,NULL)`; under SUS_SU also conditional. `filename` is `const char __user *` (4.19 signature). | `ksu_handle_faccessat(int *, const char __user **, int *, int *)` (sucompat.c:225); reads user string via `ksu_sucompat_user_common()`; rewrites `su` → `sh` (and for exec `su` → `ksud`). |
| `fs/stat.c:169-213` | `if (CONFIG_KSU) ksu_handle_stat(&dfd,&filename,&flags)`; `filename` is `const char __user *` (4.19 `vfs_statx` signature). | `ksu_handle_stat(int *, const char __user **, int *)` (sucompat.c:235); no `vfs_fstat` hook; module-rc stat size is handled by `ksu_handle_newfstat_ret()`/`ksu_handle_fstat64_ret()` from syscall hooks. |
| `fs/read_write.c:437-450` | `if (ksu_vfs_read_hook) ksu_handle_vfs_read(&file,&buf,&count,&pos);` | `ksu_handle_vfs_read(struct file **, char __user **, size_t *, loff_t **)` is a **stub returning 0** (ksud_integration.c:34). |
| `drivers/input/input.c:381-393` | `if (ksu_input_hook) ksu_handle_input_handle_event(&type,&code,&value);` | `ksu_handle_input_handle_event(...)` is a **stub returning 0** (ksud_integration.c:29); real safe-mode logic is in the separate `input_handler` `vol_detector_event()` (lines 529-646). |
| `kernel/sys.c` | Only `CONFIG_KSU_SUSFS_SPOOF_UNAME` blocks; setuid is not patched. | Our setuid works through `task_fix_setuid` LSM: `ksu_task_fix_setuid()` → `ksu_handle_setresuid_cred(new, old)` (setuid_hook.c:1), no `ksu_handle_setresuid` symbol. |
| `kernel/reboot.c:308-320` | `ksu_handle_sys_reboot(magic1,magic2,cmd,&arg);` return ignored. | Our function returns `0` for all `KSU_INSTALL_MAGIC1` magic2 values; SUSFS commands not handled. |

### 3.4 Exact signature / symbol gaps

| 2.4.0 symbol | 2.4.0 signature | Our nearest existing symbol | Gap |
|---|---|---|---|
| `ksu_handle_execveat` | `int (int *fd, struct filename **, void *argv, void *envp, int *flags)` | same name/signature, sucompat.c:331 | Semantics differ: ours handles `su` and calls preinit/adb hooks; target expects `ksu_handle_execveat_init` first and `ksu_handle_execveat_sucompat` core. |
| `ksu_handle_execveat_sucompat` | same as above | same signature wrapper at sucompat.c:349 | Exists (1.5.11 shim); replace body with target logic. |
| `ksu_handle_post_execveat_sucompat` | `int (..., int *retval)` | **missing** | Add; install `ksu_install_su_fd()` only when `*retval >= 0`. |
| `ksu_handle_execveat_init` | `int (struct filename *, struct user_arg_ptr *, struct user_arg_ptr *)` | **missing** | Add; mark `susfs_set_current_proc_no_su()` for non-`/app_process`, `/adbd`, `/stub_zygote`; call adb root for those; handle `/data/adb/ksud` for init. |
| `ksu_handle_faccessat` | `int (int *dfd, struct filename **filename, int *mode, int *__unused_flags)` | `int (int *, const char __user **, int *, int *)` at sucompat.c:225 | Signature incompatible with 4.19 VFS; keep ours, port target `current_chrooted()` + `su→sh` semantics. |
| `ksu_handle_stat` | `int (int *dfd, struct filename **filename, int *flags)` | `int (int *, const char __user **, int *)` at sucompat.c:235 | Same 4.19 incompatibility; keep ours. |
| `ksu_su_compat_enabled` | `struct static_key_true` | `static bool` at sucompat.c:10 | fs/exec.c 50 patch references the static key; either change ours or keep our `ksu_execveat_hook` bool path. |
| `ksu_is_init_rc_hook_enabled` | `struct static_key_true` | **missing**; our `ksu_vfs_read_hook` bool | Add a static key or adapt 50 patch to our bool. |
| `ksu_is_input_hook_enabled` | `struct static_key_true` | **missing**; our `ksu_input_hook` bool | Add a static key or adapt `drivers/input/input.c`. |
| `ksu_handle_vfs_fstat` | `void (int fd, loff_t *kstat_size_ptr)` | **missing**; our `ksu_handle_newfstat_ret` syscall hook | Add and patch `vfs_fstat`, or keep syscall hook. |
| `ksu_handle_sys_read` | 50 patch: `void (unsigned int fd)`; 2.4.0 KSU: `void (unsigned int fd, char __user **, size_t *)` | `ksu_handle_vfs_read(struct file **, ...)` stub | Prefer keep our `vfs_read` hook and implement module rc proxy there; fix the 50 patch's 1-arg/3-arg mismatch in any backport. |
| `ksu_setup_preinit_file` | `int (struct filename **, struct user_arg_ptr *)` | **missing**; ours uses `ksu_stage_preinit_binary()` | Do not port; our 4.19 preinit design is safer. |
| `ksu_preinit_handle_post_execveat` | `int (..., int *retval, int *preinit_fd)` | **missing** | Do not port; keep our preinit. |
| `susfs_set_current_proc_no_su` | static inline in `susfs_def.h` (`TIF_PROC_NO_SU=34`) | **missing in 1.5.11** (`susfs_def.h` has only `TIF_PROC_UMOUNTED=33`) | Required for 2.4.0 behavior; depends on 2.4.0 `susfs_def.h`, not in 1.5.11. |
| `susfs_is_sdcard_android_data_not_decrypted` | `DEFINE_STATIC_KEY_TRUE` in 2.4.0 `fs/susfs.c` | **missing in 1.5.11** | Required only if adopting target `do_execveat_common` branch; otherwise not. |
| `susfs_extra_works` | `struct work_struct` in 2.4.0 `fs/susfs.c` | missing | Required by target setuid `ksu_handle_extra_susfs_work()`. |

### 3.5 Key behavioral differences to preserve/fix

1. **fd lifetime:** 2.4.0 installs the su-session fd in `ksu_handle_post_execveat_sucompat()` only after `bprm_execve()` returns `>= 0`; our 1.5.11 code calls `ksu_install_su_fd()` inside `ksu_sucompat_kernel_common()` *before* the actual exec. On a failed exec, our fd may leak into the calling process. Port the post-exec hook if changing this path.
2. **`su` → `sh` vs `su` → `ksud`:** 2.4.0 `ksu_handle_faccessat`/`ksu_handle_stat` always rewrite `su` to `sh`; 2.4.0 `ksu_handle_execveat_sucompat` rewrites `su` to `ksud`. Our 1.5.11 `ksu_sucompat_user_common()` already behaves this way for non-exec syscalls and exec (`su -> ksud` if `/data/adb/ksud` exists, else `sh`), so no behavioral regression; but the target adds `current_chrooted()` guards we lack.
3. **`susfs_set_current_proc_no_su()`:** 2.4.0 marks all non-`app_process`/`adbd`/`stub_zygote` execs as “no su” to short-circuit SUSFS hooks; ours does not. Without this, the `susfs_is_current_proc_no_su()` early-outs in the 50 patch are false and every process pays SUSFS checks.
4. **Init/ksud:** 2.4.0 handles `init` executing `/data/adb/ksud` in `ksu_handle_execveat_init()` via `escape_to_root_for_init()`. Our tree already runs `/data/adb/ksud` as root from init.rc (and has LSM/bprm hooks for 4.19), so this branch is optional, but `escape_to_root_for_init()` should be added if target sucompat is used.

---

## 4. Kconfig: 2.4.0 vs our current Kconfig

2.4.0 patch changes only `kernel/Kconfig` (hunks 2/2 fail against our file; manual merge required). It also removes two KernelSU dependencies/options from the target baseline; those removals are **not applicable** to our fork.

| Option | 2.4.0 type/default/depends | Our current value | Action for 2.4.0 port |
|---|---|---|---|
| `KSU` | `tristate "KernelSU function support"`, `default y`, **no** `KPROBES`/`EXT4_FS` dependency | `tristate`, `depends on !CPU_BIG_ENDIAN`, `depends on SECURITY_SELINUX`, `select SECCOMP`, `default y` | No change (our deps already differ; don't import target's removal) |
| `KSU_X86_PATCH_SYSCALL_DISPATCHER` | 2.4.0 deletes it | absent | No change |
| `KSU_SUSFS` | `bool "KernelSU addon - SUSFS"`, `depends on KSU && THREAD_INFO_IN_TASK`, `default y` | same (line 147), `default y` | Keep |
| `KSU_SUSFS_SUS_PATH` | `bool`, `default y` | same, `default y` | Keep; update help text |
| `KSU_SUSFS_SUS_MOUNT` | `bool`, `default y` | same, `default y` | Keep; update help text |
| `KSU_SUSFS_SUS_KSTAT` | `bool`, `default y` | same, `default y` | Keep; update help text |
| `KSU_SUSFS_SPOOF_UNAME` | `bool`, `default y` | same, `default y` | Keep |
| `KSU_SUSFS_ENABLE_LOG` | `bool`, `default y` | same, `default y` | Keep |
| `KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS` | `bool`, `default y` | same, `default y` | Keep |
| `KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG` | `bool`, `default y` | same, `default y` | Keep |
| `KSU_SUSFS_OPEN_REDIRECT` | `bool`, `default y` | same, `default y` | Keep |
| `KSU_SUSFS_SUS_MAP` | `bool`, `default y` | same, `default y` | Keep |
| `KSU_SUSFS_PRIORITY_UID_FILTER` | `bool "Enable to allow adding user-defined uids ... getpriority/setpriority/ioprio..."`, `depends on KSU_SUSFS`, `default y` | **missing** | **Add to Kconfig** (Phase B feature; code is in 2.4.0 `fs/susfs.c` and 50 patch `kernel/sys.c`/`block/ioprio.c`) |
| `KSU_SUSFS_TRY_UMOUNT` | **removed** (2.4.0 deprecated `CMD_SUSFS_ADD_TRY_UMOUNT`) | present, `default n` | Drop or mark deprecated; no 2.4.0 `fs/susfs.c` references it |

Dependencies on KernelSU version:

- 2.4.0 assumes the official v3.3.1-51 tree where `KSU` does **not** require `KPROBES`/`EXT4_FS` and where `KSU_X86_PATCH_SYSCALL_DISPATCHER` exists. Our rsuntk fork has its own dependency set; applying the Kconfig hunks verbatim is wrong.
- `KSU_SUSFS`'s `depends on THREAD_INFO_IN_TASK` is satisfied on our arm64 4.19 (`arch/arm64/Kconfig:182 select THREAD_INFO_IN_TASK`).
- All `KSU_SUSFS_*` option names must exist if the 2.4.0 `fs/susfs.c` is compiled; missing options silently compile features out (`#ifdef CONFIG_KSU_SUSFS_*`).

---

## 5. Recommended integration strategy

### 5.1 Decision: do not apply the upstream patch with fuzz

Evidence: 4/106 hunks apply; 29/32 files reject; only one clean hunk; `kernel/supercall/dispatch.c` hunk #9 is inserted by fuzz 1 but creates a duplicate `ksu_handle_sys_reboot()` (our `supercall.c` already defines one) and references 2.4.0-only `susfs_*` functions. `patch --merge`/`.rej` cleanup would be more work and risk than a targeted port.

**Use cherry-pick-equivalent manual changes, one logical area at a time.**

### 5.2 Phase A — minimal viable 2.4.0 KernelSU glue

Assumption: Phase A also lands the 2.4.0 kernel-side ABI (`include/linux/susfs.h`, `include/linux/susfs_def.h`, `fs/susfs.c` + the 4.19 backport of relevant `50_add_susfs_in_kernel` call sites). Without that, the `susfs_*` symbols below do not exist. If the goal is only to keep 1.5.11 kernel-side, skip Phase A and keep the current prctl bridge; do not apply 2.4.0 KSU glue.

| Step | Files | Work | Why minimal |
|---:|---|---|---|
| A1 | `KernelSU/kernel/supercall/dispatch.c`, `KernelSU/kernel/supercall/supercall.c/.h`, main tree `kernel/reboot.c` | Add `SUSFS_MAGIC` branch to existing `ksu_handle_sys_reboot()` (or move it into `dispatch.c`); add `ksu_supercall_reboot_handler()` wrapper around our synchronous `ksu_install_fd()`; change return protocol (`-EINVAL` for unhandled; `0` for handled) and make `kernel/reboot.c` honour `ret`. Keep the 25-entry ioctl table untouched. | Without this, `ksu_susfs` 2.4.0 commands never reach the kernel. |
| A2 | `KernelSU/kernel/selinux/selinux.c`, `selinux.h`, `rules.c` | Apply the append-only SID helper block (patch fuzz 2 works; clean up whitespace), add `susfs_set_batch_sid()` call after `apply_kernelsu_rules()`/policy load. | Required by target `setuid_hook.c` (`susfs_zygote_sid`, `susfs_zygote_next_sid`). |
| A3 | `KernelSU/kernel/hook/setuid_hook.c`, `kernel/feature/kernel_umount.c` (optional), `KernelSU/kernel/policy/app_profile.c/.h` | Port target zygote logic into our existing `ksu_handle_setresuid_cred(new, old)`: manager → `ksu_install_fd()` + `disable_seccomp()`; allowed uid → `disable_seccomp()`; `is_appuid && ksu_uid_should_umount` → `susfs_set_current_proc_no_su()` + `susfs_set_current_proc_umounted()` + `ksu_handle_umount(new, old)` + `schedule_work(&susfs_extra_works)`; isolated same; else `susfs_set_current_proc_no_su()`. Add `int escape_to_root_for_init(void)` wrapper in `app_profile.c`. Keep our LSM `task_fix_setuid` path; do not patch `kernel/sys.c` setresuid. | This is the behavior 2.4.0's direct `__sys_setresuid` hook provides; our LSM callback is the 4.19 equivalent. |
| A4 | `KernelSU/kernel/feature/sucompat.c/.h`, main tree `fs/exec.c` | Keep our 4.19 handler signatures. Add `ksu_handle_execveat_init()` (mark no-su; call our `ksu_adb_root_execve_kernel()` for `/app_process`, `/adbd`, `/stub_zygote`); add `ksu_handle_post_execveat_sucompat()` and call it from our `fs/exec.c` after `__do_execve_file`/`bprm_execve`; add `current_chrooted()` guards in `ksu_handle_faccessat/stat`; keep `su→sh`/`su→ksud` behavior. Make `ksu_su_compat_enabled` a `static_key_true` only if the 50 patch expects it; otherwise keep our bool + `ksu_execveat_hook`. | Core sucompat correctness and fd-lifetime fix. |
| A5 | `KernelSU/kernel/runtime/ksud_integration.c` | Implement the existing stubs: `ksu_handle_input_handle_event()` (volume-key safe-mode detection; port logic from `vol_detector_event()`) and `ksu_handle_vfs_read()` (call `ksu_install_rc_hook(*file_ptr)`); optionally add `ksu_handle_vfs_fstat()` and patch `fs/stat.c vfs_fstat` if our syscall-hook path is retired. | These call sites already exist in our 4.19 kernel but currently do nothing; no need to rewrite the whole file. |
| A6 | `KernelSU/kernel/runtime/boot_event.c`, `KernelSU/kernel/ksu.c`, main tree `kernel/reboot.c` | On `EVENT_BOOT_COMPLETED`/`on_boot_completed()`, call `susfs_start_sdcard_monitor_fn()` (2.4.0 `fs/susfs.c`); remove/replace our 1.5.11 `susfs_is_boot_completed_triggered = true` line if switching to 2.4.0 `fs/susfs.c` (symbol does not exist there). | 2.4.0's `susfs_is_sdcard_android_data_not_decrypted` static key is driven by that monitor; without it, execve path selection and sdcard path spoofing stay in "not decrypted" mode. |
| A7 | `KernelSU/kernel/manager/pkg_observer.c`, `KernelSU/kernel/Kconfig`, `KernelSU/kernel/security` | Add `susfs_parse_priority_app_strings()`/`susfs_parse_sus_path_app_strings()` calls in our `ksu_rename_observer_slow()`/`track_throne` path; add `KSU_SUSFS_PRIORITY_UID_FILTER` Kconfig. | Required for `CMD_SUSFS_SET_SUS_PATH_UID_FILTER` and `CMD_SUSFS_SET_PRIORITY_UID_FILTER` to refresh after package changes. |
| A8 | `KernelSU/kernel/selinux/sepolicy.c` | Add `susfs_add_sus_avd_allowed_by_type()` in `add_rule_raw()` and `susfs_add_sus_type()` in `add_type()`. | 2.4.0 `fs/susfs.c` uses these tables for SELinux policy-aware hiding; without them, SUSFS's policy hooks lack type/AVD metadata. |

Minimum file list for Phase A (inside `KernelSU/` unless noted):

```
kernel/supercall/dispatch.c
kernel/supercall/supercall.c
kernel/supercall/supercall.h
kernel/selinux/selinux.c
kernel/selinux/selinux.h
kernel/selinux/rules.c
kernel/selinux/sepolicy.c
kernel/hook/setuid_hook.c
kernel/policy/app_profile.c
kernel/policy/app_profile.h
kernel/feature/sucompat.c
kernel/feature/sucompat.h
kernel/runtime/ksud_integration.c
kernel/runtime/boot_event.c
kernel/ksu.c
kernel/manager/pkg_observer.c
kernel/Kconfig
# main kernel tree:
kernel/reboot.c
fs/exec.c
# optional for fd-lifetime/module rc:
fs/stat.c, fs/read_write.c, drivers/input/input.c
```

### 5.3 Optional / deferred extras

| Feature | 2.4.0 files | Recommendation |
|---|---|---|
| `selinux_hide` | `kernel/feature/selinux_hide.c` (replaces 1,563 → 174 lines) | **Do not port the 2.4.0 replacement.** Our 377-line 4.19 implementation hooks `selinux_kernel_status_page()` and `selinux_transaction_write()` and does not rely on `selinux_state.status_page` (which 4.19 lacks). If 2.4.0's `fs/susfs.c` has its own SELinux policy hooks, keep only one of the two; test with `selinux_hide` feature off/on. |
| `adb_root` | `kernel/feature/adb_root.c/.h` (7 hunks) | **Keep ours.** Our `do_ksu_adb_root_execve_user/kernel()` already does the same job with 4.19-compatible APIs. Only add a call from `ksu_handle_execveat_init()` to `ksu_adb_root_execve_kernel()`; no need to switch to `ksu_adb_root_handle_execveat()`. |
| `preinit` | `kernel/preinit/preinit.c/.h` (2+2 hunks) | **Do not port.** Our preinit uses a custom runner payload and `ksyscall(execve)`; 2.4.0's `ksu_setup_preinit_file()` mutates `filename->name` in place (possible overflow if `fd_path` exceeds the original name) and assumes 5.10+ `struct filename` VFS entry points. Keep our implementation. |
| `sulog` | `kernel/sulog/event.c/.h` (10+2 hunks) | **Optional.** Our sulog API is newer (`ksu_sulog_capture_bprm`, `ksu_sulog_emit_bprm`). If porting target sucompat, add an adapter `ksu_sulog_capture_sucompat(const char *, struct user_arg_ptr *)` in our event.c; otherwise skip. |
| `priority uid filter` | `kernel/Kconfig`, `fs/susfs.c` (2.4.0), `kernel/sys.c` + `block/ioprio.c` (50 patch), `pkg_observer.c` | **Phase B.** Useful and low risk once Phase A is stable; not required for basic su/SUSFS. Add config default y and the two syscall hooks only after testing. |
| `sus_map` | `Kconfig` already exists; `fs/susfs.c` + `fs/proc/*` 50 patch | Already present in our Kconfig; ensure 2.4.0 `fs/susfs.c` includes `ADD_SUS_MAP` and `fs/proc` hooks after the 4.19 backport. |
| `try_umount` | 2.4.0 has no `CMD_SUSFS_ADD_TRY_UMOUNT`/struct | **Drop** the `KSU_SUSFS_TRY_UMOUNT` Kconfig option when moving to 2.4.0; our kernel `try_umount` list remains used by KSU's own module umount, not by SUSFS 2.4.0. |

### 5.4 Suggested sequencing and tests

1. **Branch / backup:** use a separate branch off `feature/ksu3.3.1-susfs-1.5.11-on-73fe` or a worktree; keep the 1.5.11 image as rollback.
2. **A1 first, alone:** compile `CONFIG_KSU_SUSFS=y` + `CONFIG_KSU_SUSFS=n`, run `ksu_susfs show version` (2.4.0 tool) and a plain `reboot`; verify that normal reboot still works (return-protocol bug is the highest-risk item).
3. **A2/A3:** boot, `su -c id`, `ksu_susfs show enabled_features`, spawn app processes, verify `ps`/`/proc` mount behavior and no zygote crash.
4. **A4:** test `su`, `which su`, `stat /system/bin/su`, failed `execve` (fd leak check), module install.
5. **A5/A6/A7/A8:** volume-key safe mode, module rc read, `/sdcard` monitor, package install/uninstall with `sus_path` filters.
6. Build matrix: `CONFIG_KSU_SUSFS` on/off, `CONFIG_KSU_SUSFS_PRIORITY_UID_FILTER` on/off, `CONFIG_KSU_SUSFS_TRY_UMOUNT` off, `CONFIG_KSU_LSM_SECURITY_HOOKS` on (our default), `CONFIG_KSU_TAMPER_SYSCALL_TABLE`/`KSU_HACK_ARM64_BRANCH_LINK` combinations.

---

## 6. Manager API / version mismatch warnings

### 6.1 What “v3.3.1-51 / uapi 6” means

- Official KernelSU `f49be695` is exactly **v3.3.1-51** (`git describe --tags --always` = `v3.3.1-51-gf49be695`), i.e. the 51st commit after the `v3.3.1` tag and the direct parent of official `main` HEAD `f4603d14`. The 2.4.0 KernelSU patch's 32 pre-image hashes all match this commit.
- That baseline has `KERNEL_SU_UAPI_VERSION = 6` and the same 25 `KSU_IOCTL_*` definitions as our tree; 2.4.0 does not change them. A manager built at v3.3.1-51 therefore remains UAPI-compatible with a 2.4.0-patched kernel **as far as the ioctl/supercall ABI is concerned**.
- Our fork's `KernelSU/kernel/Makefile` sets `-DKSU_VERSION=32725`; our manager's `Natives.kt` declares `MINIMAL_SUPPORTED_KERNEL = 32513`. `32725 >= 32513`, and our `KERNEL_SU_UAPI_VERSION` is `6` (`kernel/include/uapi/supercall.h:9`). So the manager version relationship is fine; no version bump is required by the 2.4.0 patch.

### 6.2 Actual mismatches to watch

| Area | Risk | Mitigation |
|---|---|---|
| `KSU_GET_INFO_FLAG_LATE_LOAD` / `BUNDLED` | 2.4.0's `do_get_info`/`do_get_info_legacy` hunks remove `ksu_late_loaded` and `ksu_bundled` flag handling. Our manager UI reads `isLateLoadMode`/`isLkmBundled` (`Natives.kt`), so applying these hunks would make an LKM/late-load build report wrong state. | **Do not port** the `do_get_info` hunks; keep our current flags logic. |
| `KSU_MANAGE_MARK` semantics | 2.4.0 changes `KSU_MARK_GET` from task-mark/seccomp status to `!susfs_is_current_proc_no_su()`; `KSU_MARK_MARK/UNMARK/REFRESH` become no-ops. Our manager may call `MANAGE_MARK` for “mark running processes” (JNI not in this source snapshot, but the UAPI exists). | Keep our `ksu_get_task_mark()` seccomp proxy, or return a combined value; test manager profile/umount screens. Do not blindly take 2.4.0's `do_manage_mark` block. |
| `KSU_MARK_REFRESH` | 2.4.0 makes it a no-op because task marks are gone; our `do_manage_mark` returns `-EINVAL` for it. If manager calls REFRESH, 2.4.0 is friendlier; ours may log an error. | Add a no-op `KSU_MARK_REFRESH` case if manager calls it. |
| `ksu_susfs` 2.4.0 tool vs 1.5.11 kernel ABI | 2.4.0 tool uses `syscall(SYS_reboot, KSU_INSTALL_MAGIC1, SUSFS_MAGIC, CMD_SUSFS_*, &info)`; 1.5.11 tool uses `prctl(KERNEL_SU_OPTION, CMD_SUSFS_*, arg3, arg4, &err)`. Structs also differ: `st_susfs_sus_path` 272 (1.5.11) vs 260 (2.4.0), `st_susfs_sus_kstat` 368 vs 376, `st_susfs_uname` 130 vs 136, `st_susfs_sus_mount` (1.5.11) vs `st_susfs_hide_sus_mnts_for_non_su_procs` (2.4.0), plus 2.4.0 adds `sus_map`, `avc_log_spoofing`, `enabled_features`, `variant`, `version`, `priority_uid_filter` structs. | **Never mix versions.** Install `susfs4ksu_sidex_v2.4.0` (or the 2.4.0 module) with a 2.4.0 kernel; keep 1.5.11 module/tool only with the 1.5.11 kernel. The manager app itself is version-agnostic to SUSFS. |
| `susfs_is_boot_completed_triggered` | Our 1.5.11 `ksu.c` uses this symbol (`extern bool ...; susfs_init(); susfs_is_boot_completed_triggered = true;`), but 2.4.0's `fs/susfs.c` **does not define it** (2.4.0 uses `susfs_start_sdcard_monitor_fn()` and the `susfs_is_sdcard_android_data_not_decrypted` static key instead). | When switching to 2.4.0 `fs/susfs.c`, delete that line or add a compatibility shim; otherwise link fails. |
| `susfs_*` function ABI | 2.4.0 prototypes are `void susfs_*(void __user **user_info)` and fill `info.err`; 1.5.11 prototypes are `int susfs_*(struct st_susfs_* __user *user_info)`. | The dispatch shim must match whichever kernel-side `fs/susfs.c` is linked. If 1.5.11 stays, keep `susfs_cmd.c`; if 2.4.0 lands, replace it with the reboot dispatch. |
| `KSU_IOCTL_*` table | No change in 2.4.0; our table and official v3.3.1-51 table are identical 25 entries (only formatting differs). | No manager-side change needed. |

### 6.3 Bottom line for the manager

The manager `v3.3.1-51` (uapi 6) works with the 2.4.0 KernelSU patch **provided the port preserves the ioctl-fd API and does not regress `do_get_info`/`do_manage_mark` semantics**. The actual compatibility risk is on the **SUSFS userspace tool side** (`ksu_susfs` 2.4.0 vs 1.5.11) and on **`susfs.c` ABI**, not on the KernelSU manager app.

---

## 7. Appendix: evidence and commands

### 7.1 Current 1.5.11 working-tree diff (for reproducibility)

`git -C KernelSU diff --stat`:

```
 kernel/Kconfig                    | 98 +++++++++++++++++++++++++++++++++++++++
 kernel/feature/sucompat.c         |  8 ++++
 kernel/hook/lsm_hooks_list.c      | 15 ++++++
 kernel/ksu.c                      |  9 ++++
 kernel/runtime/ksud.h             |  4 +-
 kernel/runtime/ksud_integration.c | 15 +++++-
 6 files changed, 145 insertions(+), 4 deletions(-)
```

Plus untracked `kernel/feature/susfs_cmd.c` (261 lines; prctl kprobe bridge). Existing integration patches: `patches/kernelsu-3.3.1-susfs-init.patch`, `patches/kernelsu-3.3.1-susfs-kconfig.patch`, `patches/kernelsu-3.3.1-susfs-prctl.patch`, `patches/kernelsu-3.3.1-stale-hooks-compat.patch` (504 lines total).

### 7.2 Official baseline verification

```bash
cd /tmp/official-ksu
git log -1 --format='%H %cd %s' --date=iso
# f4603d14..., 2026-11-17, manager: enhance module flashing screen (#3998)
git describe --tags --always f49be695
# v3.3.1-51-gf49be695
# All 32 patch pre-image hashes match f49be695; `git apply --check` exit 0.
```

### 7.3 Patch dry-run result per file (summarized)

| Result class | Files |
|---|---|
| **clean** | `kernel/selinux/selinux.h` |
| **fuzz 2** | `kernel/Makefile` (cosmetic), `kernel/selinux/selinux.c` (usable append) |
| **reject, file exists** | `kernel/Kconfig`, `kernel/feature/adb_root.c`, `kernel/feature/adb_root.h`, `kernel/feature/kernel_umount.c`, `kernel/feature/selinux_hide.c`, `kernel/feature/sucompat.c`, `kernel/feature/sucompat.h`, `kernel/hook/setuid_hook.c`, `kernel/include/ksu.h`, `kernel/manager/pkg_observer.c`, `kernel/policy/allowlist.c`, `kernel/policy/app_profile.c`, `kernel/policy/app_profile.h`, `kernel/policy/feature.h`, `kernel/preinit/preinit.c`, `kernel/preinit/preinit.h`, `kernel/runtime/boot_event.c`, `kernel/runtime/ksud.h`, `kernel/runtime/ksud_integration.c`, `kernel/selinux/rules.c`, `kernel/selinux/sepolicy.c`, `kernel/sulog/event.c`, `kernel/sulog/event.h`, `kernel/supercall/dispatch.c`, `kernel/supercall/supercall.c`, `kernel/supercall/supercall.h` |
| **reject, path missing** | `kernel/Kbuild`, `kernel/core/init.c`, `kernel/hook/setuid_hook.h` |

### 7.4 Limitations

- This is static analysis only. No kernel build, boot, device test, or `ksu_susfs` execution was performed. The `patch --dry-run` results are based on the current working tree (including uncommitted 1.5.11 changes) in `/tmp/ksu-check`.
- The companion kernel-side 2.4.0 patch (`50_add_susfs_in_gki-android14-6.1.patch`) is designed for GKI Android 14 / 6.1. Porting it to our 4.19 kernel is a separate, much larger effort; this report only uses it to derive the exact KernelSU-side symbol/call-site expectations.
- Line numbers refer to the current files at the time of analysis; line shifts after edits are expected.
