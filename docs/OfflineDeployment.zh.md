# LightRAG 离线部署指南

本指南提供了在互联网访问受限或不可用的离线环境中部署 LightRAG 的全面说明。

如果您使用 Docker 部署 LightRAG，则无需参考此文档，因为 LightRAG Docker 镜像已预先配置为离线操作。

> 需要 `transformers`、`torch` 或 `cuda` 的软件包将不会包含在离线依赖组中。因此，Docling 等文档提取工具以及 Hugging Face 和 LMDeploy 等本地 LLM 模型不在离线安装支持范围内。这些高计算资源需求的服务不应集成到 LightRAG 中。Docling 将被解耦并作为独立服务部署。

## 目录

- [概述](#概述)
- [快速开始](#快速开始)
- [分层依赖](#分层依赖)
- [Tiktoken 缓存管理](#tiktoken-缓存管理)
- [完整的离线部署工作流程](#完整的离线部署工作流程)
- [故障排除](#故障排除)

## 概述

LightRAG 使用动态包安装（`pipmaster`）基于文件类型和配置的可选功能。在离线环境中，这些动态安装将失败。本指南向您展示如何预安装所有必要的依赖项和缓存文件。

### 什么会动态安装？

LightRAG 动态安装以下软件包：

- **存储后端**: `redis`、`neo4j`、`pymilvus`、`pymongo`、`asyncpg`、`qdrant-client`
- **LLM 提供商**: `openai`、`anthropic`、`ollama`、`zhipuai`、`aioboto3`、`voyageai`、`llama-index`、`lmdeploy`、`transformers`、`torch`
- **Tiktoken 模型**: 从 OpenAI CDN 下载的 BPE 编码模型

**注意**: 文档处理依赖项（`pypdf`、`python-docx`、`python-pptx`、`openpyxl`）现在随 `api` 额外组预安装，不再需要动态安装。

## 快速开始

### 选项 1：使用带有离线额外选项的 pip

```bash
# 在线环境：安装所有离线依赖
pip install lightrag-hku[offline]

# 下载 tiktoken 缓存
lightrag-download-cache

# 创建离线软件包
pip download lightrag-hku[offline] -d ./offline-packages
tar -czf lightrag-offline.tar.gz ./offline-packages ~/.tiktoken_cache

# 传输到离线服务器
scp lightrag-offline.tar.gz user@offline-server:/path/to/

# 离线环境：安装
tar -xzf lightrag-offline.tar.gz
pip install --no-index --find-links=./offline-packages lightrag-hku[offline]
export TIKTOKEN_CACHE_DIR=~/.tiktoken_cache
```

### 选项 2：使用需求文件

```bash
# 在线环境：下载软件包
pip download -r requirements-offline.txt -d ./packages

# 传输到离线服务器
tar -czf packages.tar.gz ./packages
scp packages.tar.gz user@offline-server:/path/to/

# 离线环境：安装
tar -xzf packages.tar.gz
pip install --no-index --find-links=./packages -r requirements-offline.txt
```

## 分层依赖

LightRAG 为不同的用例提供灵活的依赖组：

### 可用的依赖组

| 组              | 描述               | 用例                                   |
| --------------- | ------------------ | -------------------------------------- |
| `api`           | API 服务器 + 文档处理 | 带有 PDF、DOCX、PPTX、XLSX 支持的 FastAPI 服务器 |
| `offline-storage` | 存储后端           | Redis、Neo4j、MongoDB、PostgreSQL 等    |
| `offline-llm`   | LLM 提供商         | OpenAI、Anthropic、Ollama 等            |
| `offline`       | 完整的离线软件包   | API + 存储 + LLM（所有功能）            |

**注意**: 文档处理（PDF、DOCX、PPTX、XLSX）包含在 `api` 额外组中。以前的 `offline-docs` 组已合并到 `api` 中以实现更好的集成。

> 需要 `transformers`、`torch` 或 `cuda` 的软件包将不会包含在离线依赖组中。

### 安装示例

```bash
# 安装带有文档处理的 API
pip install lightrag-hku[api]

# 安装 API 和存储后端
pip install lightrag-hku[api,offline-storage]

# 安装所有离线依赖（推荐用于离线部署）
pip install lightrag-hku[offline]
```

### 使用单独的需求文件

```bash
# 仅存储后端
pip install -r requirements-offline-storage.txt

# 仅 LLM 提供商
pip install -r requirements-offline-llm.txt

# 所有离线依赖
pip install -r requirements-offline.txt
```

## Tiktoken 缓存管理

Tiktoken 在首次使用时下载 BPE 编码模型。在离线环境中，您必须预先下载这些模型。

### 使用 CLI 命令

安装 LightRAG 后，使用内置命令：

```bash
# 下载到默认位置（有关确切路径请参见输出）
lightrag-download-cache

# 下载到特定目录
lightrag-download-cache --cache-dir ./tiktoken_cache

# 仅下载特定模型
lightrag-download-cache --models gpt-4o-mini gpt-4
```

### 默认下载的模型

- `gpt-4o-mini`（LightRAG 默认）
- `gpt-4o`
- `gpt-4`
- `gpt-3.5-turbo`
- `text-embedding-ada-002`
- `text-embedding-3-small`
- `text-embedding-3-large`

### 在离线环境中设置缓存位置

```bash
# 选项 1：环境变量（临时）
export TIKTOKEN_CACHE_DIR=/path/to/tiktoken_cache

# 选项 2：添加到 ~/.bashrc 或 ~/.zshrc（持久）
echo 'export TIKTOKEN_CACHE_DIR=~/.tiktoken_cache' >> ~/.bashrc
source ~/.bashrc

# 选项 3：复制到默认位置
cp -r /path/to/tiktoken_cache ~/.tiktoken_cache/
```

## 完整的离线部署工作流程

### 步骤 1：在在线环境中准备

```bash
# 1. 安装带有离线依赖的 LightRAG
pip install lightrag-hku[offline]

# 2. 下载 tiktoken 缓存
lightrag-download-cache --cache-dir ./offline_cache/tiktoken

# 3. 下载所有 Python 软件包
pip download lightrag-hku[offline] -d ./offline_cache/packages

# 4. 创建传输的存档
tar -czf lightrag-offline-complete.tar.gz ./offline_cache

# 5. 验证内容
tar -tzf lightrag-offline-complete.tar.gz | head -20
```

### 步骤 2：传输到离线环境

```bash
# 使用 scp
scp lightrag-offline-complete.tar.gz user@offline-server:/tmp/

# 或使用 USB/物理介质
# 将 lightrag-offline-complete.tar.gz 复制到 USB 驱动器
```

### 步骤 3：在离线环境中安装

```bash
# 1. 提取存档
cd /tmp
tar -xzf lightrag-offline-complete.tar.gz

# 2. 安装 Python 软件包
pip install --no-index \
    --find-links=/tmp/offline_cache/packages \
    lightrag-hku[offline]

# 3. 设置 tiktoken 缓存
mkdir -p ~/.tiktoken_cache
cp -r /tmp/offline_cache/tiktoken/* ~/.tiktoken_cache/
export TIKTOKEN_CACHE_DIR=~/.tiktoken_cache

# 4. 添加到 shell 配置文件以实现持久化
echo 'export TIKTOKEN_CACHE_DIR=~/.tiktoken_cache' >> ~/.bashrc
```

### 步骤 4：验证安装

```bash
# 测试 Python 导入
python -c "from lightrag import LightRAG; print('✓ LightRAG 已导入')"

# 测试 tiktoken
python -c "from lightrag.utils import TiktokenTokenizer; t = TiktokenTokenizer(); print('✓ Tiktoken 工作正常')"

# 测试可选依赖项（如果已安装）
python -c "import docling; print('✓ Docling 可用')"
python -c "import redis; print('✓ Redis 可用')"
```

## 故障排除

### 问题：Tiktoken 因网络错误而失败

**问题**: `Unable to load tokenizer for model gpt-4o-mini`

**解决方案**:
```bash
# 确保设置了 TIKTOKEN_CACHE_DIR
echo $TIKTOKEN_CACHE_DIR

# 验证缓存文件存在
ls -la ~/.tiktoken_cache/

# 如果为空，您需要首先在在线环境中下载缓存
```

### 问题：动态软件包安装失败

**问题**: `Error installing package xxx`

**解决方案**:
```bash
# 预安装您需要的特定软件包
# 对于带有文档处理的 API：
pip install lightrag-hku[api]

# 对于存储后端：
pip install lightrag-hku[offline-storage]

# 对于 LLM 提供商：
pip install lightrag-hku[offline-llm]
```

### 问题：运行时缺少依赖项

**问题**: `ModuleNotFoundError: No module named 'xxx'`

**解决方案**:
```bash
# 检查您安装了什么
pip list | grep -i xxx

# 安装缺失的组件
pip install lightrag-hku[offline]  # 安装所有离线依赖
```

### 问题：tiktoken 缓存权限被拒绝

**问题**: `PermissionError: [Errno 13] Permission denied`

**解决方案**:
```bash
# 确保缓存目录具有正确的权限
chmod 755 ~/.tiktoken_cache
chmod 644 ~/.tiktoken_cache/*

# 或使用用户可写目录
export TIKTOKEN_CACHE_DIR=~/my_tiktoken_cache
mkdir -p ~/my_tiktoken_cache
```

## 最佳实践

1. **首先在在线环境中测试**: 在离线之前，始终在在线环境中测试完整的设置。

2. **保持缓存更新**: 当新模型发布时，定期更新您的离线缓存。

3. **记录您的设置**: 记录您实际需要哪些可选依赖项。

4. **版本固定**: 考虑在生产环境中固定特定版本：
   ```bash
   pip freeze > requirements-production.txt
   ```

5. **最小化安装**: 仅安装您需要的内容：
   ```bash
   # 如果您只需要带有文档处理的 API
   pip install lightrag-hku[api]
   # 然后手动添加特定的 LLM: pip install openai
   ```

## 其他资源

- [LightRAG GitHub 仓库](https://github.com/HKUDS/LightRAG)
- [Docker 部署指南](./DockerDeployment.md)
- [API 文档](../lightrag/api/README.md)

## 支持

如果您遇到本指南未涵盖的问题：

1. 查看 [GitHub Issues](https://github.com/HKUDS/LightRAG/issues)
2. 阅读 [项目文档](../README.md)
3. 使用您的离线部署详细信息创建新 issue
