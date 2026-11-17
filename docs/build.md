# 构建说明

## 工具链

- clang 18.1.8
- LLVM binutils
- 设备内核源码：本仓库
- KernelSU 子模块：`KernelSU/`（按分支固定版本）

## 构建步骤

先初始化子模块，并应用 KernelSU 兼容补丁：

```bash
git submodule update --init --recursive
git -C KernelSU apply ../patches/kernelsu-3.3.1-stale-hooks-compat.patch
git -C KernelSU apply ../patches/kernelsu-3.3.1-susfs-kconfig.patch
git -C KernelSU apply ../patches/kernelsu-3.3.1-susfs-init.patch
git -C KernelSU apply ../patches/kernelsu-3.3.1-susfs-prctl.patch
```

然后构建：

```bash
source ./.env.sh
./build.sh -v ksu defconfig
make O=../out/picasso/ksu ARCH=arm64 \
  CC=clang CROSS_COMPILE=aarch64-linux-gnu- \
  CROSS_COMPILE_COMPAT=arm-linux-gnueabi- \
  CLANG_TRIPLE=aarch64-linux-gnu- \
  LLVM=1 LLVM_IAS=1 LD=ld.lld -j$(nproc)
```


## SUSFS 2.4.0 分支

`feature/ksu3.3.1-susfs-2.4.0-on-73fe` 只使用一个 KernelSU 集成补丁（它已包含 Kconfig、`susfs_init()`、sucompat、setuid hook、sdcard monitor 和命令分发）：

```bash
git submodule update --init --recursive
git -C KernelSU apply ../patches/kernelsu-3.3.1-susfs-2.4.0.patch
```

内核侧完整 backport 为 `patches/susfs-4.19-v2.4.0.patch`，基于 1.5.11 基线（`ef536713b553`）到当前 2.4.0 分支；分支提交已包含等价改动，无需重复应用。
具体配置和打包命令以 `build.sh` 与 `scripts/` 下的脚本为准。构建产物不要提交到 Git。
