# SUSFS v2.4.0 vs v1.5.11 对比与 4.19 移植分析

> 目标：比较上游 SUSFS v2.4.0（GKI android14-6.1）与当前 4.19 内核中的 SUSFS v1.5.11 backport，判断如果把 2.4.0 的 `fs/susfs.c`、头文件、KernelSU 集成补丁直接落到本仓库（picasso，Linux 4.19.248，KernelSU 3.3.1 fork）会遇到什么，并给出分优先级移植清单。
>
> 本报告只描述代码事实；未修改任何内核源码。所有行号基于下列固定版本。

## 0. 来源、范围与方法

### 0.1 代码来源

| 代号 | 位置 | 版本/提交 | 关键文件规模 |
|---|---|---|---|
| `[1.5.11]` | 本仓库 `/home/yueguang/DSH/安卓/repos/picasso-kernel-ksu-susfs`，分支 `feature/ksu3.3.1-susfs-1.5.11-on-73fe` | `ef536713b553`（2026-11-17）；Linux 4.19.248；KernelSU 子模块 `3b60f2af`（v1.0.0-1148） | `fs/susfs.c` 1324 行；`include/linux/susfs.h` 202 行；`include/linux/susfs_def.h` 87 行；`patches/susfs-4.19-v1.5.11.patch` 3757 行、22 个文件 |
| `[2.4.0]` | `/tmp/susfs4ksu-2.4.0`，GitLab `simonpunk/susfs4ksu` 分支 `gki-android14-6.1` | `f76b81f2a6bf`（2026-11-15） | `kernel_patches/fs/susfs.c` 2560 行；`kernel_patches/include/linux/susfs.h` 261 行；`kernel_patches/include/linux/susfs_def.h` 172 行；`kernel_patches/50_add_susfs_in_gki-android14-6.1.patch` 3438 行、27 个文件；`kernel_patches/KernelSU/10_enable_susfs_for_ksu.patch` 4411 行、32 个文件 |

`[1.5.11]` 的补丁来自 `patches/susfs-4.19-v1.5.11.patch`，当前工作树已应用该补丁并额外应用 `patches/susfs-4.19-v1.5.11-path-root-fix.patch`。`[2.4.0]` 的代码是上游 GKI 6.1 源文件 + 对应的 6.1 补丁，不是 4.19 backport。

### 0.2 方法

1. 直接比较 `fs/susfs.c`、`include/linux/susfs.h`、`include/linux/susfs_def.h` 的全部结构体和函数。
2. 用 `gcc -m64` 按真实字段定义计算 `sizeof(struct ...)`（LP64 / aarch64 同构；包含对齐填充）。
3. 对照本仓库 `fs/`、`security/selinux/`、`mm/`、`block/`、`KernelSU/kernel/` 的真实 4.19 头文件与函数签名，确认哪些 2.4.0 依赖不存在。
4. 只把“4.19 编译/链接会失败”与“语义会变”的点写进结论；不对未实现的 4.19 行为作猜测。

### 0.3 结论摘要

- **2.4.0 不是 1.5.11 的增量补丁，而是一次架构换代**：`fs/susfs.c` 从 1324 行增长到 2560 行（+93%）；头文件从 202/87 行增长到 261/172 行；KSU 集成补丁从本仓库约 504 行的 4 个本地 patch 变成上游 4411 行、32 个文件的补丁。
- **用户态 ABI 完全变了**：1.5.11 用 `prctl(KERNEL_SU_OPTION=0xDEADBEEF, CMD_SUSFS_*, arg3, arg4, arg5)`；2.4.0 用 `syscall(SYS_reboot, KSU_INSTALL_MAGIC1=0xDEADBEEF, SUSFS_MAGIC=0xFAFAFAFA, CMD_SUSFS_*, &info)`。2.4.0 的 `susfs_*()` 内核入口统一为 `void func(void __user **user_info)`，错误码写回结构体的 `err` 字段，而不是返回 `int`。
- **功能增删**：2.4.0 新增 `SUS_MAP`（1.5.11 只有死 Kconfig）、`PRIORITY_UID_FILTER`、完整的 SELinux type/av 隐藏、FUSE kstat、statfs/mnt_id 伪造、sus_path UID 位图过滤；删除了 `TRY_UMOUNT`、`SUS_SU`、三个 `AUTO_ADD_*` 开关，以及 `ADD_SUS_MOUNT` 用户列表。`ADD_SUS_MOUNT`/`SET_ANDROID_DATA_ROOT_PATH`/`SET_SDCARD_ROOT_PATH` 等旧命令在 2.4.0 只是保留常量，已标记 deprecated。
- **4.19 直接替换 `fs/susfs.c` 必然失败**：2.4.0 使用 `struct fs_context`/`vfs_create_mount`、`mnt->mnt.mnt_userns`、`mnt_stuck_children`、`i_uid_into_mnt`/vfsuid、`state->policy`、`struct fsnotify_ops.handle_inode_event`、`statfs_by_dentry`/`calculate_f_flags` 等 4.19 没有或签名不同的接口。这些不是“改几个 include”能解决的。
- **KSU 侧同样不能直接打 2.4.0 的 4411 行补丁**：该补丁针对的是带 `kernel/core/init.c`、`kernel/Kbuild`、`hook/syscall_hook_manager.c`、`hook/lsm_hook.c` 等模块化结构的旧 KernelSU；本仓库是 unity build（`kernel/Makefile` + `kernel/ksu.c` 包含 `feature/susfs_cmd.c`），且缺少 2.4.0 要求的大量 SUSFS 符号（`ksu_selinux_hide_running`、`fake_status`、`initialize_fake_status`、`susfs_set_batch_sid`、`susfs_*_sid`、`susfs_start_sdcard_monitor_fn` 调用点等）。
- **建议路线**：不要“替换成 2.4.0”，而是按“先 ABI/核心 sus_path+kstat → 任务跟踪 → mount → 可选 SUS_MAP/SELinux hide”的顺序移植；先保留当前可用的 1.5.11 prctl 桥作为过渡，再切到 `SUSFS_MAGIC` reboot 传输。完整清单见第 5 节。

---

## 1. 版本与功能开关矩阵

### 1.1 文件规模对比

| 项目 | `[1.5.11]` | `[2.4.0]` | 变化 |
|---|---:|---:|---:|
| `fs/susfs.c` | 1324 行 | 2560 行 | +93% |
| `include/linux/susfs.h` | 202 行 | 261 行 | +29% |
| `include/linux/susfs_def.h` | 87 行 | 172 行 | +98% |
| 内核侧补丁文件数 | 22 | 27 | +5 |
| KSU 侧补丁 | 本仓库 4 个 patch 合计 504 行（kconfig 107 + init 18 + prctl 320 + stale-hooks 59） | `10_enable_susfs_for_ksu.patch` 4411 行、32 文件 | 约 8.8 倍 |

### 1.2 全部 `CONFIG_KSU_SUSFS_*` 选项矩阵

说明：
- `[1.5.11]` 的 Kconfig 在 `KernelSU/kernel/Kconfig:147-245`；`[2.4.0]` 的 Kconfig 在 `10_enable_susfs_for_ksu.patch:58-159`（对应上游补丁 `kernel/Kconfig`）。
- “源码实现”一列指该配置在 `fs/susfs.c`、VFS 钩子等真实代码中是否有 `#ifdef` 使用；仅有 Kconfig 不算实现。

| 配置项 | `[1.5.11]` Kconfig | `[1.5.11]` 源码实现 | `[2.4.0]` Kconfig | `[2.4.0]` 源码实现 | 判定 |
|---|---|---|---|---|---|
| `CONFIG_KSU_SUSFS` | `KernelSU/kernel/Kconfig:147`（master，default y） | 有（`fs/Makefile:137` 编 `susfs.o`） | `10_enable…patch:58`（default y） | 有 | 保留 |
| `CONFIG_KSU_SUSFS_SUS_PATH` | `:155`（default y） | 有（`fs/susfs.c:134` 起） | `:66`（default y） | 有，新增 uid 位图过滤/FUSE | 保留，语义扩展 |
| `CONFIG_KSU_SUSFS_SUS_MOUNT` | `:166`（default y） | 有（`fs/susfs.c:503` 起；`fs/namespace.c`） | `:77`（default y） | 有，整体重写（`hide_sus_mnts_for_non_su_procs` + fake mnt id） | 保留，语义重写 |
| `CONFIG_KSU_SUSFS_SUS_KSTAT` | `:175`（default y） | 有（`fs/susfs.c:654` 起） | `:87`（default y） | 有，新增 FUSE/statfs/mnt_id/inotify/proc fd | 保留，语义扩展 |
| `CONFIG_KSU_SUSFS_SPOOF_UNAME` | `:183`（default y） | 有（`fs/susfs.c:974` 起） | `:95`（default y） | 有，seqlock+static key | 保留 |
| `CONFIG_KSU_SUSFS_TRY_UMOUNT` | `:191`（default **n**） | 有（`fs/susfs.c:838` 起） | 无 | 无 | **删除** |
| `CONFIG_KSU_SUSFS_ENABLE_LOG` | `:199`（default y） | 有（`fs/susfs.c:1009`） | `:103`（default y） | 有（`susfs_enable_log(void __user**)`） | 保留 |
| `CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS` | `:206`（default y） | 有（`kernel/kallsyms.c` 补丁） | `:112`（default y） | 有（`kernel/kallsyms.c`，过滤规则更多） | 保留 |
| `CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG` | `:214`（default y） | 有 `fs/susfs.c:1024`，但当前树无消费者钩子 | `:120`（default y） | 有 `fs/susfs.c:1390`，消费者在 `fs/proc/bootconfig.c`（4.19 无此文件） | 保留，4.19 需改到 `fs/proc/cmdline.c` |
| `CONFIG_KSU_SUSFS_OPEN_REDIRECT` | `:222`（default y） | 有（`fs/susfs.c:1092` 起） | `:128`（default y） | 有，新增 uid_scheme/双向表/readlink/maps 伪造 | 保留，语义扩展 |
| `CONFIG_KSU_SUSFS_SUS_MAP` | `:231`（default y） | **无任何 `#ifdef CONFIG_KSU_SUSFS_SUS_MAP` 代码**（死开关） | `:139`（default y） | 有（`fs/susfs.c:1749`，`mm/memory.c`、`fs/proc/task_mmu.c`、`fs/proc/base.c`） | **2.4.0 首次真正实现** |
| `CONFIG_KSU_SUSFS_PRIORITY_UID_FILTER` | 无 | 无 | `:151`（default y） | 有（`fs/susfs.c:1987`，`block/ioprio.c`、`kernel/sys.c`） | **新增** |
| `CONFIG_KSU_SUSFS_SUS_SU` | Kconfig 无定义 | 代码有 `fs/sus_su.c`、`include/linux/sus_su.h`、`fs/exec.c`/`fs/open.c` 的 `#ifdef`，但 `fs/Makefile` 不编译 `sus_su.o`，且 prctl 桥只在 `#ifdef` 下引用未定义选项 | 无（`CMD_SUSFS_SUS_SU 0x60000` 仅 deprecated 常量） | 无 | **删除 / 死代码** |
| `CONFIG_KSU_SUSFS_AUTO_ADD_SUS_BIND_MOUNT` | Kconfig 无定义 | 有 `fs/susfs.c:554`，无 Kconfig，实际不会编译进 | 无 | 无 | **删除 / 死代码** |
| `CONFIG_KSU_SUSFS_AUTO_ADD_SUS_KSU_DEFAULT_MOUNT` | Kconfig 无定义 | 有 `fs/susfs.c:577`，无 Kconfig | 无 | 无 | **删除 / 死代码** |
| `CONFIG_KSU_SUSFS_AUTO_ADD_TRY_UMOUNT_FOR_BIND_MOUNT` | Kconfig 无定义 | 有 `fs/susfs.c:888`，无 Kconfig | 无 | 无 | **删除 / 死代码** |

### 1.3 1.5.11 当前树的“配置存在但未实现/未接线”清单

这些是理解“为什么现在的 1.5.11 看起来功能很多、实际很多不生效”的关键：

1. `CONFIG_KSU_SUSFS_SUS_MAP`：`KernelSU/kernel/Kconfig:231` 有 `config KSU_SUSFS_SUS_MAP`，但 `grep -R "CONFIG_KSU_SUSFS_SUS_MAP" fs/ include/` 无结果——没有 `sus_map` 实现、没有 `mm/memory.c`/`task_mmu.c` 钩子。
2. `CONFIG_KSU_SUSFS_SUS_SU`：`fs/sus_su.c` 文件存在，但 `fs/Makefile:137` 只加 `susfs.o`，未加 `sus_su.o`；`fs/exec.c:69`、`fs/open.c` 的 `#ifdef CONFIG_KSU_SUSFS_SUS_SU` 分支实际不会编译。
3. `susfs_is_current_ksu_domain()`：`fs/susfs.c:26-29` 直接 `return false;`。`fs/namespace.c` 的 mount 隐身逻辑依赖它判断“KSU 进程创建的 mount”，因此永远不生效。2.4.0 把这个函数交给 KernelSU 的 `kernel/selinux/selinux.c` 实现（`10_enable…patch:3783-3793`）。
4. `susfs_set_current_proc_umounted()` / `TIF_PROC_UMOUNTED 33`：`include/linux/susfs_def.h:54` 有定义，但本仓库 KSU 的 `kernel/hook/setuid_hook.c`、`kernel/feature/sucompat.c` 没有任何调用；当前树也没有 `TIF_PROC_NO_SU`/`TIF_PROC_UMOUNTED_FOR_ZYGOTE_NEXT`。所有依赖 `susfs_is_current_proc_umounted()` 的 VFS 钩子实际上只对“主动调用 prctl 设置标志的进程”生效（而当前没有调用者）。
5. `susfs_is_boot_completed_triggered`：`fs/susfs.c:22` 定义，`KernelSU/kernel/ksu.c:282-286` 在 `kernelsu_init()` 里 `susfs_init(); susfs_is_boot_completed_triggered = true;`——在 init 阶段就置 true，而不是等 `EVENT_BOOT_COMPLETED`。这使 1.5.11 的“boot 前保护 KSU mount”逻辑基本失效。
6. `susfs_is_current_zygote_domain()`：`fs/namespace.c:38` 只有 `extern` 声明，整棵树没有定义；因为无调用者，不会链接失败，但这是 1.5.11 集成不完整的痕迹。`susfs_reorder_mnt_id()`（`fs/namespace.c:3795`）和 `susfs_is_mnt_devname_ksu()`（`:3814`）也没有调用者。
7. `susfs_spoof_cmdline_or_bootconfig()`：`fs/susfs.c:1053` 实现，但 1.5.11 补丁没有修改 `fs/proc/cmdline.c` 或 `fs/proc/bootconfig.c`，所以用户态设置成功、实际读 `/proc/cmdline` 仍是原值。

### 1.4 2.4.0 功能级变化摘要

- **保留并增强**：`SUS_PATH`（新增 `SET_SUS_PATH_UID_FILTER` 与按包名解析 UID）、`SUS_MOUNT`（从“按 dev/path 列表”改为“给 KSU 进程创建的 mount 分配 fake mnt_id，按 `TIF_PROC_UMOUNTED` 隐藏”）、`SUS_KSTAT`（新增 FUSE、statfs、mnt_id、inotify fdinfo、proc fd）、`SPOOF_UNAME`、`ENABLE_LOG`、`SPOOF_CMDLINE_OR_BOOTCONFIG`、`OPEN_REDIRECT`（新增 uid_scheme 0-4 与 readlink/maps 伪造）、`HIDE_KSU_SUSFS_SYMBOLS`。
- **新增**：`SUS_MAP`、`PRIORITY_UID_FILTER`、完整 SELinux type/av 隐藏（`susfs_add_sus_type`/`susfs_add_sus_avd_allowed_by_type`）、`susfs_switch_to_ksu_domain()`/`susfs_restore_creds()`、sdcard `/data/media/0/Android` 解密监控（fsnotify）。
- **删除/废弃**：`TRY_UMOUNT` 整套（`susfs_add_try_umount`/`susfs_try_umount`/auto add）、`SUS_SU` 整套（`fs/sus_su.c`/`fs/devpts/inode.c` 钩子）、`ADD_SUS_MOUNT` 用户列表与 `susfs_add_sus_mount`、`SET_ANDROID_DATA_ROOT_PATH`/`SET_SDCARD_ROOT_PATH`、`UMOUNT_FOR_ZYGOTE_ISO_SERVICE`、`SHOW_SUS_SU_WORKING_MODE`、`IS_SUS_SU_READY`。
- **重命名（命令）**：`CMD_SUSFS_HIDE_SUS_MNTS_FOR_ALL_PROCS (0x55561)` → `CMD_SUSFS_HIDE_SUS_MNTS_FOR_NON_SU_PROCS`（同一 ID）。对应的内核状态从全局 `bool susfs_hide_sus_mnts_for_all_procs` 改为 `DEFINE_STATIC_KEY_FALSE(susfs_is_hide_sus_mnts_for_non_su_procs_enabled)`（`[2.4.0] fs/susfs.c:838` 区域）。

