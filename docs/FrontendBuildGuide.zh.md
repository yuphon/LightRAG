# 前端构建指南

## 概述

LightRAG 项目包含一个基于 React 的 WebUI 前端。本指南解释前端在不同场景下的构建工作原理。

## 核心原则

- **Git 仓库**: 前端构建结果**不**包含在内（保持干净）
- **PyPI 软件包**: 前端构建结果**已**包含在内（即用型）
- **构建工具**: 使用 **Bun**（不是 npm/yarn）

## 安装场景

### 1. 最终用户（从 PyPI）✨

**命令:**
```bash
pip install lightrag-hku[api]
```

**发生了什么:**
- 前端已构建并包含在软件包中
- 无需额外步骤
- Web 界面立即可用

---

### 2. 开发模式（推荐给贡献者）🔧

**命令:**
```bash
# 克隆仓库
git clone https://github.com/HKUDS/LightRAG.git
cd LightRAG

# 以可编辑模式安装（暂不需要前端构建）
pip install -e ".[api]"

# 需要时构建前端（可以随时进行）
cd lightrag_webui
bun install --frozen-lockfile
bun run build
cd ..
```

**优点:**
- 先安装，后构建（灵活的工作流程）
- 更改立即生效（符号链接模式）
- 前端可以随时重新构建而无需重新安装

**工作原理:**
- 创建指向源目录的符号链接
- 前端构建输出到 `lightrag/api/webui/`
- 更改在已安装的软件包中立即可见

---

### 3. 正常安装（测试软件包构建）📦

**命令:**
```bash
# 克隆仓库
git clone https://github.com/HKUDS/LightRAG.git
cd LightRAG

# ⚠️ 必须先构建前端
cd lightrag_webui
bun install --frozen-lockfile
bun run build
cd ..

# 现在安装
pip install ".[api]"
```

**发生了什么:**
- 前端文件被**复制**到 site-packages
- 构建后的修改不会影响已安装的软件包
- 需要重新构建 + 重新安装才能更新

**何时使用:**
- 测试完整的安装过程
- 验证软件包配置
- 模拟 PyPI 用户体验

---

### 4. 创建分发包 🚀

**命令:**
```bash
# 首先构建前端
cd lightrag_webui
bun install --frozen-lockfile --production
bun run build
cd ..

# 创建分发包
python -m build

# 输出: dist/lightrag_hku-*.whl 和 dist/lightrag_hku-*.tar.gz
```

**发生了什么:**
- `setup.py` 检查前端是否已构建
- 如果缺失，安装失败并显示有用的错误消息
- 生成的软件包包含所有前端文件

---

## GitHub Actions（自动发布）

在 GitHub 上创建发布时：

1. **自动构建前端** 使用 Bun
2. **验证** 构建成功完成
3. **创建 Python 软件包** 包含前端
4. **发布到 PyPI** 使用现有的可信发布者设置

**无需人工干预！**

---

## 快速参考

| 场景         | 命令                           | 前端要求              | 可以稍后构建 |
| ------------ | ------------------------------ | --------------------- | ------------ |
| 从 PyPI      | `pip install lightrag-hku[api]` | 已包含               | 否（已安装） |
| 开发         | `pip install -e ".[api]"`      | 否                   | ✅ 是（随时）|
| 正常安装     | `pip install ".[api]"`         | ✅ 是（之前）         | 否（必须重新安装）|
| 创建软件包   | `python -m build`              | ✅ 是（之前）         | 不适用       |

---

## Bun 安装

如果您没有安装 Bun：

```bash
# macOS/Linux
curl -fsSL https://bun.sh/install | bash

# Windows
powershell -c "irm bun.sh/install.ps1 | iex"
```

官方文档: https://bun.sh

---

## 文件结构

```
LightRAG/
├── lightrag_webui/          # 前端源代码
│   ├── src/                 # React 组件
│   ├── package.json         # 依赖项
│   └── vite.config.ts       # 构建配置
│       └── outDir: ../lightrag/api/webui  # 构建输出
│
├── lightrag/
│   └── api/
│       └── webui/           # 前端构建输出（已 gitignore）
│           ├── index.html   # 构建的文件（运行 bun run build 后）
│           └── assets/      # 构建的资产
│
├── setup.py                 # 构建检查
├── pyproject.toml           # 软件包配置
└── .gitignore               # 排除 lightrag/api/webui/*（除了 .gitkeep）
```

---

## 故障排除

### 问：我以开发模式安装，但 Web 界面不工作

**答:** 构建前端：
```bash
cd lightrag_webui && bun run build
```

### 问：我构建了前端，但它不在我的已安装软件包中

**答:** 您可能在构建后使用了 `pip install .`。要么：
- 使用 `pip install -e ".[api]"` 进行开发
- 或重新安装: `pip uninstall lightrag-hku && pip install ".[api]"`

### 问：构建的前端文件在哪里？

**答:** 在运行 `bun run build` 后的 `lightrag/api/webui/` 中

### 问：我可以使用 npm 或 yarn 代替 Bun 吗？

**答:** 项目是为 Bun 配置的。虽然 npm/yarn 可能工作，但根据项目标准建议使用 Bun。

---

## 总结

✅ **PyPI 用户**: 无需操作，前端已包含
✅ **开发者**: 使用 `pip install -e ".[api]"`，需要时构建前端
✅ **CI/CD**: GitHub Actions 中自动构建
✅ **Git**: 前端构建输出从不提交

如有问题或疑虑，请在 GitHub 上创建 issue。
