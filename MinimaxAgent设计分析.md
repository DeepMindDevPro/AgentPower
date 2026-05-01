Mini Agent 详细系统架构设计
一、整体分层架构
┌──────────────────────────────────────────────────────────────┐
│                      接入层 (Entry Layer)                     │
│   CLI (cli.py)                  ACP Server (acp/)            │
│   prompt_toolkit 交互式终端       Zed Editor / ACP 协议集成     │
└───────────────────────┬──────────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────────┐
│                    Agent 核心层 (agent.py)                    │
│   ReAct Loop: LLM调用 → Tool执行 → 结果回填 → 迭代            │
│   Token管理 / 消息摘要 / 取消控制                              │
└────────────┬──────────────────────────┬───────────────────────┘
             │                          │
┌────────────▼──────────┐   ┌───────────▼───────────────────────┐
│    LLM 客户端层 (llm/) │   │       工具系统层 (tools/)          │
│  LLMClient (Wrapper)  │   │  File / Bash / Note / Skill / MCP │
│  AnthropicClient      │   └───────────────────────────────────┘
│  OpenAIClient         │
│  OllamaClient         │
└───────────────────────┘
        │
┌───────▼──────────────────────────────────────────────────────┐
│              基础设施层 (Infrastructure)                       │
│   config.py (配置管理)  retry.py (重试)  logger.py (日志)     │
│   schema/ (数据模型)                                           │
└──────────────────────────────────────────────────────────────┘

二、接入层
2.1 CLI 模式（cli.py）
main()
  ├─ parse_args()         # 解析 --workspace / --task / log
  └─ asyncio.run(run_agent())
       ├─ Config.from_yaml()         # 加载配置
       ├─ LLMClient()                # 初始化 LLM
       ├─ initialize_base_tools()    # 加载非 workspace 工具
       ├─ add_workspace_tools()      # 加载 workspace 相关工具
       ├─ Agent()                    # 创建 Agent 实例
       └─ 交互循环 (PromptSession)
            ├─ /help /clear /stats /log 命令处理
            ├─ Esc键监听线程 (threading) → cancel_event
            └─ asyncio.create_task(agent.run())

特性：

基于 prompt_toolkit 实现历史记录、自动补全、多行输入
Esc 键通过独立线程监听，通过 asyncio.Event 非侵入式取消
支持 --task 非交互式批量执行
2.2 ACP 模式（acp/）
将 Agent 封装为 ACP 协议服务器，通过 stdio 与 Zed Editor 通信：

run_acp_server()
  └─ AgentSideConnection(MiniMaxACPAgent)
       ├─ initialize()    # 协议握手
       ├─ newSession()    # 为每个会话创建独立 Agent 实例
       ├─ prompt()        # 处理用户输入，执行 _run_turn()
       └─ cancel()        # 设置取消标志

三、Agent 核心执行循环
agent.run()
│
├─ [循环 max_steps 次]
│   │
│   ├─ 1. 检查取消 (_check_cancelled)
│   │
│   ├─ 2. Token 检查与摘要 (_summarize_messages)
│   │       ├─ tiktoken 精确统计 (_estimate_tokens)
│   │       ├─ 或 API 返回 api_total_tokens
│   │       └─ 超限时 → _create_summary()
│   │               对每轮用户消息之间的 agent/tool 消息调用 LLM 生成摘要
│   │               替换为 [Assistant Execution Summary] 消息
│   │
│   ├─ 3. LLM 调用 (llm.generate)
│   │       └─ 返回 LLMResponse(content, thinking, tool_calls, usage)
│   │
│   ├─ 4. 若无 tool_calls → 任务完成，返回 response.content
│   │
│   └─ 5. 并行执行所有 tool_calls
│           ├─ 查找工具 → tool.execute(**arguments)
│           ├─ 记录 ToolResult 到 logger
│           └─ 追加 Message(role="tool") 到消息历史
│
└─ 超出 max_steps → 返回错误信息