---

## 2. 用户空间 ABI：命令、传输、结构体

### 2.1 传输方式（最重要差异）

| 项目 | `[1.5.11]` | `[2.4.0]` |
|---|---|---|
| 系统调用 | `prctl` | `syscall(SYS_reboot, ...)` |
| 参数 | `prctl(0xDEADBEEF /* KERNEL_SU_OPTION */, CMD_SUSFS_xxx, arg3, arg4, arg5)` | `syscall(SYS_reboot, 0xDEADBEEF /* KSU_INSTALL_MAGIC1 */, 0xFAFAFAFA /* SUSFS_MAGIC */, CMD_SUSFS_xxx, &info)` |
| magic1 | `KERNEL_SU_OPTION 0xDEADBEEF` | `KSU_INSTALL_MAGIC1 0xDEADBEEF`（`KernelSU/kernel/include/uapi/supercall.h:12`） |
| magic2 | 无（cmd 是 arg2） | `SUSFS_MAGIC 0xFAFAFAFA`（`[2.4.0] kernel_patches/include/linux/susfs_def.h:13`；`ksu_susfs/jni/susfs_defs.h` 同值） |
| 权限检查 | `susfs_cmd.c:68`：`current_uid().val == 0` | `10_enable…patch:4217`：`current_uid().val == 0` |
| 内核入口 | `ksu_handle_susfs_prctl(int option, unsigned long arg2, ...)`，每命令 `case` 分发（`KernelSU/kernel/feature/susfs_cmd.c:60-213`） | `ksu_handle_sys_reboot(int magic1, int magic2, unsigned int cmd, void __user **arg)`（`10_enable…patch:4215-4296`），先判断 magic1/magic2，再 `switch(cmd)` |
| 错误返回 | `susfs_reply_error(arg5, error)` 写 `int` 到用户第 5 参数 | 统一 `copy_to_user(&((struct ... __user*)info)->err, &info.err, sizeof(err))`；返回 0 表示 syscall 已处理 |
| 旧 KSU 兼容 | 当前 `KernelSU/kernel/ksu.c:139` `#include "feature/susfs_cmd.c"`；`:311` 调用 `susfs_prctl_kprobe_init()`，kprobe `__arm64_sys_prctl` | 依赖 `kernel/reboot.c` 的 `ksu_handle_sys_reboot()` 钩子；补丁 `10_enable…patch:2322` 起改 `kernel/reboot.c` |
| 依赖 | `CONFIG_KPROBES`（kprobe 在 `__arm64_sys_prctl`） | 不依赖 kprobe；依赖 `sys_reboot` 钩子点 |

补充：
- 2.4.0 的用户态工具 `ksu_susfs/jni` 中没有任何 `ioctl`/`prctl` 调用，全部是 `syscall(SYS_reboot, ...)`。
- 1.5.11 的 prctl 桥只覆盖 1.5.11 的命令集（`sfs_cmd.c:67-213` 共 20 个 case，其中包含 `SUS_SU`/`TRY_UMOUNT`），不包含 2.4.0 的 `0x55554`、`0x60020`、`0x60030`。
- 本仓库 KernelSU 3.3.1 的 `KSU_IOCTL_*`（`kernel/include/uapi/supercall.h:173` 起）是 manager fd 的 ioctl 接口，与 SUSFS 无关；两个版本都没有“SUSFS ioctl 命令号”。

### 2.2 命令 ID 全表

`[1.5.11]` 定义在 `include/linux/susfs_def.h:10-31`；`[2.4.0]` 定义在 `[2.4.0] kernel_patches/include/linux/susfs_def.h:14-40`。

| 命令 ID | `[1.5.11]` 名称 | `[2.4.0]` 名称 | 变化 |
|---:|---|---|---|
| `0x55550` | `CMD_SUSFS_ADD_SUS_PATH` | `CMD_SUSFS_ADD_SUS_PATH` | 保留；结构体从 `{target_ino,i_uid,path}` 变为 `{path,err}` |
| `0x55551` | `CMD_SUSFS_SET_ANDROID_DATA_ROOT_PATH` | 同名，标注 `/* deprecated */` | 2.4.0 保留常量但不再分发 |
| `0x55552` | `CMD_SUSFS_SET_SDCARD_ROOT_PATH` | 同名，标注 `/* deprecated */` | 同上 |
| `0x55553` | `CMD_SUSFS_ADD_SUS_PATH_LOOP` | 同名 | 保留；2.4.0 不再检查路径是否存在 |
| `0x55554` | — | `CMD_SUSFS_SET_SUS_PATH_UID_FILTER` | **新增**（`[2.4.0] susfs_def.h:18`；`susfs.c:606`） |
| `0x55560` | `CMD_SUSFS_ADD_SUS_MOUNT` | 同名，`/* deprecated */` | 2.4.0 不分发 |
| `0x55561` | `CMD_SUSFS_HIDE_SUS_MNTS_FOR_ALL_PROCS` | `CMD_SUSFS_HIDE_SUS_MNTS_FOR_NON_SU_PROCS` | **重命名（同 ID）**；语义从“所有进程隐藏”变为“仅非 su 进程隐藏” |
| `0x55562` | `CMD_SUSFS_UMOUNT_FOR_ZYGOTE_ISO_SERVICE` | 同名，`/* deprecated */` | 2.4.0 不分发 |
| `0x55570` | `CMD_SUSFS_ADD_SUS_KSTAT` | 同名 | 保留 |
| `0x55571` | `CMD_SUSFS_UPDATE_SUS_KSTAT` | 同名 | 保留 |
| `0x55572` | `CMD_SUSFS_ADD_SUS_KSTAT_STATICALLY` | 同名 | 2.4.0 分发改调 `susfs_add_sus_kstat()`，`is_statically` 在结构体中 |
| `0x55580` | `CMD_SUSFS_ADD_TRY_UMOUNT` | 同名，`/* deprecated */` | 2.4.0 不分发 |
| `0x55590` | `CMD_SUSFS_SET_UNAME` | 同名 | 保留 |
| `0x555a0` | `CMD_SUSFS_ENABLE_LOG` | 同名 | 保留 |
| `0x555b0` | `CMD_SUSFS_SET_CMDLINE_OR_BOOTCONFIG` | 同名 | 保留 |
| `0x555c0` | `CMD_SUSFS_ADD_OPEN_REDIRECT` | 同名 | 保留；semantics 扩展 |
| `0x555e1` | `CMD_SUSFS_SHOW_VERSION` | 同名 | 保留；2.4.0 返回 `st_susfs_version` 结构（16 字节 + err） |
| `0x555e2` | `CMD_SUSFS_SHOW_ENABLED_FEATURES` | 同名 | 保留；buffer 从 4096 → 8192 |
| `0x555e3` | `CMD_SUSFS_SHOW_VARIANT` | 同名 | 保留；2.4.0 返回 `st_susfs_variant` 结构 |
| `0x555e4` | `CMD_SUSFS_SHOW_SUS_SU_WORKING_MODE` | 同名，`/* deprecated */` | 2.4.0 不分发 |
| `0x555f0` | `CMD_SUSFS_IS_SUS_SU_READY` | 同名，`/* deprecated */` | 2.4.0 不分发 |
| `0x60000` | `CMD_SUSFS_SUS_SU` | 同名，`/* deprecated */` | 2.4.0 不分发 |
| `0x60010` | `CMD_SUSFS_ENABLE_AVC_LOG_SPOOFING` | 同名 | 保留 |
| `0x60020` | — | `CMD_SUSFS_ADD_SUS_MAP` | **新增**（`[2.4.0] susfs_def.h:37`；`susfs.c:1749`） |
| `0x60030` | — | `CMD_SUSFS_SET_PRIORITY_UID_FILTER` | **新增**（`[2.4.0] susfs_def.h:38`；`susfs.c:1987`） |

### 2.3 结构体布局与大小（LP64）

以下大小按真实字段定义用 `gcc -m64` 计算（含 padding）：

| 结构体 | `[1.5.11]` 大小 | `[2.4.0]` 大小 | 关键变化 |
|---|---:|---:|---|
| `st_susfs_sus_path` | 272 | 260 | 1.5.11: `target_ino`(8) + `target_pathname[256]` + `i_uid`(4)；2.4.0: 只有 `target_pathname[256]` + `err`(4) |
| `st_susfs_sus_path_list` | 552 | 536 | 1.5.11 有 `path_len`(size_t)；2.4.0 删掉 `path_len`，`info` 也变小 |
| `st_susfs_sus_path_uid_filter` | — | 65540 | 新增：`char app_strings[65535]` + `err` |
| `st_android_data_path` | 257 | — | 2.4.0 删除 |
| `st_sdcard_path` | 257 | — | 2.4.0 删除 |
| `st_susfs_sus_mount` | 264 | — | 2.4.0 删除用户列表 |
| `st_susfs_sus_mount_list` | 280 | — | 同上 |
| `st_susfs_hide_sus_mnts_for_non_su_procs` | — | 8 | 新增：`bool enabled` + `int err` |
| `st_susfs_sus_kstat` | 368 | 376 | `is_statically` int→bool；`spoofed_*_nsec` signed long→unsigned long；追加 `flags`(KSTAT_SPOOF_*) + `err`；`spoofed_blocks` unsigned long long→long long |
| `st_susfs_sus_kstat_hlist` | 392 | 536 | 2.4.0 增加 `target_dev`、`spoofed_kstatfs`（`struct kstatfs`）、`spoofed_mnt_id`、`is_fuse`；字段顺序改变（2.4.0 为 `node, target_ino, target_dev, kstatfs, spoofed_mnt_id, is_fuse, info`） |
| `st_susfs_try_umount` | 260 | — | 删除 |
| `st_susfs_try_umount_list` | 280 | — | 删除 |
| `st_susfs_uname` | 130 | 136 | 2.4.0 追加 `int err` |
| `st_susfs_log` | — | 8 | 新增（2.4.0 用结构体；1.5.11 只用 `int` 参数） |
| `st_susfs_spoof_cmdline_or_bootconfig` | — | 8196 | 新增：buffer 8192 + `err`（1.5.11 用裸 `char*`，最大 4096） |
| `st_susfs_open_redirect` | 520 | 520 | 字段语义改变：1.5.11 `{target_ino, target_pathname, redirected_pathname}`；2.4.0 `{target_pathname, redirected_pathname, uid_scheme, err}` |
| `st_susfs_open_redirect_hlist` | 536 | 704 | 2.4.0 增加 `target_dev/redirected_dev`、`spoofed_kstatfs`、`spoofed_mnt_id`、`reversed_lookup_only` |
| `st_susfs_sus_map` | — | 260 | 新增：`target_pathname[256]` + `err` |
| `st_susfs_priority_uid_filter` | — | 65540 | 新增：`app_strings[65535]` + `err` |
| `st_susfs_avc_log_spoofing` | — | 8 | 新增：bool + err |
| `st_susfs_enabled_features` | — | 8196 | 新增：8192 buffer + err |
| `st_susfs_variant` | — | 20 | 新增：16 字节 + err |
| `st_susfs_version` | — | 20 | 新增：16 字节 + err |
| `st_sus_su` | 4 | — | 删除 |

### 2.4 关键字段/语义差异

1. **`err` 内嵌 + 整结构 copy_to_user**：2.4.0 的每个命令都先 `copy_from_user` 整个结构到局部变量，完成后再把 `info.err` 写回用户结构。1.5.11 则用函数返回值 + `arg5` 输出错误。两者不能混用；用户态 `ksu_susfs` 版本必须与内核 ABI 匹配。
2. **`sus_path` 去掉 `target_ino`/`i_uid`**：1.5.11 允许“只对某个 inode + 指定 app uid 隐藏路径”；2.4.0 改为“按路径给 inode 打标 + 全局 UID 位图（包名列表）决定哪些 app 可见”。
3. **`sus_kstat` 新增 `flags`**：2.4.0 用 `KSTAT_SPOOF_INO(1<<0)`、`KSTAT_SPOOF_DEV(1<<1)`、`KSTAT_SPOOF_NLINK(1<<2)`、`KSTAT_SPOOF_SIZE(1<<3)`、`KSTAT_SPOOF_ATIME_TV_SEC(1<<4)`、`KSTAT_SPOOF_ATIME_TV_NSEC(1<<5)`、`KSTAT_SPOOF_MTIME_TV_SEC(1<<6)`、`KSTAT_SPOOF_MTIME_TV_NSEC(1<<7)`、`KSTAT_SPOOF_CTIME_TV_SEC(1<<8)`、`KSTAT_SPOOF_CTIME_TV_NSEC(1<<9)`、`KSTAT_SPOOF_BLOCKS(1<<10)`、`KSTAT_SPOOF_BLKSIZE(1<<11)` 逐字段控制；1.5.11 总是整组覆盖。
4. **`open_redirect` 新增 `uid_scheme`**：2.4.0 定义 `enum UID_SCHEME { UID_NON_APP_PROC=0, UID_ROOT_PROC_EXCEPT_SU_PROC, UID_NON_SU_PROC, UID_UMOUNTED_APP_PROC, UID_UMOUNTED_PROC }`（`[2.4.0] susfs.h:23-29`）；1.5.11 没有这个 enum。
5. **`uname` buffer 长度**：两者都用 `__NEW_UTS_LEN+1 = 65` 字节数组，但 2.4.0 追加 `err`，且 `susfs_set_uname()` 对 `release`/`version` 都要求非空；1.5.11 允许 `"default"` 复位。
6. **`enabled_features` 从 4096 到 8192**：2.4.0 `SUSFS_ENABLED_FEATURES_SIZE 8192`（`susfs_def.h:43`），1.5.11 `SUSFS_FAKE_CMDLINE_OR_BOOTCONFIG_SIZE 4096`、`get_enabled_features` 的 bridge 也只允许 4096（`sfs_cmd.c:199-204`）。

### 2.5 `susfs_def.h` 常量对照

