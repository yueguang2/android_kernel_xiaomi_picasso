# 构建说明

## 工具链

- clang 18.1.8
- LLVM binutils
- 设备内核源码：本仓库
- KernelSU 子模块：`KernelSU/`（按分支固定版本）

## clang 18 兼容

上游 `staging` 的 Makefile 对 clang 开启 MLGO `-regalloc-enable-advisor`，clang 18 不支持该选项，会报 `Requested regalloc eviction advisor analysis could not be created`。本仓库在 `Makefile` 中为 clang 18 关闭该选项；如果使用 clang 19+ 可以恢复。

## 构建步骤

```bash
export PATH=/path/to/clang/bin:/path/to/host/usr/bin:$PATH
make O=out ARCH=arm64 \
  CC=clang CROSS_COMPILE=aarch64-linux-gnu- \
  CROSS_COMPILE_COMPAT=arm-linux-gnueabi- \
  CLANG_TRIPLE=aarch64-linux-gnu- \
  LLVM=1 LLVM_IAS=1 LD=ld.lld -j$(nproc)
```

具体配置和打包命令以 `scripts/` 下的脚本为准。构建产物不要提交到 Git。
