# LightRAG Docker 部署

一个轻量级知识图谱检索增强生成系统，支持多种 LLM 后端。

## 🚀 准备工作

### 克隆仓库：

```bash
# Linux/MacOS
git clone https://github.com/HKUDS/LightRAG.git
cd LightRAG
```
```powershell
# Windows PowerShell
git clone https://github.com/HKUDS/LightRAG.git
cd LightRAG
```

### 配置环境：

```bash
# Linux/MacOS
cp .env.example .env
# 编辑 .env 文件，配置您的偏好设置
```
```powershell
# Windows PowerShell
Copy-Item .env.example .env
# 编辑 .env 文件，配置您的偏好设置
```

LightRAG 可以使用 `.env` 文件中的环境变量进行配置：

**服务器配置**

- `HOST`: 服务器主机（默认：0.0.0.0）
- `PORT`: 服务器端口（默认：9621）

**LLM 配置**

- `LLM_BINDING`: 使用的 LLM 后端（lollms/ollama/openai）
- `LLM_BINDING_HOST`: LLM 服务器主机 URL
- `LLM_MODEL`: 使用的模型名称

**嵌入配置**

- `EMBEDDING_BINDING`: 嵌入后端（lollms/ollama/openai）
- `EMBEDDING_BINDING_HOST`: 嵌入服务器主机 URL
- `EMBEDDING_MODEL`: 嵌入模型名称

**RAG 配置**

- `MAX_ASYNC`: 最大异步操作数
- `MAX_TOKENS`: 最大 token 大小
- `EMBEDDING_DIM`: 嵌入维度

## 🐳 Docker 部署

Docker 说明在所有安装了 Docker Desktop 的平台上工作方式相同。

### 构建优化

Dockerfile 使用 BuildKit 缓存挂载来显著提高构建性能：

- **自动缓存管理**: 通过 `# syntax=docker/dockerfile:1` 指令自动启用 BuildKit
- **更快的重建**: 仅在修改 `uv.lock` 或 `bun.lock` 文件时下载更改的依赖项
- **高效的包缓存**: UV 和 Bun 包下载在构建之间被缓存
- **无需手动配置**: 在 Docker Compose 和 GitHub Actions 中开箱即用

### 启动 LightRAG 服务器：

```bash
docker compose up -d
```

LightRAG 服务器使用以下路径进行数据存储：

```
data/
├── rag_storage/    # RAG 数据持久化
└── inputs/         # 输入文档
```

### 更新

要更新 Docker 容器：
```bash
docker compose pull
docker compose down
docker compose up
```

### 离线部署

需要 `transformers`、`torch` 或 `cuda` 的软件包不会在 Docker 镜像中预安装。因此，Docling 等文档提取工具以及 Hugging Face 和 LMDeploy 等本地 LLM 模型无法在离线环境中使用。这些高计算资源需求的服务不应集成到 LightRAG 中。Docling 将被解耦并作为独立服务部署。

## 📦 构建 Docker 镜像

### 用于本地开发和测试

```bash
# 使用 Docker Compose 构建和运行（自动启用 BuildKit）
docker compose up --build

# 或者，如果需要，显式启用 BuildKit
DOCKER_BUILDKIT=1 docker compose up --build
```

**注意**: BuildKit 通过 Dockerfile 中的 `# syntax=docker/dockerfile:1` 指令自动启用，确保最佳缓存性能。

### 用于生产发布

 **多架构构建和推送**：

```bash
# 使用提供的构建脚本
./docker-build-push.sh
```

**构建脚本将**：

- 检查 Docker 注册表登录状态
- 自动创建/使用 buildx 构建器
- 为 AMD64 和 ARM64 架构构建
- 推送到 GitHub Container Registry (ghcr.io)
- 验证多架构清单

**前置条件**：

在构建多架构镜像之前，请确保您拥有：

- Docker 20.10+ 支持 Buildx
- 足够的磁盘空间（离线镜像建议 20GB+）
- 注册表访问凭据（如果推送镜像）