| 常量 | `[1.5.11]` | `[2.4.0]` | 说明 |
|---|---|---|---|
| `SUSFS_MAGIC` | 无 | `0xFAFAFAFA`（`:13`） | 2.4.0 新增 reboot magic2 |
| `SUSFS_MAX_LEN_PATHNAME` | 256（`:33`） | 256（`:42`） | 相同 |
| `SUSFS_FAKE_CMDLINE_OR_BOOTCONFIG_SIZE` | 4096（`:34`） | 8192（`:43`） | 翻倍 |
| `SUSFS_ENABLED_FEATURES_SIZE` | 无 | 8192（`:44`） | 新增 |
| `SUSFS_MAX_VERSION_BUFSIZE` / `SUSFS_MAX_VARIANT_BUFSIZE` | 无 | 16 / 16（`:45-46`） | 新增 |
| `TRY_UMOUNT_DEFAULT` / `TRY_UMOUNT_DETACH` | 0 / 1（`:36-37`） | 0 / 1（`:48-49`） | 2.4.0 仅保留常量，功能删除 |
| `SUS_SU_DISABLED/WITH_OVERLAY/WITH_HOOKS` | 0/1/2（`:39-41`） | 无 | 删除 |
| `DEFAULT_KSU_MNT_ID` | 300000（`:43`） | 2000000000（`:51`） | 大幅提高，避开真实 mnt_id |
| `DEFAULT_KSU_MNT_GROUP_ID` | 3000（`:44`） | 200000（`:52`） | 大幅提高 |
| `DEFAULT_KSU_MNT_MINOR_DEV` | 无 | `(1<<12)`（`:53`） | 新增；`get_anon_bdev()` 从 4097 起分配 minor |
| `FUSE_SUPER_MAGIC` fallback | 无 | `0x65735546`（`:55-57`） | 新增 |
| `SUSFS_APP_STR_BUF_SIZE` | 无 | 65535（`:59`） | 新增 |
| UID filter 范围 | 无 | `10000..19999`，`SUSFS_UID_FILTER_BITMAP_LEN=BITS_TO_LONGS(10000)`（`:60-62`） | 新增 |
| `VFSMOUNT_MNT_FLAGS_KSU_UNSHARED_MNT` | 无 | `0x80000000`（`:72`） | 新增 |
| `TIF_PROC_UMOUNTED` | 33（`:54`） | 33（`:74`） | 相同 |
| `TIF_PROC_NO_SU` | 无 | 34（`:75`） | 新增 |
| `TIF_PROC_UMOUNTED_FOR_ZYGOTE_NEXT` | 无 | 35（`:76`） | 新增 |
| `AS_FLAGS_SUS_PATH` | 24（`:56`） | 33（`:78`） | **位移整体后移 9 位** |
| `AS_FLAGS_SUS_MOUNT` | 25（`:57`） | 34（`:79`） | 同上 |
| `AS_FLAGS_SUS_KSTAT` | 26（`:58`） | 35（`:80`） | 同上 |
| `AS_FLAGS_OPEN_REDIRECT` | 27（`:59`） | 36（`:81`） | 同上 |
| `AS_FLAGS_ANDROID_DATA_ROOT_DIR` | 28（`:60`） | 无 | 删除 |
| `AS_FLAGS_SDCARD_ROOT_DIR` | 29（`:61`） | 无 | 删除 |
| `AS_FLAGS_SUS_MAP` | 无 | 39（`:82`） | 新增 |
| `ND_STATE_LOOKUP_LAST` / `ND_STATE_OPEN_LAST` / `ND_STATE_LAST_SDCARD_SUS_PATH` | 32 / 64 / 128（`:66-68`） | 32 / 64 / 无（`:84-85`） | 2.4.0 删除 `LAST_SDCARD_SUS_PATH` |
| `ND_FLAGS_LOOKUP_LAST` | `0x2000000`（`:69`） | `0x2000000`（`:86`） | 相同 |
| `STATX_SUS_KSTAT` / `STATX_SUS_KSTAT_FUSE` | 无 | `0x10000000` / `0x20000000`（`:88-89`） | 新增，内部 `result_mask` 哨兵 |

### 2.6 函数原型 ABI 变化

`[1.5.11] include/linux/susfs.h:137-202` 中大部分函数是 `int susfs_xxx(struct st_xxx __user *)`；`[2.4.0] include/linux/susfs.h:195-257` 全部改为：

```c
void susfs_add_sus_path(void __user **user_info);
void susfs_add_sus_path_loop(void __user **user_info);
void susfs_set_sus_path_uid_filter(void __user **user_info);
void susfs_set_hide_sus_mnts_for_non_su_procs(void __user **user_info);
void susfs_add_sus_kstat(void __user **user_info);
void susfs_update_sus_kstat(void __user **user_info);
void susfs_set_uname(void __user **user_info);
void susfs_enable_log(void __user **user_info);
void susfs_set_cmdline_or_bootconfig(void __user **user_info);
void susfs_add_open_redirect(void __user **user_info);
void susfs_add_sus_map(void __user **user_info);
void susfs_set_priority_uid_filter(void __user **user_info);
void susfs_set_avc_log_spoofing(void __user **user_info);
void susfs_get_enabled_features(void __user **user_info);
void susfs_show_variant(void __user **user_info);
void susfs_show_version(void __user **user_info);
```

这是“drop-in 编译失败”的第一层：2.4.0 的 `fs/susfs.c` 函数签名与 1.5.11 的 `susfs_cmd.c` 桥完全不兼容；必须重写桥（或整个 `susfs.c` 的入口层）。

---

## 3. `fs/susfs.c` 架构差异

### 3.1 初始化与生命周期

| 项目 | `[1.5.11]` | `[2.4.0]` |
|---|---|---|
| `susfs_init()` | `fs/susfs.c:1314-1321`：`spin_lock_init(&susfs_spin_lock)`、`susfs_my_uname_init()`、打印版本；无 workqueue | `[2.4.0] fs/susfs.c:2552-2559`：仅 `INIT_WORK(&susfs_extra_works, susfs_run_extra_works)`；大量状态用 `DEFINE_STATIC_KEY_*`/静态变量初始化 |
| boot completed | `bool susfs_is_boot_completed_triggered`（`fs/susfs.c:22`），实际在 `kernelsu_init()` 里提前置 true（`KernelSU/kernel/ksu.c:282-286`） | 无此全局；用 `susfs_is_sdcard_android_data_not_decrypted` static key（`[2.4.0] susfs.c:2373`）表示“boot 早期 / sdcard 未解密” |
| 额外 work | 无 | `susfs_run_extra_works()`（`[2.4.0] susfs.c:2543-2550`）遍历 `SUS_PATH_LOOP`，对每个路径用 `ksu_cred`/`override_creds` 重新 `kern_path` + 打 `AS_FLAGS_SUS_PATH`；由 KSU setuid 路径通过 `susfs_run_extra_works` 调度 |
| sdcard 监控 | 无 | `DEFINE_STATIC_KEY_TRUE(susfs_is_sdcard_android_data_not_decrypted)`（`:2373`）；`susfs_sdcard_cleanup_fn()`（`:2394`）、`susfs_sdcard_monitor_fn()`（`:2492`）、`susfs_start_sdcard_monitor_fn()`（`:2531`）启动 kthread + fsnotify，监听 `/data/media/0/Android` 出现后关闭 static key |
| 版本打印 | `SUSFS_VERSION "v1.5.11"` | `SUSFS_VERSION "v2.4.0"`；`susfs_show_variant()`/`susfs_show_version()` 返回结构体 |

**4.19 影响**：`susfs_init()` 本身可移植；但 sdcard monitor 的 `struct fsnotify_ops` 在 4.19 只有 `handle_event` 回调，没有 `.handle_inode_event`（`include/linux/fsnotify_backend.h:116-127`）。2.4.0 的 `fsnotify_ops` 定义（`[2.4.0] susfs.c:2430-2452`）在 4.19 会编译失败；必须改写成 4.19 的 `int (*handle_event)(struct fsnotify_group *, struct inode *, u32, const void *, int, const unsigned char *, u32, struct fsnotify_iter_info *)` 签名。`fsnotify_alloc_group()` 在 4.19 是 1 参数，2.4.0 已有版本分支，可保留。`fsnotify_init_mark(mark, group)` 在 4.19 是 2 参数、`fsnotify_destroy_group()` 返回 void，2.4.0 的调用需核对。

### 3.2 任务/进程跟踪

| 项目 | `[1.5.11]` | `[2.4.0]` |
|---|---|---|
| TIF 标志 | 仅 `TIF_PROC_UMOUNTED 33`（`include/linux/susfs_def.h:54`） | `TIF_PROC_UMOUNTED 33`、`TIF_PROC_NO_SU 34`、`TIF_PROC_UMOUNTED_FOR_ZYGOTE_NEXT 35`（`[2.4.0] susfs_def.h:74-76`） |
| 读 helper | `susfs_is_current_proc_umounted()` 用 `test_ti_thread_flag(&current->thread_info, ...)`（`1.5.11 def.h:80-82`） | `test_thread_flag()`（`2.4.0 def.h:121-123`）；新增 `susfs_is_current_proc_no_su()`、`susfs_is_current_proc_umounted_app()`、`susfs_is_current_proc_umounted_for_zygote_next()` |
| 写 helper | 只有 `susfs_set_current_proc_umounted()`，无调用者 | `susfs_set/clear_current_proc_umounted()`、`..._no_su()`、`..._for_zygote_next()` 全套（`2.4.0 def.h:125-170`） |
| UID 判断 | `susfs_is_inode_sus_path()` 比较 `inode->i_uid` 与 `current_uid()`（`1.5.11 fs/susfs.c:428-500`） | `susfs_is_current_app_uid()`：`current_uid().val % 100000 >= 10000`（`2.4.0 def.h:113-115`）；sus_path 再叠加 `susfs_is_uid_allowed_for_sus_path()` 位图 |
| KSU 设置点 | 无（本仓库 `setuid_hook.c` 不调用） | `10_enable…patch:2594-2790` 改 `kernel/hook/setuid_hook.c`：`handle_zygote_setresuid()` / `handle_zygote_next_setresuid()` 按 SID 设置上述 TIF；`susfs_set_current_proc_no_su()` 在 sucompat/exec 路径设置 |

**4.19 影响**：`test_thread_flag()` 在 4.19 存在；TIF 33-35 在 arm64 64 位 `thread_info.flags` 中可用，且与当前 KSU 使用的 `TIF_KSU_UNMOUNTABLE(61)`/`TIF_KSU_MANAGED(60)`（`KernelSU/kernel/policy/app_profile.h:1-10`）不冲突。关键缺口不是位定义，而是**没有调用者**：必须把 2.4.0 的 setuid hook 逻辑移植到本仓库 `KernelSU/kernel/hook/setuid_hook.c` 的 `ksu_handle_setresuid_cred()`（当前签名 `(struct cred *new, const struct cred *old)`，见 `:7-27`）以及 `KernelSU/kernel/feature/sucompat.c` 的 `ksu_handle_execveat()`（`:331-353`）。否则 2.4.0 `susfs.c` 的所有 `susfs_is_current_proc_umounted()/no_su()` 分支永远是 false。

### 3.3 `sus_path`

`[1.5.11]` 实现（`fs/susfs.c:134-500`）：

- 三个列表：`LH_SUS_PATH_LOOP`、`LH_SUS_PATH_ANDROID_DATA`、`LH_SUS_PATH_SDCARD`（`fs/susfs.c:120-132`）。
- `susfs_set_i_state_on_external_dir()`（`:63-117`）设置 `AS_FLAGS_ANDROID_DATA_ROOT_DIR`/`AS_FLAGS_SDCARD_ROOT_DIR`；`susfs_add_sus_path()`（`:134-258`）把路径拆成 `d_name` 存到对应列表，并检查 `android_data_path.pathname`/`sdcard_path.pathname` 前缀。
- `susfs_add_sus_path_loop()`（`:260-346`）检查路径存在后加入 loop 列表；`susfs_run_sus_path_loop(uid_t uid)`（`:348-377`）在 KSU 域下重新解析路径并设置 inode 的 `AS_FLAGS_SUS_PATH`。
- `susfs_is_base_dentry_android_data_dir()`（`:379`）、`susfs_is_sus_android_data_d_name_found()`（`:387`）、`susfs_is_inode_sus_path()`（`:428/438/448`，按内核版本分支）。
- 读路径在 `fs/namei.c`/`fs/readdir.c` 中大量依赖 `ND_STATE_LAST_SDCARD_SUS_PATH`（`1.5.11 def.h:66-68`）和 `susfs_is_base_dentry_android_data_dir()`。

`[2.4.0]` 实现（`fs/susfs.c:228-830`）：

- 只有一个 RCU list `LH_SUS_PATH_LOOP` + `DEFINE_STATIC_SRCU(susfs_srcu_sus_path_loop)`（`:229-231`），没有 android_data/sdcard 两个特殊列表，没有 `st_android_data_path`/`st_sdcard_path`，没有 `AS_FLAGS_ANDROID_DATA_ROOT_DIR`/`AS_FLAGS_SDCARD_ROOT_DIR`。
- `susfs_add_sus_path(void __user **)`（`:234-284`）：`kern_path()` + 对普通 inode 直接 `set_bit(AS_FLAGS_SUS_PATH, &inode->i_mapping->flags)`；FUSE 时用 `get_fuse_inode(inode)` 同时设置 `fi->inode.i_mapping`（`:250-270`）。不再需要 `target_ino`/`i_uid`。
- `susfs_add_sus_path_loop()`（`:286-319`）：只把路径字符串放进 `st_susfs_sus_path_list`，不调用 `kern_path`、不检查存在性。
- `susfs_run_sus_path_loop()`（`:321-359`）：在 `SUS_PATH_LOOP` 上遍历并设置 inode 标志；受 `susfs_is_current_proc_umounted()` 控制。
- `susfs_is_inode_sus_path()`（`:364-400`）：只检查 `test_bit(AS_FLAGS_SUS_PATH, &inode->i_mapping->flags)`；若 `current_uid().val >= 10000`，再调用 `susfs_is_uid_allowed_for_sus_path(uid)`。
- UID 位图：`susfs_set_sus_path_uid_filter()`（`:606-700`）、包名解析 `susfs_parse_sus_path_app_names()`（`~722`）、`susfs_parse_sus_path_app_strings()`（`:793-806`）、`susfs_is_uid_allowed_for_sus_path()`（`:808-830`）；`SUSFS_UID_FILTER_MIN_UID/MAX_UID` 为 10000/19999，bitmap 用 RCU 发布，包名列表来自 `/data/system/packages.list`。
- 读路径不再处理 `android_data`/`sdcard` 的 `d_name`，`fs/namei.c`/`fs/readdir.c` 只对每个 inode 检查 `susfs_is_inode_sus_path()`（`[2.4.0] 50_add…patch:196-638`）。

**4.19 影响**：
- `susfs_add_sus_path()` 的 FUSE 分支依赖 `fs/fuse/fuse_i.h` 的 `struct fuse_inode` 和 `get_fuse_inode()`；4.19 有这些符号，结构布局为 `struct fuse_inode { struct inode inode; ... }`，可移植。
- 2.4.0 的 `susfs_is_inode_sus_path()` 只用一个参数；1.5.11 当前树按 `LINUX_VERSION_CODE` 有 3 个版本（`fs/susfs.c:428/438/448`），4.19 使用第三版。移植时应直接采用 2.4.0 的单版本实现，删除 1.5.11 的 `i_uid` 比较。
- 关键行为差异：2.4.0 不设置 `AS_FLAGS_ANDROID_DATA_ROOT_DIR`/`AS_FLAGS_SDCARD_ROOT_DIR`，所以 4.19 的 `fs/namei.c`/`fs/readdir.c` 中所有基于这两个标志的 1.5.11 分支必须删除，否则这些标志永远为 0，代码只是无效但不会错；更大的风险是保留 1.5.11 的 `susfs_is_base_dentry_android_data_dir()` 调用却已无 `st_android_data_path` 数据来源（编译会引用已删除结构）。

### 3.4 `sus_mount`

`[1.5.11]`（`fs/susfs.c:503-653`；`fs/namespace.c:130-420, 3790-3830`）：

- 用户列表 `LH_SUS_MOUNT`，每项 `st_susfs_sus_mount { target_pathname, target_dev }`；`susfs_add_sus_mount()` 检查 `target_dev` 是否等于当前 mount 的 `mnt->mnt_sb->s_dev`。
- `susfs_update_sus_mount_inode()` 设置 `AS_FLAGS_SUS_MOUNT`。
- `susfs_auto_add_sus_bind_mount()`（`:554`）、`susfs_auto_add_sus_ksu_default_mount()`（`:577`）根据 `/data/adb/susfs_no_auto_add_*` 文件决定是否自动加；这两个函数没有 Kconfig，实际死代码。
- `fs/namespace.c:3795` `susfs_reorder_mnt_id()`、`:3814` `susfs_is_mnt_devname_ksu()` 也未被调用。
- `mnt_free_id()`（`fs/namespace.c:130-150`）依赖 `mnt->mnt.susfs_mnt_id_backup`；`DEFAULT_KSU_MNT_ID 300000`、`DEFAULT_KSU_MNT_GROUP_ID 3000`（`1.5.11 def.h:43-44`）；`susfs_alloc_sus_vfsmnt()`（`fs/namespace.c:293-355`）把所有 fake mount 的 `mnt_id` 固定为 `DEFAULT_KSU_MNT_ID`。
- `fs/namespace.c:167/198` 用 `susfs_is_boot_completed_triggered` 控制 KSU mount id/group id 的初始化；`fs/namespace.c:1139/1211` 在 `mntput`/`mnt_free_id` 路径处理 backup。

