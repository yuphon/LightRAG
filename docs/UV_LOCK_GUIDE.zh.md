# uv.lock 更新指南

## 什么是 uv.lock？

`uv.lock` 是 uv 的锁文件。它捕获每个依赖项的确切版本，包括传递依赖，类似于：
- Node.js `package-lock.json`
- Rust `Cargo.lock`
- Python Poetry `poetry.lock`

将 `uv.lock` 保留在版本控制中可确保每个人都安装相同的依赖集。

## uv.lock 何时会更改？

### 不会自动更改的情况

- 运行 `uv sync --frozen`
- 构建调用 `uv sync --frozen` 的 Docker 镜像
- 编辑源代码而不触及依赖元数据

### 会更改的情况

1. **`uv lock` 或 `uv lock --upgrade`**

   ```bash
   uv lock                # 根据当前约束解析
   uv lock --upgrade      # 重新解析并升级到最新的兼容版本
   ```

   在修改 `pyproject.toml` 后使用这些命令，当您需要新的依赖版本时，或者锁文件被删除或损坏时。

2. **`uv add`**

   ```bash
    uv add requests           # 添加依赖项并更新两个文件
    uv add --dev pytest       # 添加开发依赖
   ```

   `uv add` 一步完成编辑 `pyproject.toml` 和刷新 `uv.lock`。

3. **`uv remove`**

   ```bash
   uv remove requests
   ```

   这会从 `pyproject.toml` 中删除依赖项并重写 `uv.lock`。

4. **不带 `--frozen` 的 `uv sync`**

   ```bash
   uv sync
   ```

   通常这只会安装已锁定的内容。但是，如果 `pyproject.toml` 和 `uv.lock` 不一致或锁文件丢失，uv 将重新生成并更新 `uv.lock`。在 CI 和生产构建中，您应该首选 `uv sync --frozen` 以防止意外更新。

## 示例工作流程

### 场景 1：添加新依赖

```bash
# 推荐：让 uv 处理两个文件
uv add fastapi
git add pyproject.toml uv.lock
git commit -m "添加 fastapi 依赖"

# 手动替代方案
# 1. 编辑 pyproject.toml
# 2. 重新生成锁文件
uv lock
git add pyproject.toml uv.lock
git commit -m "添加 fastapi 依赖"
```

### 场景 2：放宽或收紧版本约束

```bash
# 1. 在 pyproject.toml 中编辑需求，
#    例如 openai>=1.0.0,<2.0.0 -> openai>=1.5.0,<2.0.0

# 2. 重新解析锁文件
uv lock

# 3. 提交两个文件
git add pyproject.toml uv.lock
git commit -m "将 openai 更新到 >=1.5.0"
```

### 场景 3：将所有内容升级到最新的兼容版本

```bash
uv lock --upgrade
git diff uv.lock
git add uv.lock
git commit -m "升级依赖到最新兼容版本"
```

### 场景 4：团队成员同步项目

```bash
git pull               # 获取最新代码和锁文件
uv sync --frozen       # 完全按照 uv.lock 指定的安装
```

## 在 Docker 中使用 uv.lock

```dockerfile
RUN uv sync --frozen --no-dev --extra api
```

`--frozen` 保证可重现的构建，因为 uv 将拒绝偏离锁定版本。
`--extra api` 安装 API 服务器

## 生成包含离线依赖的锁文件

如果您需要 `uv.lock` 捕获可选的离线依赖项，请启用相关的额外选项来重新生成它：

```bash
uv lock --extra api --extra offline
```

此命令解析基本项目需求以及 `api` 和 `offline` 可选依赖集，确保下游的 `uv sync --frozen --extra api --extra offline` 安装无需进一步解析即可工作。

## 常见问题

- **`uv.lock` 几乎 1 MB。这有问题吗？**
  没有。该文件仅在依赖解析期间读取。

- **我们应该提交 `uv.lock` 吗？**
  是的。提交它，以便协作者和 CI 作业共享相同的依赖图。

- **不小心删除了锁文件？**
  运行 `uv lock` 从 `pyproject.toml` 重新生成它。

- **`uv.lock` 和 `requirements.txt` 可以共存吗？**
  可以，但维护两者是多余的。尽可能仅依赖 `uv.lock`。

- **如何检查锁定的版本？**
  ```bash
  uv tree
  grep -A5 'name = "openai"' uv.lock
  ```

## 最佳实践

### 推荐

1. 将 `uv.lock` 与 `pyproject.toml` 一起提交。
2. 在 CI、Docker 和其他可重现环境中使用 `uv sync --frozen`。
3. 在本地开发期间使用普通的 `uv sync`，如果您希望 uv 为您协调锁。
4. 定期运行 `uv lock --upgrade` 以获取最新的兼容版本。
5. 在更改依赖约束后立即重新生成锁文件。

### 避免

1. 在 CI 或生产管道中运行不带 `--frozen` 的 `uv sync`。
2. 手动编辑 `uv.lock`——uv 会覆盖手动编辑。
3. 在代码审查中忽略锁文件差异——意外的依赖更改可能会破坏构建。

## 总结

| 命令                  | 更新 `uv.lock` | 典型用途                                  |
| --------------------- | -------------- | ----------------------------------------- |
| `uv lock`             | ✅ 是          | 编辑约束后                                |
| `uv lock --upgrade`   | ✅ 是          | 升级到最新的兼容版本                      |
| `uv add <pkg>`        | ✅ 是          | 添加依赖项                                |
| `uv remove <pkg>`     | ✅ 是          | 删除依赖项                                |
| `uv sync`             | ⚠️ 可能        | 本地开发；可以重新生成锁                  |
| `uv sync --frozen`    | ❌ 否          | CI/CD、Docker、可重现构建                 |

记住：`uv.lock` 只在您运行告诉它更改的命令时才会更改。保持它与项目同步，并在更改时提交。
