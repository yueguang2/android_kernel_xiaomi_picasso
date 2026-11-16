# patches

这里放可复用的补丁系列，按上游可提交的最小改动组织。

计划中的文件：

- `susfs-4.19-v2.4.0-backport.patch`：SUSFS v2.4.0 内核侧 4.19 backport
- `kernelsu-3.3.1-susfs-integration.patch`：KernelSU 3.3.1 的 SUSFS 接入
- `build-clang18-compat.patch`：仅为本地 clang 18 构建关闭 MLGO 选项，不改变内核功能

每个补丁必须注明：

- 适用的内核版本/commit
- 适用的 KernelSU 版本
- 是否经过真机 boot 测试
- 上游来源或原始作者