消息历史结构：

[system] → [user₁] → [assistant] → [tool]... → [user₂] → ...
                ↑
         摘要后替换为:
         [user₁] → [user: "Assistant Execution Summary..."] → [user₂]

四、LLM 客户端层
LLMClient (llm_wrapper.py)
  ├─ 检测 MiniMax 域名 → 自动拼接 /anthropic 或 /v1
  └─ 路由到具体实现:
       ├─ AnthropicClient   使用 anthropic SDK，支持 thinking blocks
       ├─ OpenAIClient      使用 httpx，兼容 OpenAI 格式
       └─ OllamaClient      本地 Ollama 服务

LLMClientBase (base.py) 定义抽象接口:
  ├─ generate(messages, tools) → LLMResponse
  ├─ _prepare_request()        # 构造请求体
  └─ _convert_messages()       # 内部格式 → API 格式转换

消息格式转换 (AnthropicClient._convert_messages):
  内部 Message.role="tool"
    → Anthropic: role="user", content=[{type:"tool_result"}]
  内部 Message.thinking
    → Anthropic: content=[{type:"thinking"}, {type:"text"}, {type:"tool_use"}]

重试机制 (retry.py):
  async_retry 装饰器
  └─ 指数退避: delay = initial_delay × base^attempt (上限 max_delay)
  └─ 超限抛出 RetryExhaustedError(last_exception, attempts)

五、工具系统层
所有工具继承 Tool 基类，实现统一接口：

Tool (base.py)
  ├─ name: str (property)
  ├─ description: str (property)
  ├─ parameters: dict (JSON Schema)
  ├─ execute(**kwargs) → ToolResult
  ├─ to_schema()        → Anthropic 格式
  └─ to_openai_schema() → OpenAI 格式

工具分类
| 类别 | 工具 | 特点 |
|------|------|------|
| 文件操作 | ReadTool / WriteTool / EditTool | 相对路径基于 workspace_dir；读文件带行号；tiktoken 限制 32K token |
| Shell | BashTool / BashOutputTool / BashKillTool | 支持前台/后台进程；跨平台(bash/PowerShell)；后台进程通过 bash_id 管理 |
| 记忆 | SessionNoteTool / RecallNoteTool | JSON 文件持久化到 .agent_memory.json；带时间戳和分类标签 |
| 技能 | GetSkillTool | 渐进式披露（Progressive Disclosure）：System Prompt 注入摘要，按需加载完整 SKILL.md |
| MCP | MCPTool | 动态代理 MCP Server 工具；支持 stdio/SSE/HTTP 三种传输；超时保护 |

MCP 加载流程
load_mcp_tools_async(config_path)
  ├─ 读取 mcp.json (fallback: mcp-example.json)
  ├─ 对每个 server:
  │   └─ MCPServerConnection.connect()
  │       ├─ asyncio.timeout(connect_timeout)
  │       ├─ stdio_client / sse_client / streamablehttp_client
  │       ├─ ClientSession.initialize()
  │       └─ ClientSession.list_tools() → 生成 MCPTool 列表
  └─ 全局注册 _mcp_connections (用于 cleanup)

Skills 加载流程
SkillLoader.discover_skills()
  └─ 递归扫描 skills/ 目录中的 SKILL.md
       ├─ 解析 YAML frontmatter (name, description, allowed-tools)
       ├─ _process_skill_paths() 将相对路径替换为绝对路径
       └─ 生成 Skill 对象存入 loaded_skills 字典

get_skills_metadata_prompt()
  └─ 只输出 name + description → 注入 System Prompt (Level 1)

GetSkillTool.execute(skill_name)
  └─ skill.to_prompt() → 返回完整 SKILL.md 内容 (Level 2)

