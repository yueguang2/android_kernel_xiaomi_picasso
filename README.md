# picasso-kernel-ksu-susfs

Redmi K30 5G (**picasso**, SM7250/lito) 的 **4.19 非 GKI** 内核分支，目标是把 KernelSU、SUSFS 和 Vector 做成可复现、可回滚、别人也能使用的方案。

仓库地址：<https://github.com/yueguang2/android_kernel_xiaomi_picasso>（原仓库 fork）

## 当前状态

| 分支 | 内容 | 状态 |
| --- | --- | --- |
| `stable-ksu-1.0.5-susfs-1.5.5` | KernelSU 1.0.5 + SUSFS 1.5.5 + NeoZygisk + Vector | 设备已验证 |
| `next/ksu-3.3.1-susfs-2.4.0` | KernelSU 3.3.1+ + SUSFS 4.19 backport | 开发中 |
| `feature/ksu3.3.1-on-73fe` | KernelSU 3.3.1 + 73fe 稳定基线 | 编译通过；真机启动、`su -c id` 已验证；SUSFS 待接入 |
| `feature/ksu3.3.1-susfs-1.5.11-on-73fe` | KernelSU 3.3.1 + SUSFS 1.5.11 + 73fe 稳定基线 | 真机启动、root、`susfs is initialized v1.5.11`、Vector/Xposed 模块、微信/银行登录和 `ksu_susfs` 命令桥接已验证 |

## 设备与系统

- 设备：Redmi K30 5G / `picasso`
- SoC：SM7250 / lito
- 内核：`4.19.248-NonLa-2.0-g73fe7678e96a`
- ROM：YAAP 16（Android 16 / SDK 36）
- 分区：单 boot，无 A/B，dynamic super
- 启动：bootloader 已解锁

## 上游关系

- 内核上游：[august-aosp/android_kernel_xiaomi_picasso](https://github.com/august-aosp/android_kernel_xiaomi_picasso)
- 上游 draft PR：[susfs: backport v1.5.11 to 4.19 (#1)](https://github.com/august-aosp/android_kernel_xiaomi_picasso/pull/1)
- KernelSU：[tiann/KernelSU](https://github.com/tiann/KernelSU)；旧内核兼容参考 [backslashxx/KernelSU](https://github.com/backslashxx/KernelSU)
- SUSFS：[simonpunk/susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu)
- Zygisk：[NeoZygisk](https://github.com/Dr-TSNG/NeoZygisk) + [Vector](https://github.com/JingMatrix/Vector)

## 构建

构建脚本和依赖说明见 `docs/build.md`。构建产物不提交到仓库，boot 镜像通过 Releases 发布。

当前测试版 Release：[v2.0.1-ksu3.3.1-susfs1.5.11](https://github.com/yueguang2/android_kernel_xiaomi_picasso/releases/tag/v2.0.1-ksu3.3.1-susfs1.5.11)

SUSFS 模块建议使用 Release 中的 `susfs4ksu_sidex_v1.5.2-R28.zip`；安装脚本会读取内核 SUSFS 版本，安装后模块版本自动显示为 `v1.5.11-R28`。

## 刷入与回滚

见 `docs/flashing.md`。任何实验镜像都必须保留原 boot 备份，确认能进入 fastboot 后再刷入。

## 许可证

内核及 KernelSU/SUSFS 相关代码遵循各自上游许可证；本仓库保留原始版权声明。详见 `LICENSE` 与 `NOTICE`。
