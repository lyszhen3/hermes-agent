# Hermes Agent 工程分析报告

> 分析时间：2026-04-30 | 版本：**v0.11.0** | Python 文件：**1264 个** | 总文件数：**2599 个**
> 作者：**Nous Research** | 许可证：**MIT** | Python 要求：**>= 3.11**（开发环境 **3.13.7**）
> 仓库地址：[github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)

---

## 一、项目概述

**Hermes Agent** 是由 [Nous Research](https://nousresearch.com) 开发的开源 AI 智能助手框架，定位为"自我进化的 AI 代理——从经验中创造技能、在使用中不断改进、随时随地运行"。核心特性包括：**闭环学习**（自动创建/优化技能）、**跨平台持久化**（Telegram / Discord / Slack / WhatsApp / Signal / 钉钉 / 企业微信 / 飞书等 25+ 平台）、**六种终端后端**（本地 / Docker / SSH / Daytona / Singularity / Modal），以及内置的 **cron 定时调度**和**子代理委派**能力。

项目经 GitNexus 索引包含 **79,399 个符号、166,563 条关系、300 条执行流程**，测试套件约 **700 个测试文件**（AGENTS.md 自述 15k+ 测试用例），覆盖 18 个测试目录。

---

## 二、整体技术架构

```
┌─────────────────────────────────────────────────────────────┐
│                      用户交互层                              │
│  CLI (prompt_toolkit)  │  TUI (Ink/React)  │  Web Dashboard │
│  ──────────────────────┼───────────────────┼────────────────│
│  cli.py                │  ui-tui/           │  web/          │
│                         │  tui_gateway/      │  PTY Bridge    │
├─────────────────────────────────────────────────────────────┤
│                      消息网关层                              │
│  gateway/run.py + gateway/platforms/ (23个平台适配器)        │
│  Telegram │ Discord │ Slack │ WhatsApp │ Signal │ 钉钉 │ ...│
├─────────────────────────────────────────────────────────────┤
│                      核心引擎层                              │
│  run_agent.py (AIAgent)  ← 会话管理 + 模型调用 + 工具编排    │
│  model_tools.py           ← 工具发现 + 调用处理              │
│  hermes_state.py          ← SQLite FTS5 会话存储             │
├─────────────────────────────────────────────────────────────┤
│                      工具 & 技能层                           │
│  tools/*.py (自动注册)    │  skills/ (内置)                  │
│                           │  optional-skills/ (可选)         │
├─────────────────────────────────────────────────────────────┤
│                      插件 & 扩展层                           │
│  plugins/memory/  │  plugins/context_engine/                 │
│  plugins/image_gen/  │  plugins/observability/              │
├─────────────────────────────────────────────────────────────┤
│                      基础设施层                              │
│  environments/ (RL训练)  │  cron/ (定时任务)                │
│  scripts/ (测试/发布)    │  tests/ (700+ 测试文件)          │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、核心模块详解

### 3.1 核心引擎 (`run_agent.py`)

**AIAgent 类** 是整个系统的核心（AGENTS.md 自述约 12,000 行），承担以下职责：

| 职责 | 说明 |
|------|------|
| 会话管理 | 维护对话上下文、消息历史、迭代计数 |
| 模型调用 | 支持多 Provider（OpenAI / Anthropic / HuggingFace / Mistral / Bedrock 等）|
| 工具编排 | 自动发现和调用工具，处理 tool_calls 响应 |
| 预算控制 | `iteration_budget` 限制 API 调用次数（默认 90 次）|
| 中断处理 | `_interrupt_requested` 支持外部中断 |
| 子代理 | `_run_single_child()` 支持委托子代理执行 |

**核心对话循环**（简化版）：

```python
while api_call_count < max_iterations:
    response = client.chat.completions.create(messages=messages, tools=tool_schemas)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args)
            messages.append(tool_result_message(result))
    else:
        return response.content