`[2.4.0]`（`[2.4.0] fs/susfs.c:838-868`；`50_add…patch:638-1090`）：

- 完全没有用户列表和 `susfs_add_sus_mount()`；只剩 `susfs_set_hide_sus_mnts_for_non_su_procs(void __user **)`（`susfs.c:838-868`），设置 `DEFINE_STATIC_KEY_FALSE(susfs_is_hide_sus_mnts_for_non_su_procs_enabled)`。
- `fs/namespace.c` 在 `vfs_create_mount()`/`alloc_vfsmnt()`/`clone_mnt()` 中给“KSU 域进程创建的 mount”分配 fake mnt_id：`susfs_alloc_non_unshare_ksu_vfsmnt()`（`50_add…patch:758-818`）用 `ida_alloc_min(&mnt_id_ida, DEFAULT_KSU_MNT_ID=2000000000)`；`susfs_alloc_unshare_ksu_vfsmnt()`（`:690-756`）复用 `old->mnt_id` 并置 `VFSMOUNT_MNT_FLAGS_KSU_UNSHARED_MNT 0x80000000`。
- `mnt_alloc_group_id()` 改为 `ida_alloc_min(&mnt_group_ida, DEFAULT_KSU_MNT_GROUP_ID=200000)`（`50_add…patch:638-690`）。
- `__lookup_mnt()`（`:821-840`）在 `susfs_is_current_proc_umounted()` 时跳过 fake mnt；`copy_mnt_ns()` 加 `CL_COPY_MNT_NS`（`:3730`）；`get_anon_bdev()` 从 minor `1<<12=4096` 起分配（`:2222` 起，`fs/super.c`）。
- 提供 `susfs_get_non_sus_mnt_id_from_mnt()` / `susfs_get_non_sus_vfsmnt_from_vfsmnt()`（`50_add…patch:4980-5030` 末尾），供 kstat/statfs/proc fd 使用。
- 由 `susfs_is_sdcard_android_data_not_decrypted` static key 控制：boot 早期为 true，`/data/media/0/Android` 可访问后关闭；此后不再给新 mount 分配 fake id（性能）。

**4.19 影响（最高风险区）**：
1. `fs/namespace.c` 的 4.19 `struct mount` 没有 `mnt_userns`、`mnt_stuck_children`；2.4.0 的 `susfs_alloc_*` 直接写 `mnt->mnt.mnt_userns = &init_user_ns`、`INIT_HLIST_HEAD(&mnt->mnt_stuck_children)`，**编译失败**。必须删掉这些字段初始化（4.19 没有 user namespace mount 字段）。
2. 4.19 没有 `struct fs_context`/`vfs_create_mount()`；2.4.0 的 `vfs_create_mount()` 钩子（`50_add…patch:1179-1210`）不存在。4.19 的挂载入口是 `vfs_kern_mount()`/`sget_userns()`/`attach_recursive_mnt()`，必须把 fake mnt 分配逻辑迁到 `vfs_kern_mount()`/`clone_mnt()` 或 `alloc_vfsmnt()` 调用点。
3. 4.19 的 `alloc_vfsmnt()` 会 `init_fs_pin(&mnt->mnt_umount, drop_mountpoint)`（`fs/namespace.c:341-342`）；2.4.0 的 `susfs_alloc_*` 副本没有这一行，直接照抄会导致 `mnt_umount` 未初始化，后续 `mntput`/umount 崩。必须补 `init_fs_pin()`（以及 4.19 的 `mnt->mnt.data = NULL` 等初始化）。
4. 4.19 的 `mnt_free_id()` 已经有 1.5.11 的 `susfs_mnt_id_backup` 逻辑；2.4.0 的 `mnt_free_id()` 只判断 `mnt_flags & VFSMOUNT_MNT_FLAGS_KSU_UNSHARED_MNT`。合并时必须同时处理：`susfs_mnt_id_backup == DEFAULT_KSU_MNT_ID`、`susfs_mnt_id_backup != 0`、`mnt_flags & KSU_UNSHARED` 三种情况，否则会 `ida_free` 错 id 或泄漏 id。
5. 4.19 的 `mnt_id` 是 `int`（`include/linux/mount.h`/`fs/mount.h`），`DEFAULT_KSU_MNT_ID 2000000000` 仍在 int 范围内（2e9 < 2^31-1），可用；`mnt_group_id` 200000 也安全。
6. 4.19 的 `copy_mnt_ns()` 有 `CL_COPY_*` 但是否有 `CL_COPY_MNT_NS` 需新增；`fs/namespace.c` 的 `struct mnt_namespace` 字段与 6.1 不同，不能照抄 hunk。
7. 4.19 的 `fs/proc_namespace.c` show_mountinfo 使用 `mnt->mnt_id`/`mnt_group_id`；2.4.0 的伪造逻辑必须适配，否则 `/proc/self/mountinfo` 泄露 fake id。

### 3.5 `sus_kstat`

`[1.5.11]`（`fs/susfs.c:654-837`）：

- 哈希表 `SUS_KSTAT_HLIST`，每项 `st_susfs_sus_kstat_hlist { target_ino, info, node }`；以 `target_ino` 为 key。
- `susfs_update_sus_kstat_inode()` 设置 `AS_FLAGS_SUS_KSTAT`；`susfs_add_sus_kstat()` 检查路径/inode；`susfs_update_sus_kstat()` 更新。
- `susfs_sus_ino_for_generic_fillattr()`（`:800-820`）在 `generic_fillattr` 中替换 `ino`/`dev`/`mode`/`nlink`/`size`/`blocks`/时间戳；`susfs_sus_ino_for_show_map_vma()`（`:822-836`）只在 `/proc/pid/maps` 的 `show_map_vma` 中替换 `dev`/`ino`。
- 只支持“整组字段覆盖”，没有 `flags` 位掩码；不支持 FUSE、statfs、mnt_id、inotify fdinfo、proc fd。

`[2.4.0]`（`fs/susfs.c:832-1305`）：

- 表项 `st_susfs_sus_kstat_hlist` 增加 `target_dev`、`spoofed_kstatfs`、`spoofed_mnt_id`、`is_fuse`；key 从 `target_ino` 升级为 `(target_ino, target_dev)`（FUSE 双 inode 兼容）。
- `susfs_mark_inode_sus_kstat()`（`:870-941`）：支持普通 inode 和 FUSE inode，保存原始 `struct kstatfs`（`spoofed_kstatfs`）、`spoofed_mnt_id`，并设置 `AS_FLAGS_SUS_KSTAT`。
- `susfs_add_sus_kstat()`（`:942-1054`）/`susfs_update_sus_kstat()`（`:1056-1108`）：`st_susfs_sus_kstat` 的 `flags` 决定哪些字段覆盖；`is_statically` 允许“静态 inode”模式；FUSE 时检测 `is_fuse`。
- 伪造函数：
  - `susfs_sus_kstat_spoof_generic_fillattr()`（`:1136-1199`）：`generic_fillattr`。
  - `susfs_sus_kstat_spoof_show_map_vma()`（`:1201-1242`）：`/proc/pid/maps`。
  - `susfs_sus_kstat_spoof_vfs_statfs()`（`:1244-1263`）：`statfs` 时把 fake mount 映射回真实 mount，并用 `spoofed_kstatfs` 覆盖。
  - `susfs_sus_kstat_spoof_inotify_fdinfo()`（`:1265-1280`）：`/proc/pid/fdinfo` 的 inotify 信息。
  - `susfs_sus_kstat_spoof_proc_fd_seq_show()`（`:1282-1304`）：`/proc/pid/fd` 的 `mnt_id`/`ino`。
- `susfs_is_inode_sus_kstat(struct inode *, bool *out_is_fuse)`（`:1110-1134`）供 `fs/notify/fdinfo.c`、`fs/proc/fd.c` 调用。

**4.19 影响**：
- `struct kstat` 在 4.19 没有 `mnt_id` 字段（`include/linux/stat.h:24-50`），2.4.0 中给 `stat->mnt_id` 赋值必须放在 `#if LINUX_VERSION_CODE >= KERNEL_VERSION(5,10,0)` 内或直接删掉；`stat->blksize` 是 `u32`（4.19），2.4.0 用 `long`，赋值会截断但不至于编译失败。
- `statfs_by_dentry()`/`calculate_f_flags()` 在 4.19 `fs/statfs.c` 是 static（`fs/statfs.c:57`、`:197` 起），2.4.0 在 `fs/statfs.c` 末尾添加 `statfs_by_dentry_wrapper()`/`calculate_f_flags_wrapper()` 并 extern 给 `susfs.c`（`[2.4.0] fs/susfs.c:867-868`）。4.19 可移植，但 `calculate_f_flags()` 的参数在 4.19 是 `struct vfsmount *`（6.x 是 `struct mount *`/`mnt_idmap`），需按 4.19 签名改写。
- `vfs_statfs()` 在 4.19 存在但 1.5.11 已经改过 `fs/statfs.c`；2.4.0 的 `susfs_statfs_by_dentry()` 需要重新接线，不能直接用 6.1 的 `dentry->d_sb->s_op->statfs` 版本。
- `fs/notify/fdinfo.c` 在 4.19 已经有 `struct file *file` 参数（本仓库当前 1.5.11 补丁已改过，`fs/notify/fdinfo.c:28-33`），2.4.0 的 `show_fdinfo` 改写可复用；`inotify_fdinfo` 的签名与 4.19 匹配（`:90-92`）。
- `fs/proc/fd.c` 的 `seq_show` 在 4.19 存在，2.4.0 增加 `mnt_id`/`ino` 输出；`real_mount()`、`file_inode()` 可用。
- `fs/proc/task_mmu.c` 的 `show_map_vma` 在 4.19 有；`VMA_PAD_START` 在本 4.19 树**不存在**（`grep -R VMA_PAD_START` 无结果），2.4.0 的 open redirect/maps 输出格式不能照抄，需用 `vma->vm_start` 直接输出。
- `fsnotify` 的 `susfs_is_inode_sus_kstat()` 调用点（2.4.0 `fs/notify/fdinfo.c`）在 4.19 可编译，但需把 `<linux/fsnotify_backend.h>` 的 `fsnotify_mark` 结构差异（4.19 有 `mask` 字段）对齐。

### 3.6 SELinux hiding

`[1.5.11]`：

- `fs/susfs.c:1306-1313` 只有 `susfs_set_avc_log_spoofing()`，保存一个全局 bool；`security/selinux/avc.c` 的 `avc_dump_query` 在该 bool 为真时把 `tsid` 从 `susfs_ksu_sid` 替换成 `susfs_kernel_sid`（1.5.11 补丁的 `security/selinux/avc.c` hunk）。
- 没有 `susfs_add_sus_type()`、没有 `susfs_add_sus_avd_allowed_by_type()`、没有 `susfs_switch_to_ksu_domain()`/`susfs_restore_creds()`；`susfs_is_current_ksu_domain()` 是 `fs/susfs.c:26-29` 的 `return false`。
- 没有修改 `security/selinux/hooks.c`、`selinuxfs.c`、`ss/services.c`、`ss/status.c`；SELinux hide 完全由 KernelSU 的 `kernel/feature/selinux_hide.c` 自带运行时 hook 完成（本仓库 `KernelSU/kernel/feature/selinux_hide.c`，377 行）。

`[2.4.0]`（`fs/susfs.c:53-225` + `50_add…patch:2458-3430`）：

- `susfs_add_sus_type(const char *secctx, u32 type_value)`（`:58-83`）：把 KSU 新增的 SELinux type 注册到 `SUS_TYPE_HLIST`；`susfs_is_sus_type_rcu()`（`:85-92`）判断某 type 是否为“SUS 类型”。
- `susfs_add_sus_avd_allowed_by_type(u32 s_type, u32 t_type, u32 c_type, u32 orig_avd_allowed)`（`:121-160`）：记录原始 `avd->allowed`；`susfs_spoof_avd_allowed_rcu()`（`:162-176`）在 `security_compute_av` 时恢复原始 allowed，实现“策略查询时看不到 KSU 添加的规则”。
- `susfs_switch_to_ksu_domain()`（`:186-214`）/`susfs_restore_creds()`（`:220-225`）：用 `prepare_creds()` + `setup_selinux("u:r:ksu:s0", cred)` + `commit_creds()` 切换到 KSU 域，读取 `/data/system/packages.list` 后恢复。
- `security/selinux/ss/services.c`：新增 `susfs_string_to_context_struct()`、`susfs_security_context_to_sid*()`、`susfs_security_compute_av_user()` 等一整套函数，并在 `security_context_to_sid_core()`、`security_compute_av()` 中插入 `susfs_is_sus_type_rcu()`/`susfs_spoof_avd_allowed_rcu()`；同时修改 `selinux_policy_commit()`/`security_set_bools()` 的 `latest_granting` 更新逻辑，在 KSU 域内不递增 seqno（`50_add…patch:3155-3215`）。
- `security/selinux/hooks.c`：新增 `susfs_selinux_inode_setxattr()`、`susfs_selinux_inode_post_setxattr()`、`susfs_selinux_setprocattr()`，在 `current_uid().val >= 10000 && ksu_selinux_hide_running` 时绕过 SELinux 检查（`50_add…patch:2501-2870`）。
- `security/selinux/selinuxfs.c`：新增 `susfs_sel_write_context()`、`susfs_sel_write_access()`、`handle_sel_write_access()`；`SEL_STATUS` 的 fake status page 由 KSU `fake_status`/`initialize_fake_status()` 提供（`50_add…patch:2872-3075`）。
- `security/selinux/ss/status.c`：在 KSU 域内跳过 status page 的 seqno 更新（`50_add…patch:3390-3430`）。
- `security/selinux/avc.c`：`tsid` 从 ksu sid 替换为 `priv_app_sid`（不是 1.5.11 的 `kernel_sid`）；`susfs_set_avc_log_spoofing()` 仍保留命令（`fs/susfs.c:2214-2249`）。

**4.19 影响（第二大风险区）**：
1. `struct selinux_state` 在 4.19 只有 `avc`、`ss` 指针（`security/selinux/include/security.h:101-116`），没有 `policy`、`policy_mutex`、`status_page`；2.4.0 的 `state->policy` 访问全部编译失败，必须改为 `state->ss->policydb` / `state->ss->sidtab` + `policy_rwlock`。
2. 4.19 的 status page 在 `security/selinux/ss/status.c`（不是 `security/selinux/status.c`），字段是 `state->ss->status_page`；2.4.0 补丁路径 `security/selinux/status.c` 在本树不存在，必须映射到 `ss/status.c` 并改字段。
3. 4.19 的 `selinux_inode_setxattr()` 签名是 `(struct dentry *dentry, const char *name, const void *value, size_t size, int flags)`，没有 6.1 的 `struct user_namespace *mnt_userns` 首参；`selinux_inode_post_setxattr()` 同样没有 `mnt_userns`；`inode_owner_or_capable()` 是 1 参。2.4.0 的 hooks 必须去掉 `mnt_userns` 参数（`security/selinux/hooks.c:3237-3345`）。
4. 4.19 的 `selinux_setprocattr()` 签名与 2.4.0 相同（`(const char *name, void *value, size_t size)`，`hooks.c:6407`），可保留，但内部 `current_sid()`/`selinux_cred()` 的字段名需核对。
5. 4.19 的 `security_read_state_kernel()` 在 `ss/services.c` 可能不存在；2.4.0 在文件末尾追加的大段函数需改挂到 4.19 的 `security_context_to_sid_core()` 实现位置。
6. 本仓库的 `KernelSU/kernel/feature/selinux_hide.c` 已有一套自己的 fake status page/runtime hook（`ksu_fake_status_page`、`ksu_prepare_fake_status_page()`、`ksu_sel_open_handle_status()`，见 `selinux_hide.c:182-240`），与 2.4.0 的 `fake_status`/`initialize_fake_status()` 设计**不兼容**。移植时必须二选一：要么改造本仓库的 `selinux_hide.c` 暴露 2.4.0 需要的符号（`ksu_selinux_hide_running`、`fake_status`、`initialize_fake_status`、`fake_status_initialize_key`、`ksu_selinux_hide_handle_second_stage/post_fs_data`），要么放弃 2.4.0 的 `selinuxfs.c`/`hooks.c` 补丁，仅保留本仓库现有实现。两者同时启用会重复 hook/竞争。

