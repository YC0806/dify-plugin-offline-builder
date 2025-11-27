# Dify Plugin Offline Builder

## Dify 插件离线打包工具

[![Python Version](https://img.shields.io/badge/python-3.12%2B-blue.svg)](https://www.python.org/downloads/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

🚀 一键将 Dify 插件转换为包含预编译依赖的离线安装包

[简体中文](#) | [English](README_EN.md)

---

## 快速开始

```bash
# 安装依赖
pip install requests pyyaml

# 从 Dify 市场打包插件
python plugin_repackaging.py market xcsf my-plugin 0.1.0

# 从本地文件打包
python plugin_repackaging.py local ./your-plugin.difypkg

# 为 Linux AMD64 平台交叉打包
python plugin_repackaging.py -p manylinux2014_x86_64 -s linux-amd64 local ./your-plugin.difypkg
```

---

### 简介

`plugin_repackaging.py` 是一个专为 Dify 插件离线部署设计的自动化打包工具。它能够将 Dify 插件（.difypkg）转换为包含所有预编译依赖的离线安装包，解决在无网络或受限网络环境下的插件部署问题。

**核心特性：**

- **完全离线部署**：将所有 Python 依赖预构建为 wheel 格式并打包到插件中
- **跨平台支持**：支持为不同平台（Linux/Windows/macOS、x86_64/ARM64）交叉打包依赖
- **智能预编译**：使用 `pip wheel` 将源码包自动转换为预编译 wheel，避免目标环境缺少编译工具
- **多源下载**：支持从 Dify 市场、GitHub Release 或本地文件获取插件
- **自动化流程**：一键完成下载、解压、依赖处理、重新打包全流程

**主要改进：**

相比传统的 `pip download` 方式，本工具使用 `pip wheel` 进行依赖处理，具有以下优势：

- ✅ 预构建所有源码包（.tar.gz、.zip）为 wheel 格式，无需目标环境具备编译能力
- ✅ 自动下载并打包构建依赖（setuptools、wheel）
- ✅ 验证所有依赖均为 wheel 格式，确保离线安装成功率
- ✅ 支持大型插件包（最大 5GB）打包

### 先决条件

- Python 3.12+
- `pip` 可用
- 安装 Python 库：`requests`、`PyYAML`
- `dify-plugin-<os>-<arch>` 工具位于脚本同目录（脚本会根据宿主机的 OS 与架构自动选择文件名，例如 `dify-plugin-windows-amd64` 或 `dify-plugin-linux-arm64`）。
- 对于 Linux，若系统安装了 `unzip`，脚本会优先使用系统 `unzip`，否则使用 Python 标准库解压。

可选环境变量（覆盖默认值）：

- `GITHUB_API_URL` - GitHub API/主机前缀，默认 `https://github.com`
- `MARKETPLACE_API_URL` - Dify 市场 API 地址，默认 `https://marketplace.dify.ai`
- `PIP_MIRROR_URL` - pip 镜像源，默认 `https://mirrors.aliyun.com/pypi/simple`

### 安装依赖

在 PowerShell 下（Windows）：

```powershell
python -m pip install --upgrade pip
python -m pip install requests pyyaml
```

或者使用虚拟环境：

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1

pip install requests pyyaml
```

### 用法

基本命令行：

```powershell
python plugin_repackaging.py [-p 平台] [-s 后缀] <source> <args...>

# source: market | github | local
```

参数说明：

- `-p, --platform`：为 pip 下载指定平台，比如 `manylinux2014_x86_64`。脚本会把其转成 pip 参数：`--platform <platform> --only-binary=:all:`，并在 `pip download` 时使用。可用于交叉打包依赖。
- `-s, --suffix`：输出包后缀，默认 `offline`，最终输出文件名样式为 `<plugin>-<suffix>.difypkg`。

source 依赖的附加参数：

- market: `market [author] [name] [version]`
  - 示例：`python plugin_repackaging.py market xcsf my-plugin 0.1.0`
- github: `github [repo] [release] [asset_name]`
  - `repo` 支持带或不带 `https://github.com` 前缀。`asset_name` 需包含 `.difypkg` 后缀。
  - 示例：`python plugin_repackaging.py github xcsf/my-plugin v1.0.0 my-plugin-1.0.0.difypkg`
- local: `local [difypkg_path]`
  - 示例：`python plugin_repackaging.py local ./langgenius-openai_api_compatible_0.0.23.difypkg`

示例（Windows PowerShell，指定平台并自定义后缀）：

```powershell
python plugin_repackaging.py -p manylinux2014_x86_64 -s linux-amd64 local .\your-plugin.difypkg
```

示例（从 Dify 市场下载并打包）：

```powershell
python plugin_repackaging.py market xcsf plugin-name 0.1.0
```

示例（从 GitHub Release 下载并打包）：

```powershell
python plugin_repackaging.py github xcsf/my-repo v1.2.3 plugin-1.2.3.difypkg
```

### Dify 平台配置

为了成功安装离线打包的插件，需要修改 Dify 平台的 `.env` 配置文件：

```bash
# 允许安装未在 Marketplace 上架的插件
FORCE_VERIFYING_SIGNATURE=false

# 允许安装最大 500MB 的插件包
PLUGIN_MAX_PACKAGE_SIZE=524288000

# 允许 Nginx 上传最大 500MB 的内容
NGINX_CLIENT_MAX_BODY_SIZE=500M
```

修改后需要重启 Dify 服务以使配置生效。

### 执行流程概览

1. **解压插件**：将目标 `.difypkg` 解压到以包名命名的文件夹
2. **修改元数据**：
   - 修改 `manifest.yaml` 中的 `author` 字段为 `xcsf`
   - 规范化 `manifest.yaml` 中的 `created_at` 时间格式（统一为 ISO 8601 格式）
   - 修改 `.verification.dify.json` 中的 `authorized_category` 字段为 `xcsf`
3. **预下载构建依赖**：使用 `pip download` 下载 setuptools 和 wheel 到 `./wheels` 目录
4. **预构建所有依赖**：使用 `pip wheel` 将 `requirements.txt` 中的所有依赖（包括源码包）构建为 wheel 格式
5. **验证依赖格式**：检查 `wheels/` 目录中所有文件是否为 `.whl` 格式
6. **修改安装配置**：将 `requirements.txt` 改为离线安装模式（添加 `--no-index --find-links=./wheels/`）
7. **清理忽略规则**：移除 `.difyignore` 或 `.gitignore` 中对 `wheels/` 目录的忽略
8. **重新打包**：调用本地 `dify-plugin-<os>-<arch>` 工具生成带后缀的离线 `.difypkg` 文件

### 常见问题与排查建议

**依赖构建相关：**

- **pip wheel 构建失败**：
  - 检查网络或镜像源是否可用。可用 `PIP_MIRROR_URL` 环境变量替换默认镜像。
  - 确保本地 Python 环境已安装必要的编译工具（如 gcc、build-essential）。
  - 若某些包需要系统依赖（如 libxml2、libffi），请先安装相应的开发包。
  - 若需要更详细日志，手动运行 `pip wheel -r requirements.txt -w ./wheels --index-url <mirror>` 查看错误。

- **非 wheel 格式警告**：
  - 脚本会验证所有依赖是否为 wheel 格式，如果发现非 wheel 文件，可能需要在目标环境编译。
  - 解决方案：确保源环境具备编译能力，或使用 `-p` 参数指定目标平台进行交叉编译。

**工具与环境相关：**

- **找不到 `dify-plugin-...` 可执行文件**：
  - 确认可执行文件位于脚本同目录且命名匹配（脚本会根据 `platform.system()` 与 `platform.machine()` 决定名称）。
  - 在 Windows 上文件通常为 `dify-plugin-windows-amd64`（无扩展名），确认存在并可执行；在 Linux/macOS 上需有可执行权限（`chmod +x`）。

- **解压失败**：
  - Linux 环境下优先使用系统 `unzip`（若已安装），否则使用 Python 的 `zipfile` 模块。
  - 确保 `.difypkg` 文件完整且未损坏。

- **权限或路径问题**：
  - 请在脚本所在目录运行，或者使用绝对路径指定本地 `.difypkg` 文件。

**Dify 平台配置：**

- **插件安装签名验证失败**：
  - 在 .env 配置文件将 `FORCE_VERIFYING_SIGNATURE` 改为 `false`，Dify 平台将允许安装所有未在 Dify Marketplace 上架（审核）的插件。

- **插件包过大无法上传**：
  - 在 .env 配置文件将 `PLUGIN_MAX_PACKAGE_SIZE` 增大为 `524288000`（500MB），Dify 平台将允许安装 500M 大小以内的插件。
  - 在 .env 配置文件将 `NGINX_CLIENT_MAX_BODY_SIZE` 增大为 `500M`，Nginx 客户端将允许上传 500M 大小以内的内容。

### 技术细节

**为什么使用 `pip wheel` 而不是 `pip download`？**

传统的 `pip download` 只能下载包文件，无法处理源码分发包（Source Distribution）。这意味着：

- ❌ 下载的 `.tar.gz` 或 `.zip` 文件需要在目标环境编译
- ❌ 目标环境必须具备编译工具（gcc、make 等）
- ❌ 某些平台（如 Alpine Linux）可能编译失败

使用 `pip wheel` 的优势：

- ✅ 在打包环境预编译所有源码包，生成二进制 wheel
- ✅ 目标环境无需编译工具，直接安装预编译的 wheel
- ✅ 安装速度更快，成功率更高

**时间格式规范化：**

脚本会自动将 `manifest.yaml` 中的 `created_at` 字段统一为 ISO 8601 格式（`YYYY-MM-DDTHH:MM:SS+TZ`），避免因时间格式不一致导致的解析错误。

### 可定制点（代码层面）

- **作者字段修改**：`manifest.yaml` 中的 `author` 和 `.verification.dify.json` 中的 `authorized_category` 目前硬编码为 `xcsf`。如需自定义，请修改脚本第 90 行和第 103 行。
- **pip 镜像源**：可通过 `PIP_MIRROR_URL` 环境变量指定不同的 PyPI 镜像源。
- **最大包大小**：脚本中默认最大包大小为 5120MB（第 177 行），可根据需要调整 `--max-size` 参数。

### 使用场景

- **内网部署**：企业内网无法访问外部 PyPI，需要离线安装插件
- **跨平台打包**：在 Windows/macOS 上为 Linux 服务器打包插件
- **安全合规**：需要审计所有依赖包，不允许运行时动态下载
- **加速部署**：预编译依赖可显著加快插件安装速度

### 贡献

欢迎提交 issue 或 pull request 来改进工具：

- 将 `author` 和 `authorized_category` 改为命令行参数
- 支持更多平台的交叉编译配置
- 改进错误处理和日志输出
- 增加对更多包格式的支持

### 许可证

本项目遵循 MIT 许可证。使用或分发前请查看 LICENSE 文件。

---

**注意**：本工具仅用于合法授权的 Dify 插件打包。请确保遵守插件原作者的许可协议和 Dify 平台的使用条款。