六、配置系统
Config (config.py, Pydantic)
  ├─ LLMConfig:   api_key / api_base / model / provider / retry
  ├─ AgentConfig: max_steps / workspace_dir / system_prompt_path
  └─ ToolsConfig: enable_* 开关 / skills_dir / mcp_config_path / MCPConfig(超时)

配置文件优先级搜索 (find_config_file):
  1. ./mini_agent/config/{file}   (开发模式)
  2. ~/.mini-agent/config/{file}  (用户全局)
  3. <package>/config/{file}      (已安装包)

七、数据流向总览
用户输入
   │
   ▼
CLI/ACP 接入层
   │  Message(role="user", content=...)
   ▼
Agent.run()  ←──────────────────────────┐
   │  messages=[system, user, ...]       │
   ▼                                     │
LLMClient.generate()                     │
   │  LLMResponse(content, thinking,     │
   │              tool_calls, usage)     │
   ▼                                     │
有 tool_calls?                           │
  YES → tool.execute(**args)             │
         ToolResult(success, content)    │
         → Message(role="tool")──────────┘
  NO  → 返回最终回答

八、关键设计决策
| 设计 | 决策 | 原因 |
|------|------|------|
| 消息摘要策略 | 按用户对话轮次分段摘要 | 保留用户意图，压缩执行过程 |
| Token 计数 | tiktoken + API 上报双重检查 | 精确控制，避免超限 |
| 工具取消 | asyncio.Event + 独立线程 | 不阻塞事件循环，优雅退出 |
| MCP 连接 | AsyncExitStack 全局管理 | 确保进程退出时资源释放 |
| 技能加载 | 渐进式披露（Level 1/2） | System Prompt 不膨胀，按需加载 |
| LLM 路由 | 域名检测自动选择 API 格式 | 用户只需填 api_base，无需关心格式 |



## 以下是其核心架构设计的详细分析：

### 1. 核心架构概览 (High-Level Architecture)

Mini-Agent 采用了经典的 **Orchestrator-Worker** 模式，其中 `mini_agent/` 目录下的核心逻辑充当协调器（Orchestrator），负责调度、状态管理和决策循环，而外部模块（Tools, Skills, LLM Providers）作为可插拔组件（Workers）被动态加载。

*   **中心枢纽 (`mini_agent/`)**: 包含 `agent.py` (核心循环), `cli.py` (交互入口), `config.py` (配置管理)。
*   **能力层**: 分为通用工具 (`tools/`)、领域技能 (`skills/`) 和 上下文协议 (`mcp/`)。
*   **感知层 (`llm/`)**: 抽象了多种大语言模型接口，实现模型无关性。
*   **持久化与测试**: 完善的 `tests/` 目录和 `workspace/` 示例库，保障了系统的稳定性和可验证性。

### 2. 核心组件深度解析

#### A. 智能体核心引擎 (`agent.py`)
这是整个框架的“大脑”，设计重点在于**鲁棒性**和**上下文控制**。
*   **执行循环 (ReAct Loop)**:
    1.  **意图感知**: 接收用户指令，结合系统提示词。
    2.  **Token 管理**: 在调用 LLM 前进行上下文评估，若超出限制触发 `_summarize_messages`。
    3.  **决策生成**: LLM 输出 `thinking` (推理), `content` (回复), `tool_calls` (工具调用)。
    4.  **工具执行**: 安全地执行工具，捕获异常并转换为 LLM 可理解的错误反馈。
    5.  **状态更新**: 更新历史对话，准备下一轮迭代。
*   **上下文压缩策略**:
    *   **动态摘要**: 当 token 使用接近阈值（如默认 500）时，保留所有“用户意图”消息，仅对“智能体思考/工具调用/结果”进行摘要，确保任务连贯性。
    *   **滑动窗口保护**: 强制保留最近 10 条消息 + 系统提示词，防止上下文无限膨胀导致显存溢出或幻觉。
*   **可中断设计**: 支持优雅取消，执行中断时自动清理临时状态和历史记录。