### 3.7 priority UID filter

`[1.5.11]`：无对应实现。

`[2.4.0]`（`fs/susfs.c:1983-2249`）：

- `st_susfs_priority_uid_filter { char app_strings[65535]; int err; }`；位图 `susfs_priority_uid_filter_bitmap`（`rcu_head + bits[157]`，10000-19999）。
- `susfs_set_priority_uid_filter()`（`:1987-2100`）：`kvmalloc` 读取用户结构，校验 NUL、解析包名、切换 KSU 域调用 `susfs_parse_priority_app_names()` 得到 UID，再 RCU 发布位图。
- `susfs_parse_priority_app_names()` 用 `/data/system/packages.list`（`open`/`read`/`strtok`）解析 uid；`susfs_priority_app_strings_worker()` 在包变化时异步刷新；`susfs_parse_priority_app_strings()`（`:2174-2187`）由 KSU 的 `pkg_observer.c` 调用。
- `susfs_is_priority_uid_filtered(uid_t uid)`（`:2189-2212`）：uid 10000-19999 内查位图；供 `block/ioprio.c` 的 `ioprio_get/ioprio_set` 与 `kernel/sys.c` 的 `getpriority/setpriority` 使用（`50_add…patch` 中对应文件 hunk）。
- 语义：被过滤的 UID 从 `getpriority`/`ioprio_get` 结果中“消失”，用于防止 app 枚举已安装应用/后台进程。

**4.19 影响**：
- `block/ioprio.c` 在 4.19 有 `SYSCALL_DEFINE2(ioprio_get...)`/`ioprio_set`，但 6.1 的 `ioprio_get` 已改为 `struct ioprio` 等；2.4.0 hunk 需按 4.19 的 `int which, int who` 重写。
- `kernel/sys.c` 的 `SYSCALL_DEFINE2(getpriority...)` 在 4.19 存在，但 `setpriority` 的 `struct cred`/`kuid_t` 处理不同；需要逐行核对。
- 包名解析依赖 `/data/system/packages.list` 的 4.19 格式与 SELinux 访问；本仓库 KSU 的 `pkg_observer.c` 是 inode_rename 观察器（`KernelSU/kernel/manager/pkg_observer.c:26-104`），不是 2.4.0 的 `ksu_track_throne_tw_func`，刷新调用点要插在 `track_throne(false)` 之后。

### 3.8 open redirect

`[1.5.11]`（`fs/susfs.c:1090-1164`）：

- `st_susfs_open_redirect` 以 `target_ino` 为 key，`susfs_update_open_redirect_inode()` 设置 `AS_FLAGS_OPEN_REDIRECT`；`susfs_get_redirected_path(ino)` 返回目标路径。
- `fs/namei.c` 只在 `do_filp_open`/`vfs_readlink` 少量位置使用重定向；没有 uid_scheme，没有 `/proc/pid/maps`/`/proc/pid/map_files` 伪造，没有反向查找。

`[2.4.0]`（`fs/susfs.c:1463-1745`）：

- `st_susfs_open_redirect` 去掉 `target_ino`，加 `uid_scheme`（0-4）与 `err`；`st_susfs_open_redirect_hlist` 保存 `target_dev/redirected_dev`、`redirected_ino`、`spoofed_kstatfs`、`spoofed_mnt_id`、`reversed_lookup_only`。
- `susfs_add_open_redirect()`（`:1465-1623`）：双向哈希（target→redirected 与 redirected→target），用 `SUSFS_IS_INODE_OPEN_REDIRECT`/`..._WITHOUT_UID_CHECK`；FUSE 直接拒绝（`-EINVAL`）。
- 伪造函数：`susfs_open_redirect_spoof_do_sys_openat()`（`:1625-1668`，`do_sys_openat` 前替换路径）、`susfs_open_redirect_spoof_vfs_readlink()`（`:1670-1696`，readlink）、`susfs_open_redirect_spoof_do_proc_readlink()`（`:1698-1721`，`/proc/pid/fd/*`）、`susfs_open_redirect_spoof_show_map_vma_srcu()`（`:1723-1745`，`/proc/pid/maps`）。
- `uid_scheme` 语义：`UID_NON_APP_PROC(0)`、`UID_ROOT_PROC_EXCEPT_SU_PROC(1)`、`UID_NON_SU_PROC(2)`、`UID_UMOUNTED_APP_PROC(3)`、`UID_UMOUNTED_PROC(4)`（`[2.4.0] susfs.h:23-29`），控制哪些调用者看到重定向后的路径。

**4.19 影响**：
- `vfs_readlink()` 在 4.19 存在（`fs/namei.c`），签名 `int vfs_readlink(struct dentry *, char __user *, int)`，2.4.0 的 `susfs_open_redirect_spoof_vfs_readlink(struct inode *, char __user *, int)` 可在其内部调用；但 `do_proc_readlink()` 在 4.19 `fs/proc/base.c` 的签名与 6.1 可能不同（4.19 是 `static int do_proc_readlink(const struct path *, char __user *, int)`），2.4.0 的 `(struct inode *, char *, int)` 需改。
- `VMA_PAD_START` 不存在（见 3.5），`show_map_vma` 的重定向输出需按 4.19 格式重写。
- `do_sys_openat` 在 4.19 是 `SYSCALL_DEFINE4(openat...)` 内部调用 `do_filp_open`；2.4.0 的 `do_sys_openat` 钩子位置在 4.19 没有同名函数，需挂到 `do_filp_open()`/`path_openat()`（1.5.11 当前就是在 `path_openat` 里做）。
- `path_openat()`/`do_filp_open()` 的签名在 4.19 与 6.1 不同（`struct open_flags`），不能照抄 hunk。

### 3.9 sus_map

`[1.5.11]`：无实现；`CONFIG_KSU_SUSFS_SUS_MAP` 是死 Kconfig（见 1.3）。

`[2.4.0]`（`fs/susfs.c:1749-1795`）：

- `susfs_add_sus_map(void __user **)`：`kern_path()` + `set_bit(AS_FLAGS_SUS_MAP=39, &inode->i_mapping->flags)`。
- `SUSFS_IS_INODE_SUS_MAP(inode)` 宏要求 `susfs_is_current_proc_umounted_app()`（`2.4.0 def.h:159-163`）。
- 钩子：
  - `mm/memory.c` 的 `__access_remote_vm()`（`50_add…patch:2432-2455`）：`vma->vm_file` 的 inode 是 SUS_MAP 时 `break`，防止 `/proc/pid/mem` 读取。
  - `fs/proc/task_mmu.c`：`show_map_vma()` 返回；`show_smap()` 返回 0；`show_smaps_rollup()` 跳过 `smap_gather_stats`；`pagemap_read()` 跳过（`50_add…patch:1336-1712`）。
  - `fs/proc/base.c`：`proc_map_files_readdir()` 跳过（`50_add…patch:1155-1200`）。
- 4.19 影响：`__access_remote_vm()` 在 4.19 签名是 `(struct task_struct *tsk, struct mm_struct *mm, unsigned long addr, void *buf, int len, unsigned int gup_flags)`（`mm/memory.c:4472`），6.1 没有 `tsk`；`get_user_pages_remote()` 也多了 `tsk` 参数。`show_smaps_rollup` 在本 4.19 树存在（`fs/proc/task_mmu.c:900`），但 `pagemap_read` 的 4.19 实现不同；`VMA_PAD_START` 不存在。SUS_MAP 可以移植，但需要逐函数适配，不能直接打 2.4.0 的 `task_mmu.c` hunk。

### 3.10 uname / cmdline

`[1.5.11]`：

- `susfs_set_uname()`（`fs/susfs.c:974-997`）：保存 `release`/`version`（`my_uname` 静态 + spinlock）；`susfs_spoof_uname()`（`:999-1007`）在 `kernel/sys.c` 的 `newuname` 中复制。
- `susfs_set_cmdline_or_bootconfig()`（`:1024-1051`）：把用户字符串存入全局 `fake_cmdline`（4096）；`susfs_spoof_cmdline_or_bootconfig()`（`:1053-1090`）用 `seq_printf` 输出，但**没有 VFS/proc 消费者**（见 1.3）。
- `susfs_set_log()`（`:1009-1022`）设置全局 log 开关。

`[2.4.0]`：

- `susfs_set_uname()`（`:1306-1357`）：`seqlock_t susfs_uname_seqlock` + `DEFINE_STATIC_KEY_FALSE(susfs_is_uname_spoof_buffer_set)`；`"default"` 时清除 static key；`susfs_spoof_uname()`（`:1298-1305`）在 `kernel/sys.c` 的 `newuname` 中用 `static_branch_likely` 判断。
- `susfs_set_cmdline_or_bootconfig()`（`:1390-1450`）：`struct st_susfs_spoof_cmdline_or_bootconfig`（8196 字节）+ static key `susfs_is_fake_cmdline_or_bootconfig_buffer_set`；消费者是 `fs/proc/bootconfig.c`（`50_add…patch:1211-1240`）的 `boot_config_proc_show()`。
- `susfs_enable_log()`（`:1359-1388`）：`struct st_susfs_log`（bool+err），static key `susfs_is_log_enabled`。
- 4.19 影响：`fs/proc/bootconfig.c` 在 4.19 不存在（`ls fs/proc` 无 `bootconfig.c`）；必须改挂到 `fs/proc/cmdline.c` 的 `cmdline_proc_show()`，且 4.19 的 `/proc/cmdline` 读取 `saved_command_line`，需要用 `seq_replace`/`seq_puts` 输出 fake buffer。`newuname` 在 4.19 的 `SYSCALL_DEFINE1(newuname)` 与 6.1 基本一致，可移植；`seqlock_t`/`static_key` 在 4.19 可用。

### 3.11 boot completed / sdcard monitor

- `[1.5.11]`：`susfs_is_boot_completed_triggered` 只是普通 bool（`fs/susfs.c:22`），在 `kernelsu_init()` 提前置 true；没有 sdcard 监控；`fs/namespace.c:167/198` 的 KSU mount 逻辑因此从 init 后就不再走“boot 前”分支。
- `[2.4.0]`：`susfs_is_sdcard_android_data_not_decrypted` static key（`:2373`）在 boot 早期为 true；`susfs_start_sdcard_monitor_fn()`（`:2531-2541`）启动 `susfs_sdcard_monitor_fn` kthread（`:2492-2529`），用 fsnotify 监听 `/data/media/0/Android` 的创建/移动；`susfs_sdcard_cleanup_fn()`（`:2394-2412`）在检测到后清理 mark、关闭 static key。KSU 在 `EVENT_BOOT_COMPLETED` 的 `do_report_event()` 中调用 `susfs_start_sdcard_monitor_fn()`（`10_enable…patch:4125-4131`）。
- 4.19 影响：`fsnotify` API 差异见 3.1；另外 4.19 的 `/data/media/0/Android` 可能由 FUSE/SDCARDFS 提供，`fsnotify` 对 FUSE 目录事件的可靠性需实测；可先不启用该监控，用 `susfs_is_current_ksu_domain()` + mount 分配逻辑的静态开关替代。

### 3.12 4.19 直接替换的编译/链接阻断点（汇总）

| 2.4.0 依赖 | 2.4.0 位置 | 4.19 现实 | 影响 | 适配方向 |
|---|---|---|---|---|
| `struct fs_context` / `vfs_create_mount()` | `fs/namespace.c` 补丁 `:1179-1210` | 4.19 无 `fs_context.h`，用 `vfs_kern_mount()`/`sget_userns()` | 编译失败 | 在 `vfs_kern_mount()`/`clone_mnt()` 内插入 fake mnt 分配 |
| `mnt->mnt.mnt_userns`、`mnt_stuck_children` | `susfs_alloc_*` (`:690-818`) | 4.19 `struct mount` 无这些字段 | 编译失败 | 删除/改用 4.19 字段 |
| `struct mount` 的 `mnt_id`/`mnt_group_id` 类型 | `fs/namespace.c` 全部 | 4.19 为 `int`，可用 | 无编译问题；需保证 2e9 不溢出 | 保留 `DEFAULT_KSU_MNT_ID 2000000000` |
| `i_uid_into_mnt()`/`i_uid_into_vfsuid()`/`vfsuid_t` | `fs/namei.c`/`fs/stat.c` 补丁 | 4.19 无 idmap/vfsuid | 编译失败 | 用 `inode->i_uid.val`/`current_uid().val` 替代 |
| `state->policy`/`struct selinux_policy` | `security/selinux/ss/services.c` | 4.19 `state->ss->policydb`/`sidtab` | 编译失败 | 重写 SELinux 辅助函数 |
| `security/selinux/status.c` | 2.4.0 补丁路径 | 4.19 是 `security/selinux/ss/status.c`；`state->ss->status_page` | 文件不存在 | 映射到 `ss/status.c` |
| `selinux_inode_setxattr(..., struct user_namespace *mnt_userns, ...)` | `security/selinux/hooks.c` | 4.19 无 `mnt_userns` 首参；`inode_owner_or_capable()` 1 参 | 编译失败 | 去掉 `mnt_userns` 参数 |
| `struct fsnotify_ops.handle_inode_event` | `fs/susfs.c` sdcard monitor | 4.19 只有 `handle_event` | 编译失败 | 改为 4.19 `handle_event` 签名 |
| `statfs_by_dentry()`/`calculate_f_flags()` 非 static 包装 | `[2.4.0] fs/susfs.c:867-868`、`fs/statfs.c` | 4.19 原函数 static，签名/调用点不同 | 链接失败/编译失败 | 在 4.19 `fs/statfs.c` 增加 wrapper 并按 4.19 参数改写 |
| `VMA_PAD_START` | `fs/proc/task_mmu.c` open redirect | 本 4.19 树无定义 | 编译失败 | 用 `vma->vm_start`/现有格式 |
| `do_faccessat(dfd, struct filename *, ...)` 4 参 | `fs/open.c` 补丁 `:1094-1155` | 4.19 `do_faccessat(int dfd, const char __user *filename, int mode)` 3 参 | 编译失败 | 保留 1.5.11 的 3 参 hook 结构，按 2.4.0 语义加 `susfs_is_current_proc_no_su()` 早退 |
| `get_user_pages_remote(mm, ...)` | `mm/memory.c` | 4.19 多 `struct task_struct *tsk` 首参 | 编译失败 | 传入 `tsk` |
| `initial_idmapping()`/`mnt_idmapping.h` | `fs/namespace.c`/`fs/open.c` | 4.19 无 | 编译失败 | 删除/用 `&init_user_ns` 或无需判断 |
| `fsnotify_alloc_group(ops, 0)` | `fs/susfs.c` | 4.19 1 参（2.4.0 已有版本分支） | 视分支而定 | 固定 1 参 |
| `struct kstat.mnt_id` | `fs/stat.c` | 4.19 无该字段 | 编译失败（若未 ifdef） | 用 `#if LINUX_VERSION_CODE >= 5.10` 包裹或删除 |
| `struct kstatfs` 字段 | `susfs.c` | 4.19 字段基本一致（`f_flags` 等） | 可编译 | 保留 |
| `strscpy`/`kvmalloc`/`strim`/`ida_alloc_min`/`seqlock`/`static_key` | 全文件 | 4.19 全部可用 | 无 | 保留 |