```

消息格式遵循 OpenAI 标准：`{"role": "system/user/assistant/tool", ...}`，推理内容存储在 `assistant_msg["reasoning"]` 中。

### 3.2 工具系统 (`tools/`)

工具系统采用 **自动注册** 机制，任何 `tools/*.py` 文件中的 `registry.register()` 调用都会被自动发现。

- **自动发现**：无需手动维护导入列表
- **Schema 管理**：统一的 JSON Schema 定义
- **环境验证**：`check_fn` 检查 API Key 等依赖
- **结果格式**：所有 handler 返回 JSON 字符串

**已注册的 40+ 工具示例**：

| 工具类别 | 具体工具 |
|----------|----------|
| 浏览器 | `browser_tool`, `browser_cdp_tool`, `browser_camofox` |
| 代码执行 | `code_execution_tool` |
| 文件操作 | `file_tools`, `file_operations`, `file_state` |
| MCP | `mcp_tool`, `mcp_oauth`, `mcp_oauth_manager` |
| 审批 | `approval`, `clarify_tool` |
| 委托 | `delegate_tool` |
| 第三方集成 | `feishu_doc_tool`, `feishu_drive_tool`, `discord_tool` |

### 3.3 CLI 架构 (`cli.py`)

HermesCLI 类（AGENTS.md 自述约 11,000 行），基于 **Rich + prompt_toolkit** 构建：

- **KawaiiSpinner**：动画表情指示器
- **皮肤引擎**：数据驱动的 CLI 主题系统（`hermes_cli/skin_engine.py`）
- **斜杠命令**：中央注册表 `COMMAND_REGISTRY`，自动推导到所有下游

**命令分发链路**：
```
COMMAND_REGISTRY → CLI → Gateway → Telegram Menu → Slack → Autocomplete
```

### 3.4 TUI 架构 (`ui-tui/` + `tui_gateway/`)

基于 **Ink (React)** 的现代化终端 UI，通过 JSON-RPC over stdio 与 Python 后端通信。TypeScript 负责屏幕渲染，Python 负责会话、工具和模型调用。

```
Node (Ink)  ──stdio JSON-RPC──  Python (tui_gateway)
    │                               └─ AIAgent + tools + sessions
    └─ 渲染：transcript, composer, prompts, activity
```

**核心组件**：
- `app.tsx` - 主应用组件，含聊天流式渲染和本地斜杠命令处理
- `messageLine.tsx` - 消息流式渲染（delta/complete 事件）
- `thinking.tsx` - 工具活动显示（tool.start/progress/complete 事件）
- `prompts.tsx` - 审批/确认/澄清/密钥弹窗 UI

**Dashboard 嵌入模式**：Web Dashboard（`hermes dashboard` → `/chat`）通过 `hermes_cli/pty_bridge.py` + WebSocket（`/api/pty`）将 `hermes --tui` 的 PTY 输出桥接到浏览器中的 xterm.js 终端，而非重新实现聊天 UI。这意味着 TUI 的任何改进都会自动同步到 Web 端。

### 3.5 消息网关 (`gateway/`)

支持 **23 个平台适配器**的统一接入层：

| 平台 | 适配器文件 | 说明 |
|------|-----------|------|
| Telegram | `telegram.py`, `telegram_network.py` | 支持 webhooks 和网络代理 |
| Discord | `discord.py` | 含语音支持 |
| Slack | `slack.py` | 支持 Bolt SDK |
| WhatsApp | `whatsapp.py` | — |
| Signal | `signal.py` | 端到端加密 |
| Matrix | `matrix.py` | 联邦协议 |
| Mattermost | `mattermost.py` | 企业聊天 |
| 钉钉 | `dingtalk.py` | 阿里生态 |
| 企业微信 | `wecom.py`, `wecom_callback.py`, `wecom_crypto.py` | 含回调加解密 |
| 飞书 | `feishu.py`, `feishu_comment.py`, `feishu_comment_rules.py` | 含评论互动 |
| 微信 | `weixin.py` | — |
| 邮件 | `email.py` | — |
| SMS | `sms.py` | — |
| 元宝 | `yuanbao.py`, `yuanbao_media.py`, `yuanbao_proto.py`, `yuanbao_sticker.py` | 含多媒体与贴纸 |
| HomeAssistant | `homeassistant.py` | 智能家居 |
| BlueBubbles | `bluebubbles.py` | iMessage 桥接 |
| API Server | `api_server.py` | 通用 HTTP API |
| Webhook | `webhook.py` | 通用 Webhook |

网关包含两条消息守卫：
1. **基础适配器** (`gateway/platforms/base.py`) — 排队管理活跃会话消息
2. **网关运行器** (`gateway/run.py`) — 拦截 `/stop`、`/new`、`/approve`、`/deny` 等控制命令

---

## 四、技能与插件生态

### 4.1 内置技能 (`skills/` - 25 个类别)

```
apple, autonomous-ai-agents, creative, data-science, devops, diagramming,
dogfood, domain, email, gaming, gifs, github, inference-sh, mcp, media,
mlops, note-taking, productivity, red-teaming, research, smart-home,
social-media, software-development, yuanbao
```

每个技能通过 `SKILL.md` 配置，支持 frontmatter 元数据。

### 4.2 可选技能 (`optional-skills/` - 15 个类别)

重量级或利基技能，需手动安装：

```
autonomous-ai-agents, blockchain, communication, creative, devops, dogfood,
email, health, mcp, migration, mlops, productivity, research, security,
web-development
```

### 4.3 插件系统 (`plugins/`)

| 插件类型 | 功能 | 实现 |
|----------|------|------|
| 记忆提供者 | 多种记忆后端 | honcho, mem0, supermemory, byterover, hindsight, holographic, openviking, retaindb |
| 上下文引擎 | 上下文管理 | `context_engine/` |
| 图片生成 | AI 图片生成 | `image_gen/` |
| 可观测性 | 监控/追踪 | `observability/` |
| 仪表盘 | Dashboard 示例 | `example-dashboard/` |
| 第三方集成 | Spotify, Google Meet | `spotify/`, `google_meet/` |

---

## 五、环境与部署

### 5.1 终端后端 (`tools/environments/`)

| 后端 | 适用场景 |
|------|----------|
| Local | 本地终端 |
| Docker | 容器隔离 |
| SSH | 远程执行 |
| Modal | Serverless |
| Daytona | 云端开发环境 |
| Singularity | HPC 容器 |

### 5.2 Profiles 多实例

支持 **完全隔离的多实例**，每个 Profile 拥有独立的：
- 配置文件 (`config.yaml`)
- API Key (`.env`)
- 记忆存储
- 会话数据
- 技能管理
- 网关配置

---

## 六、开发规范与质量保障

### 6.1 测试体系

- **测试文件**：~700 个测试文件，覆盖 18 个测试目录
- **测试执行**：`scripts/run_tests.sh` 保证 CI 环境一致性
- **关键约束**：
  - `xdist` 固定 4 workers（避免排序问题）
  - `TZ=UTC, LANG=C.UTF-8`
  - 所有 `*_API_KEY`/`*_TOKEN` 环境变量被清除

### 6.2 核心设计原则

| 原则 | 说明 |
|------|------|
| **Prompt 缓存完整性** | 不改变中间上下文，不中途切换工具集 |
| **Profile 安全** | 所有路径使用 `get_hermes_home()`，禁止硬编码 `~/.hermes` |
| **不做变更检测器测试** | 测试行为而非数据快照 |
| **GitNexus 强制** | 编辑前必须运行影响分析 |

### 6.3 配置管理

- `config.yaml`：非敏感设置（超时、特性开关、显示偏好）
- `.env`：仅 API Key、Token、密码等敏感信息
- 三类配置加载器：CLI、Gateway、CLI Subcommands

---

## 七、技术栈与依赖管理

### 7.1 技术栈总览

| 层级 | 技术 |
|------|------|
| 后端语言 | Python >= 3.11（开发环境 3.13.7）|
| 前端 TUI | TypeScript + Ink (React) |
| 终端 CLI | Rich + prompt_toolkit |
| 数据存储 | SQLite (FTS5) |
| 代码分析 | GitNexus 知识图谱 |
| 测试框架 | PyTest + xdist + pytest-asyncio |
| 消息网关 | 多平台适配器（25+）|
| 构建打包 | setuptools >= 61.0 |
| 部署 | Nix, Docker, scripts/release.py |

### 7.2 核心依赖（`pyproject.toml`）

| 依赖 | 版本 | 用途 |
|------|------|------|
| `openai` | >=2.21.0,<3 | OpenAI API 客户端 |
| `anthropic` | >=0.39.0,<1 | Anthropic Claude API |
| `rich` | >=14.3.3,<15 | CLI 美化渲染 |
| `prompt_toolkit` | >=3.0.52,<4 | 交互式 CLI 输入 |
| `httpx[socks]` | >=0.28.1,<1 | HTTP 客户端（含 SOCKS 代理）|
| `pyyaml` | >=6.0.2,<7 | YAML 配置解析 |
| `pydantic` | >=2.12.5,<3 | 数据验证 |
| `tenacity` | >=9.1.4,<10 | 重试机制 |
| `jinja2` | >=3.1.5,<4 | 模板引擎 |
| `croniter` | >=6.0.0,<7 | Cron 表达式解析 |
| `edge-tts` | >=7.2.7,<8 | 免费 TTS（无需 API Key）|

### 7.3 可选依赖组

| 组名 | 说明 | 关键依赖 |
|------|------|----------|
| `messaging` | 消息平台集成 | python-telegram-bot, discord.py, slack-bolt |
| `voice` | 本地语音 | faster-whisper, sounddevice |
| `rl` | RL 训练环境 | atroposlib, tinker, wandb |
| `web` | Web Dashboard | fastapi, uvicorn |
| `modal` | Modal Serverless | modal |
| `daytona` | 云端开发环境 | daytona |
| `dingtalk` | 钉钉集成 | dingtalk-stream |
| `feishu` | 飞书集成 | lark-oapi |
| `google` | Google Workspace | google-api-python-client |
| `bedrock` | AWS Bedrock | boto3 |
| `mistral` | Mistral AI | mistralai |
| `acp` | VS Code/JetBrains 集成 | agent-client-protocol |
| `all` | 全量安装 | 聚合上述所有组 |

### 7.4 CLI 入口点

| 命令 | 模块入口 | 说明 |
|------|----------|------|
| `hermes` | `hermes_cli.main:main` | 主 CLI 入口 |
| `hermes-agent` | `run_agent:main` | 核心 Agent 入口 |
| `hermes-acp` | `acp_adapter.entry:main` | ACP 协议（IDE 集成）|

### 7.5 支持的平台

| 操作系统 | 状态 |
|----------|------|
| **Linux** | ✅ 完全支持 |
| **macOS** | ✅ 完全支持 |
| **WSL2** | ✅ 完全支持 |
| **Android / Termux** | ✅ 支持（`[termux]` 精简安装）|
| **原生 Windows** | ❌ 不支持，建议通过 WSL2 使用 |

安装方式：`curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash`

---

## 八、项目亮点

1. **高度模块化**：工具、技能、插件均采用自动注册机制，添加新功能零耦合
2. **多端统一**：同一套核心引擎支持 CLI/TUI/Web/25+消息平台
3. **缓存友好**：精心设计的 prompt 缓存策略，大幅降低 API 调用成本
4. **Profile 隔离**：原生支持多实例完全隔离，适应多场景部署
5. **工程纪律**：严格的 GitNexus 影响分析 + 测试规范，保障代码质量
6. **生态丰富**：40+ 工具、25 个技能类别、8 种记忆后端，开箱即用

---

> 本报告基于 `AGENTS.md` 及项目实际结构分析生成。
