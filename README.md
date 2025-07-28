# rclone 云存储挂载完整教程

> 基于实际配置经验的详细 rclone 使用指南

[![](https://img.shields.io/badge/rclone-v1.60+-blue.svg)](https://rclone.org/)
[![](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![](https://img.shields.io/badge/language-中文-red.svg)]()

## 📋 目录

- [前置准备](#前置准备)
- [配置云存储](#配置云存储)
- [主流云存储配置](#主流云存储配置)
- [验证配置](#验证配置)
- [挂载到本地文件系统](#挂载到本地文件系统)
- [创建系统服务](#创建系统服务)
- [性能优化参数](#性能优化参数)
- [常用管理命令](#常用管理命令)
- [故障排除](#故障排除)
- [最佳实践](#最佳实践)
- [配置示例](#配置示例)

## 🚀 快速开始

```bash
# 1. 安装 rclone
curl https://rclone.org/install.sh | sudo bash

# 2. 配置云存储
rclone config

# 3. 测试连接
rclone ls your-remote:your-bucket

# 4. 挂载到本地
sudo mkdir -p /opt/video/mount-point
rclone mount your-remote:your-bucket /opt/video/mount-point --daemon
```

## 前置准备

### 安装 rclone

```bash
# 使用官方脚本安装
curl https://rclone.org/install.sh | sudo bash

# 验证安装
rclone version
```

### 准备云存储信息

在开始配置前，请准备：

| 信息项 | 说明 | 示例 |
|--------|------|------|
| Access Key ID | 访问密钥 ID | `AKIAIOSFODNN7EXAMPLE` |
| Secret Access Key | 访问密钥 | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` |
| 存储桶名称 | bucket 名称 | `my-media-bucket` |
| 区域 | 存储区域 | `us-east-1` |
| 端点 | 自定义端点（可选） | `s3.amazonaws.com` |

## 配置云存储

### 进入配置界面

```bash
rclone config
```

### 配置流程

```
n) New remote
name> your-config-name
Storage> 4  # S3 兼容存储
provider> [选择对应的云服务商]
```

## 主流云存储配置

<details>
<summary>🌐 Cloudflare R2</summary>

```ini
[cfr2-media]
type = s3
provider = Cloudflare
access_key_id = your_r2_access_key
secret_access_key = your_r2_secret_key
region = auto
endpoint = https://your-account-id.r2.cloudflarestorage.com
acl = private
```

**配置步骤：**
1. 登录 Cloudflare Dashboard
2. 进入 R2 Object Storage
3. 创建 API 令牌
4. 获取账户 ID 和端点信息

</details>

<details>
<summary>🌊 DigitalOcean Spaces</summary>

```ini
[do-spaces]
type = s3
provider = DigitalOcean
access_key_id = your_do_access_key
secret_access_key = your_do_secret_key
endpoint = sfo3.digitaloceanspaces.com
location_constraint = sfo3
acl = private
```

**可用区域：**
- `nyc3` - New York 3
- `sfo3` - San Francisco 3
- `sgp1` - Singapore 1
- `fra1` - Frankfurt 1

</details>

<details>
<summary>☁️ Amazon S3</summary>

```ini
[aws-s3]
type = s3
provider = AWS
access_key_id = your_aws_access_key
secret_access_key = your_aws_secret_key
region = us-east-1
```

</details>

<details>
<summary>🇨🇳 阿里云 OSS</summary>

```ini
[aliyun-oss]
type = s3
provider = Alibaba
access_key_id = your_aliyun_access_key
secret_access_key = your_aliyun_secret_key
endpoint = oss-cn-hangzhou.aliyuncs.com
region = cn-hangzhou
```

</details>

## 验证配置

```bash
# 查看配置
rclone config show

# 测试连接
rclone lsd your-config:
rclone ls your-config:your-bucket

# 上传测试文件
echo "test" | rclone rcat your-config:your-bucket/test.txt
rclone cat your-config:your-bucket/test.txt
```

## 挂载到本地文件系统

### 创建挂载目录

```bash
sudo mkdir -p /opt/video/your-mount-point
```

### 前台测试挂载

```bash
rclone mount your-config:your-bucket /opt/video/your-mount-point \
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

### 验证挂载

在新终端中执行：

```bash
# 检查挂载状态
df -h | grep your-mount-point
mount | grep your-mount-point

# 测试文件操作
echo "mount test" > /opt/video/your-mount-point/test.txt
ls -la /opt/video/your-mount-point/
rm /opt/video/your-mount-point/test.txt
```

### 挂载参数说明

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `--allow-other` | 允许其他用户访问 | 必需 |
| `--vfs-cache-mode` | 缓存模式 | `minimal` |
| `--vfs-cache-max-size` | 最大缓存大小 | `1G` |
| `--buffer-size` | 缓冲区大小 | `16M` |
| `--dir-cache-time` | 目录缓存时间 | `30m` |
| `--transfers` | 并发传输数 | `2` |
| `--checkers` | 并发检查数 | `2` |

## 创建系统服务

### 停止前台挂载

按 `Ctrl+C` 停止前台测试

### 创建服务文件

```bash
sudo nano /etc/systemd/system/rclone-your-service.service
```

### 服务配置

```ini
[Unit]
Description=RClone mount for your-service
After=network.target

[Service]
Type=simple
ExecStartPre=/bin/mkdir -p /opt/video/your-mount-point
ExecStart=/usr/bin/rclone mount your-config:your-bucket /opt/video/your-mount-point \
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
  --gid 0
ExecStop=/bin/fusermount -u /opt/video/your-mount-point
Restart=on-failure
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
```

### 启用服务

```bash
# 重新加载配置
sudo systemctl daemon-reload

# 启用开机自启
sudo systemctl enable rclone-your-service.service

# 启动服务
sudo systemctl start rclone-your-service.service

# 检查状态
sudo systemctl status rclone-your-service.service
```

## 性能优化参数

### 低配置服务器 (2C2G)

```bash
--vfs-cache-mode minimal \
--vfs-cache-max-size 500M \
--buffer-size 8M \
--dir-cache-time 30m \
--transfers 1 \
--checkers 1
```

### 高配置服务器 (4C8G+)

```bash
--vfs-cache-mode writes \
--vfs-cache-max-size 2G \
--buffer-size 32M \
--dir-cache-time 1h \
--transfers 4 \
--checkers 4
```

### 网络环境差

```bash
--retries 10 \
--retries-sleep 10s \
--timeout 300s \
--contimeout 60s
```

## 常用管理命令

### 服务管理

```bash
# 查看状态
sudo systemctl status rclone-*.service

# 重启服务
sudo systemctl restart rclone-your-service.service

# 查看日志
journalctl -fu rclone-your-service.service

# 查看最近日志
journalctl -fu rclone-your-service.service --since "10 minutes ago"
```

### 挂载管理

```bash
# 查看所有挂载
df -h | grep rclone
mount | grep rclone

# 手动卸载
sudo fusermount -u /opt/video/your-mount-point

# 强制卸载
sudo umount -l /opt/video/your-mount-point
```

### 文件操作

```bash
# 直接操作云存储
rclone ls your-config:your-bucket
rclone copy local-file your-config:your-bucket/remote-path
rclone sync local-dir your-config:your-bucket/remote-dir

# 通过挂载点操作
cp local-file /opt/video/your-mount-point/
mv local-file /opt/video/your-mount-point/
```

## 故障排除

### 常见错误

#### 🔧 挂载失败：directory already mounted

```bash
sudo fusermount -u /opt/video/your-mount-point
sudo systemctl restart rclone-your-service.service
```

#### 🔧 权限错误

```bash
# 检查权限参数
--dir-perms 0755 --file-perms 0644 --uid 0 --gid 0

# 检查用户权限
id
groups
```

#### 🔧 连接超时

```bash
# 增加超时参数
--timeout 300s --contimeout 60s --retries 10

# 检查网络
ping your-endpoint
telnet your-endpoint 443
```

#### 🔧 配置文件错误

```bash
# 检查配置
rclone config show your-config

# 重新配置
rclone config
```

### 性能问题

```bash
# 查看资源使用
ps aux | grep rclone
htop -p $(pgrep rclone)

# 查看网络使用
iftop
ss -tuln | grep :443

# 测试速度
rclone test speed your-config:your-bucket
```

## 最佳实践

### ✅ 命名规范

| 组件 | 格式 | 示例 |
|------|------|------|
| 配置名 | `provider-purpose` | `cfr2-media` |
| 服务名 | `rclone-config.service` | `rclone-cfr2-media.service` |
| 挂载点 | `/opt/video/config` | `/opt/video/cfr2-media` |

### 🔒 安全建议

- ✅ 使用 `acl = private`
- ✅ 定期轮换密钥
- ✅ 限制 API 权限
- ✅ 使用 IAM 用户
- ✅ 启用访问日志

### 📊 监控建议

```bash
# 状态检查脚本
#!/bin/bash
services=$(systemctl list-units --type=service | grep rclone | awk '{print $1}')
for service in $services; do
    status=$(systemctl is-active "$service")
    echo "$service: $status"
done
```

### 💾 备份配置

```bash
# 备份 rclone 配置
cp ~/.config/rclone/rclone.conf ~/rclone.conf.backup

# 备份服务配置
sudo cp /etc/systemd/system/rclone-*.service ~/services-backup/
```

## 配置示例

### 多云存储配置

```ini
# ~/.config/rclone/rclone.conf

[cfr2-media]
type = s3
provider = Cloudflare
access_key_id = cfr2_access_key
secret_access_key = cfr2_secret_key
region = auto
endpoint = https://account.r2.cloudflarestorage.com
acl = private

[s3do-backup]
type = s3
provider = DigitalOcean
access_key_id = do_access_key
secret_access_key = do_secret_key
endpoint = sfo3.digitaloceanspaces.com
location_constraint = sfo3
acl = private

[aws-archive]
type = s3
provider = AWS
access_key_id = aws_access_key
secret_access_key = aws_secret_key
region = us-east-1
storage_class = GLACIER
```

### 批量部署脚本

```bash
#!/bin/bash
# setup-multi-rclone.sh

configs=("cfr2-media" "s3do-backup" "aws-archive")

for config in "${configs[@]}"; do
    mount_point="/opt/video/$config"
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

    # 启用服务
    sudo systemctl daemon-reload
    sudo systemctl enable "$service_name"
    sudo systemctl start "$service_name"
    
    echo "✅ $service_name configured!"
done

echo "🎉 All rclone services ready!"
```

## 🤝 贡献指南

欢迎提交 Issue 和 Pull Request！

### 贡献类型
- 📝 改进文档
- 🐛 修复错误
- ✨ 添加新的云存储配置
- 🔧 优化脚本和配置

### 提交规范
- 使用清晰的提交信息
- 遵循现有的文档格式
- 测试所有代码示例

## 📄 许可证

MIT License - 详见 [LICENSE](LICENSE) 文件

## 🙏 致谢

- [rclone](https://rclone.org/) - 优秀的云存储同步工具