### 3.13 其他架构差异

- **锁模型**：1.5.11 用单一 `static spinlock_t susfs_spin_lock`（`fs/susfs.c:20`）保护所有 list；2.4.0 每个子系统独立 mutex + RCU/SRCU：`susfs_mutex_lock_sus_path`、`..._sus_path_uid_filter`、`..._sus_kstat`、`..._open_redirect`、`..._priority_uid_filter`；路径遍历用 `DEFINE_STATIC_SRCU(susfs_srcu_sus_path_loop)`（`[2.4.0] susfs.c:229-231`），open redirect 用 `DEFINE_SRCU(susfs_srcu_open_redirect)`（`[2.4.0] fs/susfs.c:1463`）。4.19 有 SRCU，可移植，但需要重新审计所有读写侧。
- **内存模型**：2.4.0 的 sus_path UID 位图、priority 位图、kstat 表、open redirect 表都使用 `kfree_rcu()`；4.19 有 `kfree_rcu()`（2 参数宏/函数），可用。
- **FUSE 支持**：2.4.0 统一在 `susfs_add_sus_path()`/`susfs_mark_inode_sus_kstat()`/`susfs_add_open_redirect()` 中处理 `FUSE_SUPER_MAGIC`；1.5.11 没有 FUSE 分支。4.19 的 FUSE 头文件有 `get_fuse_inode()`，但 Android 4.19 可能用 sdcardfs 而非 FUSE，需确认实际挂载类型。
- **mount id 策略**：1.5.11 固定 `mnt_id=300000` 的“单值”模型；2.4.0 每 mount 从 2e9 起分配唯一 id，且 unshare 复用旧 id + flag。4.19 的 `mnt_id` 是 `int`，2e9 可行但离 `INT_MAX` 仅 14.7 亿，足够。
- **`susfs_is_current_ksu_domain()`**：1.5.11 是 false stub；2.4.0 是 KSU 提供的 SID 比较函数（`10_enable…patch:3783-3793`）。4.19 本仓库的 `kernel/selinux/selinux.c` 目前只有 `is_ksu_domain()`/`is_task_ksu_domain()`，没有 `susfs_is_current_ksu_domain`/`susfs_is_sid_equal`/`susfs_set_batch_sid`/`susfs_*_sid`；必须新增。
- **`susfs_extra_works`**：2.4.0 在 `susfs_set_current_proc_umounted()` 后调度（`10_enable…patch:2594-2790`），用于把 `SUS_PATH_LOOP` 的路径在 KSU 域下重新打标；1.5.11 没有，`susfs_run_sus_path_loop(uid_t uid)` 也没有被调用（`fs/susfs.c:348`）。4.19 移植必须补这条“loop 路径”生效链，否则 `susfs_add_sus_path_loop` 只是存字符串。

---

## 4. KernelSU 集成接口差异（`10_enable_susfs_for_ksu.patch`）

### 4.1 2.4.0 补丁假设的 KernelSU 树结构 vs 本仓库

2.4.0 的 `10_enable_susfs_for_ksu.patch` 修改 32 个文件，其上下文明显是针对一个“模块化”的 KernelSU 树；本仓库是 unity build（`KernelSU/kernel/Makefile`：`kernelsu-y := ksu.o`，各 `.c` 通过 `#include` 串联）。对照如下：

| 2.4.0 补丁中的文件 | 本仓库对应 | 状态 |
|---|---|---|
| `kernel/Kbuild`（`:1-36`） | `KernelSU/kernel/Makefile`（unity build，`kernelsu-y := ksu.o`） | **不存在**；2.4.0 的 Kbuild 修改无意义 |
| `kernel/Kconfig`（`:37-161`） | `KernelSU/kernel/Kconfig:147-245` | 存在，但选项内容不同（见 1.2） |
| `kernel/Makefile`（`:162-180`） | 同 `KernelSU/kernel/Makefile` | 存在但结构不同（2.4.0 打印 `SUSFS_VERSION`，可加到 `ksu.c` 编译日志） |
| `kernel/core/init.c`（`:181-300`） | `KernelSU/kernel/ksu.c` 的 `kernelsu_init()` | **无 `core/init.c`**；2.4.0 在此调用 `susfs_init()`，本仓库已在 `ksu.c:282-286` 调用 |
| `kernel/feature/adb_root.c`（`:410-449`） | `KernelSU/kernel/feature/adb_root.c` | 存在，但 2.4.0 只用 `susfs_starts_with/ends_with` 改写 adbd 判断；可择机应用 |
| `kernel/feature/kernel_umount.c`（`:566-640`） | `KernelSU/kernel/feature/kernel_umount.c:73-126` | 存在；2.4.0 把 `ksu_handle_umount(uid_t, uid_t)` 的判定删掉并改调用者；本仓库是 `(struct cred *new, const struct cred *old)` |
| `kernel/feature/selinux_hide.c`（`:601-1300`） | `KernelSU/kernel/feature/selinux_hide.c`（377 行） | 存在但 API/实现不同（见 3.6/4.3） |
| `kernel/feature/sucompat.c`（`:2144-2590`） | `KernelSU/kernel/feature/sucompat.c:331-353` | 存在；2.4.0 新增 `ksu_handle_execveat_init()`/`ksu_handle_execveat()`/`ksu_handle_post_execveat_sucompat()`，本仓库只有 `ksu_handle_execveat()` + `ksu_handle_execveat_sucompat()` |
| `kernel/hook/setuid_hook.c`（`:2594-2790`） | `KernelSU/kernel/hook/setuid_hook.c:1-27` | 存在；2.4.0 的 `handle_zygote_setresuid()`/`handle_zygote_next_setresuid()` 本仓库没有 |
| `kernel/hook/lsm_hooks_list.c` | `KernelSU/kernel/hook/lsm_hooks_list.c` | 存在；本仓库通过 `ksu_task_prctl`/`ksu_task_fix_setuid` 注册，2.4.0 补丁未改此文件（说明 2.4.0 对应树用 `hook/lsm_hook.c` 另一套机制） |
| `kernel/manager/pkg_observer.c`（`:2791-2840`） | `KernelSU/kernel/manager/pkg_observer.c:26-104` | 存在但机制不同：2.4.0 用 `ksu_track_throne_tw_func()` fsnotify 回调，本仓库用 inode_rename 观察器 |
| `kernel/policy/app_profile.c`（`:2840-2920`） | `KernelSU/kernel/policy/app_profile.c` | 存在；2.4.0 的 `disable_seccomp()` 变非 static 等改动需对本仓库同位置适配 |
| `kernel/policy/feature.h`（`:3000-3050`） | `KernelSU/kernel/policy/feature.h` | 存在；可能需要 `KSU_FEATURE_*` 枚举/声明补充 |
| `kernel/preinit.c`/`preinit.h`（`:3050-3170`） | `KernelSU/kernel/preinit/preinit.c`/`.h` | 存在（路径为 `kernel/preinit/`）；2.4.0 增加 `ksu_handle_execveat_init()` 相关声明，本仓库无 |
| `kernel/runtime/boot_event.c`（`:3179-3220`） | `KernelSU/kernel/runtime/boot_event.c` | 存在；2.4.0 未在此直接改？`do_report_event` 在 `supercall/dispatch.c` |
| `kernel/selinux/rules.c`（`:3642-3710`） | `KernelSU/kernel/selinux/rules.c:174-260` | 存在；2.4.0 在 `apply_kernelsu_rules()` 末尾调用 `susfs_set_batch_sid()` |
| `kernel/selinux/selinux.c`（`:3710-3825`） | `KernelSU/kernel/selinux/selinux.c` | 存在；缺少全部 `susfs_*` SID 函数 |
| `kernel/selinux/sepolicy.c`（`:3825-3900`） | `KernelSU/kernel/selinux/sepolicy.c:280-350, 729-800` | 存在；`add_rule_raw()`/`add_type()` 钩子位置存在 |
| `kernel/sulog/event.c`（`:3900-3970`） | `KernelSU/kernel/sulog/event.c` | 存在；2.4.0 改动可能与 SUSFS 无关，需逐条筛 |
| `kernel/supercall/dispatch.c`（`:4043-4180`） | `KernelSU/kernel/supercall/dispatch.c:94-100` | 存在；2.4.0 在 `do_report_event` 的 boot_complete 分支调用 `susfs_start_sdcard_monitor_fn()`，本仓库无 |
| `kernel/supercall/supercall.c`（`:4309-4400`） | `KernelSU/kernel/supercall/supercall.c:81` | 存在；2.4.0 新增 `ksu_supercall_reboot_handler()` 并删除 kprobe；本仓库 `ksu_handle_sys_reboot()` 内联处理 `KSU_INSTALL_MAGIC2` |
| `kernel/supercall/supercall.h`（`:4402-4411`） | `KernelSU/kernel/supercall/supercall.h` | 存在；2.4.0 增加 handler 声明 |

### 4.2 超级调用/命令分发

`[2.4.0]` 的 `ksu_handle_sys_reboot()`（`10_enable…patch:4215-4296`）是本补丁的核心：

```c
int ksu_handle_sys_reboot(int magic1, int magic2, unsigned int cmd, void __user **arg)
{
    if (magic1 != KSU_INSTALL_MAGIC1)
        return -EINVAL;

    if (magic2 == SUSFS_MAGIC && current_uid().val == 0) {
        switch (cmd) {
        #ifdef CONFIG_KSU_SUSFS_SUS_PATH
        case CMD_SUSFS_ADD_SUS_PATH: susfs_add_sus_path(arg); return 0;
        ...
        #endif
        default: return -EINVAL;
        }
    }

    if (magic2 == KSU_INSTALL_MAGIC2)
        return ksu_supercall_reboot_handler(arg);

    return -EINVAL;
}
```

它与本仓库现状的差异：

1. 本仓库 `KernelSU/kernel/supercall/supercall.c:81` 的 `ksu_handle_sys_reboot(int magic1, int magic2, unsigned int cmd, void __user **arg)` 只处理 `magic1 == KSU_INSTALL_MAGIC1 && magic2 == KSU_INSTALL_MAGIC2` 的 fd 安装；没有 `SUSFS_MAGIC` 分支，也没有 `ksu_supercall_reboot_handler`。
2. 本仓库 `KernelSU/kernel/supercall/dispatch.c` 的 `do_report_event()`（`:94-100`）只调用 `on_boot_completed()`；2.4.0 在其后追加 `susfs_start_sdcard_monitor_fn()`（`10_enable…patch:4125-4131`）。
3. 2.4.0 修改 `kernel/reboot.c` 调用 `ksu_handle_sys_reboot`（`10_enable…patch:2322` 起）；本仓库的 reboot 钩子在哪需核对（`KernelSU/kernel/downstream/toolkit.h` 有 `toolkit_handle_sys_reboot`，`kernel/supercall/supercall.c` 里注册），但本仓库没有把 `SUSFS_MAGIC` 接入。
4. 2.4.0 的 `ksu_supercall_reboot_handler()` 使用 `task_work_add(current, &tw->cb, TWA_RESUME)`，本仓库 `supercall.c:28-76` 的 fd 安装逻辑已经内联，需按本仓库结构保留。

### 4.3 2.4.0 要求但本仓库缺失的 KSU 符号/函数

| 符号 | 2.4.0 使用处 | 本仓库状态 | 影响 | 建议实现位置 |
|---|---|---|---|---|
| `susfs_is_current_ksu_domain()` | `fs/susfs.c` 多处、`10_enable…patch:3783` | 无；`fs/susfs.c:26-29` 只有 false stub | mount/uid 解析永远 false | `KernelSU/kernel/selinux/selinux.c`：用 `cached_su_sid` + `selinux_cred()->sid` 实现并导出 |
| `susfs_is_sid_equal(const struct cred *cred, u32 sid)` | `10_enable…patch:2594-2790` setuid hook | 无（本仓库有 `is_sid_match()` static） | setuid 判定无法写 | 在 `selinux.c` 暴露 `is_sid_match()` 或新增同名函数 |
| `susfs_ksu_sid` / `susfs_init_sid` / `susfs_zygote_sid` / `susfs_zygote_next_sid` / `susfs_priv_app_sid` | `security/selinux/avc.c`、`selinuxfs.c`、`services.c` | `fs/susfs.c:23-24` 只有 1.5.11 的 `susfs_ksu_sid`/`susfs_kernel_sid`；`KernelSU/kernel/selinux/selinux.c` 没有 2.4.0 的 5 个全局（仅有 static `cached_su_sid`/`cached_zygote_sid`/`cached_init_sid`/`cached_adbd_sid`） | AVC/status/SELinux hide 无法链接；替换 `fs/susfs.c` 时旧 `susfs_kernel_sid` 定义需删除，AVC 逻辑改指向 `susfs_priv_app_sid` | `KernelSU/kernel/selinux/selinux.c` 新增 5 个全局并 `cache_sid()` 填充 |
| `susfs_set_batch_sid()` | `kernel/selinux/rules.c:apply_kernelsu_rules()`（`10_enable…patch:3642-3710`） | 无 | policy 应用后无法批量设置 SID | `kernel/selinux/selinux.c` 实现；在 `rules.c` 的 4.19 `#else` 路径末尾调用 |
| `ksu_selinux_hide_running` | `security/selinux/hooks.c`、`selinuxfs.c`（`50_add…patch:2501-3075`） | 本仓库 `selinux_hide.c` 只有 static `ksu_selinux_hide_enabled`，无 `running` | SELinux hook 无法判断 hide 是否运行 | 改造 `KernelSU/kernel/feature/selinux_hide.c` 暴露全局 |
| `fake_status` / `initialize_fake_status()` / `fake_status_initialize_key` | `security/selinux/selinuxfs.c` | 本仓库是 `static struct page *ksu_fake_status_page` + `ksu_prepare_fake_status_page()` + `ksu_sel_open_handle_status()`（`selinux_hide.c:182-240`） | 2.4.0 的 fake status 补丁无法链接 | 二选一：暴露别名或重写本仓库实现 |
| `ksu_selinux_hide_handle_second_stage()` / `..._post_fs_data()` | `10_enable…patch:1087-1120`（KSU 侧） | 无 | SELinux hide 的 second stage/post-fs-data 时机缺失 | 在本仓库 `selinux_hide.c` 增加 |
| `susfs_start_sdcard_monitor_fn()` | `do_report_event` boot_complete（`10_enable…patch:4125-4131`） | 无 | sdcard 监控永不启动 | `KernelSU/kernel/runtime/boot_event.c:on_boot_completed()` 或 `dispatch.c` 增加调用 |
| `susfs_parse_priority_app_strings()` / `susfs_parse_sus_path_app_strings()` | `kernel/manager/pkg_observer.c`（`10_enable…patch:2791-2840`） | 无；本仓库 `pkg_observer.c` 是 inode_rename 观察器 | UID 位图不会随安装/卸载刷新 | 在 `KernelSU/kernel/manager/pkg_observer.c` 的 `track_throne(false)` 后调用 |
| `susfs_set_current_proc_no_su()` / `umounted()` / `..._for_zygote_next()` | `kernel/hook/setuid_hook.c`、`sucompat.c` | 头文件定义在 `susfs_def.h` 但无调用 | 所有进程跟踪失效 | `KernelSU/kernel/hook/setuid_hook.c:ksu_handle_setresuid_cred()` + `feature/sucompat.c` |
| `ksu_handle_execveat_init()` / `ksu_handle_post_execveat_sucompat()` | 2.4.0 `feature/sucompat.c`（`:2144-2590`） | 无 | sucompat 的 SUSFS 早/晚钩子缺失 | `feature/sucompat.c` 按 4.19 签名补 |
| `ksu_handle_faccessat(..., struct filename **filename, ...)` | 2.4.0 `fs/open.c` | 本仓库是 `const char __user **`（`sucompat.c:225`） | 4.19 `do_faccessat` 3 参，不能直接用 | 保留本仓库旧签名，内部转换 |
| `ksu_handle_umount(uid_t, uid_t)` | 2.4.0 `feature/kernel_umount.c` | 本仓库是 `(struct cred *new, const struct cred *old)`（`:73`） | 签名不匹配 | 保留本仓库签名，在内部用 `new->uid`/`old->uid` 取 uid |
| `disable_seccomp()`（非 static） | 2.4.0 `setuid_hook.c` | 本仓库 `policy/app_profile.c:59` 是 `static` | 2.4.0 代码无法调用 | 改为非 static 或复制逻辑 |

