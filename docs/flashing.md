# 刷入与回滚

## 前置条件

- bootloader 已解锁
- 已安装 adb/fastboot
- 已保存当前可启动的 boot 镜像

## 刷入测试镜像

```bash
fastboot flash boot ksu_test.img
fastboot reboot
```

## 回滚

```bash
fastboot flash boot ksu_boot.img
fastboot reboot
```

不要在没有备份的情况下刷入实验镜像。首次测试建议先用 `fastboot boot` 临时启动（设备支持时），确认稳定后再写入 boot 分区。
