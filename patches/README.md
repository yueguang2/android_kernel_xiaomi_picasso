# patches

这里放可复用的补丁系列，按上游可提交的最小改动组织。

当前实际文件：

- `susfs-4.19-v1.5.11.patch`：上游 `88836` 的 SUSFS v1.5.11 内核侧 4.19 补丁（目前在 `feature/susfs-1.5.11` 上验证编译）
- `susfs-4.19-v1.5.11-path-root-fix.patch`：修复未配置 Android data / sdcard root 时，空路径匹配导致 `add_sus_path` 失败的问题
- `susfs-4.19-v2.4.0.patch`：SUSFS 2.4.0 内核侧 4.19 完整 backport，基于 `feature/ksu3.3.1-susfs-1.5.11-on-73fe`（`ef536713b553`）生成；包含 SUS_PATH / SUS_MOUNT / SUS_KSTAT / OPEN_REDIRECT / SUS_MAP / SPOOF_UNAME / AVC log spoof 等路径，已在 73fe 真机启动、root、挂载隐藏和连续重启验证
- `kernelsu-3.3.1-stale-hooks-compat.patch`：为 upstream `staging` 的旧 manual hook 引用补 KSU 兼容符号
- `kernelsu-3.3.1-susfs-kconfig.patch`：为 KernelSU Kconfig 增加 SUSFS 选项
- `kernelsu-3.3.1-susfs-init.patch`：在 KernelSU 初始化时调用 `susfs_init()`
- `kernelsu-3.3.1-susfs-prctl.patch`：旧 1.5.11 的 LSM `task_prctl` 桥（仅用于 1.5.11 分支）
- `kernelsu-3.3.1-susfs-2.4.0.patch`：KernelSU 3.3.1 的 SUSFS 2.4.0 集成，包含 Kconfig、`susfs_init()`、sdcard monitor、setuid hook 的 app 标记、`sys_reboot` + `SUSFS_MAGIC` 命令分发

说明：`susfs-4.19-v2.4.0.patch` 应用在 1.5.11 工作树之上（或先应用 1.5.11 系列补丁），不要在未打 1.5.11 补丁的 73fe 基线上直接套用。`kernelsu-3.3.1-susfs-2.4.0.patch` 应用在 KernelSU 3.3.1（submodule `3b60f2af951d1dd9540421ac9b57156912ddedaf`）的 `KernelSU/` 目录下。2.4.0 分支的真机验证结果和已知限制见 `docs/analysis/`。

每个补丁必须注明：

- 适用的内核版本/commit
- 适用的 KernelSU 版本
- 是否经过真机 boot 测试
- 上游来源或原始作者