### 4.4 必须插入的 feature hooks（本仓库版本）

| Hook | 2.4.0 位置 | 本仓库插入点 | 具体动作 |
|---|---|---|---|
| 初始化 | `kernel/core/init.c` `kernelsu_init()` | `KernelSU/kernel/ksu.c:282-286` | 已有 `susfs_init()`；需确认 `susfs_is_boot_completed_triggered` 不再提前置 true（若保留旧字段） |
| boot complete | `supercall/dispatch.c:do_report_event()` | `KernelSU/kernel/supercall/dispatch.c:94-100` 或 `runtime/boot_event.c:on_boot_completed()` | 调用 `susfs_start_sdcard_monitor_fn()`；同时置 boot completed 状态 |
| setuid/zygote | `hook/setuid_hook.c` | `KernelSU/kernel/hook/setuid_hook.c:ksu_handle_setresuid_cred()` | 按 `new->uid`/`old->uid` 与 SID 设置 `susfs_set_current_proc_umounted()/no_su()/...zygote_next()`，并 `schedule_work(&susfs_extra_works)` |
| sucompat exec | `feature/sucompat.c` | `KernelSU/kernel/feature/sucompat.c:ksu_handle_execveat()` | 在 exec 前标记 `TIF_PROC_NO_SU`（init/ksud 路径）；在 exec 后清/设 umounted |
| SELinux policy | `selinux/rules.c:apply_kernelsu_rules()` | `KernelSU/kernel/selinux/rules.c:174-235`（4.19 `#else` 分支） | `susfs_set_batch_sid()`；若用 2.4.0 的 `susfs_add_sus_type()`，需在 `sepolicy.c:add_type()` 中登记 |
| sepolicy rule | `selinux/sepolicy.c:add_rule_raw()` | `KernelSU/kernel/selinux/sepolicy.c:280-350` | 记录 `susfs_add_sus_avd_allowed_by_type(db->p_types... )` 的原始 avd allowed |
| package change | `manager/pkg_observer.c:ksu_track_throne_tw_func()` | `KernelSU/kernel/manager/pkg_observer.c:ksu_rename_observer()` / `..._slow()` | `track_throne(false)` 后调用 `susfs_parse_priority_app_strings()` 和 `susfs_parse_sus_path_app_strings()` |
| 命令传输 | `supercall/supercall.c:ksu_handle_sys_reboot()` | `KernelSU/kernel/supercall/supercall.c:81-110` | 加 `SUSFS_MAGIC` 分支；或在 `feature/susfs_cmd.c` 保持 prctl 桥并扩展命令 |
| 日志/版本 | `KernelSU/kernel/Makefile` 打印 | `KernelSU/kernel/ksu.c` 初始化日志 | 可选：`pr_info("SUSFS %s", SUSFS_VERSION)` |

### 4.5 其他 KSU 行为改动（2.4.0 补丁里容易被忽略的）

- `kernel/policy/allowlist.c`：2.4.0 删除 `ksu_uid_should_umount()` 中对 manager uid 的豁免（`10_enable…patch:2840-2860`），因为 `setuid_hook` 现在负责。本仓库的 `KernelSU/kernel/policy/allowlist.c:253` 仍有该豁免；移植时要确认是否跟随（否则 manager 的 mount 不会被卸载，可能影响 SUS_MOUNT 隐藏）。
- `kernel/feature/kernel_umount.c`：2.4.0 删除 `is_appuid`/`is_isolated_process`/`is_zygote` 三组判定（`10_enable…patch:566-600`），把判定搬到 `setuid_hook.c`。本仓库 `kernel_umount.c:73-126` 仍保留这些判定；如果直接套用 2.4.0 的 setuid hook，会出现“setuid hook 已决定 umount、kernel_umount 再次判断拒绝”的双重判定，需统一。
- `kernel/feature/adb_root.c`：2.4.0 用 `susfs_starts_with()`/`susfs_ends_with()` 替换 `strncmp` 风格判断（`10_enable…patch:410-449`）。这两个 inline 在 2.4.0 `susfs_def.h:90-112`；本仓库 1.5.11 的 `susfs_def.h` 没有，需要把 helper 放入头文件或本地复制。
- `kernel/sulog/event.c`：2.4.0 的改动可能只是上下文同步；不要盲目套用，先 `diff` 确认是否与 SUSFS 相关。
- `kernel/supercall/dispatch.c` 的 `do_manage_mark()`：2.4.0 用 `susfs_is_current_proc_no_su()` 决定是否记录 mark（`10_enable…patch:4153` 附近）；本仓库 `dispatch.c` 的 `do_manage_mark` 逻辑不同，需要按本仓库 `ksu_handle_setresuid_cred` 的 `TIF_KSU_MANAGED` 语义重写。

### 4.6 Build system 差异

- 2.4.0 的 `kernel/Kbuild` hunk 增加 `kernelsu-y += feature/selinux_hide.o` 等；本仓库 `kernel/Makefile` 用 `kernelsu-y := ksu.o`，所有 `.c` 通过 `#include` 进入 `ksu.o`（例如 `KernelSU/kernel/ksu.c:139` 包含 `feature/susfs_cmd.c`）。因此 2.4.0 的 Makefile/Kbuild hunk **全部不能直接 apply**，只能手工把新增文件加入 `ksu.c` 的 include 列表，或者把 unity build 改成多对象编译（不推荐，会破坏本仓库现有布局）。
- 2.4.0 的 `KernelSU/kernel/Kconfig` 选项依赖 `KSU_SUSFS` 与 `THREAD_INFO_IN_TASK`；本仓库 `KernelSU/kernel/Kconfig:147-245` 已有 `KSU_SUSFS` 菜单，但缺少 `PRIORITY_UID_FILTER`，且 `SUS_MAP` 是无代码的死开关。移植时先补 Kconfig（`PRIORITY_UID_FILTER`）并决定 `SUS_MAP` 是否保留。

### 4.7 结论：2.4.0 的 KSU 补丁不能直接打

- 4411 行补丁中至少 20 个文件的上下文在本仓库不存在或结构不同；`git apply` 会大面积失败。
- 即使强行“按 hunk 手工合”，也会因为缺少 `ksu_selinux_hide_running`、`fake_status`、`susfs_set_batch_sid`、`susfs_*_sid`、`susfs_start_sdcard_monitor_fn` 等符号而**链接失败**。
- 正确做法：把 2.4.0 的 KSU 补丁当作“接口需求清单”，在本仓库现有 `ksu.c`/`supercall`/`selinux`/`setuid_hook`/`pkg_observer` 上逐项实现（见 4.3/4.4），而不是复制补丁。

---

## 5. 移植检查清单（按优先级）

> 目标定义：
> - **P0（must-have）**：没有它，2.4.0 的核心 `sus_path`/`sus_kstat`/`sus_mount` 功能无法工作，或内核无法编译/启动。
> - **P1（should-have）**：2.4.0 的标志性增强功能，缺失不影响系统启动，但影响“功能对齐”。
> - **P2（optional / drop）**：可以明确不移植的历史功能或高风险功能。

### 5.1 P0：必须先做

#### P0-1 ABI 与头文件

- [ ] 用 2.4.0 的 `include/linux/susfs.h` 和 `include/linux/susfs_def.h` 替换 1.5.11 版本；保留 1.5.11 的 `SUSFS_VERSION` 改成本仓库版本号（或加 `SUSFS_VARIANT`）。
- [ ] 决定传输层：
  - 方案 A（推荐过渡）：保留 `KernelSU/kernel/feature/susfs_cmd.c` 的 prctl kprobe，增加 2.4.0 新命令 `0x55554`/`0x60020`/`0x60030`；对 `susfs_*()` 的 `void __user **` 原型写适配 wrapper。
  - 方案 B（目标态）：在 `KernelSU/kernel/supercall/supercall.c:ksu_handle_sys_reboot()` 加 `SUSFS_MAGIC` 分支；在 `KernelSU/kernel/include/uapi/supercall.h` 或 `susfs_def.h` 定义 `SUSFS_MAGIC`；同步改用户态 `ksu_susfs`。
- [ ] 修复 `susfs_get_enabled_features()`/`susfs_set_cmdline_or_bootconfig()` 的 8192 字节 buffer 的 `access_ok`/`copy_to_user` 限制（当前 `sfs_cmd.c:199-204` 只允许 4096）。

#### P0-2 `fs/susfs.c` 核心

- [ ] 以 2.4.0 `fs/susfs.c` 为底稿，删除 1.5.11 的 `st_android_data_path`/`st_sdcard_path`/`susfs_set_i_state_on_external_dir()`/`susfs_add_sus_mount()`/`susfs_add_try_umount()` 分支。
- [ ] 保留 2.4.0 的 `susfs_add_sus_path()`（FUSE 双映射）、`susfs_add_sus_path_loop()`、`susfs_run_sus_path_loop()`、`susfs_is_inode_sus_path()`。
- [ ] 移植 2.4.0 的 `susfs_is_uid_allowed_for_sus_path()` 位图与 `susfs_set_sus_path_uid_filter()`；包名解析函数依赖 `/data/system/packages.list`，4.19 需确认 `kernel_read`/`filp_open` 权限。
- [ ] 实现 `susfs_set_hide_sus_mnts_for_non_su_procs()` + `DEFINE_STATIC_KEY_FALSE`；删除 1.5.11 的 `susfs_hide_sus_mnts_for_all_procs` 全局。
- [ ] 移植 `susfs_set_uname()`/`susfs_spoof_uname()`、`susfs_enable_log()`、`susfs_set_cmdline_or_bootconfig()`、`susfs_get_enabled_features()`、`susfs_show_variant()/version()`。
- [ ] 移植 `susfs_add_open_redirect()`/`susfs_add_sus_map()`（如果 P1 做）。
- [ ] 统一锁：把 1.5.11 的 `susfs_spin_lock` 替换为 2.4.0 的 per-subsystem mutex/RCU/SRCU；4.19 支持 `DEFINE_STATIC_SRCU`/`kfree_rcu`。

#### P0-3 任务跟踪（否则所有 VFS 钩子都是死代码）

- [ ] `include/linux/susfs_def.h` 增加 `TIF_PROC_NO_SU 34`、`TIF_PROC_UMOUNTED_FOR_ZYGOTE_NEXT 35`、`AS_FLAGS_SUS_MAP 39`（如果做 SUS_MAP）。
- [ ] `KernelSU/kernel/selinux/selinux.c`：
  - 新增 `u32 susfs_ksu_sid`、`susfs_init_sid`、`susfs_zygote_sid`、`susfs_zygote_next_sid`、`susfs_priv_app_sid`；在 `cache_sid()` 中 `security_secctx_to_secid()` 填充。
  - 新增 `bool susfs_is_current_ksu_domain(void)`（`selinux_cred(current_cred())->sid == susfs_ksu_sid`）。
  - 新增 `bool susfs_is_sid_equal(const struct cred *cred, u32 sid)`。
  - 新增 `void susfs_set_batch_sid(void)`（在 policy 加载后批量设置）。
- [ ] `KernelSU/kernel/hook/setuid_hook.c:ksu_handle_setresuid_cred()`：按 2.4.0 `handle_zygote_setresuid()` 逻辑设置 TIF；`schedule_work(&susfs_extra_works)`。
- [ ] `KernelSU/kernel/feature/sucompat.c:ksu_handle_execveat()`：按 2.4.0 `ksu_handle_execveat_init()`/`post_execveat_sucompat()` 设置/清除 TIF。
- [ ] `KernelSU/kernel/feature/kernel_umount.c:ksu_handle_umount()`：与 setuid hook 统一判定，避免双重过滤；保留 `TIF_KSU_UNMOUNTABLE` 给 hostsredirect/getpriority（`KernelSU/kernel/policy/app_profile.h:7-10`）。

#### P0-4 VFS 钩子（按 4.19 签名）

- [ ] `fs/namei.c`：按 2.4.0 `lookup_dcache`/`lookup_one_qstr_excl`/`lookup_fast`/`__lookup_slow`/`walk_component`/`link_path_walk`/`lookup_last`/`lookup_open`/`open_last_lookups`/`do_tmpfile`/`path_openat`/`vfs_readlink` 的 hunk，裁剪成 4.19 可用的子集；删除 1.5.11 的 `ND_STATE_LAST_SDCARD_SUS_PATH` 分支。
- [ ] `fs/readdir.c`：按 2.4.0 `fillonedir`/`filldir`/`filldir64`/`compat_*` 的 `susfs_is_inode_sus_path()` 判断替换 1.5.11 的 d_name 判断。
- [ ] `fs/open.c`：保留 1.5.11 的 3 参 `do_faccessat` 结构，在 `ksu_handle_faccessat()` 前加 `if (likely(susfs_is_current_proc_no_su())) goto orig_flow;`。
- [ ] `fs/exec.c`：在当前 `do_execveat_common()` 的 KSU 钩子块（`fs/exec.c:1943-1974`）前加 `susfs_is_current_proc_no_su()` 快路径；删除 `CONFIG_KSU_SUSFS_SUS_SU` 分支（`fs/exec.c:69` 起）或保留为死代码但不要编译。
- [ ] `fs/stat.c`/`fs/statfs.c`：把 2.4.0 的 `susfs_sus_kstat_spoof_generic_fillattr()`/`vfs_statfs`/`statfs_by_dentry_wrapper`/`calculate_f_flags_wrapper` 接到 4.19 的 `vfs_getattr_nosec()`/`vfs_statfs()`；`stat->mnt_id` 用 `#if` 排除。
- [ ] `fs/proc/fd.c`/`fs/notify/fdinfo.c`/`fs/proc/task_mmu.c`：按 2.4.0 hunk 加 `susfs_is_inode_sus_kstat()` 与 `susfs_sus_kstat_spoof_*`；`task_mmu.c` 的 `VMA_PAD_START` 改 `vma->vm_start`。
- [ ] `fs/proc_namespace.c`/`fs/proc/fd.c`：sus_mount 的 `mnt_id` 伪造；4.19 `show_mountinfo` 的 `mnt_id`/`mnt_group_id` 与 2.4.0 6.1 字段兼容性需核对。

#### P0-5 `fs/namespace.c`（mount）

- [ ] 在 4.19 `fs/namespace.c` 中新增 `susfs_alloc_non_unshare_ksu_vfsmnt()`/`susfs_alloc_unshare_ksu_vfsmnt()`，但必须：
  - 删除 `mnt->mnt.mnt_userns = &init_user_ns`；
  - 删除 `INIT_HLIST_HEAD(&mnt->mnt_stuck_children)`；
  - 补 `mnt->mnt.data = NULL;` 和 `init_fs_pin(&mnt->mnt_umount, drop_mountpoint);`
  - 保留 `mnt->mnt.susfs_mnt_id_backup` 初始化逻辑。
- [ ] 在 `vfs_kern_mount()`/`alloc_vfsmnt()`/`clone_mnt()` 的 4.19 调用点插入 fake id 分配，触发条件用 `susfs_is_current_ksu_domain()` + `susfs_is_sdcard_android_data_not_decrypted`。
- [ ] 修改 `mnt_alloc_group_id()` 从 `DEFAULT_KSU_MNT_GROUP_ID=200000` 起分配；修改 `get_anon_bdev()` 的 minor 起点 4096。
- [ ] 修改 `mnt_free_id()` 同时处理 1.5.11 的 `susfs_mnt_id_backup` 与 2.4.0 的 `VFSMOUNT_MNT_FLAGS_KSU_UNSHARED_MNT`。
- [ ] 在 `__lookup_mnt()` 加 umounted 进程跳过 fake mnt；`copy_mnt_ns()` 加 `CL_COPY_MNT_NS`；新增 `susfs_get_non_sus_mnt_id_from_mnt()`/`susfs_get_non_sus_vfsmnt_from_vfsmnt()`。

