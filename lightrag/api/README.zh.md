# LightRAG 服务器和 WebUI

LightRAG 服务器旨在提供 Web UI 和 API 支持。WebUI 便于文档索引、知识图谱探索和简单的 RAG 查询界面。LightRAG 服务器还提供了与 Ollama 兼容的接口，旨在将 LightRAG 模拟为 Ollama 聊天模型。这使得 AI 聊天机器人（如 Open WebUI）能够轻松访问 LightRAG。

![image-20250323122538997](./README.assets/image-20250323122538997.png)

![image-20250323122754387](./README.assets/image-20250323122754387.png)

![image-20250323123011220](./README.assets/image-20250323123011220.png)

## 快速开始

### 安装

* 从 PyPI 安装

```bash
### 使用 uv 安装 LightRAG Server 工具（推荐）
uv tool install "lightrag-hku[api]"

### 或使用 pip
# python -m venv .venv
# source .venv/bin/activate  # Windows: .venv\Scripts\activate
# pip install "lightrag-hku[api]"
```

* 从源代码安装

```bash
# 克隆仓库
git clone https://github.com/HKUDS/lightrag.git

# 进入仓库目录
cd lightrag

# 使用 uv（推荐）
# 注意：uv sync 会自动在 .venv/ 中创建虚拟环境
uv sync --extra api
source .venv/bin/activate  # 激活虚拟环境 (Linux/macOS)
# 或在 Windows 上: .venv\Scripts\activate

# 或使用 pip 配合虚拟环境
# python -m venv .venv
# source .venv/bin/activate  # Windows: .venv\Scripts\activate
# pip install -e ".[api]"

# 构建前端资源
cd lightrag_webui
bun install --frozen-lockfile
bun run build
cd ..
```

### 启动 LightRAG 服务器前的准备

LightRAG 需要集成 LLM（大语言模型）和 Embedding 模型才能有效地执行文档索引和查询操作。在首次部署 LightRAG 服务器之前，必须配置 LLM 和 Embedding 模型的设置。LightRAG 支持绑定各种 LLM/Embedding 后端：

* ollama
* lollms
* openai 或 openai 兼容
* azure_openai
* aws_bedrock
* gemini

建议使用环境变量来配置 LightRAG 服务器。项目根目录中有一个名为 `env.example` 的示例环境变量文件。请将此文件复制到启动目录并重命名为 `.env`。然后，您可以修改 `.env` 文件中与 LLM 和 Embedding 模型相关的参数。需要注意的是，LightRAG 服务器每次启动时都会将 `.env` 中的环境变量加载到系统环境变量中。**LightRAG 服务器会优先考虑系统环境变量中的设置，而非 .env 文件**。

> 由于带有 Python 扩展的 VS Code 可能会在集成终端中自动加载 .env 文件，因此每次修改 .env 文件后请打开一个新的终端会话。

以下是一些 LLM 和 Embedding 模型的常见配置示例：

* OpenAI LLM + Ollama Embedding：

```
LLM_BINDING=openai
LLM_MODEL=gpt-4o
LLM_BINDING_HOST=https://api.openai.com/v1
LLM_BINDING_API_KEY=your_api_key

EMBEDDING_BINDING=ollama
EMBEDDING_BINDING_HOST=http://localhost:11434
EMBEDDING_MODEL=bge-m3:latest
EMBEDDING_DIM=1024
# EMBEDDING_BINDING_API_KEY=your_api_key
```

> 使用 Google Gemini 时，设置 `LLM_BINDING=gemini`，选择一个模型如 `LLM_MODEL=gemini-flash-latest`，并通过 `LLM_BINDING_API_KEY`（或 `GEMINI_API_KEY`）提供您的 Gemini 密钥。

* Ollama LLM + Ollama Embedding：

```
LLM_BINDING=ollama
LLM_MODEL=mistral-nemo:latest
LLM_BINDING_HOST=http://localhost:11434
# LLM_BINDING_API_KEY=your_api_key
###  Ollama 服务器上下文长度（必须大于 MAX_TOTAL_TOKENS+2000）
OLLAMA_LLM_NUM_CTX=16384

EMBEDDING_BINDING=ollama
EMBEDDING_BINDING_HOST=http://localhost:11434
EMBEDDING_MODEL=bge-m3:latest
EMBEDDING_DIM=1024
# EMBEDDING_BINDING_API_KEY=your_api_key
```

> **重要提示**：Embedding 模型必须在文档索引之前确定，并且在文档查询阶段必须使用相同的模型。对于某些存储解决方案（如 PostgreSQL），向量维度必须在初始创建表时定义。因此，当更改 embedding 模型时，需要删除现有的向量相关表，并允许 LightRAG 使用新的维度重新创建它们。

