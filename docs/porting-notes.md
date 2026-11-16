# 移植笔记

## 目标版本

- KernelSU：官方最新稳定版或 4.19 兼容的 backslashxx legacy 分支
- SUSFS：v2.4.0（目前只有 GKI 5.10+ 分支），需要 backport 到 4.19
- Vector：保持当前 NeoZygisk 方案

## 现状

- 当前稳定组合是 **KernelSU 1.0.5 + SUSFS 1.5.5**，已在 picasso 上验证。
- 上游 `august-aosp/android_kernel_xiaomi_picasso` 的 `staging` 已包含 `KernelSU: Switch to backslashxx fork (v3.3.1+)`（commit `920c3be1ccd4`），说明最新 KernelSU 的旧内核适配已有现成基础。
- 上游历史中有 `treewide: Backport SUSFS v1.5.11 for k4.19`（commit `88836b6906c1`），后来被 revert；可作为 4.19 SUSFS 的参考，但不能直接当作稳定实现。
- `susfs4ksu` 的 `kernel-4.19` 分支停在 v1.5.5；v2.4.0 只在 GKI 5.10+ 分支。
- `rsuntk/KernelSU` 的 `susfs-rksu-master` 分支内置了 SUSFS 调用和 `KernelSU - SUSFS` 配置，内核侧仍需要单独的 `fs/susfs.c` 补丁；可作为“旧内核 + SUSFS”的参考实现，但它对应的 KernelSU 版本较旧，不能直接当作最新版方案。

## 移植难点

1. SUSFS v2.x 的 VFS 钩子（`fs/namei.c`、`fs/namespace.c`、`fs/stat.c`、`fs/readdir.c` 等）基于 5.10+ 的接口，4.19 需要逐项回移。
2. KernelSU 3.3.1 使用新的 supercall/LSM 结构，旧版 SUSFS 的 prctl 接入不能直接复用。
3. 需要真机测试 boot、root、SUSFS、Vector、模块安装，不能只看编译通过。

## 测试清单

- [ ] 正常开机
- [ ] `su -c id` 返回 root
- [ ] `ksu_susfs show version`
- [ ] `ksu_susfs show enabled_features`
- [ ] Vector 显示已激活
- [ ] Xposed 模块安装并重启生效
- [ ] 连续重启 3 次
- [ ] 常用检测类 App 不异常
- [ ] 记录回滚镜像与步骤
