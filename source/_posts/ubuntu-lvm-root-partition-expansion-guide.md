---
title: 【生产环境】Ubuntu LVM 根分区扩容操作指南
date: 2026-06-30 10:00:00
tags:
  - Ubuntu
  - LVM
  - 磁盘扩容
  - ext4
  - 运维
categories: 运维
---

# 【生产环境】Ubuntu LVM 根分区扩容操作指南

**目标**：/home 缩减至 500G，/ 根分区扩容至 1.2T
**适用系统**：Ubuntu + LVM + ext4
**停机时间**：约 15~30 分钟
**风险等级**：低（/home 仅使用 4.7G，数据量极小）

## ⚠️ 执行前必读（非常重要）

1. 本次操作仅涉及文件系统大小调整，不删除业务数据。
2. PG/Redis 数据卷位于根分区，缩小 /home 对其零影响。
3. 操作需要 umount /home，期间 /home 不可用，请务必安排停机窗口。
4. 建议先执行一次 PG 全量备份（命令已包含在步骤中）。

## 一、前置确认（现在就执行，把结果截屏/保存）

```bash
# 1. 确认 VG/LV 结构
lsblk
lvdisplay
df -h / /home

# 2. 确认 /home 实际用量
du -sh /home

# 3. 确认 PG 容器名
docker ps | grep postgres
```

你应该看到的关键信息：
- /home 对应 LV 名称（通常是 `/dev/ubuntu-vg/lv-0` 或类似）
- /home 使用率极低（约 4~5G）
- 根 LV 名称是 `/dev/ubuntu-vg/ubuntu-lv`

下面指南中，统一用以下占位符，执行时请替换为你的真实值：

- `HOME_LV = /dev/ubuntu-vg/lv-0`
- `ROOT_LV = /dev/ubuntu-vg/ubuntu-lv`

## 二、停机 & 备份（安全第一）

### 1️⃣ 停止所有 Docker 服务

```bash
cd /home/disco-devops/local-setup-db
docker compose down
```

### 2️⃣ 确认所有容器已停止

```bash
docker ps
# 应为空
```

### 3️⃣ PG 数据备份（可选但强烈建议）

```bash
PG_CONTAINER=$(docker ps -a | grep postgres | awk '{print $1}')
docker start $PG_CONTAINER
sleep 5
docker exec -it $PG_CONTAINER pg_dumpall -U postgres > /root/pg_backup_$(date +%F).sql
docker stop $PG_CONTAINER
```

## 三、卸载 /home（关键步骤）

### 1️⃣ 确保 /home 没有被占用

```bash
lsof +D /home
# 如果有输出，kill 对应进程
```

### 2️⃣ 卸载 /home

```bash
umount /home
```

✅ 如果失败，说明仍有进程占用，用：

```bash
fuser -km /home
umount /home
```

## 四、缩小 /home 文件系统 & LV

### 1️⃣ 强制检查文件系统（ext4 必须）

```bash
e2fsck -f HOME_LV
```

> 这一步 **不可跳过**，否则数据可能损坏

### 2️⃣ 缩小文件系统到 480G（留一点余量）

```bash
resize2fs HOME_LV 480G
```

### 3️⃣ 缩小 LV 到 500G

```bash
lvreduce -L 500G HOME_LV
```

✅ 系统会提示确认，输入 `y`

## 五、重新挂载 /home 并验证

```bash
mount /home
df -h /home
```

✅ 应显示 /home 约为 500G，且数据完整

## 六、扩容根分区（核心目标）

### 1️⃣ 查看 VG 剩余空间

```bash
vgs
```

✅ 此时 VFree 应显示约 1.3T+

### 2️⃣ 扩容根 LV 到 1.2T

```bash
lvextend -L 1.2T ROOT_LV
```

### 3️⃣ 在线扩容根文件系统（ext4 支持）

```bash
resize2fs ROOT_LV
```

✅ 这一步 **不需要卸载根分区**，非常安全

## 七、验证最终结果

```bash
df -h /
df -h /home
vgs
lvs
```

期望结果：

| 分区 | 大小 | 使用率 |
| --- | --- | --- |
| / | ~1.2T | ~3% |
| /home | ~500G | <1% |
| VFree | ≈ 100G+ | 预留缓冲 |

## 八、恢复业务

```bash
cd /home/disco-devops/local-setup-db
docker compose up -d
docker ps
docker logs postgres
```

✅ 确认 PG / Redis / Mosquitto 均正常运行

## 九、设置开机自动挂载（确认）

```bash
cat /etc/fstab
```

应仍包含 /home 的挂载项（UUID 或 /dev/mapper 路径），无需修改。

## 十、回滚方案（以防万一）

### 若根扩容失败

```bash
lvextend -L +100G ROOT_LV
resize2fs ROOT_LV
```

### 若 /home 无法挂载

```bash
mount HOME_LV /home
# 或恢复到原大小
lvresize -L 1.7T HOME_LV
resize2fs HOME_LV
```

## 十一、后续建议（避免再踩坑）

1. 根分区 1.2T 足够 PG 长期增长
2. 定期清理无用镜像：

   ```bash
   docker image prune
   ```

3. 监控磁盘：

   ```bash
   df -h / /home
   ```

4. 不要再把业务数据放到根分区
5. 可考虑将 PG 数据目录显式挂载到 `/home/pgdata`（长期优化）

## ✅ 执行清单（Checklist）

- [ ] 已安排停机窗口
- [ ] 已备份 PG 数据
- [ ] 已确认 LV 名称
- [ ] 已停止 Docker
- [ ] 已卸载 /home
- [ ] 已执行 e2fsck
- [ ] 已缩小 /home
- [ ] 已扩容根分区
- [ ] 业务已恢复

> 如果你愿意，在执行前可以把下面三条命令的输出贴出来，我帮你二次确认 LV 名称和大小，确保万无一失：
>
> ```bash
> lsblk
> lvdisplay
> df -h / /home
> ```

你可以把这页文档当成"操作手册"，一步一步执行，有任何异常立刻停下来问我 👍
