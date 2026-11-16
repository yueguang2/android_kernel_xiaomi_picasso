# patches

这里放可复用的补丁系列，按上游可提交的最小改动组织。

当前实际文件：

- `susfs-4.19-v1.5.11.patch`：上游 `88836` 的 SUSFS v1.5.11 内核侧 4.19 补丁（目前在 `feature/susfs-1.5.11` 上验证编译）
- `kernelsu-3.3.1-stale-hooks-compat.patch`：为 upstream `staging` 的旧 manual hook 引用补 KSU 兼容符号
- `kernelsu-3.3.1-susfs-kconfig.patch`：为 KernelSU Kconfig 增加 SUSFS 选项
- `kernelsu-3.3.1-susfs-init.patch`：在 KernelSU 初始化时调用 `susfs_init()`

说明：以上 KernelSU 三个补丁都已通过编译验证；尚未做真机 boot/root 测试。SUSFS v2.4.0 的 4.19 backport 仍是目标，不是当前已合并状态。

每个补丁必须注明：

- 适用的内核版本/commit
- 适用的 KernelSU 版本
- 是否经过真机 boot 测试
- 上游来源或原始作者
