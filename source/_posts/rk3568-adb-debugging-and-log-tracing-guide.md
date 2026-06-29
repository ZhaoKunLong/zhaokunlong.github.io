---
title: RK3568 设备 ADB 调试与日志追踪手册
date: 2026-06-29 10:00:00
tags:
  - Android
  - ADB
  - RK3568
  - 调试
  - 日志
categories: 技术
---

# RK3568 设备 ADB 调试与日志追踪手册

**设备信息**
- 型号：Rockchip RK3568，Android 11
- 序列号：`3e093fcc15cd5416`
- ADB 地址：`127.0.0.1:15555`
- 设备 IP：`10.1.1.26`

---

## 一、本次问题根因

**现象**：设备 2026-06-26 约 14:00 掉线，约 17:15 经 POE 交换机断电重启后恢复。

**结论**：设备被 `com.android.rk.service`（Rockchip OEM 管理服务）触发了软件重启（`reboot,userrequested`），但重启过程卡死，设备无法自行完成重启。POE 断电提供了强制硬重置，设备才恢复。

**证据来源**：`/sys/fs/pstore/console-ramoops-0`（前次启动内核日志）

```
[1598s] init: Received sys.powerctl='reboot,userrequested' from pid:441 (system_server)
[1599s] reboot: Restarting system with command 'userrequested'
```

---

## 二、连接设备

```bash
adb connect 127.0.0.1:15555
```

验证连接：
```bash
adb -s 127.0.0.1:15555 shell uptime
```

---

## 三、持久化日志配置（已完成，重启永久生效）

> **本次已配置完毕，无需重复执行。**

```bash
adb -s 127.0.0.1:15555 shell setprop persist.logd.logpersistd logcatd
adb -s 127.0.0.1:15555 shell setprop persist.logd.logpersistd.buffer all
adb -s 127.0.0.1:15555 shell setprop persist.logd.logpersistd.rotate_kbytes 102400
adb -s 127.0.0.1:15555 shell setprop persist.logd.logpersistd.size 5
```

**效果**：日志写入 `/data/misc/logd/`，上限 500MB（100MB × 5个文件滚动），覆盖约 33-100 小时，跨重启保留，无需 ADB 持续连接。

验证配置：
```bash
adb -s 127.0.0.1:15555 shell getprop persist.logd.logpersistd
adb -s 127.0.0.1:15555 shell getprop persist.logd.logpersistd.rotate_kbytes
adb -s 127.0.0.1:15555 shell ls -lh /data/misc/logd/
```

---

## 四、下次掉线后的排查步骤

### 步骤 1：连接设备

```bash
adb connect 127.0.0.1:15555
```

### 步骤 2：查看前次启动的关键信息

```bash
# 重启原因
adb -s 127.0.0.1:15555 shell getprop sys.boot.reason

# 前次启动末尾的 Android 日志（快速查看）
adb -s 127.0.0.1:15555 shell logcat -L -b all | grep -E "(reboot|died|crash|killed|shutdown)"

# 前次启动的内核日志（查看重启原因）
adb -s 127.0.0.1:15555 shell cat /sys/fs/pstore/console-ramoops-0 | grep -E "(reboot|panic|watchdog|powerctl)"
```

### 步骤 3：拉取完整历史日志

```bash
# 先复制到 sdcard（绕过权限限制）
adb -s 127.0.0.1:15555 shell "mkdir -p /sdcard/device_logs && cp /data/misc/logd/logcat* /sdcard/device_logs/"

# 拉取到本地
adb -s 127.0.0.1:15555 pull /sdcard/device_logs/ ~/Desktop/

# 清理设备上的临时文件
adb -s 127.0.0.1:15555 shell "rm -rf /sdcard/device_logs/"
```

### 步骤 4：在日志中搜索关键词

```bash
# 在设备上直接搜索（不用拉下来）
adb -s 127.0.0.1:15555 shell "grep -h 'died\|reboot\|userrequested\|crash\|killed' /data/misc/logd/logcat* 2>/dev/null | grep '掉线时间段'"

# 例如搜索特定时间段
adb -s 127.0.0.1:15555 shell "grep -h '06-26 14:' /data/misc/logd/logcat* 2>/dev/null"
```

---

## 五、常用快速检查命令

```bash
# 当前开机时长
adb -s 127.0.0.1:15555 shell uptime

# 查看存储空间
adb -s 127.0.0.1:15555 shell df -h /data /sdcard

# 查看日志文件大小
adb -s 127.0.0.1:15555 shell ls -lh /data/misc/logd/

# 实时查看日志（调试用）
adb -s 127.0.0.1:15555 shell logcat -v time | grep -E "(reboot|died|crash)"
```

---

## 六、注意事项

- `logcat.id` 文件无读取权限，忽略即可，不影响日志内容
- 日志文件命名：`logcat`（最新）→ `logcat.001` → `logcat.002`（最旧）
- 设备上运行的 `wput` 进程会定期上传 `board.txt` 到 `123.207.119.35`，属于 Rockchip OEM 服务行为，目前未确认是否为预期行为，后续可排查
