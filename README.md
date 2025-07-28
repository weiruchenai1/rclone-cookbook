```markdown
# rclone 云存储挂载完整教程

基于实际配置经验的详细 rclone 使用指南

## 目录

- [1. 前置准备](#1-前置准备)
- [2. 配置云存储](#2-配置云存储)
- [3. 主流云存储配置参数](#3-主流云存储配置参数)
- [4. 验证配置](#4-验证配置)
- [5. 挂载到本地文件系统](#5-挂载到本地文件系统)
- [6. 创建系统服务](#6-创建系统服务)
- [7. 性能优化参数](#7-性能优化参数)
- [8. 常用管理命令](#8-常用管理命令)
- [9. 故障排除](#9-故障排除)
- [10. 最佳实践](#10-最佳实践)
- [11. 配置示例](#11-配置示例)

## 1. 前置准备

### 1.1 安装 rclone

```bash
# 安装 rclone
curl https://rclone.org/install.sh | sudo bash

# 验证安装
rclone version
```

### 1.2 准备云存储账户信息

在开始配置前，请准备以下信息：

- **Access Key ID** / **Secret Access Key**
- **存储桶名称**
- **区域信息**
- **端点地址**（如果需要）

## 2. 配置云存储

### 2.1 进入配置界面

```bash
rclone config
```

### 2.2 通用配置流程

```
n) New remote
name> 你的配置名称（建议格式：服务商-用途，如 s3do-media）
Storage> 4  # 选择 S3 兼容存储
provider> [根据服务商选择对应选项]
```

## 3. 主流云存储配置参数

### 3.1 Cloudflare R2

```ini
[cfr2-media]
type = s3
provider = Cloudflare
access_key_id = 你的R2_Access_Key_ID
secret_access_key = 你的R2_Secret_Access_Key
region = auto
endpoint = https://你的账户ID.r2.cloudflarestorage.com
acl = private
```

### 3.2 DigitalOcean Spaces

```ini
[do-spaces]
type = s3
provider = DigitalOcean
access_key_id = 你的DO_Access_Key_ID
secret_access_key = 你的DO_Secret_Access_Key
endpoint = 区域.digitaloceanspaces.com  # 如 sfo3.digitaloceanspaces.com
location_constraint = 区域代码  # 如 sfo3
acl = private
```

### 3.3 Amazon S3

```ini
[aws-s3]
type = s3
provider = AWS
access_key_id = 你的AWS_Access_Key_ID
secret_access_key = 你的AWS_Secret_Access_Key
region = us-east-1  # 你的区域
```

### 3.4 阿里云 OSS

```ini
[aliyun-oss]
type = s3
provider = Alibaba
access_key_id = 你的阿里云_Access_Key_ID
secret_access_key = 你的阿里云_Secret_Access_Key
endpoint = oss-cn-hangzhou.aliyuncs.com  # 你的区域端点
region = cn-hangzhou
```

### 3.5 腾讯云 COS

```ini
[tencent-cos]
type = s3
provider = TencentCOS
access_key_id = 你的腾讯云_Access_Key_ID
secret_access_key = 你的腾讯云_Secret_Access_Key
endpoint = cos.ap-guangzhou.myqcloud.com  # 你的区域端点
region = ap-guangzhou
```

## 4. 验证配置

### 4.1 测试连接

```bash
# 查看存储桶列表
rclone lsd 配置名称:

# 查看存储桶内容
rclone ls 配置名称:存储桶名称

# 测试文件操作
echo "test" | rclone rcat 配置名称:存储桶名称/test.txt
rclone cat 配置名称:存储桶名称/test.txt
```

### 4.2 查看配置信息

```bash
# 查看所有配置
rclone config show

# 查看配置文件位置
rclone config file

# 查看配置文件内容
cat ~/.config/rclone/rclone.conf
```

## 5. 挂载到本地文件系统

### 5.1 创建挂载目录

```bash
sudo mkdir -p /opt/video/你的挂载目录名
```

### 5.2 前台测试挂载

```bash
rclone mount 配置名称:存储桶名称 /opt/video/挂载目录 \
  --allow-other \
  --vfs-cache-mode minimal \
  --vfs-cache-max-size 1G \
  --buffer-size 16M \
  --dir-cache-time 30m \
  --transfers 2 \
  --checkers 2 \
  --dir-perms 0755 \
  --file-perms 0644 \
  --uid 0 \
  --gid 0 \
  -v
