# patches

这里放可复用的补丁系列，按上游可提交的最小改动组织。

当前实际文件：

- `susfs-4.19-v1.5.11.patch`：上游 `88836` 的 SUSFS v1.5.11 内核侧 4.19 补丁（目前在 `feature/susfs-1.5.11` 上验证编译）
- `kernelsu-3.3.1-stale-hooks-compat.patch`：为 upstream `staging` 的旧 manual hook 引用补 KSU 兼容符号
- `kernelsu-3.3.1-susfs-kconfig.patch`：为 KernelSU Kconfig 增加 SUSFS 选项
- `kernelsu-3.3.1-susfs-init.patch`：在 KernelSU 初始化时调用 `susfs_init()`
- `kernelsu-3.3.1-susfs-prctl.patch`：在 KSU 3.3.1 的 LSM `task_prctl` 上桥接 SUSFS 用户态命令（`ksu_susfs`），真机已验证 `enable_log`

说明：以上 KernelSU 补丁均已通过编译验证；`kernelsu-3.3.1-susfs-prctl.patch` 已在 73fe 真机验证 root、SUSFS 初始化和 `ksu_susfs enable_log`。SUSFS v2.4.0 的 4.19 backport 仍是目标，不是当前已合并状态。

每个补丁必须注明：

- 适用的内核版本/commit
- 适用的 KernelSU 版本
- 是否经过真机 boot 测试
- 上游来源或原始作者
