# SUSFS 2.4.0 / 73fe 完整功能测试记录

日期：2026-11-17

## 构建

- 分支：`feature/ksu3.3.1-susfs-2.4.0-on-73fe`
- 基线：`feature/ksu3.3.1-susfs-1.5.11-on-73fe`（`ef536713b553`）
- KernelSU：3.3.1（submodule `3b60f2af951d1dd9540421ac9b57156912ddedaf`）
- 内核：`4.19.248-NonLa-2.0-ge73bc7cba03d-dirty`
- 编译：clang 18.1.8 + LLD 18.1.8，arm64
- 内核源码目录：`/home/yueguang/DSH/安卓/repos/picasso-kernel-ksu-susfs-24`
- 构建 O：`/home/yueguang/DSH/安卓/repos/out/picasso/ksu24`

最终产物：

- `ksu331_73fe_susfs240_full_boot.img`（fastboot 直刷，sha256 `ba77c0e8abd307f4ba8c8e24a7f8efb230b0f119ef6ba026f6f3e5fe57695d0f`）
- `ksu331_73fe_susfs240_full.zip`（AnyKernel3，Recovery 刷入，sha256 `5d7faa2b93780ca33c97160bd8ec3aeb1f9ab4c8bfc8c551c1d719a1d02dc39a`）

## 开启的功能

`ksu_susfs show enabled_features`：

- `CONFIG_KSU_SUSFS_SUS_PATH`
- `CONFIG_KSU_SUSFS_SUS_MOUNT`
- `CONFIG_KSU_SUSFS_SUS_KSTAT`
- `CONFIG_KSU_SUSFS_SPOOF_UNAME`
- `CONFIG_KSU_SUSFS_ENABLE_LOG`
- `CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS`
- `CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG`
- `CONFIG_KSU_SUSFS_OPEN_REDIRECT`
- `CONFIG_KSU_SUSFS_SUS_MAP`
- `CONFIG_KSU_SUSFS_PRIORITY_UID_FILTER`

`show version`：`v2.4.0`；`show variant`：`NON-GKI`。

## 真机验证（Redmi K30 5G / picasso / YAAP 16）

- `fastboot flash boot` 持久化后多次重启均 `sys.boot_completed=1`
- `su -c id` 返回 `uid=0(root) ... context=u:r:ksu:s0`；普通 shell 的 `su` 兼容路径正常
- `/proc/version`：`4.19.248-NonLa-2.0-ge73bc7cba03d-dirty`
- `/data/adb/modules/susfs4ksu/module.prop`：`v2.4.0-R28`
- `SUS_PATH`：`add_sus_path /data/local/tmp/susfs_secret2` 后，root 可读，普通应用 uid 10238 读取返回 `No such file or directory`
- `SUS_KSTAT`：`add_sus_kstat_statically ... 999 ...` 后，root 看到真实大小 4，应用 uid 10238 看到 999
- `SUS_MOUNT`：在 `/sdcard/Android` 解密前创建 KSU 测试挂载 `/data/local/tmp/mnttest3`，`/proc/self/mountinfo` 中 mnt_id 为 `2000000003`；root 可见，普通应用上下文不可见；`hide_sus_mnts_for_non_su_procs 1` 时 `/proc/self/mountinfo` 过滤生效
- `OPEN_REDIRECT`：`add_open_redirect /system/etc/hosts /data/local/tmp/hosts_fake2 0` 后，root 打开 `/system/etc/hosts` 得到重定向内容；uid 10238 应用仍看到原文件
- `SPOOF_UNAME`：`set_uname 4.19.248-test SUSFS-TEST` 后应用看到测试值，`set_uname default default` 可恢复
- `SUS_MAP` / `add_sus_path_loop` / `set_priority_uid_filter` 命令均返回成功并写入 dmesg
- `lookup_fast` RCU 分支问题已修复；SUS_PATH 查询不再卡死
- 最终镜像连续启动、应用启动、打开普通文件路径无 kernel panic / BUG；Vector `vectord` 正常运行

## 已知限制

- 4.19 不支持 `vfs_create_mount` / `mnt_userns`，`SUS_MOUNT` 采用 4.19 兼容的 `vfs_kern_mount`/`clone_mnt` 分支与 `mnt_id_ida` 区间分配
- SELinux 策略级 `susfs_add_sus_type` 未接入；`enable_avc_log_spoofing` 的 AVC `tsid`/`tcontext` 伪装已接入并完成构建验证，未做完整策略场景回归
- 极端 `O_TMPFILE` / `O_PATH` 与 `map_files` 组合场景未做完整回归；普通 open/readlink/maps 路径已接入
- 测试机上的 SUSFS 模块 `config.sh` 曾出现被模块自身重置为全零的情况，已恢复默认配置；这是 sidex 模块行为，不属于内核问题