#### P0-6 启动/挂载监控

- [ ] `fs/susfs.c` 的 `susfs_start_sdcard_monitor_fn()`：按 4.19 `fsnotify_ops.handle_event` 改写；`fsnotify_alloc_group(&ops)` 1 参；`fsnotify_destroy_group()` void。
- [ ] `KernelSU/kernel/supercall/dispatch.c:do_report_event()` 的 `EVENT_BOOT_COMPLETED` 分支调用 `susfs_start_sdcard_monitor_fn()`（或 `runtime/boot_event.c:on_boot_completed()`）。
- [ ] `KernelSU/kernel/ksu.c`：`susfs_init()` 保持；如果保留 `susfs_is_boot_completed_triggered`，不要在 init 时无条件置 true（当前 `ksu.c:282-286` 需改）。

### 5.2 P1：功能对齐（可分批）

- [ ] **SUS_KSTAT 完整版**：FUSE inode 检测、`spoofed_kstatfs`、`spoofed_mnt_id`、`inotify_fdinfo`、`proc_fd_seq_show`；这是 2.4.0 最实用的反检测增强。涉及 `fs/susfs.c:832-1305`、`fs/stat.c`、`fs/statfs.c`、`fs/notify/fdinfo.c`、`fs/proc/fd.c`、`fs/proc/task_mmu.c`。
- [ ] **SUS_MAP**：`fs/susfs.c:1749-1795`、`mm/memory.c:__access_remote_vm()`、`fs/proc/task_mmu.c:show_map_vma/show_smap/show_smaps_rollup/pagemap_read`、`fs/proc/base.c:proc_map_files_readdir`。4.19 需要 `tsk` 参数适配。
- [ ] **PRIORITY_UID_FILTER**：`fs/susfs.c:1983-2249`、`block/ioprio.c`、`kernel/sys.c`、`KernelSU/kernel/manager/pkg_observer.c`。
- [ ] **OPEN_REDIRECT 完整版**：`fs/susfs.c:1463-1745`、`fs/namei.c`（`vfs_readlink`、`path_openat`）、`fs/proc/base.c:do_proc_readlink`、`fs/proc/task_mmu.c:show_map_vma`；4.19 `VMA_PAD_START` 替代。
- [ ] **SELinux AVC log spoofing**：2.4.0 把 `tsid` 替换为 `priv_app_sid`（不是 1.5.11 的 `kernel_sid`）；需要 `susfs_priv_app_sid` 与 `susfs_is_current_ksu_domain()`。`fs/susfs.c:2214-2249`、`security/selinux/avc.c`。
- [ ] **完整 SELinux hide**：`fs/susfs.c:53-225`、`security/selinux/ss/services.c`、`security/selinux/hooks.c`、`selinuxfs.c`、`ss/status.c`。这是最高风险项，建议放在最后；如果本仓库现有 `selinux_hide.c` 已能覆盖 `ksu_susfs` 的 SELinux 隐藏需求，可只做 AVC 日志伪装，不做完整 policy 层。
- [ ] **cmdline/bootconfig**：`fs/proc/cmdline.c` 挂钩；`fs/proc/bootconfig.c` 在 4.19 不存在，不要照搬。

### 5.3 P2：可以明确不移植/删除

- [ ] `CONFIG_KSU_SUSFS_TRY_UMOUNT` 及 `susfs_add_try_umount()`/`susfs_try_umount()`/`susfs_auto_add_try_umount_for_bind_mount()`：2.4.0 已删除；保留只会增加 4.19 mount 代码复杂度。
- [ ] `CONFIG_KSU_SUSFS_SUS_SU` 及 `fs/sus_su.c`/`include/linux/sus_su.h`/`fs/devpts/inode.c` 钩子：2.4.0 已删除；本仓库当前也是死代码，建议直接删除补丁中的相关 hunk。
- [ ] `CONFIG_KSU_SUSFS_AUTO_ADD_SUS_BIND_MOUNT`/`AUTO_ADD_SUS_KSU_DEFAULT_MOUNT`/`AUTO_ADD_TRY_UMOUNT_FOR_BIND_MOUNT`：Kconfig 从未定义，实际死代码；2.4.0 不需要。
- [ ] `CMD_SUSFS_SET_ANDROID_DATA_ROOT_PATH`/`SET_SDCARD_ROOT_PATH`/`ADD_SUS_MOUNT`/`UMOUNT_FOR_ZYGOTE_ISO_SERVICE`/`SHOW_SUS_SU_WORKING_MODE`/`IS_SUS_SU_READY`/`SUS_SU`：2.4.0 只保留 deprecated 常量，不实现分发；本仓库也不需要继续支持。
- [ ] `susfs_is_current_zygote_domain()` 的 extern（`fs/namespace.c:38`，无定义）、`susfs_reorder_mnt_id()`（`:3795`）、`susfs_is_mnt_devname_ksu()`（`:3814`）：2.4.0 没有这些函数；如果移植 2.4.0 的 mount 模型，应删除而不是补定义。

### 5.4 建议实施顺序（里程碑）

1. **M1 编译通过**：替换头文件 + `fs/susfs.c` 核心 + `fs/namespace.c` 4.19 适配 + KSU 侧 `selinux.c` SID 函数；先关闭所有可选 feature（`SUS_MAP`/`PRIORITY`/`OPEN_REDIRECT`/完整 SELinux），只保留 `SUS_PATH`/`SUS_KSTAT`/`SUS_MOUNT`/`SPOOF_UNAME`。
2. **M2 设备启动**：验证 `susfs_init()`、`cmdline` 不崩、`mountinfo` 正常；确认 `TIF_PROC_UMOUNTED` 在 `setuid_hook` 中真实置位。
3. **M3 基础功能**：`ksu_susfs add_sus_path` + `add_sus_kstat` + `hide_sus_mnts_for_non_su_procs` 端到端；验证 app 通过 `/proc/self/maps`、`/proc/self/mountinfo`、`stat` 看到的是伪造值。
4. **M4 可选功能**：`SUS_MAP` → `PRIORITY_UID_FILTER` → `OPEN_REDIRECT` → `SELinux hide`；每项单独 Kconfig 开关，出问题可回退。
5. **M5 清理**：删除 1.5.11 的死代码/配置（`TRY_UMOUNT`、`SUS_SU`、`AUTO_ADD_*`、android_data/sdcard 列表），把 `fs/susfs.c` 与上游 2.4.0 的 diff 控制在“只保留 4.19 适配差异”。

### 5.5 验证清单

- [ ] 编译：`make ARCH=arm64 ...` 无 `fs/susfs.c`/`fs/namespace.c`/`security/selinux` 警告/错误；重点检查 `struct mount`、`fsnotify_ops`、`statfs_by_dentry`、`selinux_state`。
- [ ] 启动：`dmesg | grep susfs` 显示 `susfs is initialized! version: v2.4.0`（或本仓库版本）；无 `BUG`/`WARNING`。
- [ ] `cat /proc/self/mountinfo` 中 app 进程看不到 fake mnt；`/proc/self/maps` 中 sus_map/sus_kstat 的 inode 被替换/隐藏。
- [ ] `ksu_susfs show` 返回 enabled_features 包含实际启用的 feature；`show_version`/`show_variant` 正常。
- [ ] `ksu_susfs add_sus_path /data/local/tmp/test` 后，app 读取 `/proc/self/maps`/`stat` 看不到目标路径（按 uid filter 行为）。
- [ ] 重启/OTA 后 sdcard monitor 正常关闭（`dmesg | grep susfs_sdcard`）。
- [ ] SELinux permissive/enforcing 两种模式下均无 denial 风暴；`avc: denied` 中的 `tsid` 按 2.4.0 规则伪装为 priv_app（如果启用）。

---

## 6. 附录：关键函数/行号索引

### 6.1 `[1.5.11] fs/susfs.c`

| 函数 | 行号 |
|---|---:|
| `susfs_is_current_ksu_domain`（false stub） | 26 |
| `susfs_set_i_state_on_external_dir` | 63 |
| `susfs_add_sus_path` | 134 |
| `susfs_add_sus_path_loop` | 260 |
| `susfs_run_sus_path_loop` | 348 |
| `susfs_is_base_dentry_android_data_dir` | 379 |
| `susfs_is_sus_android_data_d_name_found` | 387 |
| `susfs_is_inode_sus_path`（三版本） | 428 / 438 / 448 |
| `susfs_add_sus_mount` | 503 |
| `susfs_auto_add_sus_bind_mount` | 554 |
| `susfs_auto_add_sus_ksu_default_mount` | 577 |
| `susfs_add_sus_kstat` | 654 |
| `susfs_update_sus_kstat` | 749 |
| `susfs_sus_ino_for_generic_fillattr` | 800 |
| `susfs_sus_ino_for_show_map_vma` | 822 |
| `susfs_add_try_umount` | 838 |
| `susfs_try_umount` | 871 |
| `susfs_auto_add_try_umount_for_bind_mount` | 888 |
| `susfs_set_uname` | 974 |
| `susfs_spoof_uname` | 999 |
| `susfs_set_log` | 1009 |
| `susfs_set_cmdline_or_bootconfig` | 1024 |
| `susfs_spoof_cmdline_or_bootconfig` | 1053 |
| `susfs_add_open_redirect` | 1092 |
| `susfs_get_redirected_path` | 1143 |
| `susfs_sus_su` | 1166 |
| `susfs_get_enabled_features` | 1223 |
| `susfs_set_avc_log_spoofing` | 1306 |
| `susfs_init` | 1314 |

### 6.2 `[2.4.0] fs/susfs.c`

| 函数 | 行号 |
|---|---:|
| `susfs_add_sus_type` | 58 |
| `susfs_is_sus_type_rcu` | 85 |
| `susfs_add_sus_avd_allowed_by_type` | 121 |
| `susfs_spoof_avd_allowed_rcu` | 162 |
| `susfs_switch_to_ksu_domain` | 186 |
| `susfs_restore_creds` | 220 |
| `susfs_add_sus_path` | 234 |
| `susfs_add_sus_path_loop` | 286 |
| `susfs_run_sus_path_loop` | 321 |
| `susfs_is_inode_sus_path` | 364 |
| `susfs_set_sus_path_uid_filter` | 606 |
| `susfs_parse_sus_path_app_strings` | 793 |
| `susfs_is_uid_allowed_for_sus_path` | 808 |
| `susfs_set_hide_sus_mnts_for_non_su_procs` | 838 |
| `susfs_mark_inode_sus_kstat` | 870 |
| `susfs_is_inode_sus_kstat` | 1110 |
| `susfs_sus_kstat_spoof_generic_fillattr` | 1136 |
| `susfs_sus_kstat_spoof_show_map_vma` | 1201 |
| `susfs_sus_kstat_spoof_vfs_statfs` | 1244 |
| `susfs_sus_kstat_spoof_inotify_fdinfo` | 1265 |
| `susfs_sus_kstat_spoof_proc_fd_seq_show` | 1282 |
| `susfs_set_uname` | 1306 |
| `susfs_enable_log` | 1359 |
| `susfs_set_cmdline_or_bootconfig` | 1390 |
| `susfs_add_open_redirect` | 1465 |
| `susfs_open_redirect_spoof_do_sys_openat` | 1625 |
| `susfs_open_redirect_spoof_vfs_readlink` | 1670 |
| `susfs_open_redirect_spoof_do_proc_readlink` | 1698 |
| `susfs_open_redirect_spoof_show_map_vma_srcu` | 1723 |
| `susfs_add_sus_map` | 1749 |
| `susfs_set_priority_uid_filter` | 1987 |
| `susfs_parse_priority_app_strings` | 2174 |
| `susfs_is_priority_uid_filtered` | 2189 |
| `susfs_set_avc_log_spoofing` | 2214 |
| `susfs_get_enabled_features` | 2251 |
| `susfs_show_variant` | 2335 |
| `susfs_show_version` | 2353 |
| `susfs_sdcard_monitor_fn` | 2492 |
| `susfs_start_sdcard_monitor_fn` | 2531 |
| `susfs_run_extra_works` | 2543 |
| `susfs_init` | 2552 |

### 6.3 `[2.4.0]` 主要补丁 hunk 定位

| 内容 | `50_add_susfs_in_gki-android14-6.1.patch` 行 |
|---|---:|
| `fs/Makefile` | ~1-40 |
| `fs/exec.c` | 92 起 |
| `fs/namei.c` | 196 起 |
| `fs/namespace.c` | 638 起（fake mnt id 在 690-900；`__lookup_mnt` 821；`copy_mnt_ns` 3730；helper 4980） |
| `fs/open.c` | 1094 起 |
| `fs/proc/base.c` | 1155 起 |
| `fs/proc/bootconfig.c` | 1211 起 |
| `fs/proc/task_mmu.c` | 1336 起 |
| `fs/readdir.c` | 1714 起 |
| `fs/stat.c` | 1982 起 |
| `fs/statfs.c` | 2114 起 |
| `fs/super.c`（`get_anon_bdev`） | 2222 起 |
| `kernel/reboot.c`（`ksu_handle_sys_reboot` 调用） | 2322 起 |
| `kernel/sys.c` | 2352 起 |
| `mm/memory.c`（`__access_remote_vm`） | 2432 起 |
| `security/selinux/avc.c` | 2458 起 |
| `security/selinux/hooks.c` | 2501 起 |
| `security/selinux/selinuxfs.c` | 2872 起 |
| `security/selinux/ss/services.c` | 3077 起 |
| `security/selinux/ss/status.c`（上游路径为 `security/selinux/status.c`） | 3390 起 |

### 6.4 `[2.4.0]` KSU 补丁主要 hunk 定位

| 内容 | `10_enable_susfs_for_ksu.patch` 行 |
|---|---:|
| `kernel/Kbuild` | 1 起 |
| `kernel/Kconfig`（SUSFS 菜单） | 37-161（SUSFS 选项从 58 起） |
| `kernel/Makefile`（版本打印） | 162-180 |
| `kernel/core/init.c`（`susfs_init()`） | 181-300（调用约 287） |
| `kernel/feature/adb_root.c` | 410-449 |
| `kernel/feature/kernel_umount.c` | 566-600 |
| `kernel/feature/selinux_hide.c` | 601-1300 |
| `kernel/feature/sucompat.c` | 2144-2590 |
| `kernel/hook/setuid_hook.c` | 2594-2790 |
| `kernel/manager/pkg_observer.c` | 2791-2840 |
| `kernel/policy/app_profile.c` | 2840-2920 |
| `kernel/policy/feature.h` | 3000-3050 |
| `kernel/preinit.c`/`preinit.h` | 3050-3170 |
| `kernel/runtime/boot_event.c` | 3179-3220 |
| `kernel/selinux/rules.c` | 3642-3710 |
| `kernel/selinux/selinux.c` | 3710-3825 |
| `kernel/selinux/sepolicy.c` | 3825-3900 |
| `kernel/sulog/event.c` | 3900-3970 |
| `kernel/supercall/dispatch.c` | 4043-4180 |
| `kernel/supercall/supercall.c` | 4309-4400 |
| `kernel/supercall/supercall.h` | 4402-4411 |

---

## 7. 一句话结论

2.4.0 的 `fs/susfs.c` 可以移植到 4.19，但必须视为“以 2.4.0 为蓝本、按 4.19 VFS/SELinux/fsnotify API 重写”的工程，而不是替换文件；KSU 侧更要按本仓库 3.3.1 unity build 重写接口。先完成 P0（头文件/传输、sus_path+sus_kstat、TIF 任务跟踪、mount fake id、4.19 VFS 钩子），再按 P1 逐项加 `SUS_MAP`/`PRIORITY_UID_FILTER`/`OPEN_REDIRECT`/SELinux hide；`TRY_UMOUNT`/`SUS_SU`/`AUTO_ADD_*` 直接放弃。

（报告完）
