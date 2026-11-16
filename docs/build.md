# 构建说明

## 工具链

- clang 18.1.8
- LLVM binutils
- 设备内核源码：本仓库
- KernelSU 子模块：`KernelSU/`（按分支固定版本）

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
