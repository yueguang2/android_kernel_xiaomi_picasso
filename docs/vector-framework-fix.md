# Vector 框架状态异常修复记录

设备：Redmi K30 5G / picasso / YAAP 16  
内核：KernelSU 3.3.1 + SUSFS 2.4.0 4.19 backport

## 现象

Vector Manager 首页显示“异常”，系统状态页显示：

- `SELinux 策略：未加载`
- `系统框架：未注入`
- `Dex 优化器包装：已崩溃` 或“不可用”

实际 `vectord` 进程可以启动，但 Vector 的 `SELinux.checkSELinuxAccess()` 返回 false，导致 Dex2OAT wrapper 选择错误的文件上下文；同时启动时机过晚，system_server 已经完成 specialization。

## 根因与修复

### 1. 模块 sepolicy 仍使用旧版 ksud

设备上此前的 `/data/adb/post-fs-data.d/00_apply_sepolicy.sh` 使用了 `/data/adb/ksud.official`（KernelSU 1.0.5），与 3.3.1 内核不匹配，导致 Vector 的 `sepolicy.rule` 实际没有加载。

修复为使用当前 KernelSU 的 `/data/adb/ksu/bin/ksud`：

```sh
#!/system/bin/sh
for f in /data/adb/modules/*/sepolicy.rule; do
  [ -f "$f" ] && /data/adb/ksu/bin/ksud sepolicy apply "$f" >/dev/null 2>&1
done
exit 0
```

### 2. Vector daemon 启动太晚

`/data/adb/post-fs-data.d/99vector.sh` 原来延迟 25 秒启动 `service.sh`，system_server 已经完成注入窗口；改为约 2 秒启动。`service.sh` 增加幂等判断，避免出现两个 `vectord` 进程：

```sh
if pgrep -x vectord >/dev/null 2>&1; then
    exit 0
fi
```

并改用 KernelSU 自带的 busybox `unshare`（系统 toybox 的 `unshare` 不支持 `--propagation`）：

```sh
/data/adb/ksu/bin/busybox unshare --propagation slave -m "$MODDIR/daemon" --system-server-max-retry=3 "$@" &
```

### 3. SUSFS AVD 伪装范围过宽

SUSFS 2.4.0 的 `susfs_spoof_avd_allowed_rcu()` 会对所有 KernelSU 添加的策略规则恢复“原始 allowed”。Vector 的 `dex2oat -> dex2oat_exec` 规则虽然是实际生效的，但通过 `SELinux.checkSELinuxAccess()` 查询时被错误地伪装成不允许，导致 Vector 判断“策略未加载”。

在 `security/selinux/ss/services.c` 中把伪装限制到 **涉及 SUSFS type 的策略对**；普通模块策略（例如 Vector 的 dex2oat 规则）保持对 userspace 可见。实际 SELinux 内核执行路径不受影响。

## 验证

修复并重启后，Vector Manager 系统状态页显示：

- `SELinux 策略：已加载`
- `系统框架：已注入`
- `Dex 优化器包装：支持`

同时 `vectord` 只保留一个进程，system_server 日志出现 `Injected Vector framework into system_server.`。本次修复对应的内核改动随 SUSFS 2.4.0 完整镜像发布。
