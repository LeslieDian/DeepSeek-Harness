# DeepSeek-Harness 在 Windows 本地的部署实验

> 本仓库记录了一次完整的本地化实验：在 Windows 11 上从源码构建
> **DeepSeek Harness (dsh) v0.2.0-rc.2** 桌面版 `.exe`，并接入自定义的
> **MiniMax M3** 模型（OpenAI 兼容协议）。
>
> 实验目标不是分发二进制，而是**沉淀一套可复现的、面向受限网络环境的本地构建流程**。

---

## 目录

1. [实验结论速览](#1-实验结论速览)
2. [实验背景与目标](#2-实验背景与目标)
3. [架构理解](#3-架构理解)
4. [环境与工具链](#4-环境与工具链)
5. [网络问题与镜像方案](#5-网络问题与镜像方案)
6. [源码获取](#6-源码获取)
7. [自定义模型配置：MiniMax M3](#7-自定义模型配置minimax-m3)
8. [从源码构建桌面版 .exe](#8-从源码构建桌面版-exe)
9. [安装与运行桌面版](#9-安装与运行桌面版)
10. [端到端验证](#10-端到端验证)
11. [故障排查](#11-故障排查)
12. [安全提醒](#12-安全提醒)
13. [引用与致谢](#13-引用与致谢)

---

## 1. 实验结论速览

✅ **已完成**

- 在 Windows 上从源码构建出 `deepseek-harness-0.2.0-rc.2-win-x64-unsigned.exe`（274.6 MB）
- 通过 NSIS 静默安装参数 `/S` 完成本地安装
- 桌面版内嵌的 dsh 服务在 `http://127.0.0.1:19387/` 上正常启动（端口每次随机）
- 通过 Cordis 补丁层为 `desktop` profile 接入 MiniMax M3，提供方显示正常
- 接入方式的最小化改动（两个文件即可让任意 profile 用上自定义模型）

📁 **本仓库产物**

| 文件 | 用途 |
|------|------|
| `package.json` | 工作区声明 dsh 依赖（用于跨 profile 引用） |
| `schema.json` | Cordis 配置 JSON Schema 参考（`dsh --dump-config` 生成） |
| `.gitignore` | 排除 `node_modules` / `harness-src/` / `.exe` 等大文件 |
| `README.md` | 本文件，实验全流程 |

> `harness-src/`（3.9 GB 源码）、`node_modules/`、构建产物 `.exe` 都不进 git，
> 它们要么能通过镜像克隆，要么是用户机器上的中间产物。

---

## 2. 实验背景与目标

### 2.1 背景

DeepSeek 在 2026 年开源了 **DeepSeek Harness**（项目代号 `dsh`，仓库
`github.com/deepseek-ai/deepseek-harness`），一个面向多模型、多入口的 agent harness：

- 多入口：`web` / `headless` / `tui` / `sdk` / `acp` / `desktop`（Electron）
- 多协议 LLM：内置 pi-ai 适配器，支持 OpenAI Completions / OpenAI Responses /
  Anthropic Messages 等多种协议
- 多项目扩展：基于 Cordis 插件框架，所有能力（工具、LLM、文件访问、agent 循环）
  都是插件

### 2.2 目标

1. ✅ 在 Windows 上跑通 `dsh web`（验证 CLI 与 web UI）
2. ✅ 把第三方模型 **MiniMax M3** 接入为默认模型（替换 DeepSeek 官方）
3. ✅ 从源码打包出桌面版 `.exe` 安装器
4. ✅ 安装并启动桌面版，确认 MiniMax M3 在桌面 profile 下也生效

### 2.3 约束

- **网络受限**：所在网络对 GitHub 大文件下载极不稳定（Connection reset），
  但 npm registry / 国内镜像可用
- **未签名构建**：本环境无代码签名证书，使用 `unsigned` 目标产物

---

## 3. 架构理解

理解这三件事之后，构建与配置就不再是黑盒：

### 3.1 Profile 是有序插件栈

dsh 的核心抽象是 **profile**：一个目录 + 一份 bundle 列表。运行时把以下层叠加：

```
bundle 层（包内置）
  ↓
~/.dsh/profiles/<name>/cordis.yml        ← 用户基线
  ↓
~/.dsh/profiles/<name>/cordis.patch.yml  ← 用户补丁层（用户编辑这个）
  ↓
--patch CLI 覆盖                       ← 临时覆盖
```

所有能力以 **Cordis 插件**（TS 模块，导出 `apply(ctx)`）的形式接入。
`cordis.patch.yml` 的每个条目是**针对某个插件 id 的配置覆盖**。

### 3.2 pi-ai LLM 适配器

```
@deepseek-ai/dsh-llm-pi-ai  ← 插件 id: "llm-pi-ai"
```

负责把 provider 抽象成统一的 LLM 接口。Provider 通过 `providers` dict 注册：

```yaml
providers:
  <provider-id>:
    apiKeyEnv: ENV_VAR_NAME   # ← 引用环境变量，不存明文
    displayName: 显示名
    api: openai-completions | openai-responses | anthropic-messages
    baseURL: https://...
    models:
      - id: model-id
        name: 显示名
```

要设为默认：`@deepseek-ai/dsh-agent` 的 `agent-default-model` 插件，
覆盖其 `provider` + `model` 字段。

### 3.3 桌面版架构

```
DeepSeek Harness.exe          ← Electron 主进程
  └─ 内嵌 dsh runtime        ← 真正的 agent 服务
       └─ 内嵌 web UI         ← http://127.0.0.1:<随机端口>/
```

桌面版读取的 profile 名是 **`desktop`**（保留给 Electron shell），
不在 `web` 或 `headless`。所以要让 MiniMax M3 在桌面版生效，**必须**
给 `~/.dsh/profiles/desktop/cordis.patch.yml` 加相同的补丁。

---

## 4. 环境与工具链

| 工具 | 版本 | 备注 |
|------|------|------|
| OS | Windows 11 / PowerShell 5.1 | — |
| Node.js | v24.14.1（自定义路径 `D:\node.js`） | 满足 `engines` |
| pnpm | 11.7.0（通过 corepack 管理） | `packageManager` 字段声明 |
| Git | 2.53.0 | — |
| Python | 3.13.5 / 3.14.3（系统） | 仅供 uv 探测使用 |
| uv | 0.12.3（`~/.local/bin/uv.exe`，**不在 PATH**） | 用绝对路径调用 |
| Electron | 44.0.0 | 桌面版运行时 |
| electron-builder | 26.15.3 | 打 NSIS 安装器 |

环境变量：

- `MINIMAX_API_KEY` — MiniMax M3 的 API key（**用户级**设置，重启后所有进程可见）
- `DSH_DESKTOP_NPM_REGISTRY` — 桌面版打包时 runtime install 走哪个 npm registry

---

## 5. 网络问题与镜像方案

这是本环境**最具特色**的部分。直接 `curl` GitHub 大文件几乎必然失败：

```
curl: (35) OpenSSL SSL_connect: Connection was reset
```

60 秒只能下 1 MB 的下载速度，让 `gh release download` 完全不可用。

### 5.1 实测可用的源

| 用途 | 直连 GitHub | 镜像/代理 |
|------|-------------|-----------|
| **克隆 dsh 源码** | ❌ 多次失败 | ✅ `https://gitcode.com/gh_mirrors/de/deepseek-harness.git`（13s 完成） |
| **下载 GitHub release 资产** | ❌ Connection reset | ✅ `https://ghfast.top/https://github.com/.../release.tar.gz` |
| **下载 Electron 二进制** | ❌ CDN 185.199.110.133 卡死 | ✅ `https://registry.npmmirror.com/-/binary/electron/<v>-win32-x64.zip` 手动放到 `%LOCALAPPDATA%\electron\Cache\` |
| **下载 npm 包** | ⚠️ 大包失败 | ✅ `https://registry.npmmirror.com`（npmmirror，287 包 12.1s） |
| **下载 Python wheel** | ❌ files.pythonhosted.org 卡死 | ✅ `https://pypi.tuna.tsinghua.edu.cn/simple`（清华，10 MB/s） |

### 5.2 关键技巧：缓存复用

dsh 桌面版的打包脚本 `apps/desktop/scripts/prepare-dsh.ts` 实现了**缓存优先**：

```
downloadPrimaryRuntimeAsset(url, sha256)
  → 检查 .desktop-build/downloads/<sha256> 是否存在
    → 命中：直接复用，零网络
    → 未命中：下载到 downloads/<sha256>，再校验
```

只要把文件**按 sha256 命名**放到这个目录，所有运行时组件（Node、Python、
wheels）就**零网络复用**。这一点让构建脚本对网络中断具有极强的鲁棒性。

### 5.3 缓存就位清单（实测）

```
%LOCALAPPDATA%\electron\Cache\
  electron-v44.0.0-win32-x64.zip          150.2 MB  ← 手放
  SHASUMS256.txt                          ← sha256 校验

harness-src/apps/desktop/.desktop-build/downloads/
  158f7685…   Node 24.21.0                35.9 MB
  7c45c962…   Python 3.12.14              21.0 MB
  86945f2e…   numpy wheel                 12.2 MB
  536232a5…   pandas wheel                 9.3 MB
  a2b55dd6…   pillow wheel                 6.9 MB
  3e9a00d1…   lxml wheel                   3.8 MB
```

每个文件 SHA256 校验通过后才放入缓存。

---

## 6. 源码获取

不要尝试从 GitHub 直拉（参见第 5 节），用 gitcode 镜像：

```powershell
cd "D:\vs project\deepssek harness"
git clone https://gitcode.com/gh_mirrors/de/deepseek-harness.git harness-src
```

约 3.9 GB，预计 1 分钟内完成（gitcode CDN 稳定）。

如果中途网络抖动，可以断点续传：`git fetch` 会自动继续。

---

## 7. 自定义模型配置：MiniMax M3

> ⚠️ 下面的代码块展示了**结构**，**不要**把你的真实 API key 粘贴到任何配置文件中。
> API key 只能通过环境变量提供。

### 7.1 MiniMax 协议信息

| 字段 | 值 |
|------|---|
| `baseURL` | `https://api.minimaxi.com/v1` |
| 模型 id | `MiniMax-M3` |
| 协议 | OpenAI 兼容（`openai-completions`） |
| Auth | Bearer Token（环境变量 `MINIMAX_API_KEY` 提供） |

### 7.2 补丁模板

`~/.dsh/profiles/<profile-name>/cordis.patch.yml` 写入：

```yaml
# 注册 MiniMax 提供方（id: minimax）
- id: llm-pi-ai
  config:
    providers:
      minimax:
        apiKeyEnv: MINIMAX_API_KEY       # ← 引用环境变量，不存明文
        displayName: MiniMax M3
        api: openai-completions
        baseURL: https://api.minimaxi.com/v1
        models:
          - id: MiniMax-M3
            name: MiniMax-M3

# 把 agent 默认模型指向 MiniMax M3
- id: agent-default-model
  config:
    provider: minimax
    model: MiniMax-M3
```

### 7.3 需要打补丁的 profile

| profile | 用途 | 是否需要补丁 |
|---------|------|--------------|
| `web` | `dsh web` CLI 启动的 Web UI | ✅ |
| `headless` | `dsh --profile headless` 无头执行 | ✅ |
| `desktop` | 桌面版 `.exe` 内嵌 dsh | ✅ **必加**（很多教程只改了 web，结果桌面版无效） |

desktop profile 的内容是空的（`[]`），构建桌面版后**第一次启动会自动生成**
`~/.dsh/profiles/desktop/`，但里面只有注释。

### 7.4 提供 API key

**最推荐：用户级环境变量**

```powershell
[Environment]::SetEnvironmentVariable('MINIMAX_API_KEY', '<你的key>', 'User')
```

设置后**新启动的进程**（包括桌面版）都会自动读到。无需重启系统。

**备选：UI 手动输入**

桌面版 → 设置 → 模型 → 点击 "MiniMax M3" 行 → "编辑" → 粘贴 key → 保存。
dsh 会把 key 存到 `~/.dsh/.credentials.yaml`（加密），下次启动自动加载。

---

## 8. 从源码构建桌面版 .exe

### 8.1 准备 Node 与 pnpm

```powershell
# Node v24+ 必须从自定义 PATH
$env:PATH = "D:\node.js;$env:PATH"
node -v   # 应该是 v24.x

# 启用 corepack（一次性）
corepack enable
corepack prepare pnpm@11.7.0 --activate
pnpm -v
```

### 8.2 切换 npm registry

```powershell
# 一次性，全局生效
pnpm config set registry https://registry.npmmirror.com
```

### 8.3 装依赖 + 构建

```powershell
cd "D:\vs project\deepssek harness\harness-src"
pnpm install                 # 287 包，约 12s（npmmirror）
pnpm run build               # tsc 编译所有包，EXIT 0
```

### 8.4 桌面版 .env.windows 配置

`apps/desktop/.env.windows` 写入：

```ini
DSH_DESKTOP_APP_ID=com.deepseek.harness
DSH_DESKTOP_AUTO_UPDATE_ENV=test
DOWNLOAD_TEST_ORIGIN=https://download.deepseek.com
DOWNLOAD_TEST_RELEASE_ID=00000000000000000000000000000000
DSH_DESKTOP_NPM_REGISTRY=https://registry.npmmirror.com   # ← 关键
DSH_DESKTOP_MANDATORY_UPDATE_TEST_ORIGIN=https://download.deepseek.com
DSH_DESKTOP_MANDATORY_UPDATE_CONFIG={"allowedAuthOrigins":["https://download.deepseek.com"]}
DSH_DESKTOP_WINDOWS_SIGNATURE_CACHE_CONCURRENCY=4
```

**关键**：最后两行让 unsigned 构建通过——只要 `allowedAuthOrigins` 非空
（任何字符串都行），electron-builder 就不会因为没签名而失败。

### 8.5 触发打包

```powershell
pnpm run package:desktop:win:x64:unsigned
```

输出：

```
apps/desktop/.desktop-build/targets/win-x64/unsigned-artifacts/
  deepseek-harness-0.2.0-rc.2-win-x64-unsigned.exe   274.6 MB
  latest.yml                                           1.5 KB
  deepseek-harness-0.2.0-rc.2-win-x64-unsigned.exe.blockmap
```

> 💡 `runtime:lockfile` 阶段偶发崩溃（`PostQueuedCompletionStatus: (6)` +
> `pnpm exited with 2147483651`）—— **重试一次就好**，是 pnpm 偶发问题，
> 与本环境网络无关。

### 8.6 复制安装器到工作区根

```powershell
Copy-Item "harness-src\apps\desktop\.desktop-build\targets\win-x64\unsigned-artifacts\deepseek-harness-0.2.0-rc.2-win-x64-unsigned.exe" "D:\vs project\deepssek harness\"
```

---

## 9. 安装与运行桌面版

### 9.1 静默安装

```powershell
$exe = "D:\vs project\deepssek harness\deepseek-harness-0.2.0-rc.2-win-x64-unsigned.exe"
$p = Start-Process $exe -ArgumentList '/S' -PassThru
$p.WaitForExit()   # 约 60-90 秒
# exit code = 0 即成功
```

NSIS 静默参数 `/S` 会跳过 SmartScreen 交互弹窗（注意未签名警告需要手动确认
或先在 SmartScreen 中点"更多信息 → 仍要运行"，再启动）。

### 9.2 安装位置

```
C:\Users\<user>\AppData\Local\Programs\DeepSeek Harness\
  DeepSeek Harness.exe     233 MB    ← 主程序
  chrome_*.pak, *.dll                  ← Electron 运行时
  icudtl.dat, *.dat                    ← Chromium 资源
  resources/                           ← Electron 资源目录
```

开始菜单也会创建快捷方式。

### 9.3 启动

```powershell
Start-Process "C:\Users\<user>\AppData\Local\Programs\DeepSeek Harness\DeepSeek Harness.exe"
```

正常启动会有 6 个进程（主进程 + GPU + 渲染器 + utility），并**在 stdout 输出**：

```
dsh web: http://127.0.0.1:19387/?token=<base64-token>
```

端口每次启动随机，token 每次变化。

---

## 10. 端到端验证

### 10.1 设置模型补丁并重启

```powershell
# 写入 desktop profile 的 patch（首次启动会自动创建 profile 目录）
# 内容见 7.2 节

# 关闭所有 dsh 桌面版进程
Get-Process | Where-Object { $_.ProcessName -match 'DeepSeek Harness' } | Stop-Process -Force

# 重新启动
Start-Process "C:\Users\<user>\AppData\Local\Programs\DeepSeek Harness\DeepSeek Harness.exe"
```

### 10.2 UI 验证清单

打开 `http://127.0.0.1:19387/?token=<token>`（token 在桌面版 stdout）：

- [ ] 中文 UI 正常加载
- [ ] 没有"预览版说明"弹窗（`welcomeNoticeVersion: 2026-09-28.1` 生效）
- [ ] 点"设置"→"模型"，列表里有 **"MiniMax M3"** 行
- [ ] 该行不显示红色"API 密钥缺失"图标（前提：`MINIMAX_API_KEY` 已设为用户级）

### 10.3 命令行验证（更彻底）

```powershell
$env:MINIMAX_API_KEY = "<你的key>"
cd "D:\vs project\deepssek harness\harness-src"
node_modules\.bin\dsh.cmd --profile headless "用一句话回答：2+2=几？只回答数字"
```

期望 stdout 出现 `4`。这会真正调用 MiniMax M3 API，验证整条链路：
dsh → pi-ai → minimax provider → OpenAI 协议 → MiniMax。

---

## 11. 故障排查

### 11.1 大文件下载 Connection reset

→ 用第 5 节列出的镜像。**永远不要直连 GitHub CDN**。

### 11.2 Electron 缓存目录缺 SHASUMS256.txt

打包脚本读 `electron-v44.0.0-win32-x64.zip` 时校验 sha256，需**同时**放
`SHASUMS256256.txt` 进 `%LOCALAPPDATA%\electron\Cache\`。从 npmmirror
binary 目录下载 zip 时**手动**校验一次：

```powershell
Get-FileHash .\electron-v44.0.0-win32-x64.zip -Algorithm SHA256
```

对比 npmmirror 提供的哈希。

### 11.3 桌面版 UI 显示"预览版说明"反复弹窗

→ 给 `cordis.patch.yml` 加：

```yaml
- id: ui-settings-general
  name: "@deepseek-ai/dsh-client-ui-settings-general"
  config:
    welcomeNoticeVersion: 2026-09-28.1
```

### 11.4 桌面版模型页显示"API 密钥缺失"

→ 说明进程读不到 `MINIMAX_API_KEY`。检查：

```powershell
[Environment]::GetEnvironmentVariable('MINIMAX_API_KEY', 'User')
```

未设置 → 用第 7.4 节命令设置后**重启桌面版**。

### 11.5 打包时 `runtime:lockfile` 崩溃

→ 重试一次。这是 pnpm 偶发，不是本环境问题。

### 11.6 unsigned 构建失败

→ 确认 `apps/desktop/.env.windows` 中
`DSH_DESKTOP_MANDATORY_UPDATE_CONFIG` 存在且 `allowedAuthOrigins` 非空。

---

## 12. 安全提醒

### 12.1 API Key 永不入代码

本仓库**不包含**任何真实 API key。所有引用都通过环境变量：

```yaml
apiKeyEnv: MINIMAX_API_KEY     ← 引用变量名，不存明文
```

`.gitignore` 已排除：

- `.env` / `.env.*`（除 `.env.example`）
- `apps/desktop/.env.windows`
- `~/.dsh/profiles/*/cordis.patch.yml`（用户配置）

### 12.2 建议轮换 key

如果你之前在聊天/截图/日志中暴露过 API key，**强烈建议在 MiniMax 控制台
重新生成一次 key**，让旧的失效。

### 12.3 Windows SmartScreen

未签名安装器会触发 SmartScreen 警告。建议：

- 个人开发机：在 SmartScreen 弹窗选"更多信息 → 仍要运行"
- 生产/分发：用代码签名证书（`signtool.exe` + EV 证书）重签

---

## 13. 引用与致谢

- **DeepSeek Harness**：<https://github.com/deepseek-ai/deepseek-harness>
- **Cordis 插件框架**：vendored 在 `vendor/cordis/`
- **pi-ai LLM 适配器**：vendored 在 `packages/dsh-llm-pi-ai/`
- **镜像源**：
  - gitcode 镜像（GitHub 克隆）
  - ghfast.top（GitHub release 下载）
  - npmmirror（npm 包 / Electron 二进制）
  - 清华 PyPI（Python wheels）

本仓库不包含 dsh 源码（请按第 6 节克隆），仅记录部署流程与配置示例。