#### B. 能力扩展生态 (Capability Ecosystem)

框架通过分层设计支持不同复杂度的任务：

| 层级 | 目录/模块 | 功能描述 | 设计哲学 |
| :--- | :--- | :--- | : |
| **通用工具** | `tools/` | Bash, File, Note, MCP 基础功能 | 原子化操作，提供基础 API 能力 |
| **领域技能** | `skills/` | 文档处理 (docx/pdf), 艺术生成，工作流自动化 | 组合型任务，将多个工具封装为单一接口 |
| **MCP 集成** | `mcp/` | Model Context Protocol 适配器 | 标准化外部工具接入，兼容主流生态 |

*   **技能 (Skills) 系统**: 这是该架构的一大亮点。`skills/` 目录下包含 `document-skills` (处理 Office 文件), `slack-gif-creator` (特定场景自动化) 等。这种设计允许用户通过配置即可加载复杂的业务能力，而无需深入代码底层。
*   **工具 (Tools) 系统**: 位于 `mini_agent/tools/`，提供了 Bash 执行、文件读写等基础原语。每个工具通常包含参数 Schema，便于 LLM 准确调用。

#### C. 模型抽象层 (`llm/`)

*   **多提供者支持**: 实现了 `OpenAI`, `Anthropic`, `Ollama`, `Qwen` 等客户端。
*   **统一接口**: 无论底层模型如何，上层 `Agent` 调用的接口是一致的，便于实验不同模型的效果。
*   **本地优先**: 明确支持本地部署 (Ollama/Qwen)，体现了对数据隐私和低延迟场景的关注。

### 3. 关键技术决策分析

1.  **Token 估算机制**:
    *   优先使用 `tiktoken` (encoding `cl100k_base`) 进行精确计数。
    *   容错设计：若库不可用，自动降级为字符计数，保证系统可用性。
2.  **错误处理与透明度**:
    *   工具执行异常被捕获并转化为 `tool_failure` 消息返回给 LLM，引导其自我修正。
    *   详细的日志系统记录每个工具的参数、结果和 Token 消耗，便于调试和审计。
3.  **模块化与解耦**:
    *   `skills` 与 `tools` 分离。Tool 是原子操作，Skill 是业务逻辑封装。这种分层降低了维护成本，使得新增业务场景只需编写 Skill 模块。
4.  **测试驱动 (TDD)**:
    *   `tests/` 目录覆盖了 Agent 逻辑、LLM 客户端、工具函数及 MCP 集成，表明该项目重视生产级代码质量。

### 4. 架构优势与潜在改进点

**优势**:
*   **高度可扩展**: 新增 LLM 或工具只需实现标准接口。
*   **上下文安全**: 主动的 Token 管理和摘要策略有效防止了长对话中的上下文丢失或溢出。
*   **生态兼容**: 支持 MCP 协议，可轻松接入大量现有的外部工具生态。
*   **文档完善**: 中英双语文档和详细的使用指南降低了上手门槛。

**潜在改进点 (基于架构推测)**:
*   **并发处理**: 当前描述侧重于单线程循环。对于需要并行调用多个工具的场景，可能需引入异步 (`asyncio`) 支持。
*   **Skill 发现机制**: 虽然展示了静态的 Skills 目录，若需支持动态注册（如从远程加载 Skill），可能需要引入插件注册中心。

### 5. 总结

`Mini-Agent` 是一个**工程化成熟度较高**的 AI Agent 框架。它不仅仅是一个简单的脚本合集，而是一个具备**状态管理**、**上下文压缩**、**模块化插件**和**多模型支持**的完整系统。其架构设计平衡了灵活性与稳定性，特别适合作为企业级应用或复杂自动化任务的基础底座。特别是其对 **Skills** 的封装和 **MCP** 的支持，显示出该框架在设计之初就考虑了长远的生态扩展性。