```

### 5.3 验证挂载（新终端）

```bash
# 检查挂载状态
df -h | grep 挂载目录
mount | grep 挂载目录

# 测试文件操作
echo "mount test" > /opt/video/挂载目录/test.txt
ls -la /opt/video/挂载目录
rm /opt/video/挂载目录/test.txt
```

### 5.4 挂载参数说明

| 参数 | 说明 |
|------|------|
| `--allow-other` | 允许其他用户访问挂载点 |
| `--vfs-cache-mode` | 缓存模式：off/minimal/writes/full |
| `--vfs-cache-max-size` | 最大缓存大小 |
| `--buffer-size` | 读写缓冲区大小 |
| `--dir-cache-time` | 目录缓存时间 |
| `--transfers` | 并发传输数 |
| `--checkers` | 并发检查数 |
| `--dir-perms` | 目录权限 |
| `--file-perms` | 文件权限 |

## 6. 创建系统服务

### 6.1 停止前台挂载

按 `Ctrl+C` 停止前台运行的 rclone

### 6.2 创建服务文件

```bash
sudo nano /etc/systemd/system/rclone-你的服务名.service
```

### 6.3 服务配置模板

```ini
[Unit]
Description=RClone mount 云存储描述
After=network.target

[Service]
Type=simple
ExecStartPre=/bin/mkdir -p /opt/video/挂载目录
ExecStart=/usr/bin/rclone mount 配置名称:存储桶名称 /opt/video/挂载目录 --allow-other --vfs-cache-mode minimal --vfs-cache-max-size 1G --buffer-size 16M --dir-cache-time 30m --transfers 2 --checkers 2 --dir-perms 0755 --file-perms 0644 --uid 0 --gid 0
ExecStop=/bin/fusermount -u /opt/video/挂载目录
Restart=on-failure
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
```

### 6.4 启用和启动服务

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 启用开机自启
sudo systemctl enable rclone-你的服务名.service

# 启动服务
sudo systemctl start rclone-你的服务名.service

# 检查状态
sudo systemctl status rclone-你的服务名.service
```

## 7. 性能优化参数

### 7.1 低配置服务器（2c2g）

```bash
--vfs-cache-mode minimal \
--vfs-cache-max-size 500M \
--buffer-size 8M \
--dir-cache-time 30m \
--transfers 1 \
--checkers 1
```

### 7.2 高配置服务器（4c8g+）

```bash
--vfs-cache-mode writes \
--vfs-cache-max-size 2G \
--buffer-size 32M \
--dir-cache-time 1h \
--transfers 4 \
--checkers 4
```

### 7.3 网络环境差的情况

```bash
--retries 10 \
--retries-sleep 10s \
--timeout 300s \
--contimeout 60s
```

### 7.4 缓存模式对比

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| `off` | 无缓存 | 内存极其有限 |
| `minimal` | 最小缓存 | 低配置服务器 |
| `writes` | 写入缓存 | 中等配置服务器 |
| `full` | 完全缓存 | 高配置服务器 |

## 8. 常用管理命令

### 8.1 服务管理

```bash
# 查看服务状态
sudo systemctl status rclone-服务名.service

# 重启服务
sudo systemctl restart rclone-服务名.service

# 停止服务
sudo systemctl stop rclone-服务名.service

# 启动服务
sudo systemctl start rclone-服务名.service

# 查看服务日志
journalctl -fu rclone-服务名.service

# 查看最近的日志
journalctl -fu rclone-服务名.service --since "10 minutes ago"
```

### 8.2 挂载管理

```bash
# 查看所有挂载
df -h | grep rclone
mount | grep rclone

# 手动卸载
sudo fusermount -u /opt/video/挂载目录

# 强制卸载
sudo umount -l /opt/video/挂载目录

# 检查挂载点是否被占用
lsof /opt/video/挂载目录
```

### 8.3 文件操作

```bash
# 直接操作云存储
rclone ls 配置名称:存储桶名称
rclone copy 本地路径 配置名称:存储桶名称/远程路径
rclone sync 本地路径 配置名称:存储桶名称/远程路径
rclone move 配置名称:存储桶名称/文件 配置名称:存储桶名称/新位置

# 通过挂载点操作（推荐）
cp 本地文件 /opt/video/挂载目录/
mv 本地文件 /opt/video/挂载目录/
rsync -av 本地目录/ /opt/video/挂载目录/
```

