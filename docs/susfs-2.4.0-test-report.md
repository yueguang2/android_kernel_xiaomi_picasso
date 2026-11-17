# SUSFS 2.4.0 / 73fe 测试记录

日期：2026-11-17

## 构建

- 分支：`feature/ksu3.3.1-susfs-2.4.0-on-73fe`
- 基线：`feature/ksu3.3.1-susfs-1.5.11-on-73fe`（`ef536713b553`）
- KernelSU：3.3.1（submodule `3b60f2af951d1dd9540421ac9b57156912ddedaf`）
- 内核：`4.19.248-NonLa-2.0-gef536713b553-dirty`
- 编译：clang 18.1.8 + LLD 18.1.8，arm64

产物：

- `ksu331_73fe_susfs240_boot.img`（fastboot 直刷）
- `ksu331_73fe_susfs240.zip`（AnyKernel3，Recovery 刷入）

## 真机验证（Redmi K30 5G / picasso / YAAP 16）

- `fastboot boot` 测试通过；`fastboot flash boot` 持久化后连续 3 次重启，`sys.boot_completed=1`
- `su -c id` 返回 `uid=0(root) ... context=u:r:ksu:s0`
- `/proc/version`：`4.19.248-NonLa-2.0-gef536713b553-dirty`
- `ksu_susfs show version`：`v2.4.0`
- `ksu_susfs show variant`：`NON-GKI`
- `ksu_susfs show enabled_features`：SUS_PATH / SUS_MOUNT / SUS_KSTAT / SPOOF_UNAME / ENABLE_LOG / HIDE_KSU_SUSFS_SYMBOLS / SPOOF_CMDLINE_OR_BOOTCONFIG / OPEN_REDIRECT / SUS_MAP / PRIORITY_UID_FILTER
- `/data/adb/modules/susfs4ksu/module.prop` 显示 `v2.4.0-R28`（sidex 模块自动按内核版本显示）
- `add_sus_path /system/etc/hosts` 后，root 可见，普通应用 uid 10238 读取返回 `No such file or directory`
- `add_sus_kstat_statically`、`add_sus_path_loop`、`add_sus_map`、`set_priority_uid_filter` 命令均返回成功并写入 dmesg
- `set_uname default default` 后普通应用/root 的 `uname -r`、`uname -v` 正常
- 修复 `lookup_fast` RCU 分支对 `__d_lookup_rcu()` 返回 dentry 误用 `dput()` 的问题；修复后 `add_sus_path` 后应用侧 `ls /system/etc/hosts` 快速返回 `No such file or directory`，不再卡死
- 最终镜像 `#9`（2026-11-17 18:46:25 CST）连续重启 2 次均 `sys.boot_completed=1`
- Vector `vectord` 正常运行；开机无 kernel panic / BUG

## 已知限制

- `SUS_MAP` 已接入 maps/smaps/smaps_rollup/map_files 的主要读取路径，`pagemap` 尚未覆盖
- `SUS_MOUNT` 的 `/proc/self/mounts|mountinfo|mountstat` 过滤已接新 static key，但仍需要按模块实际挂载场景做进一步隐蔽性验证
- `OPEN_REDIRECT` 已接入 namei/proc readlink 路径，极端 `map_files` 场景未做完整回归
- SELinux AVC log spoofing 代码已接入，但未做 SELinux 策略级场景测试