### 启动 LightRAG 服务器

LightRAG 服务器支持两种运行模式：
* 简单高效的 Uvicorn 模式：

```
lightrag-server
```
* 多进程 Gunicorn + Uvicorn 模式（生产模式，Windows 环境不支持）：

```
lightrag-gunicorn --workers 4
```

启动 LightRAG 时，当前工作目录必须包含 `.env` 配置文件。**故意将 `.env` 文件设计为必须放在启动目录中**。这样做的目的是允许用户同时启动多个 LightRAG 实例，并为不同的实例配置不同的 `.env` 文件。**修改 `.env` 文件后，需要重新打开终端才能使新设置生效**。这是因为每次 LightRAG 服务器启动时，都会将 `.env` 文件中的环境变量加载到系统环境变量中，而系统环境变量的优先级更高。

在启动过程中，`.env` 文件中的配置可以被命令行参数覆盖。常见的命令行参数包括：

- `--host`：服务器监听地址（默认：0.0.0.0）
- `--port`：服务器监听端口（默认：9621）
- `--timeout`：LLM 请求超时时间（默认：150 秒）
- `--log-level`：日志级别（默认：INFO）
- `--working-dir`：数据库持久化目录（默认：./rag_storage）
- `--input-dir`：上传文件目录（默认：./inputs）
- `--workspace`：工作区名称，用于在多个 LightRAG 实例之间逻辑隔离数据（默认：空）

### 使用 Docker 启动 LightRAG 服务器

使用 Docker Compose 是部署和运行 LightRAG 服务器最方便的方式。

