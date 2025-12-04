# LiteLLM + Open WebUI Docker 部署

本项目使用 Docker Compose 部署了一个完整的 LLM 代理服务栈，包括 LiteLLM 代理、PostgreSQL 数据库和 Open WebUI 界面。

## 服务架构

### LiteLLM 代理服务
- **端口**: 4000
- **功能**: 提供 LLM 模型代理服务，支持多种模型管理和 API 调用
- **配置**: 通过外部配置文件 `litellmconfig.yaml` 进行配置
- **特性**:
  - 支持通过 UI 界面添加模型
  - 健康检查机制

### PostgreSQL 数据库
- **端口**: 5432
- **功能**: 存储 LiteLLM 的配置和模型数据
- **数据持久化**: 使用命名卷 `litellm_postgres_data` 保存数据
- **连接信息**:
  - 数据库名: litellm
  - 用户名: llmproxy
  - 密码: dbpassword9090

### Open WebUI 界面
- **端口**: 3000
- **功能**: 提供用户友好的 Web 界面，与 LiteLLM 代理集成
- **自动连接**: 通过 Docker 内部网络自动连接到 LiteLLM (http://litellm:4000/v1)
- **数据持久化**: 使用 `open-webui` 卷保存用户数据
- **依赖关系**: 自动等待 LiteLLM 服务启动后再启动

## 使用方法

### 启动所有服务
```bash
docker compose up -d
```

**注意**：`-d` 参数表示后台运行模式（detach），服务会在后台运行并释放终端。如果不加 `-d`，服务会在前台运行并实时显示日志，按 `Ctrl+C` 可停止服务。

### 查看服务状态
```bash
docker compose ps
```

### 停止所有服务
```bash
docker compose down
```

### 重启所有服务
```bash
docker compose down && docker compose up -d
```

### 查看服务日志
```bash
# 查看所有服务日志
docker compose logs

# 查看特定服务日志
docker compose logs litellm
docker compose logs open-webui
```

## 访问地址

### 本地访问
- **LiteLLM Dashboard**: http://localhost:4000/ui/
- **Open WebUI**: http://localhost:3000/

### 局域网访问

#### 获取服务器 IP 地址

首先获取运行 Docker 服务的电脑的局域网 IP 地址：

**Windows 系统**：
```bash
ipconfig
# 查找 "IPv4 地址"，通常是 192.168.x.x 格式
```

**Linux/macOS 系统**：
```bash
ip addr show
# 或
ifconfig
# 查找 inet 地址，通常是 192.168.x.x 格式
```

#### 局域网访问地址

假设服务器 IP 地址为 `192.168.1.111`，则局域网访问地址为：

- **LiteLLM Dashboard**: http://192.168.1.111:4000/ui/
- **Open WebUI**: http://192.168.1.111:3000/

#### 端口说明

根据 [`docker-compose.yml`](docker-compose.yml) 配置：
- **3000 端口**：Open WebUI 服务（容器内部 8080 端口映射到主机 3000 端口）
- **4000 端口**：LiteLLM 服务

#### 故障排除

如果无法从局域网其他设备访问，请检查以下项目：

1. **防火墙设置**：
   - **Windows**：在 Windows 防火墙中创建**入站规则**，允许端口 3000、4000
     - 打开 Windows 防火墙高级设置
     - 选择"入站规则" > "新建规则"
     - 选择"端口"，点击"下一步"
     - 选择"TCP"，输入特定端口：3000,4000
     - 选择"允许连接"，点击"下一步"
     - 选择适用的网络类型（通常全部选中），点击"下一步"
     - 输入规则名称（如"LiteLLM 服务"），完成创建
   - **Linux**：使用 `ufw allow 3000`、`ufw allow 4000`（默认为入站规则）
   - **macOS**：在系统偏好设置 > 安全性与隐私 > 防火墙中配置**入站连接**

2. **Docker 服务状态**：
   ```bash
   docker compose ps
   # 确保所有服务状态为 "running"
   ```

3. **端口绑定检查**：
   - 确认 [`docker-compose.yml`](docker-compose.yml:12) 中的端口绑定使用 `0.0.0.0`（已正确配置）
   - 这表示服务绑定到所有网络接口，而不仅仅是 localhost

4. **网络连通性测试**：
   ```bash
   # 在服务器上测试
   curl http://192.168.1.111:3000
   
   # 在其他设备上测试
   ping 192.168.1.111
   ```

5. **路由器设置**：
   - 确保局域网设备之间可以互相访问
   - 检查是否有 AP 隔离或访客网络限制

#### 安全建议

1. **修改默认密码**：首次访问 Open WebUI 时，请立即修改默认管理员密码
2. **网络隔离**：如果可能，将服务部署在独立的网络段中
3. **VPN 访问**：如需从外网访问，建议使用 VPN 而非直接暴露端口到公网
4. **定期更新**：保持 Docker 镜像和系统更新

## 配置说明

### 在安装了 ShellCrash 的云虚拟机上配置代理

如果你的云虚拟机安装了 [ShellCrash](https://github.com/juewuy/ShellCrash) 用于翻墙，需要在 `docker-compose.yml` 中为 LiteLLM 服务配置代理，使其能够访问外部 API。

#### 配置步骤

1. **获取服务器 IP 地址**：
   ```bash
   hostname -I | awk '{print $1}'
   ```
   假设返回的 IP 地址为 `172.31.219.189`

2. **检查 ShellCrash 配置**：
   ```bash
   cat /tmp/ShellCrash/config.yaml | grep "mixed-port"
   ```
   确认混合端口，通常为是自己手动设置的比如 8964

3. **修改 docker-compose.yml**：

   在 `litellm` 服务的 `environment` 部分添加以下三行：

   ```yaml
   services:
     litellm:
       # ... 其他配置 ...
       environment:
         DATABASE_URL: "postgresql://llmproxy:dbpassword9090@db:5432/litellm"
         STORE_MODEL_IN_DB: "True"
         LITELLM_ENABLE_PROMETHEUS: "True"
         HTTP_PROXY: "http://172.31.219.189:8964"  # 使用ShellCrash代理
         HTTPS_PROXY: "http://172.31.219.189:8964"  # 使用ShellCrash代理
         NO_PROXY: "localhost,127.0.0.1,db,open-webui"  # 不代理本地服务
         # ... 其他环境变量 ...
   ```

4. **重启服务**：
   ```bash
   docker compose down && docker compose up -d
   ```

#### 注意事项

- 将 `172.31.219.189` 替换为你的实际服务器 IP
- 将 `8964` 替换为你的 ShellCrash 混合端口
- `NO_PROXY` 设置确保本地 Docker 网络通信不通过代理
- 修改配置后必须重启服务才能生效

### 环境变量
- 通过 `.env` 文件加载环境变量
- 重要配置项包括 API 密钥、数据库连接信息等

### 配置文件
- **LiteLLM 配置**: `litellmconfig.yaml`

### 数据卷
- `postgres_data`: PostgreSQL 数据持久化
- `open-webui`: Open WebUI 用户数据持久化

## Docker Image 版本管理

### 查看最新版本

LiteLLM 的 Docker 镜像托管在 GitHub Packages，可以在以下网址查看最新版本：
https://github.com/berriai/litellm/pkgs/container/litellm

### 更新 Docker Image 版本

当前使用的版本：`ghcr.io/berriai/litellm-database:main-v1.80.0-nightly`

如需更新到最新版本，请按照以下步骤操作：

1. **访问 GitHub Packages 页面**，查看最新可用的版本标签
2. **编辑 [`docker-compose.yml`](docker-compose.yml:3) 文件**，修改 `litellm` 服务的 `image` 字段：

```yaml
services:
  litellm:
    image: ghcr.io/berriai/litellm:main-v1.80.0.dev2  # 替换为最新版本
    # ... 其他配置保持不变
```

3. **拉取新镜像并重启服务**：

```bash
# 拉取最新的镜像
docker compose pull litellm

# 重启服务以使用新版本
docker compose up -d

# 验证版本是否更新成功
docker compose ps
```

### 版本标签说明

- `main-v1.80.0-nightly`:  nightly 版本，包含最新功能但可能不够稳定
- `main-v1.80.0.dev2`: 开发版本，包含特定版本的更新
- `main-stable`: 稳定版本，适合生产环境使用

**建议**：生产环境使用稳定版本（`main-stable`），开发测试环境可以使用最新版本获取新功能。

## 配置文件管理

### 配置文件位置说明

本项目默认将 [`litellmconfig.yaml`](litellmconfig.yaml:1) 配置文件放在**项目目录**中，这是推荐的做法。

#### 默认配置（推荐）
```yaml
volumes:
  # 挂载项目文件夹中的配置文件（推荐）
  - ./litellmconfig.yaml:/app/config.yaml
```

### 使用项目目录中的配置文件（默认方式）

1. 确保 [`litellmconfig.yaml`](litellmconfig.yaml:1) 文件存在于项目根目录
2. 使用默认的 docker-compose.yml 配置（无需修改）
3. 启动服务：
   ```bash
   docker compose up -d
   ```

### 使用电脑其他位置的配置文件

如果你希望将配置文件放在电脑的其他位置（如文档文件夹），请按照以下步骤操作：

#### 1. 确定配置文件路径

**Windows 系统示例**：
```yaml
volumes:
  # 绝对路径示例（C盘用户文档文件夹）
  - /c/Users/你的用户名/Documents/litellmconfig.yaml:/app/config.yaml
  
  # 或者使用 Windows 路径格式（需要额外引号）
  - "C:\Users\你的用户名\Documents\litellmconfig.yaml:/app/config.yaml"
```

**Linux/macOS 系统示例**：
```yaml
volumes:
  # 绝对路径示例
  - /home/你的用户名/documents/litellmconfig.yaml:/app/config.yaml
  
  # 或者使用 ~ 表示用户主目录
  - ~/documents/litellmconfig.yaml:/app/config.yaml
```

#### 2. 修改 docker-compose.yml

编辑 [`docker-compose.yml`](docker-compose.yml:21) 文件，找到 `litellm` 服务的 `volumes` 配置段，修改左侧路径为你的配置文件实际路径：

```yaml
services:
  litellm:
    # ... 其他配置 ...
    volumes:
      # 将左侧路径修改为你的配置文件实际路径
      - /你的/配置文件/完整路径/litellmconfig.yaml:/app/config.yaml
    # ... 其他配置 ...
```

**重要提示**：
- 右侧的 `/app/config.yaml` 必须保持不变
- 左侧路径必须是你的配置文件在宿主机上的完整路径
- Windows 用户注意使用正斜杠 `/` 或双反斜杠 `\\`

#### 3. 重启服务

修改配置后，需要重启服务才能生效：

```bash
# 停止当前运行的服务
docker compose down

# 重新启动服务
docker compose up -d

# 验证配置是否生效
docker compose logs litellm
```

### 配置文件优先级

1. **外部配置文件**（通过 volumes 挂载）优先级最高
2. **环境变量** 次之
3. **默认配置** 优先级最低

### 配置文件结构说明

[`litellmconfig.yaml`](litellmconfig.yaml:1) 文件包含以下主要部分：

- **general_settings**: 通用设置，如 master_key、数据库连接等
- **litellm_settings**: LiteLLM 特定设置，如重试次数、超时时间等
- **model_list**: 模型列表，定义可用的 AI 模型及其配置

### 快速开始建议

**对于新用户**，强烈推荐使用默认方式：
1. 将 [`litellmconfig.yaml`](litellmconfig.yaml:1) 文件放在项目目录
2. 根据需求修改配置文件中的 API 密钥和模型设置
3. 直接使用 `docker compose up -d` 启动服务

**对于高级用户**，如果需要多个项目共享配置文件，可以将配置文件放在统一的位置，然后按照上述步骤修改路径。

## 注意事项

1. 首次启动前，请确保已正确配置 `.env` 文件中的 API 密钥
2. Windows 用户需要确认 `litellmconfig.yaml` 文件路径是否正确
3. 服务启动顺序已通过 `depends_on` 配置确保依赖关系
4. 所有服务都配置了健康检查，确保服务可用性
5. 修改配置文件后需要重启服务才能生效