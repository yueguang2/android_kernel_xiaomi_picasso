# SUSFS 2.4.0 / 73fe 完整功能测试记录

日期：2026-11-17

## 构建

- 分支：`feature/ksu3.3.1-susfs-2.4.0-on-73fe`
- 基线：`feature/ksu3.3.1-susfs-1.5.11-on-73fe`（`ef536713b553`）
- KernelSU：3.3.1（submodule `3b60f2af951d1dd9540421ac9b57156912ddedaf`）
- 内核：`4.19.248-NonLa-2.0-g5a3c4dfc0258-dirty`
- 编译：clang 18.1.8 + LLD 18.1.8，arm64
- 内核源码目录：`/home/yueguang/DSH/安卓/repos/picasso-kernel-ksu-susfs-24`
- 构建 O：`/home/yueguang/DSH/安卓/repos/out/picasso/ksu24`

最终产物：

- `ksu331_73fe_susfs240_full_boot.img`（fastboot 直刷，sha256 `789f6ede7e752ebfdc0fa003b3e8a91705c0a39754f2db49972a0836be1baa5c`）
- `ksu331_73fe_susfs240_full.zip`（AnyKernel3，Recovery 刷入，sha256 `0af87b2e36f4c6a5bf80256770744f20ff0f82e8fb8155a77daf22453872e3c1`）

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
- `/proc/version`：`4.19.248-NonLa-2.0-g5a3c4dfc0258-dirty`
- `/data/adb/modules/susfs4ksu/module.prop`：`v2.4.0-R28`
- `SUS_PATH`：`add_sus_path /data/local/tmp/susfs_secret2` 后，root 可读，普通应用 uid 10238 读取返回 `No such file or directory`
- `SUS_KSTAT`：`add_sus_kstat_statically ... 999 ...` 后，root 看到真实大小 4，应用 uid 10238 看到 999
- `SUS_MOUNT`：在 `/sdcard/Android` 解密前创建 KSU 测试挂载 `/data/local/tmp/mnttest3`，`/proc/self/mountinfo` 中 mnt_id 为 `2000000003`；root 可见，普通应用上下文不可见；`hide_sus_mnts_for_non_su_procs 1` 时 `/proc/self/mountinfo` 过滤生效
- `OPEN_REDIRECT`：`add_open_redirect /system/etc/hosts /data/local/tmp/hosts_fake2 0` 后，root 打开 `/system/etc/hosts` 得到重定向内容；uid 10238 应用仍看到原文件
- `SPOOF_UNAME`：`set_uname 4.19.248-test SUSFS-TEST` 后应用看到测试值，`set_uname default default` 可恢复
- `SPOOF_CMDLINE_OR_BOOTCONFIG`：`set_cmdline_or_bootconfig /data/local/tmp/fake_cmdline` 后读取 `/proc/cmdline` 得到伪内容（root/应用侧均走同一 proc 输出路径）
- SELinux policy 层：KSU `sepolicy.c` 已调用 `susfs_add_sus_type` / `susfs_add_sus_avd_allowed_by_type`，`security/selinux/ss/services.c` 已接入 SUS 类型解析跳过与 AVD allowed 恢复；开机 `Enforcing` 正常
- Vector 框架：修复 sepolicy 加载、daemon 启动时机和 SUSFS AVD 伪装范围后，Vector Manager 系统状态显示 `SELinux 策略：已加载`、`系统框架：已注入`、`Dex 优化器包装：支持`（详见 `docs/vector-framework-fix.md`）
- `SUS_MAP`：用 zygote 域自测程序在 `setresuid(10242)` 后测试，`add_sus_map /data/local/tmp/susfs_target` 前 `/proc/self/maps`、`/proc/self/smaps` 均可见目标映射（退出码 1），添加后均不可见（退出码 0）；`/proc/self/map_files` 的条目数由 4 变为 3，证明 maps/smaps/map_files 过滤路径端到端生效
- `AVC log spoofing`：`enable_avc_log_spoofing 1` 后，以 `u:r:untrusted_app:s0` 对 ksu 域进程执行 `kill -0`，dmesg 记录 `replace tsid: 567 with priv_app_sid: 571`，AVC 行显示 `tcontext=u:r:priv_app:s0:c512,c768`，伪装生效
- `add_sus_path_loop` / `set_priority_uid_filter` 命令均返回成功并写入 dmesg；包安装/更新后通过 `pkg_observer` 刷新 SUS_PATH/priority UID 过滤器
- `lookup_fast` RCU 分支问题已修复；SUS_PATH 查询不再卡死
- 最终镜像连续启动、应用启动、打开普通文件路径无 kernel panic / BUG；Vector `vectord` 正常运行

## 已知限制

- 4.19 不支持 `vfs_create_mount` / `mnt_userns`，`SUS_MOUNT` 采用 4.19 兼容的 `vfs_kern_mount`/`clone_mnt` 分支与 `mnt_id_ida` 区间分配
- `susfs_add_sus_type` / `susfs_add_sus_avd_allowed_by_type` 已接入 KSU sepolicy 与 `security/selinux/ss/services.c`；AVD 伪装只针对 SUSFS type，普通模块策略不被误伤；完整策略级隐藏场景（多策略切换/动态 type 重载）未做端到端回归
- `SUS_MAP` 的 `/proc/self/maps`、`/proc/self/smaps`、`/proc/self/map_files` 已端到端验证；`pagemap` / `mem` 和 fdinfo 组合在自测进程中受 SELinux/权限限制，未全部端到端回归
- 极端 `O_TMPFILE` / `O_PATH` 与 `map_files` 组合场景未做完整回归；普通 open/readlink/maps 路径已接入
- 测试机上的 SUSFS 模块 `config.sh` 曾出现被模块自身重置为全零的情况，已恢复默认配置；这是 sidex 模块行为，不属于内核问题
