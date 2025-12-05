# XBow Competition - AI 驱动的 CTF 自动解题系统

一个完整的 AI Agent 自动化 XBOW 解题方案，结合 MCP 服务器和智能 CLI 客户端，实现自主渗透测试和 XBOW 挑战。

## 项目概述

本项目由两个核心组件组成：

1. [ez-xbow-platform-mcp](https://github.com/m-sec-org/ez-xbow-platform-mcp) - 模型上下文协议 (MCP) 服务器

   - 提供 XBOW 挑战管理、知识库、Kali 容器等能力
   - 作为 AI Agent 与 XBow 平台之间的桥梁
   - 支持本地模拟测试和真实平台对接

2. [kimi-cli-for-xbow](https://github.com/m-sec-org/kimi-cli-for-xbow) - CTF 专用 AI Agent CLI

   - 基于 Kimi CLI 深度定制的自动解题客户端
   - 支持多种 AI 模型（DeepSeek、通义千问等）
   - Daemon 模式实现无人值守自动解题


### 工作原理

````
┌─────────────────────┐
│   Kimi CLI Agent    │  ← 用户交互层
│  (AI 决策引擎)       │
└──────────┬──────────┘
           │ MCP 协议
           ↓
┌─────────────────────┐
│  XBow MCP Server    │  ← 能力抽象层
│  (工具和知识库)      │
└──────────┬──────────┘
           │
           ├→ XBow Platform (真实/模拟)
           ├→ Kali Container (工具执行)
           └→ Knowledge Base (技术文档)
````

## 核心特性

### 🛡️ XBow MCP Server

- **挑战管理** - 列出、尝试和提交 CTF 挑战的解决方案
- **知识库** - 内建涵盖 9 种漏洞类别的 CTF 技术文档（XSS、SQL、SSTI、SSRF、IDOR、XXE、LFI、CodeI、AFR&PT）
- **持久化 Kali 容器** - 在隔离环境中执行安全工具（nmap、sqlmap、gobuster 等）
- **尝试历史** - 通过自动笔记管理，跨会话跟踪和共享发现
- **多种协议** - 支持 stdio、SSE 和 HTTP/2 流模式

### 🤖 Kimi CLI Agent

- **自定义模型支持** - 支持 DeepSeek、通义千问等 OpenAI 兼容 API
- **专用 Agent 模式** - 针对 CTF 竞赛优化的 Agent（ctfer、security、security_beta）
- **Daemon 自动解题** - 无人值守自动获取题目并解题
- **命令执行防沉迷** - 防止无限循环和过度命令执行
- **Session 隔离** - 按工作目录维护独立的对话上下文
- **Shell 集成** - 支持 Zsh 集成和 Shell 命令执行

## 快速开始

### 前提条件

- **XBow MCP Server**:

  - Go 1.24.7+
  - Docker (支持 buildx)

- **Kimi CLI**:

  - Python 3.13+
  - [uv](https://docs.astral.sh/uv/) 包管理器


### 第一步：安装 Kimi CLI

```bash
# 安装 uv（如果未安装）
curl -LsSf https://astral.sh/uv/install.sh | sh

# 安装 Kimi CLI
uv tool install --python 3.13 kimi-cli

# 验证安装
kimi --help
```

或者从源码安装：

```bash
cd kimi-cli-for-xbow
uv sync
uv run kimi
```

### 第二步：构建 XBow MCP Server

```bash
cd ez-xbow-platform-mcp
go build -o xbow-mcp ./cmd/main.go
```

### 第三步：配置 MCP 连接

在工作目录创建 `mcp.json` 配置文件：

```json
{
    "mcpServers": {
        "xbow": {
            "url": "http://127.0.0.1:8080"
        }
    }
}
```

### 第四步：启动 MCP 服务器

#### 使用模拟平台（测试环境）

在 `ez-xbow-platform-mcp/mock-challenges` 目录下创建模拟题目：

```json
{
    "challenge_code": "XBEN-123-24",
    "difficulty": "easy",
    "points": 200,
    "target_info": {
        "ip": "192.168.5.37",
        "port": [32820]
    },
    "hint_viewed": false,
    "solved": false,
    "hint_content": "Sometimes the AuthN/AuthZ checks are vulnerable to TOCTOU issues",
    "solution": "flag{xxxx}"
}
```

启动模拟服务器：

```bash
./xbow-mcp --mock -listen 127.0.0.1:8080
```

#### 使用真实平台（生产环境）

```bash
./xbow-mcp \
  -xbow-url https://your-xbow-platform.com \
  -xbow-token YOUR_AUTH_TOKEN \
  -mode streamable \
  -listen 127.0.0.1:8080
```

### 第五步：配置 Kimi CLI

1. 启动 Kimi CLI 进行初始配置：

```bash
kimi
```

2. 发送 `/setup` 命令配置 AI 模型：

选择 Custom API，根据提示配置：

**DeepSeek 示例**：

- API Base: `https://api.deepseek.com/v1`
- Provider: OpenAI Legacy
- Model: `deepseek-chat`

**通义千问示例**：

- API Base: `https://dashscope.aliyuncs.com/compatible-mode/v1`
- Provider: OpenAI Legacy
- Model: `qwen-plus`

### 第六步：开始自动解题

#### 交互式解题

```bash
kimi -a security -m deepseek-chat
```

然后在 CLI 中输入你的解题需求。

#### Daemon 模式（自动解题）

```bash
kimi -a security -m deepseek-chat --daemon --verbose \
  -c "优先尝试没有做过的题目，解决的题禁止尝试做和验证，如果 list_challenges 没有题目就说明完成任务了"
```

**参数说明**：

- `-a security`：使用 CTF 专用 Agent
- `-m deepseek-chat`：指定 AI 模型
- `--daemon`：启用自动化模式
- `--verbose`：显示详细日志
- `-c "..."`：初始任务指令

## MCP Server 可用工具

| 工具                   | 描述                                                                |
| ---------------------- | ------------------------------------------------------------------- |
| `list_challenges`      | 获取当前阶段的挑战，包括难度和目标信息                              |
| `do_challenge`         | 标记挑战为进行中，并增加尝试计数器                                  |
| `get_challenge_hint`   | 检索提示（会扣除分数）                                              |
| `submit_answer`        | 提交 Flag 并接收评分结果                                            |
| `get_ctf_skill`        | 访问技术文档（xss、sql、ssti、ssrf、idor、xxe、lfi、codei、afr&pt） |
| `write_challenge_note` | 保存发现和尝试记录，供将来参考                                      |
| `read_challenge_note`  | 查看历史笔记（每 9 次尝试后自动重置）                               |
| `kail_terminal`        | 在持久化 Kali 容器中执行命令                                        |
| `get_terminal_history` | 通过 ID 检索命令执行结果                                            |

## 高级功能

### 多实例并行解题

构建独立二进制实现多开：

```bash
cd kimi-cli-for-xbow
make build
```

在不同目录同时运行多个实例：

```bash
# 终端 1：专注 Web 题目
cd /path/to/project1
./kimi -a security -m deepseek-chat --daemon --verbose -c "优先做 web 题"

# 终端 2：专注 PWN 题目
cd /path/to/project2
./kimi -a security_beta -m qwen-plus --daemon --verbose -c "优先做 pwn 题"
```

### Session 隔离

每个工作目录自动维护独立的 session：

```bash
# 新 session
kimi

# 继续上次 session
kimi --continue
# 或
kimi -C

# 指定工作目录
kimi -w /path/to/project --continue
```

Session 数据存储在当前目录的 `.kimi` 文件夹中。

### Zsh 集成

将 Kimi CLI 集成到 Zsh：

```bash
git clone https://github.com/MoonshotAI/zsh-kimi-cli.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/kimi-cli
```

在 `~/.zshrc` 中添加：

```bash
plugins=(... kimi-cli)
```

重启 Zsh 后，按 `Ctrl-X` 切换到 Agent 模式。

### Security Beta Agent

使用增强的 security_beta Agent 需要先克隆 PayloadsAllTheThings 仓库：

```bash
git clone https://github.com/swisskyrepo/PayloadsAllTheThings.git
```

然后使用：

```bash
kimi -a security_beta -m deepseek-chat
```

CLI 会利用文件工具自动筛选和使用相关的渗透测试技巧。

## MCP Server 命令行选项

```bash
# 服务器模式
-mode, -m [stdio|sse|streamable]       MCP 服务器协议 (默认: streamable)
-listen, -l ADDR:PORT                  监听地址 (默认: 127.0.0.1:8080)

# 平台配置
-xbow-url, -u URL                      XBow API 基础 URL
-xbow-token, -t TOKEN                  认证 Token

# Docker 配置
-docker-container, -c NAME             容器名称 (默认: xbow-kail)
-docker-image, -i IMAGE:TAG            Docker 镜像 (默认: xbow-kail:latest)
-dockerfile-dir, -f PATH               Dockerfile 路径 (默认: ./Dockerfile)
-docker-exec-log-dir, -d DIR           执行日志目录 (默认: ./.kail-history)

# 模拟平台（本地测试）
--mock                                 启用模拟平台服务器
-mock-addr ADDR:PORT                   模拟服务器地址 (默认: 127.0.0.1:8000)
-mock-dir PATH                         模拟挑战目录 (默认: ./mock-challenges)
```

## 本地存储

### XBow MCP Server

- `.challenge_history/{challenge_code}/` - 尝试元数据、笔记和历史记录
- `.kail-history/` - 命令执行记录

### Kimi CLI

- `.kimi/` - Session 数据、对话历史和配置

## 项目结构

````
xbow-competition/
├── ez-xbow-platform-mcp/       # MCP 服务器
│   ├── cmd/                    # 主程序入口
│   ├── mock-challenges/        # 模拟题目
│   ├── Dockerfile              # Kali 容器配置
│   └── README.MD
├── kimi-cli-for-xbow/          # CLI 客户端
│   ├── src/                    # 源代码
│   ├── agents/                 # Agent 配置
│   ├── mcp.json                # MCP 配置示例
│   ├── start.sh                # 快速启动脚本
│   └── README.md
└── README.md                   # 本文件
````

## 开发

### XBow MCP Server

```bash
cd ez-xbow-platform-mcp
go build -o xbow-mcp ./cmd/main.go
```

### Kimi CLI

```bash
cd kimi-cli-for-xbow
make prepare  # 准备开发环境
uv run kimi   # 运行 CLI

# 其他命令
make format   # 格式化代码
make check    # 代码检查
make test     # 运行测试
make help     # 显示所有命令
```

## 故障排查

### MCP 连接失败

- 确认 MCP 服务器已启动且监听正确端口
- 检查 `mcp.json` 配置是否正确
- 验证防火墙规则

### Kali 容器无法启动

- 确认 Docker 服务正在运行
- 检查 Dockerfile 路径是否正确
- 验证是否有足够的磁盘空间

### AI 模型调用失败

- 验证 API Key 是否有效
- 检查 API Base URL 是否正确
- 确认模型名称拼写正确
- 查看 API 配额是否充足

### Daemon 模式无响应

- 使用 `--verbose` 查看详细日志
- 检查是否触发了防沉迷保护
- 验证 MCP 服务器是否正常响应

## 贡献

欢迎提出 Issue 和 Pull Request！

<p align="center">
<a href="https://github.com/m-sec-org/ez-xbow-platform-mcp/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=m-sec-org/ez-xbow-platform-mcp&max=36" alt="sponsors-list" >
</a>
</p>


### XBow MCP Server

项目地址：[m-sec-org/ez-xbow-platform-mcp](https://github.com/m-sec-org/ez-xbow-platform-mcp)

### Kimi CLI for XBow

基于官方 [MoonshotAI/kimi-cli](https://github.com/MoonshotAI/kimi-cli) 定制开发。

项目地址：[m-sec-org/kimi-cli-for-xbow](https://github.com/m-sec-org/kimi-cli-for-xbow)

## 许可

本项目各组件遵循各自的开源许可协议，详见各子项目的 LICENSE 文件。

## 相关资源

- [Model Context Protocol (MCP) 规范](https://modelcontextprotocol.io/)
- [Kimi API 文档](https://platform.moonshot.cn/docs)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

## 致谢

感谢 [Moonshot AI](https://www.moonshot.cn/) 提供的 Kimi CLI 基础框架。

***

**注意**：本项目仅供授权的安全测试、CTF 竞赛和教育目的使用。请勿用于非法活动。