### 8.4 监控命令

```bash
# 查看 rclone 进程
ps aux | grep rclone

# 查看资源使用
htop
top -p $(pgrep rclone)

# 查看网络使用
iftop
nethogs

# 测试传输速度
rclone test speed 配置名称:存储桶名称
```

## 9. 故障排除

### 9.1 常见错误及解决

#### 挂载失败：directory already mounted

```bash
sudo fusermount -u /opt/video/挂载目录
sudo systemctl restart rclone-服务名.service
```

#### 权限错误

```bash
# 添加权限参数
--dir-perms 0755 --file-perms 0644 --uid 0 --gid 0

# 检查用户权限
id
groups
```

#### 连接超时

```bash
# 增加超时参数
--timeout 300s --contimeout 60s --retries 10

# 检查网络连接
ping 端点域名
telnet 端点域名 443
```

#### 配置文件错误

```bash
# 检查配置
rclone config show
cat ~/.config/rclone/rclone.conf

# 验证配置语法
rclone config show 配置名称

# 重新配置
rclone config
```

#### 磁盘空间不足

```bash
# 检查磁盘空间
df -h

# 清理缓存
sudo rm -rf /tmp/rclone-*
sudo rm -rf ~/.cache/rclone/

# 调整缓存大小
--vfs-cache-max-size 500M
```

### 9.2 性能问题诊断

```bash
# 查看挂载进程资源使用
ps aux | grep rclone
htop -p $(pgrep rclone)

# 查看网络使用
iftop
ss -tuln | grep :443

# 查看缓存使用
du -sh ~/.cache/rclone/
df -h /tmp | grep rclone

# 测试传输速度
rclone test speed 配置名称:存储桶名称
time rclone copy 测试文件 配置名称:存储桶名称/
```

### 9.3 调试模式

```bash
# 详细日志模式
rclone mount 配置名称:存储桶名称 /opt/video/挂载目录 -vv

# 极详细模式
rclone mount 配置名称:存储桶名称 /opt/video/挂载目录 -vvv

# 输出到日志文件
rclone mount 配置名称:存储桶名称 /opt/video/挂载目录 --log-file /var/log/rclone.log -v
```

## 10. 最佳实践

### 10.1 命名规范

| 组件 | 命名格式 | 示例 |
|------|----------|------|
| 配置名称 | `服务商-用途` | `cfr2-media`, `s3do-backup` |
| 服务名称 | `rclone-配置名称.service` | `rclone-cfr2-media.service` |
| 挂载目录 | `/opt/video/配置名称` | `/opt/video/cfr2-media` |

### 10.2 安全建议

- ✅ 使用 `acl = private` 确保文件私有
- ✅ 定期轮换访问密钥
- ✅ 限制 API 权限范围
- ✅ 使用 IAM 用户而非根用户
- ✅ 启用访问日志记录
- ✅ 设置适当的存储桶策略

### 10.3 监控建议

```bash
# 定期检查服务状态
sudo systemctl status rclone-*.service

# 监控磁盘空间
df -h
du -sh /opt/video/*

# 监控网络使用
iftop
vnstat

# 设置日志轮转
sudo nano /etc/logrotate.d/rclone
```

### 10.4 备份配置

```bash
# 备份 rclone 配置
cp ~/.config/rclone/rclone.conf ~/rclone.conf.backup.$(date +%Y%m%d)

# 备份服务配置
sudo cp /etc/systemd/system/rclone-*.service ~/services-backup/

# 创建配置备份脚本
#!/bin/bash
DATE=$(date +%Y%m%d)
cp ~/.config/rclone/rclone.conf ~/backups/rclone.conf.$DATE
sudo cp /etc/systemd/system/rclone-*.service ~/backups/
```

### 10.5 性能调优

```bash
# 根据带宽调整参数
# 1Gbps 网络
--transfers 8 --checkers 8 --buffer-size 32M

# 100Mbps 网络  
--transfers 4 --checkers 4 --buffer-size 16M

# 10Mbps 网络
--transfers 2 --checkers 2 --buffer-size 8M
```

## 11. 配置示例

### 11.1 完整的三云存储配置示例