- 创建一个项目目录。
- 从 LightRAG 仓库中将 `docker-compose.yml` 文件复制到您的项目目录。
- 准备 `.env` 文件：复制示例文件 [`env.example`](https://ai.znipower.com:5013/c/env.example) 创建自定义的 `.env` 文件，并根据您的具体要求配置 LLM 和 embedding 参数。
- 使用以下命令启动 LightRAG 服务器：

```shell
docker compose up
# 如果您希望程序在启动后在后台运行，请在命令末尾添加 -d 参数。
```

您可以从这里获取官方的 docker compose 文件：[docker-compose.yml](https://raw.githubusercontent.com/HKUDS/LightRAG/refs/heads/main/docker-compose.yml)。如需获取历史版本的 LightRAG docker 镜像，请访问此链接：[LightRAG Docker Images](https://github.com/HKUDS/LightRAG/pkgs/container/lightrag)。有关 docker 部署的更多详细信息，请参阅 [DockerDeployment.md](./../../docs/DockerDeployment.md)。

### Nginx 反向代理配置

在 LightRAG 服务器前使用 Nginx 作为反向代理时，需要为 `/documents/upload` 端点配置 `client_max_body_size` 以处理大文件上传。如果没有此配置，Nginx 将拒绝大于 1MB（默认限制）的文件，并返回 `413 Request Entity Too Large` 错误，而请求无法到达 LightRAG。

**推荐配置：**

```nginx
server {
    listen 80;
    server_name your-domain.com;

    # 全局默认：8MB，用于具有长上下文的 LLM 查询
    client_max_body_size 8M;

    # 上传端点：100MB，用于大文件上传
    location /documents/upload {
        client_max_body_size 100M;

        proxy_pass http://localhost:9621;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 增加大文件上传的超时时间
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }

    # 流式端点：LLM 响应流式传输
    location ~ ^/(query/stream|api/chat|api/generate) {
        gzip off;  # 对流式响应禁用压缩

        proxy_pass http://localhost:9621;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # LLM 生成的长超时时间
        proxy_read_timeout 300s;
    }

    # 其他端点
    location / {
        proxy_pass http://localhost:9621;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**关键点：**

1. **全局限制（8MB）**：足以处理具有长对话历史和上下文的 LLM 查询（128K tokens ≈ 512KB + JSON 开销）。
2. **上传端点（100MB）**：必须与 `.env` 文件中的 `MAX_UPLOAD_SIZE` 匹配或超过它。默认的 `MAX_UPLOAD_SIZE` 是 100MB。
3. **流式端点**：对流式端点禁用 gzip 压缩（`gzip off`）以确保实时响应传递。LightRAG 自动设置 `X-Accel-Buffering: no` 头来禁用响应缓冲。
4. **超时设置**：大文件上传和 LLM 生成需要更长的超时时间；相应地调整 `proxy_read_timeout` 和 `proxy_send_timeout`。
5. **大小验证层**：
   - Nginx 首先验证 `Content-Length` 头
   - LightRAG 在上传期间执行流式验证
   - 在两层设置适当的限制可以确保更好的错误消息和安全性

### 离线部署

官方 LightRAG Docker 镜像完全支持离线或隔离环境。如果您想构建自己的离线环境，请参阅[离线部署指南](./../../docs/OfflineDeployment.md)。

### 启动多个 LightRAG 实例

有两种方法可以启动多个 LightRAG 实例。第一种方法是为每个实例配置一个完全独立的工作环境。这需要为每个实例创建一个单独的工作目录，并在该目录中放置一个专用的 `.env` 配置文件。不同实例配置文件中的服务器监听端口不能相同。然后，您可以通过在工作目录中运行 `lightrag-server` 来启动服务。

第二种方法是所有实例共享同一组 `.env` 配置文件，然后使用命令行参数为每个实例指定不同的服务器监听端口和工作区。您可以使用不同的命令行参数在同一工作目录中启动多个 LightRAG 实例。例如：

```
# 启动实例 1
lightrag-server --port 9621 --workspace space1

# 启动实例 2
lightrag-server --port 9622 --workspace space2
```

工作区的目的是实现不同实例之间的数据隔离。因此，不同实例的 `workspace` 参数必须不同；否则，会导致数据混乱和损坏。

通过 Docker Compose 启动多个 LightRAG 实例时，只需在 `docker-compose.yml` 中为每个容器指定唯一的 `WORKSPACE` 和 `PORT` 环境变量。即使所有实例共享一个公共的 `.env` 文件，Compose 中定义的容器特定环境变量也将优先，确保每个实例的独立配置。

### LightRAG 实例之间的数据隔离

为每个实例配置一个独立的工作目录和一个专用的 `.env` 配置文件，通常可以确保内存数据库中的本地持久化文件保存在各自的工作目录中，从而实现数据隔离。默认情况下，LightRAG 使用所有内存数据库，这种数据隔离方法就足够了。但是，如果您使用的是外部数据库，并且不同实例访问同一个数据库实例，则需要使用工作区来实现数据隔离；否则，不同实例的数据会发生冲突并被破坏。

命令行 `workspace` 参数和 `.env` 文件中的 `WORKSPACE` 环境变量都可以用来指定当前实例的工作区名称，命令行参数的优先级更高。以下是不同类型存储的工作区实现方式：

- **对于基于本地文件的数据库，通过工作区子目录实现数据隔离**：`JsonKVStorage`、`JsonDocStatusStorage`、`NetworkXStorage`、`NanoVectorDBStorage`、`FaissVectorDBStorage`。
- **对于以集合形式存储数据的数据库，通过在集合名称前添加工作区前缀来实现**：`RedisKVStorage`、`RedisDocStatusStorage`、`MilvusVectorDBStorage`、`MongoKVStorage`、`MongoDocStatusStorage`、`MongoVectorDBStorage`、`MongoGraphStorage`、`PGGraphStorage`。
- **对于 Qdrant 向量数据库，通过基于负载的分区实现数据隔离（Qdrant 推荐的多租户方法）**：`QdrantVectorDBStorage` 使用共享集合和负载过滤，实现无限的工作区可扩展性。
- **对于关系数据库，通过在表中添加 `workspace` 字段来实现逻辑数据隔离**：`PGKVStorage`、`PGVectorStorage`、`PGDocStatusStorage`。
- **对于图数据库，通过标签实现逻辑数据隔离**：`Neo4JStorage`、`MemgraphStorage`

为了保持与旧数据的兼容性，当未配置工作区时，PostgreSQL 的默认工作区是 `default`，Neo4j 的默认工作区是 `base`。对于所有外部存储，系统提供了专用的工作区环境变量来覆盖通用的 `WORKSPACE` 环境变量配置。这些存储特定的工件区环境变量是：`REDIS_WORKSPACE`、`MILVUS_WORKSPACE`、`QDRANT_WORKSPACE`、`MONGODB_WORKSPACE`、`POSTGRES_WORKSPACE`、`NEO4J_WORKSPACE`、`MEMGRAPH_WORKSPACE`。

### Gunicorn + Uvicorn 的多工作进程

LightRAG 服务器可以在 `Gunicorn + Uvicorn` 预加载模式下运行。Gunicorn 的多工作进程（多进程）功能可以防止文档索引任务阻塞 RAG 查询。在纯 Uvicorn 模式下，使用 CPU 密集型文档提取工具（如 docling）可能会导致整个系统被阻塞。

虽然 LightRAG 服务器使用一个工作进程来处理文档索引流程，但借助 Uvicorn 的异步任务支持，可以并行处理多个文件。文档索引速度的瓶颈主要在于 LLM。如果您的 LLM 支持高并发性，可以通过提高 LLM 的并发级别来加速文档索引。以下是几个与并发处理相关的环境变量及其默认值：

```
### 工作进程数，不超过 (2 x 核心数) + 1
WORKERS=2
### 一批中并行处理的文件数
MAX_PARALLEL_INSERT=2
### LLM 的最大并发请求数
MAX_ASYNC=4
```

### 将 LightRAG 安装为 Linux 服务

从示例文件 `lightrag.service.example` 创建您的服务文件 `lightrag.service`。修改服务文件中的启动选项：

```text
# 将环境设置为您的 Python 虚拟环境
Environment="PATH=/home/netman/lightrag-xyj/venv/bin"
WorkingDirectory=/home/netman/lightrag-xyj
# ExecStart=/home/netman/lightrag-xyj/venv/bin/lightrag-server
ExecStart=/home/netman/lightrag-xyj/venv/bin/lightrag-gunicorn
```

> ExecStart 命令必须是 `lightrag-gunicorn` 或 `lightrag-server`；不允许使用包装脚本。这是因为服务终止需要主进程是这两个可执行文件之一。

安装 LightRAG 服务。如果您的系统是 Ubuntu，以下命令将起作用：

```shell
sudo cp lightrag.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl start lightrag.service
sudo systemctl status lightrag.service
sudo systemctl enable lightrag.service
```

## Ollama 模拟

我们为 LightRAG 提供了与 Ollama 兼容的接口，旨在将 LightRAG 模拟为 Ollama 聊天模型。这使得支持 Ollama 的 AI 聊天前端（如 Open WebUI）能够轻松访问 LightRAG。

### 将 Open WebUI 连接到 LightRAG

启动 lightrag-server 后，您可以在 Open WebUI 管理面板中添加 Ollama 类型的连接。然后，名为 `lightrag:latest` 的模型将出现在 Open WebUI 的模型管理界面中。用户可以通过聊天界面向 LightRAG 发送查询。对于此用例，您应该将 LightRAG 安装为服务。

Open WebUI 使用 LLM 来生成会话标题和会话关键字。因此，Ollama 聊天完成 API 会检测并转发 OpenWebUI 会话相关请求直接到底层 LLM。来自 Open WebUI 的屏幕截图：

![image-20250323194750379](./README.assets/image-20250323194750379.png)

### 在聊天中选择查询模式

如果您从 LightRAG 的 Ollama 界面发送消息（查询），默认的查询模式是 `hybrid`。您可以通过发送带有查询前缀的消息来选择查询模式。

查询字符串中的查询前缀可以决定使用哪种 LightRAG 查询模式来生成查询的响应。支持的前缀包括：

```
/local
/global
/hybrid
/naive
/mix

/bypass
/context
/localcontext
/globalcontext
/hybridcontext
/naivecontext
/mixcontext
```

例如，聊天消息 `/mix What's LightRAG?` 将触发 LightRAG 的混合模式查询。没有查询前缀的聊天消息将默认触发混合模式查询。

`/bypass` 不是 LightRAG 查询模式；它将告诉 API 服务器将查询直接传递给底层 LLM，包括聊天历史记录。因此，用户可以使用 LLM 根据聊天历史记录来回答问题。如果您使用 Open WebUI 作为前端，您可以直接将模型切换到普通的 LLM，而不是使用 `/bypass` 前缀。

`/context` 也不是 LightRAG 查询模式；它将告诉 LightRAG 仅返回为 LLM 准备的上下文信息。您可以检查上下文是否符合您的要求，或者自己处理上下文。

### 在聊天中添加用户提示

使用 LightRAG 进行内容查询时，避免将搜索过程与无关的输出处理结合在一起，因为这会显著影响查询效果。用户提示专门用于解决这个问题——它不参与 RAG 检索阶段，而是在查询完成后指导 LLM 如何处理检索结果。我们可以在查询前缀后面附加方括号来为 LLM 提供用户提示：

```
/[Use mermaid format for diagrams] Please draw a character relationship diagram for Scrooge
/mix[Use mermaid format for diagrams] Please draw a character relationship diagram for Scrooge
```

## API 密钥和身份验证

默认情况下，LightRAG 服务器可以在没有任何身份验证的情况下访问。我们可以使用 API 密钥或帐户凭据来配置服务器以确保其安全。

* API 密钥：

```
LIGHTRAG_API_KEY=your-secure-api-key-here
WHITELIST_PATHS=/health,/api/*
```

> 健康检查和 Ollama 模拟端点默认情况下不受 API 密钥检查的限制。出于安全原因，如果不需要 Ollama 服务，请从 `WHITELIST_PATHS` 中删除 `/api/*`。

API 密钥通过请求头 `X-API-Key` 传递。以下是通过 API 访问 LightRAG 服务器的示例：

```
curl -X 'POST' \
  'http://localhost:9621/documents/scan' \
  -H 'accept: application/json' \
  -H 'X-API-Key: your-secure-api-key-here-123' \
  -d ''
```

* 帐户凭据（Web UI 需要登录才能访问）：

LightRAG API 服务器使用 HS256 算法实现基于 JWT 的身份验证。要启用安全访问控制，需要以下环境变量：

```bash
# 用于 jwt 身份验证
AUTH_ACCOUNTS='admin:admin123,user1:pass456'
TOKEN_SECRET='your-key'
TOKEN_EXPIRE_HOURS=4
```

> 目前，仅支持配置管理员帐户和密码。全面的帐户系统尚待开发和实施。

如果未配置帐户凭据，Web UI 将以访客身份访问系统。因此，即使仅配置了 API 密钥，所有 API 仍可以通过访客帐户访问，这仍然不安全。因此，为了保护 API，需要同时配置两种身份验证方法。

## 对于 Azure OpenAI 后端

可以使用 Azure CLI 中的以下命令创建 Azure OpenAI API（您需要先从 [https://docs.microsoft.com/en-us/cli/azure/install-azure-cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) 安装 Azure CLI）：

```bash
# 根据需要更改资源组名称、位置和 OpenAI 资源名称
RESOURCE_GROUP_NAME=LightRAG
LOCATION=swedencentral
RESOURCE_NAME=LightRAG-OpenAI

az login
az group create --name $RESOURCE_GROUP_NAME --location $LOCATION
az cognitiveservices account create --name $RESOURCE_NAME --resource-group $RESOURCE_GROUP_NAME  --kind OpenAI --sku S0 --location swedencentral
az cognitiveservices account deployment create --resource-group $RESOURCE_GROUP_NAME  --model-format OpenAI --name $RESOURCE_NAME --deployment-name gpt-4o --model-name gpt-4o --model-version "2024-08-06"  --sku-capacity 100 --sku-name "Standard"
az cognitiveservices account deployment create --resource-group $RESOURCE_GROUP_NAME  --model-format OpenAI --name $RESOURCE_NAME --deployment-name text-embedding-3-large --model-name text-embedding-3-large --model-version "1"  --sku-capacity 80 --sku-name "Standard"
az cognitiveservices account show --name $RESOURCE_NAME --resource-group $RESOURCE_GROUP_NAME --query "properties.endpoint"
az cognitiveservices account keys list --name $RESOURCE_NAME -g $RESOURCE_GROUP_NAME

```

最后一个命令的输出将为您提供 OpenAI API 的端点和密钥。您可以使用这些值来设置 `.env` 文件中的环境变量。

```
# .env 中的 Azure OpenAI 配置：
LLM_BINDING=azure_openai
LLM_BINDING_HOST=your-azure-endpoint
LLM_MODEL=your-model-deployment-name
LLM_BINDING_API_KEY=your-azure-api-key
### API 版本是可选的，默认为最新版本
AZURE_OPENAI_API_VERSION=2024-08-01-preview

### 如果使用 Azure OpenAI 进行 embedding
EMBEDDING_BINDING=azure_openai
EMBEDDING_MODEL=your-embedding-deployment-name
```

## LightRAG 服务器配置详解

API 服务器可以通过三种方式进行配置（优先级从高到低）：

* 命令行参数
* 环境变量或 .env 文件
* Config.ini（仅用于存储配置）

大多数配置都有默认设置；请查看示例文件 `.env.example` 中的详细信息。数据存储配置也可以通过 config.ini 设置。为了方便起见，提供了示例文件 `config.ini.example`。

### 支持的 LLM 和 Embedding 后端

LightRAG 支持绑定各种 LLM/Embedding 后端：

* ollama
* openai（包括 openai 兼容）
* azure_openai
* lollms
* aws_bedrock

使用环境变量 `LLM_BINDING` 或 CLI 参数 `--llm-binding` 来选择 LLM 后端类型。使用环境变量 `EMBEDDING_BINDING` 或 CLI 参数 `--embedding-binding` 来选择 Embedding 后端类型。

有关 LLM 和 embedding 配置示例，请参阅项目根目录中的 `env.example` 文件。要查看 OpenAI 和 Ollama 兼容 LLM 接口的完整可配置选项列表，请使用以下命令：
```
lightrag-server --llm-binding openai --help
lightrag-server --llm-binding ollama --help
lightrag-server --embedding-binding ollama --help
```

> 请使用 OpenAI 兼容的方法来访问由 OpenRouter 或 vLLM/SGLang 部署的 LLM。您可以通过 `OPENAI_LLM_EXTRA_BODY` 环境变量向 OpenRouter 或 vLLM/SGLang 传递额外参数，以禁用推理模式或实现其他个性化控制。

设置 max_tokens 以**防止大语言模型 (LLM) 响应在实体关系提取阶段产生过长或无休止的输出循环**。设置 max_tokens 参数的目的是在超时发生之前截断 LLM 输出，从而防止文档提取失败。这解决了某些文本块（例如表格或引用）包含大量实体和关系可能导致 LLM 输出过长甚至无休止循环的问题。对于本地部署的小参数模型，此设置尤其重要。Max tokens 值可以通过以下公式计算：`LLM_TIMEOUT * llm_output_tokens/second`（即 `180s * 50 tokens/s = 9000`）

```
# 对于 vLLM/SGLang 部署的模型，或大多数 OpenAI 兼容的 API 提供商
OPENAI_LLM_MAX_TOKENS=9000

# 对于 Ollama 部署的模型
OLLAMA_LLM_NUM_PREDICT=9000

# 对于 OpenAI o1-mini 或更新的模型
OPENAI_LLM_MAX_COMPLETION_TOKENS=9000
```

### 实体提取配置

* ENABLE_LLM_CACHE_FOR_EXTRACT：为实体提取启用 LLM 缓存（默认：true）

对于测试环境，将 `ENABLE_LLM_CACHE_FOR_EXTRACT` 设置为 true 以减少 LLM 调用成本是非常常见的做法。

### 支持的存储类型

LightRAG 使用 4 种类型的存储用于不同的目的：

* KV_STORAGE：llm 响应缓存、文本块、文档信息
* VECTOR_STORAGE：实体向量、关系向量、块向量
* GRAPH_STORAGE：实体关系图
* DOC_STATUS_STORAGE：文档索引状态

LightRAG 服务器提供各种存储实现，默认情况下是内存数据库，将数据持久化到 WORKING_DIR 目录。此外，LightRAG 支持广泛的存储解决方案，包括 PostgreSQL、MongoDB、FAISS、Milvus、Qdrant、Neo4j、Memgraph 和 Redis。有关支持的存储选项的详细信息，请参阅根目录中 README.md 文件的存储部分。

您可以通过配置环境变量来选择存储实现。例如，在 API 服务器首次启动之前，您可以设置以下环境变量来指定所需的存储实现：

```
LIGHTRAG_KV_STORAGE=PGKVStorage
LIGHTRAG_VECTOR_STORAGE=PGVectorStorage
LIGHTRAG_GRAPH_STORAGE=PGGraphStorage
LIGHTRAG_DOC_STATUS_STORAGE=PGDocStatusStorage
```

在向 LightRAG 添加文档后，您不能更改存储实现选择。尚不支持从一个存储实现到另一个存储的数据迁移。有关更多信息，请阅读示例 env 文件或 config.ini 文件。

### 存储类型之间的 LLM 缓存迁移

在 LightRAG 中切换存储实现时，可以将 LLM 缓存从现有存储迁移到新存储。随后，当将文件重新上传到新存储时，预先存在的 LLM 缓存将显著加快文件处理速度。有关使用 LLM 缓存迁移工具的详细说明，请参阅 [README_MIGRATE_LLM_CACHE.md](../tools/README_MIGRATE_LLM_CACHE.md)

### LightRAG API 服务器命令行选项

| 参数             | 默认值       | 描述                                                                                                                     |
| --------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| --host                | 0.0.0.0       | 服务器主机                                                                                                                     |
| --port                | 9621          | 服务器端口                                                                                                                     |
| --working-dir         | ./rag_storage | RAG 存储的工作目录                                                                                               |
| --input-dir           | ./inputs      | 包含输入文档的目录                                                                                             |
| --max-async           | 4             | 最大异步操作数                                                                                              |
| --log-level           | INFO          | 日志级别 (DEBUG, INFO, WARNING, ERROR, CRITICAL)                                                                           |
| --verbose             | -             | 详细调试输出 (True, False)                                                                                              |
| --key                 | None          | 用于身份验证的 API 密钥。保护 LightRAG 服务器免受未经授权的访问                                            |
| --ssl                 | False         | 启用 HTTPS                                                                                                                    |
| --ssl-certfile        | None          | SSL 证书文件路径（如果启用了 --ssl，则需要）                                                                     |
| --ssl-keyfile         | None          | SSL 私钥文件路径（如果启用了 --ssl，则需要）                                                                     |
| --llm-binding         | ollama        | LLM 绑定类型 (lollms, ollama, openai, openai-ollama, azure_openai, aws_bedrock)                                                          |
| --embedding-binding   | ollama        | Embedding 绑定类型 (lollms, ollama, openai, azure_openai, aws_bedrock)                                                                   |

### 重排序配置

重排序查询召回的块可以通过基于优化的相关性评分模型重新排序文档来显著提高检索质量。LightRAG 目前支持以下重排序提供商：

- **Cohere / vLLM**：提供与 Cohere AI 的 `v2/rerank` 端点的完整 API 集成。由于 vLLM 提供了与 Cohere 兼容的重排序器 API，因此也支持通过 vLLM 部署的所有重排序器模型。
- **Jina AI**：提供与所有 Jina rerank 模型的完整实现兼容性。
- **Aliyun**：具有专为支持 Aliyun 的 rerank API 格式而设计的自定义实现。

重排序提供商通过 `.env` 文件配置。以下是使用 vLLM 本地部署的重排序模型的示例配置：

```
RERANK_BINDING=cohere
RERANK_MODEL=BAAI/bge-reranker-v2-m3
RERANK_BINDING_HOST=http://localhost:8000/v1/rerank
RERANK_BINDING_API_KEY=your_rerank_api_key_here
```

以下是使用 Aliyun 提供的 Reranker 服务的示例配置：

```
RERANK_BINDING=aliyun
RERANK_MODEL=gte-rerank-v2
RERANK_BINDING_HOST=https://dashscope.aliyuncs.com/api/v1/services/rerank/text-rerank/text-rerank
RERANK_BINDING_API_KEY=your_rerank_api_key_here
```

有关全面的重排序器配置示例，请参阅 `env.example` 文件。

### 启用重排序

可以针对每个查询启用或禁用重排序。

`/query` 和 `/query/stream` API 端点包含一个 `enable_rerank` 参数，默认设置为 `true`，控制当前查询是否激活重排序。要将 `enable_rerank` 参数的默认值更改为 `false`，请设置以下环境变量：

```
RERANK_BY_DEFAULT=False
```

### 在引用中包含块内容

默认情况下，`/query` 和 `/query/stream` 端点返回的引用仅包含 `reference_id` 和 `file_path`。为了评估、调试或引用目的，您可以请求在引用中包含实际检索的块内容。

`include_chunk_content` 参数（默认：`false`）控制是否在响应引用中包含检索块的实际文本内容。这对于以下情况特别有用：

- **RAG 评估**：测试需要访问检索上下文的系统（如 RAGAS）
- **调试**：验证实际用于生成答案的内容
- **引用显示**：向用户显示支持响应的确切文本段落
- **透明度**：提供对 RAG 检索过程的完全可见性

**重要提示**：`content` 字段是一个**字符串数组**，其中每个字符串代表来自同一文件的块。单个文件可能对应多个块，因此内容作为列表返回以保留块边界。

**API 请求示例：**

```json
{
  "query": "What is LightRAG?",
  "mode": "mix",
  "include_references": true,
  "include_chunk_content": true
}
```

**响应示例（包含块内容）：**

```json
{
  "response": "LightRAG is a graph-based RAG system...",
  "references": [
    {
      "reference_id": "1",
      "file_path": "/documents/intro.md",
      "content": [
        "LightRAG is a retrieval-augmented generation system that combines knowledge graphs with vector similarity search...",
        "The system uses a dual-indexing approach with both vector embeddings and graph structures for enhanced retrieval..."
      ]
    },
    {
      "reference_id": "2",
      "file_path": "/documents/features.md",
      "content": [
        "The system provides multiple query modes including local, global, hybrid, and mix modes..."
      ]
    }
  ]
}
```

**注意事项**：
- 此参数仅在 `include_references=true` 时起作用。在不包含引用的情况下设置 `include_chunk_content=true` 无效。
- **重大变更**：以前的版本将 `content` 作为单个连接字符串返回。现在它返回一个字符串数组以保留各个块的边界。如果需要单个字符串，可以使用您喜欢的分隔符连接数组元素（例如，`"\n\n".join(content)`）。

### .env 示例

```bash
### 服务器配置
# HOST=0.0.0.0
PORT=9621
WORKERS=2

### 文档索引设置
ENABLE_LLM_CACHE_FOR_EXTRACT=true
SUMMARY_LANGUAGE=Chinese
MAX_PARALLEL_INSERT=2

### LLM 配置（使用有效的主机。对于使用 docker 安装的本地服务，您可以使用 host.docker.internal）
TIMEOUT=150
MAX_ASYNC=4

LLM_BINDING=openai
LLM_MODEL=gpt-4o-mini
LLM_BINDING_HOST=https://api.openai.com/v1
LLM_BINDING_API_KEY=your-api-key

### Embedding 配置（使用有效的主机。对于使用 docker 安装的本地服务，您可以使用 host.docker.internal）
# 另请参阅 env.ollama-binding-options.example 以微调 ollama
EMBEDDING_MODEL=bge-m3:latest
EMBEDDING_DIM=1024
EMBEDDING_BINDING=ollama
EMBEDDING_BINDING_HOST=http://localhost:11434

### 用于 JWT 身份验证
# AUTH_ACCOUNTS='admin:admin123,user1:pass456'
# TOKEN_SECRET=your-key-for-LightRAG-API-Server-xxx
# TOKEN_EXPIRE_HOURS=48

# LIGHTRAG_API_KEY=your-secure-api-key-here-123
# WHITELIST_PATHS=/api/*
# WHITELIST_PATHS=/health,/api/*
```

## 文档和块处理

LightRAG 中的文档处理流程有些复杂，分为两个主要阶段：提取阶段（实体和关系提取）和合并阶段（实体和关系合并）。有两个关键参数控制流程并发性：并行处理的最大文件数 (MAX_PARALLEL_INSERT) 和最大并发 LLM 请求数 (MAX_ASYNC)。工作流程描述如下：

1. MAX_ASYNC 限制系统中的并发 LLM 请求总数，包括查询、提取和合并的请求。LLM 请求具有不同的优先级：查询操作的优先级最高，其次是合并，然后是提取。
2. MAX_PARALLEL_INSERT 控制提取阶段并行处理的文件数。为了获得最佳性能，建议将 MAX_PARALLEL_INSERT 设置在 2 到 10 之间，通常是 MAX_ASYNC/3。将此值设置得太高会增加不同文档中实体和关系之间在合并阶段发生命名冲突的可能性，从而降低其整体效率。
3. 在单个文件中，不同文本块的实体和关系提取是并发处理的，并发程度由 MAX_ASYNC 设置。只有在处理了 MAX_ASYNC 文本块后，系统才会继续处理同一文件中的下一批。
4. 当文件完成实体和关系提取后，它进入实体和关系合并阶段。该阶段也并发处理多个实体和关系，并发级别也由 `MAX_ASYNC` 控制。
5. 合并阶段的 LLM 请求优先于提取阶段，以确保合并阶段的文件得到快速处理，并且其结果及时更新到向量数据库中。
6. 为了防止竞态条件，合并阶段避免并发处理相同的实体或关系。当多个文件涉及需要合并的相同实体或关系时，它们将按串行方式处理。
7. 每个文件在流程中被视为一个原子处理单元。只有当文件的所有文本块都完成提取和合并后，文件才会被标记为成功处理。如果在处理过程中发生任何错误，整个文件将被标记为失败，必须重新处理。
8. 当由于错误重新处理文件时，由于 LLM 缓存，可以快速跳过以前处理的文本块。虽然在合并阶段也使用了 LLM 缓存，但合并顺序的不一致性可能会限制其在此阶段的有效性。
9. 如果在提取过程中发生错误，系统不会保留任何中间结果。如果在合并过程中发生错误，已合并的实体和关系可能会被保留；当重新处理同一文件时，重新提取的实体和关系将与现有的合并，而不会影响查询结果。
10. 在合并阶段结束时，所有实体和关系数据都会在向量数据库中更新。如果此时发生错误，一些更新可能会被保留。但是，下一次处理尝试将覆盖以前的结果，确保成功重新处理的文件不会影响未来查询结果的完整性。

大文件应分成较小的段以支持增量处理。可以通过 Web UI 上的"扫描"按钮启动失败文件的重新处理。

## API 端点

所有服务器（LoLLMs、Ollama、OpenAI 和 Azure OpenAI）为 RAG 功能提供相同的 REST API 端点。当 API 服务器运行时，访问：

- Swagger UI：http://localhost:9621/docs
- ReDoc：http://localhost:9621/redoc

您可以使用提供的 curl 命令或通过 Swagger UI 界面测试 API 端点。确保：

1. 启动适当的后端服务（LoLLMs、Ollama 或 OpenAI）
2. 启动 RAG 服务器
3. 使用文档管理端点上传一些文档
4. 使用查询端点查询系统
5. 如果将新文件放入 inputs 目录，则触发文档扫描

## 带有进度跟踪的异步文档索引

LightRAG 实现了异步文档索引，以支持前端监控和查询文档处理进度。上传文件或通过指定端点插入文本后，将返回唯一的 Track ID 以便于实时进度监控。

**支持 Track ID 生成的 API 端点：**

* `/documents/upload`
* `/documents/text`
* `/documents/texts`

**文档处理状态查询端点：**
* `/track_status/{track_id}`

此端点提供全面的状态信息，包括：
* 文档处理状态（pending/processing/processed/failed）
* 内容摘要和元数据
* 处理失败的错误消息
* 创建和更新的时间戳