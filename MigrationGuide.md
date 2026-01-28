# LiteLLM + Open WebUI Docker Compose 迁移指南（Windows Docker Desktop）

## 前提条件
- 源机器和目标机器均安装 **Docker Desktop**（最新版本，支持 WSL2）。
- 源机器上的容器正在运行，确保数据卷完整（postgres_data、open-webui）。
- 备份所有配置文件和数据卷，以防意外。

## 步骤 1: 在源机器备份配置文件和数据
### 1.1 复制配置文件
将以下文件复制到 USB 驱动器、共享文件夹或云存储（如 OneDrive），传输到目标机器：
- `docker-compose.yml`
- `.env`（如果存在，包含敏感信息如 API 密钥）
- `litellmconfig.yaml`（LiteLLM 配置）
- 其他相关文件（如 `.env_example` 用于参考）

**命令（在项目目录下）：**
```
dir  # 确认文件列表
xcopy *.* D:\backup\ /E /Y  # 示例：复制到 D:\backup\
```

### 1.2 备份数据卷
Docker Desktop on Windows 使用 WSL2 存储卷数据。路径通常为 `\\wsl$\docker-desktop-data\data\docker\volumes\`。

#### 列出卷：
```
docker volume ls
```
查找：`litellm_postgres_data`、`open-webui`。

#### 备份每个卷（使用临时容器）：
在项目目录创建 `backup` 文件夹，然后运行以下命令备份：

**Postgres 数据（litellm_postgres_data）：**
```
docker run --rm -v litellm_postgres_data:/data -v %CD%\backup:/backup alpine tar czf /backup/postgres_data.tar.gz -C /data .
```

**Open WebUI 数据：**
```
docker run --rm -v open-webui:/data -v %CD%\backup:/backup alpine tar czf /backup/open-webui.tar.gz -C /data .
```

将 `backup` 文件夹复制到目标机器。

**可选：直接访问 WSL2 卷路径备份（高级用户）**
```
# 打开 WSL：wsl.exe
# 导航：cd /var/lib/docker/volumes/
# tar czf /mnt/d/backup/litellm_postgres_data.tar.gz litellm_postgres_data/
```

## 步骤 2: 在目标机器准备环境
### 2.1 安装 Docker Desktop
- 下载并安装 [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/)。
- 启用 WSL2，确保 Docker 服务运行。

### 2.2 创建项目目录
```
mkdir C:\LiteLLM_OpenWebUI_Docker
cd C:\LiteLLM_OpenWebUI_Docker
```
将备份的配置文件粘贴到此目录。

### 2.3 恢复数据卷
在项目目录下，解压备份：

**创建卷并恢复：**
```
# 先创建空卷（如果不存在）
docker volume create litellm_postgres_data
docker volume create open-webui

# 恢复 Postgres
docker run --rm -v litellm_postgres_data:/data -v %CD%\backup:/backup alpine tar xzf /backup/postgres_data.tar.gz -C /data

# 恢复 Open WebUI
docker run --rm -v open-webui:/data -v %CD%\backup:/backup alpine tar xzf /backup/open-webui.tar.gz -C /data
```

## 步骤 3: 启动服务
### 3.1 检查端口可用性
确保端口未被占用：
- 4000 (LiteLLM)
- 5432 (Postgres)
- 9036 (Open WebUI)

如果冲突，编辑 `docker-compose.yml` 中的 `ports` 修改主机端口。

### 3.2 拉取镜像并启动
```
docker-compose pull  # 拉取最新镜像
docker-compose up -d  # 后台启动
```

### 3.3 验证启动
```
docker-compose ps  # 检查容器状态
docker-compose logs litellm  # 查看 LiteLLM 日志
docker-compose logs db  # 查看 Postgres 日志
```

- 访问：
  - LiteLLM: http://localhost:4000
  - Open WebUI: http://localhost:9036
  - Postgres: localhost:5432 (用工具如 pgAdmin 连接测试)

## 步骤 4: 常见问题排查
- **卷权限问题**：运行 `docker-compose down -v` 删除旧卷后重试。
- **镜像拉取失败**：检查网络，运行 `docker system prune -f` 清理。
- **Postgres 连接失败**：确认 `POSTGRES_PASSWORD` 一致，重启 `db` 服务。
- **LiteLLM 配置错误**：检查 `litellmconfig.yaml` 和 `.env`。
- **Windows 防火墙**：允许 Docker Desktop 通过防火墙。
- **数据不一致**：如果迁移后数据丢失，从备份重新恢复。

## 步骤 5: 清理源机器（可选）
```
docker-compose down -v  # 停止并删除卷（谨慎！）
docker volume rm litellm_postgres_data open-webui
```

**注意**：
- 迁移过程中停止源机器服务：`docker-compose down`。
- 敏感信息（如 API 密钥）在 `.env` 中，确保安全传输。
- 测试完整性：在目标机器运行几天确认稳定。

迁移完成！如果遇到问题，检查 Docker 日志或参考官方文档。