```ini
# ~/.config/rclone/rclone.conf
[cfr2-media]
type = s3
provider = Cloudflare
access_key_id = your_r2_key
secret_access_key = your_r2_secret
region = auto
endpoint = https://account.r2.cloudflarestorage.com
acl = private

[s3do-backup]
type = s3
provider = DigitalOcean
access_key_id = your_do_key
secret_access_key = your_do_secret
endpoint = sfo3.digitaloceanspaces.com
location_constraint = sfo3
acl = private

[aws-archive]
type = s3
provider = AWS
access_key_id = your_aws_key
secret_access_key = your_aws_secret
region = us-east-1
storage_class = GLACIER
```

### 11.2 多服务配置脚本

```bash
#!/bin/bash
# multi-rclone-setup.sh

# 定义配置数组
declare -A configs=(
    ["cfr2-media"]="/opt/video/cfr2-media"
    ["s3do-backup"]="/opt/video/s3do-backup"
    ["aws-archive"]="/opt/video/aws-archive"
)

# 为每个配置创建服务
for config in "${!configs[@]}"; do
    mount_point="${configs[$config]}"
    service_name="rclone-$config.service"
    
    echo "Setting up $service_name..."
    
    # 创建挂载目录
    sudo mkdir -p "$mount_point"
    
    # 创建服务文件
    sudo tee "/etc/systemd/system/$service_name" > /dev/null <<EOF
[Unit]
Description=RClone mount for $config
After=network.target

[Service]
Type=simple
ExecStartPre=/bin/mkdir -p $mount_point
ExecStart=/usr/bin/rclone mount $config: $mount_point --allow-other --vfs-cache-mode minimal --vfs-cache-max-size 1G --buffer-size 16M --dir-cache-time 30m --transfers 2 --checkers 2 --dir-perms 0755 --file-perms 0644 --uid 0 --gid 0
ExecStop=/bin/fusermount -u $mount_point
Restart=on-failure
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
EOF

    # 启用并启动服务
    sudo systemctl daemon-reload
    sudo systemctl enable "$service_name"
    sudo systemctl start "$service_name"
    
    echo "$service_name setup complete!"
done

echo "All rclone services configured!"
```

### 11.3 健康检查脚本

```bash
#!/bin/bash
# rclone-health-check.sh

# 检查所有 rclone 服务
services=$(systemctl list-units --type=service | grep rclone | awk '{print $1}')

for service in $services; do
    status=$(systemctl is-active "$service")
    if [ "$status" != "active" ]; then
        echo "WARNING: $service is $status"
        # 可以添加重启逻辑
        # sudo systemctl restart "$service"
    else
        echo "OK: $service is running"
    fi
done

# 检查挂载点
mount_points=$(mount | grep rclone | awk '{print $3}')
for mount_point in $mount_points; do
    if [ -d "$mount_point" ] && mountpoint -q "$mount_point"; then
        echo "OK: $mount_point is mounted"
        # 测试写入
        if touch "$mount_point/.health_check" 2>/dev/null; then
            rm "$mount_point/.health_check"
            echo "OK: $mount_point is writable"
        else
            echo "WARNING: $mount_point is not writable"
        fi
    else
        echo "ERROR: $mount_point is not mounted"
    fi
done
```

### 11.4 Docker Compose 集成示例

```yaml
# docker-compose.yml
version: '3.8'

services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
    volumes:
      - ./jellyfin/config:/config
      - /opt/video:/media  # 包含所有 rclone 挂载点
    ports:
      - "8096:8096"
    restart: unless-stopped
    depends_on:
      - rclone-health-check

  rclone-health-check:
    image: alpine:latest
    container_name: rclone-health-check
    command: |
      sh -c "
        apk add --no-cache curl
        while true; do
          # 检查挂载点
          if [ ! -f /media/cfr2-media/.mounted ]; then
            echo 'cfr2-media not mounted'
            exit 1
          fi
          if [ ! -f /media/s3do-backup/.mounted ]; then
            echo 's3do-backup not mounted'  
            exit 1
          fi
          echo 'All mounts healthy'
          sleep 30
        done
      "
    volumes:
      - /opt/video:/media
    restart: unless-stopped
```

---

## 📝 结语

这份教程涵盖了 rclone 的完整使用流程，从基础配置到高级优化，适用于各种云存储服务。建议：

1. **先在测试环境验证配置**
2. **根据实际需求调整参数**
3. **定期备份配置文件**
4. **监控服务运行状态**
5. **及时更新 rclone 版本**

如有问题，请参考官方文档：https://rclone.org/docs/

---

*最后更新：2025-07-28*
```