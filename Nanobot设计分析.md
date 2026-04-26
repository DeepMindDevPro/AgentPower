Nanobot 详细架构设计说明
一、整体架构总览
┌─────────────────────────────────────────────────────────────────────────────┐
│                           外部接入层 (Entry Points)                          │
│                                                                             │
│   CLI (nanobot agent)    WebUI (React+Vite)    OpenAI-Compatible API        │
│        ↓                       ↓  WebSocket          ↓ /v1/chat/completions │
│   AgentLoop.run()        BaseChannel.start()    api/server.py               │
└───────────────┬────────────────────┬─────────────────────┬──────────────────┘
                │                    │                      │
                ▼                    ▼                      │
┌─────────────────────────────────────────────────────────────────────────────┐
│                          消息总线层 (Message Bus)                             │
│                                                                             │
│   Channel → InboundMessage → MessageBus.inbound_queue                       │
│   AgentLoop ← OutboundMessage ← MessageBus.outbound_queue                   │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Agent 核心层 (Core)                               │
│                                                                             │
│   AgentLoop → CommandRouter → SessionManager → ContextBuilder               │
│       ↓                                              ↓                      │
│   AgentRunner ←──── ToolRegistry ────────→ LLMProvider                      │
│       ↓                                              ↓                      │
│   SubagentManager          AutoCompact/Consolidator/Dream                   │
└──────────┬──────────────────────────────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│             支撑服务层 (Infrastructure)                                       │
│   MemoryStore(GitStore)  CronService  HeartbeatService  SecurityNetwork      │
└─────────────────────────────────────────────────────────────────────────────┘

二、消息总线层（bus/）—— 解耦核心
设计模式：异步双向消息队列
MessageBus
 是整个系统的解耦核心，内部维护两条独立的 asyncio.Queue：

Channel → bus.publish_inbound(InboundMessage)
                ↓
        bus.inbound (asyncio.Queue)
                ↓ AgentLoop.consume_inbound()
        AgentLoop 处理 → bus.publish_outbound(OutboundMessage)
                ↓
        bus.outbound (asyncio.Queue)
                ↓ ChannelManager 路由分发到对应 Channel

关键数据结构：

InboundMessage
 — 包含 channel、sender_id、chat_id、content、media、metadata、session_key_override
OutboundMessage
 — 包含 channel、chat_id、content、reply_to、media、buttons
session_key 计算规则：

默认：f"{channel}:{chat_id}"
统一会话模式（unified_session=True）：全部路由至 "unified:default"
线程会话（如 Slack thread）：由 Channel 注入 session_key_override
三、渠道层（channels/）—— 统一抽象
3.1 BaseChannel 接口契约
BaseChannel
 定义了三个核心抽象方法：

| 方法 | 职责 |
|------|------|
| start() | 长驻异步任务，连接平台并监听消息 |
| stop() | 清理资源，断开连接 |
| send(OutboundMessage) | 发送响应到平台 |
| send_delta(chat_id, delta) | 可选：发送流式分块（子类覆盖实现） |

流式输出判断逻辑：

@property
def supports_streaming(self) -> bool:
    cfg_streaming = getattr(self.config, "streaming", False)
    return bool(cfg_streaming) and type(self).send_delta is not BaseChannel.send_delta

只有 config 开启 streaming 且子类真正覆写了 send_delta 才启用流式。

3.2 权限控制
内置 allow_from 白名单机制，每条消息进入前经过 
is_allowed(sender_id)
 检查：

空列表 → 全部拒绝（防止裸配置上线被滥用）
"*" → 全部允许
其他 → 精确 ID 比对
3.3 ChannelManager 路由策略
ChannelManager
 通过 pkgutil 扫描 + entry_points 支持插件渠道，并在出站时通过指数退避重试（1s → 2s → 4s）确保消息可靠投递。支持 14+ 渠道：Telegram、Discord、Slack、Feishu、DingTalk、WeChat、WeCom、Matrix、Email、QQ、Mochat、WhatsApp、WebSocket、MS Teams。

四、Agent 核心层 —— 三层处理架构
4.1 AgentLoop —— 主控层
AgentLoop
 负责端到端请求编排：

并发控制机制：

环境变量 NANOBOT_MAX_CONCURRENT_REQUESTS（默认 3）控制并发上限，通过 asyncio.Semaphore 实现
每个 session_key 独立拥有一把 asyncio.Lock，防止同 session 并发写
活跃任务字典 _active_tasks: dict[str, list[asyncio.Task]] 支持 /stop 精确取消
中途注入机制（Mid-turn Injection）：

当某 session 正在处理请求时，新消息不创建新任务，而是进入 _pending_queues[session_key]
AgentRunner 在每轮工具执行后通过 injection_callback 排水（drain）
上限 _MAX_INJECTIONS_PER_TURN=3，超出部分记录警告而非静默丢弃
MCP 连接策略： 懒加载 + 自动重试。首次收到消息时触发 _connect_mcp()，失败后下次消息再试，通过 AsyncExitStack 管理多 server 生命周期。

4.2 AgentRunner —— 执行层
AgentRunner
 是纯粹的工具执行循环，不含任何产品层关注点：

迭代循环中的上下文治理（每轮执行前）：

messages
  → _drop_orphan_tool_results()      # 清除孤立 tool result
  → _backfill_missing_tool_results() # 补全丢失的 tool result
  → _microcompact()                  # 微压缩：截断超长工具结果
  → _apply_tool_result_budget()      # 按字符预算截断结果
  → _snip_history()                  # 按 token 预算剪裁历史
  → _drop_orphan_tool_results()      # 再次清理（snip 可能引入新孤立）
  → _backfill_missing_tool_results() # 再次补全

鲁棒性处理策略：

| 场景 | 策略 |
|------|------|
| 空响应 | 最多 _MAX_EMPTY_RETRIES=2 次重试，然后强制 finalization |
| 输出截断（finish_reason=length） | 最多 _MAX_LENGTH_RECOVERIES=3 次续写 |
| 工具调用错误 | 记录错误后尝试排水注入，可继续执行 |
| ask_user 中断 | 截断后续 tool_calls，将 question 作为最终响应 |

Checkpoint 机制： 每个关键阶段（awaiting_tools / tools_completed / final_response）都发出 checkpoint，支持断点恢复。

4.3 ContextBuilder —— 上下文组装层
ContextBuilder
 按固定优先级顺序拼接 System Prompt：

1. identity.md（工作目录、运行时环境、平台策略）
2. Bootstrap Files（AGENTS.md、SOUL.md、USER.md、TOOLS.md）
3. MEMORY.md（长期记忆，检测到模板内容则跳过）
4. Always-Skills（始终注入的技能）
5. Available Skills 摘要
6. Recent History（dream_cursor 之后的条目，最多 50 条 / 32000 字符）

消息构建规则： Runtime Context（当前时间、Channel、Chat ID、会话摘要）与用户内容合并到同一 user 消息，避免触发部分 Provider 的连续同角色消息限制。

五、工具体系（agent/tools/）—— 统一抽象
5.1 Tool 基类设计
Tool
 抽象基类强制实现三个属性和一个方法：

@property name: str        # 工具名（LLM function call 名）
@property description: str # 工具描述
@property parameters: dict # JSON Schema 参数定义
async execute(**kwargs)    # 实际执行逻辑

附加并发属性：

read_only → 无副作用，可并行
concurrency_safe → read_only and not exclusive
exclusive → 独占，必须单独运行
@tool_parameters 装饰器： 注入静态 parameters 属性并从抽象方法集合中移除，避免子类重复实现 property 模板代码。

5.2 参数校验流程
LLM 返回 tool_call
    → ToolRegistry.prepare_call(name, params)
        → tool.cast_params(params)   # 类型强制转换（"1" → 1 等）
        → tool.validate_params()     # JSON Schema 校验
    → tool.execute(**cast_params)
    → 结果加 "[Analyze the error above and try a different approach.]" hint

5.3 ToolRegistry 缓存策略
ToolRegistry
 维护 _cached_definitions，每次 register/unregister 后失效。获取定义时按固定顺序排序：内置工具字母序 在前，MCP 工具（mcp_ 前缀）字母序 在后，保证 prompt cache 的稳定性。

5.4 MCP 集成
mcp.py
 将外部 MCP 服务的工具包装为原生 Tool 实例：

支持 stdio / SSE / streamableHttp 三种传输协议
Windows 兼容：自动包装 npx/npm 等 shell 启动器为 cmd /d /c ...
瞬态错误自动单次重试（ClosedResourceError、BrokenPipeError 等）
每个 server 独立的 AsyncExitStack 生命周期管理
六、LLM Provider 层（providers/）—— 可扩展后端
6.1 LLMProvider 抽象基类
LLMProvider
 内置精细的重试策略：

重试分类逻辑：

HTTP 429
  ├── 含 insufficient_quota / billing_hard_limit → 不重试（账单问题）
  ├── 含 rate_limit_exceeded / overloaded → 重试（临时限流）
  └── 其他 → 按状态码判断

standard 模式：指数退避 1s/2s/4s，最多 3 次
persistent 模式：最大间隔 60s，相同错误最多 10 次，超过停止

6.2 Provider 注册表（registry.py）
ProviderSpec 数据类驱动整个 Provider 生态，新增 Provider 只需两步：

在 
PROVIDERS
 列表添加一条 ProviderSpec
在 
ProvidersConfig
 添加对应字段
已支持 25+ Provider：Anthropic、OpenAI、Azure OpenAI、OpenRouter、DeepSeek、Groq、Gemini、Moonshot、MiniMax、Mistral、VolcEngine、SiliconFlow、ZhipuAI、DashScope、Ollama、vLLM、LM Studio、GitHub Copilot、OpenAI Codex 等。

模型匹配优先级：

强制指定 provider 字段 → 直接使用
API Key 前缀检测（sk-or- → OpenRouter）
API Base URL 关键词检测
model 名称关键词匹配（最长前缀优先）
6.3 流式处理（Streaming）
OpenAICompatProvider 支持两种 API 路径：

chat.completions（标准）→ delta 累积模式
responses（OpenAI Responses API）→ consume_sdk_stream 解析
Anthropic Provider 支持 extended thinking 流式块，通过 thinking_blocks 字段透传给 runner。

七、记忆系统（agent/memory.py）—— 三层记忆架构
用户对话
    ↓ 实时存储
Session（JSONL in sessions/）      ← 短期工作记忆（完整对话历史）
    ↓ 超出 token 预算时
Consolidator → history.jsonl      ← 中期摘要记忆（LLM 压缩的对话摘要）
    ↓ 定期（默认每 2 小时）
Dream → MEMORY.md                 ← 长期知识记忆（Git 版本化的事实）

7.1 MemoryStore —— 纯文件 I/O 层
MemoryStore
 管理的文件结构：

workspace/
  SOUL.md           # Agent 身份/性格定义
  USER.md           # 用户偏好/信息
  memory/
    MEMORY.md       # 长期记忆事实
    history.jsonl   # JSONL 格式的对话摘要历史（append-only）
    .cursor         # Consolidator 游标（已摘要位置）
    .dream_cursor   # Dream 游标（已处理位置）
  sessions/
    telegram_12345.jsonl  # 按 channel:chat_id 隔离的会话文件

Git 版本管理： 使用 dulwich 对 SOUL.md、USER.md、memory/MEMORY.md 进行 Git 追踪，Dream 写入时自动 commit，支持 git blame 行龄标注（← Nd 后缀），帮助 LLM 识别陈旧信息。

7.2 Consolidator —— 基于 Token 预算的压缩
当 估算 prompt tokens >= context_window_tokens  时触发：

估算当前 prompt tokens
    ↓ 超出预算
pick_consolidation_boundary()  # 找合法的 user-turn 边界
    ↓
archive(old_messages)          # LLM 摘要旧消息 → history.jsonl
    ↓
更新 session.last_consolidated
    ↓ 最多循环 5 轮直到 tokens <= budget/2

失败降级：LLM 摘要失败时 raw_archive()，原始格式写入 history.jsonl，记录警告。

7.3 Dream —— 两阶段自主记忆蒸馏
Phase 1（读取+评估）： 读取 history.jsonl 中未处理的条目（since dream_cursor），附带 git-blame 行龄标注，交给 LLM 决定需要更新哪些 MEMORY.md 内容。

Phase 2（执行写入）： 使用完整 AgentRunner（含 FileSystem 工具），让 LLM 实际读写 MEMORY.md，最多 15 次工具迭代，完成后更新 dream_cursor。

7.4 AutoCompact —— 空闲会话自动归档
AutoCompact
 定期扫描所有 session，对超过 session_ttl_minutes 未活跃的会话：

保留最近 8 条合法消息（retain_recent_legal_suffix）
对其余消息调用 Consolidator.archive() 生成摘要
摘要缓存在内存 _summaries 字典，下次对话时注入为 session_summary
八、Session 管理（session/）
SessionManager
 采用 write-through 缓存：

内存 _cache 热路径访问
磁盘 JSONL 持久化（sessions/{safe_filename}.jsonl）
自动迁移旧版 ~/.nanobot/sessions/ 路径
JSONL 文件格式：

{"_type": "metadata", "created_at": "...", "updated_at": "...", "last_consolidated": 42}
{"role": "user", "content": "...", "timestamp": "..."}
{"role": "assistant", "content": "...", "tool_calls": [...]}
{"role": "tool", "tool_call_id": "...", "content": "..."}

合法边界对齐： get_history() 从 last_consolidated 开始，对齐到 user-turn 边界，清除前端孤立 tool result，保证 LLM 始终收到结构完整的对话序列。

九、定时任务（cron/）+ 心跳（heartbeat/）
CronService 调度三模式
| 模式 | 描述 |
|------|------|
| at | 单次定时，指定绝对毫秒时间戳 |
| every | 周期循环，每隔 N 毫秒 |
| cron | 标准 cron 表达式，支持 IANA 时区 |

使用 filelock 实现跨进程安全，任务状态持久化到 JSON 文件，支持 nanobot gateway 和 nanobot agent 多实例共享同一任务队列。

HeartbeatService 两阶段决策
Phase 1（虚拟工具调用）： 读取 HEARTBEAT.md，用 heartbeat 虚拟工具（结构化输出）让 LLM 判断是否有待执行任务（skip / run），避免自由文本解析的不可靠性。

Phase 2（按需触发）： 只有 Phase 1 返回 run 才通过 on_execute 回调触发完整 Agent 循环，on_notify 回调将结果推送到 Channel。

十、安全设计（security/）
SecurityNetwork
 实现多层 SSRF 防护：

阻断的内网地址段：

0.0.0.0/8, 10.0.0.0/8, 100.64.0.0/10（运营商 NAT）
127.0.0.0/8, 169.254.0.0/16（云元数据），172.16.0.0/12
192.168.0.0/16, ::1/128, fc00::/7, fe80::/10

两阶段校验：

validate_url_target() — 请求前：DNS 解析后对每个 IP 地址检查
validate_resolved_url() — 重定向后：防止通过公网跳板绕过检测
白名单支持： ssrf_whitelist 配置 CIDR 范围（如 Tailscale 100.64.0.0/10）可豁免阻断。

Shell Exec 工具中 contains_internal_url() 对命令行内嵌 URL 做额外扫描，防止通过 shell 命令间接发起内网请求。

十一、命令路由（command/）
CommandRouter
 实现四级命令分发，按顺序匹配：

1. Priority（优先）  → /stop、/restart（在加锁前执行）
2. Exact（精确）    → /status、/clear 等
3. Prefix（前缀）   → 最长前缀优先（如 "/team " > "/t "）
4. Interceptors     → fallback 谓词（如 team-mode 拦截）

内置命令：

| 命令 | 实现 |
|------|------|
| /stop | 取消当前 session 所有任务和子 Agent |
| /restart | os.execv() 原地重启，保留 restart notice 跨进程传递到 Channel |
| /status | 返回版本、模型、运行时间、Token 用量、活跃任务数、搜索配额 |
| /clear | 清空 session 历史 |
| /model <name> | 运行时切换模型 |

十二、子 Agent 系统（agent/subagent.py）
SubagentManager
 管理后台并行子 Agent：

每个 subagent 是独立 asyncio.Task，共享父 AgentRunner 和 LLMProvider
_SubagentHook 实时更新 SubagentStatus（phase / iteration / tool_events / usage）
_session_tasks 按 session 追踪，支持 /stop 精确取消所属 session 的所有 subagent
subagent 完成后通过 MessageBus.publish_inbound() 以普通消息形式注入父 session，被中途注入机制接管
十三、关键设计决策总结
| 决策 | 理由 |
|------|------|
| asyncio.Queue 双向总线 | Channel 与 Agent 完全解耦，可独立扩展和测试 |
| AgentRunner 与 AgentLoop 分离 | Runner 无产品层关注，可被 Subagent / Dream 复用 |
| CompositeHook 扇出 | 外部 SDK 可无侵入式注入，单个 Hook 异常不影响主流程 |
| Context 治理（每轮前 6 步修复） | 防御 LLM 幻觉工具调用 / 历史截断引入的消息结构破损 |
| 注册表驱动 ProviderSpec | 新增 Provider 零修改路由逻辑，配置、UI、环境变量全自动 |
| Git 版本化 MEMORY.md | 行龄标注帮助 LLM 识别过期知识，支持回滚 |
| session_ttl_minutes + AutoCompact | 长期运行不积累无效上下文，降低 token 成本 |
| SSRF 双阶段检查 + shell URL 扫描 | 防止通过 Agent 工具发起内网探测攻击 |
| JSONL 格式持久化 | append-only 写入，崩溃安全，支持逐行流式读取 |
| Checkpoint 机制 | 支持中断后断点恢复，长任务不丢失进度 |

现在我已经掌握了足够深度的信息，可以输出详细设计说明。

Nanobot 详细架构设计说明
一、系统总体架构
┌──────────────────────────────────────────────────────────────────────────┐
│                           外部接入层                                      │
│  ┌──────────┐  ┌──────────────────┐  ┌────────────────┐  ┌────────────┐ │
│  │   CLI    │  │  14+ Channels     │  │ OpenAI API /   │  │  WebUI     │ │
│  │(commands)│  │(TG/Discord/wx...) │  │  SSE Stream    │  │(React+WS)  │ │
│  └────┬─────┘  └────────┬─────────┘  └───────┬────────┘  └─────┬──────┘ │
└───────┼─────────────────┼────────────────────┼─────────────────┼────────┘
        │                 │                    │                 │
┌───────▼─────────────────▼────────────────────▼─────────────────▼────────┐
│                        MessageBus（异步消息总线）                         │
│          InboundQueue ←─────────────────────────→ OutboundQueue          │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼───────────────────────────────────────┐
│                         AgentLoop（主控循环）                             │
│  ┌────────────┐  ┌────────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │CommandRouter│  │ContextBuilder  │  │SessionManager│  │SubagentMgr   │ │
│  │(优先/精确  │  │(Prompt组装)    │  │(JSONL持久化) │  │(asyncio并行) │ │
│  │ /前缀路由) │  └────────────────┘  └──────────────┘  └──────────────┘ │
│  └────────────┘                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                      AgentRunner（工具执行核心）                     │ │
│  │  context-governance → LLM调用 → 工具调度 → 结果收集 → checkpoint   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────┬───────────────────────────────────────┘
              ┌─────────────────────┼──────────────────────┐
              ▼                     ▼                        ▼
   ┌──────────────────┐  ┌──────────────────┐   ┌──────────────────────┐
   │   LLMProvider    │  │   ToolRegistry   │   │    Memory System     │
   │ (OpenAI/Anthro-  │  │  Tool抽象/注册/  │   │ MemoryStore+         │
   │  pic/Azure/...)  │  │  cast/validate/  │   │ Consolidator + Dream │
   └──────────────────┘  │  execute         │   └──────────────────────┘
                         └──────────────────┘

二、消息总线层（bus/）
设计目标
彻底解耦 Channel 和 Agent 核心，使两侧可以独立演化和测试。

关键数据结构
InboundMessage
 — 入站消息：

channel / sender_id / chat_id — 路由三元组
content + media — 文本与多媒体内容
session_key_override — 支持线程级 session 隔离（如 Slack thread）
session_key 属性自动合成：channel:chat_id
OutboundMessage
 — 出站消息：

buttons 字段支持平台交互按钮（Telegram InlineKeyboard 等）
MessageBus
 — 双队列总线：

inbound: asyncio.Queue — Channel → Agent
outbound: asyncio.Queue — Agent → Channel
轻量级、无持久化，重启丢失队列中未处理消息属预期行为
三、渠道层（channels/）
抽象基类设计（BaseChannel）
BaseChannel
 定义了渠道实现必须遵守的契约：

| 抽象方法 | 职责 |
|----------|------|
| start() | 长期运行，连接平台并监听消息 |
| stop() | 优雅关闭，释放资源 |
| send(msg) | 发送完整消息，失败须抛出异常（供 manager 重试）|
| send_delta(chat_id, delta) | 可选覆写，实现流式输出 |

权限模型：
is_allowed()
 检查 allow_from 白名单，空列表拒绝一切，"*" 放行一切。

流式协议：supports_streaming 属性联合检查配置项 streaming: true 和子类是否实际覆写了 send_delta，双重守卫避免配置与实现不一致。

ChannelManager 工作流
_init_channels()
  └─ pkgutil scan + entry_points 插件扫描
       └─ 对每个 enabled channel 实例化并注入 bus

start() → asyncio.gather(每个 channel.start())
        → _dispatch_outbound_loop()  ← 持续消费 bus.outbound
             └─ 重试策略: delays=(1,2,4)s, 最多 send_max_retries 次

四、Agent 主控循环（AgentLoop）
初始化时注册的组件
AgentLoop
├── ContextBuilder      # 系统提示组装
├── SessionManager      # 会话 JSONL 持久化
├── ToolRegistry        # 工具动态注册
├── AgentRunner         # 工具执行循环
├── SubagentManager     # 后台子 Agent
├── Consolidator        # Token 预算触发整合
├── AutoCompact         # 空闲会话自动压缩
├── Dream               # 定期长期记忆蒸馏
└── CommandRouter       # Slash 命令路由

消息处理主流程
run() 主循环
│
├─ [Priority 命令检测] → dispatch_priority (无锁，立即响应 /stop /restart)
│
├─ [获取 Session 锁]  ← 防止同一会话并发写入
│
├─ [可分发命令检测]  → dispatch (精确/前缀/拦截器)
│
├─ [并发门控]  NANOBOT_MAX_CONCURRENT_REQUESTS=3
│
├─ AutoCompact.prepare_session()  ← 检查 TTL 过期，恢复压缩摘要
│
├─ Consolidator.maybe_consolidate_by_tokens()  ← 超出 context 预算时整合
│
├─ ContextBuilder.build_messages()  ← 组装完整 Prompt
│
└─ AgentRunner.run()  ← 核心执行循环

中途注入机制（Mid-turn Injection）
当会话有活跃任务时，新消息不创建新任务，而是放入 _pending_queues[session_key]。AgentRunner 每次工具执行后调用 injection_callback 消费队列，最多注入 _MAX_INJECTIONS_PER_TURN=3 条，超限丢弃并记日志。

五、AgentRunner（工具执行循环核心）
每次迭代的 Context Governance 流水线
# 每次 LLM 调用前，对 messages 执行以下操作（不修改持久化数组）：
messages_for_model = _drop_orphan_tool_results(messages)    # 删孤立 tool 结果
messages_for_model = _backfill_missing_tool_results(...)    # 补充缺失占位符
messages_for_model = _microcompact(...)                     # 压缩可压缩工具结果
messages_for_model = _apply_tool_result_budget(...)         # 截断超长工具输出
messages_for_model = _snip_history(...)                     # token 超限时裁剪历史
messages_for_model = _drop_orphan_tool_results(...)         # 裁剪后再次清理孤儿
messages_for_model = _backfill_missing_tool_results(...)

关键常量：

| 常量 | 值 | 含义 |
|------|----|------|
| _MAX_EMPTY_RETRIES | 2 | 空响应最大重试次数 |
| _MAX_LENGTH_RECOVERIES | 3 | finish_reason=length 续写次数 |
| _MAX_INJECTIONS_PER_TURN | 3 | 单轮最大注入条数 |
| _MAX_INJECTION_CYCLES | 5 | 注入循环上限 |
| _COMPACTABLE_TOOLS | read_file/exec/grep/glob/web_* | 可被 microcompact 压缩的工具 |

Checkpoint 机制
AgentRunner 在三个关键点发出 checkpoint 事件（通过 checkpoint_callback）：

awaiting_tools — LLM 决定调用工具之后
tools_completed — 所有工具执行完毕
final_response — 输出最终响应
这使长任务崩溃后可从最近 checkpoint 恢复，而非从头重跑。

六、Tool 体系设计（agent/tools/）
工具抽象层次
Schema（JSON Schema 验证）
  └─ Tool（抽象基类）
       ├─ name / description / parameters (抽象属性)
       ├─ read_only / concurrency_safe / exclusive (并发控制)
       ├─ cast_params()    # schema 驱动类型转换 (str→int, "true"→True 等)
       ├─ validate_params() # 完整 JSON Schema 校验
       └─ execute(**kwargs) # 异步执行

@tool_parameters 装饰器：注入 parameters 属性，避免每个工具手写 @property，schema 在类级别深拷贝存储，每次访问返回新副本防止意外修改。

ToolRegistry 工作机制
稳定排序：内置工具字母序在前，MCP 工具（mcp_ 前缀）字母序在后，结果缓存直至下次 register/unregister，保证 prompt cache 命中率最大化
prepare_call 三步：resolve → cast_params → validate_params，错误信息直接返回给 LLM 带上 [Analyze the error above and try a different approach.] 提示
MCP 工具集成（tools/mcp.py）
支持 stdio、sse、streamableHttp 三种传输
Windows 平台自动用 cmd /d /c 包装 Node.js 命令行（npx/yarn 等）
瞬态连接错误自动重试（ClosedResourceError / BrokenPipeError 等）
每个 MCP Server 独立 AsyncExitStack，故障隔离
七、LLM Provider 层（providers/）
统一重试策略（LLMProvider._run_with_retry）
standard 模式:  固定延迟 (1s, 2s, 4s)，最多 3 次
persistent 模式: 指数退避，最大 60s，连续相同错误超 10 次放弃

区分不可重试 429（insufficient_quota / billing_hard_limit_reached 等）和可重试 429（rate_limit_exceeded / overloaded 等），避免账单耗尽时的无限循环。

响应规范化
LLMResponse
 统一承载：

content / tool_calls / finish_reason
reasoning_content — DeepSeek-R1 / Kimi / MiMo 的思考链
thinking_blocks — Anthropic extended thinking
retry_after — provider 指定的等待时长
should_execute_tools 属性：仅当 finish_reason in ("tool_calls", "stop") 时执行工具，阻断网关注入的幻觉工具调用
OpenAI Compat Provider 特性
可选集成 Langfuse 可观测性（检测到 LANGFUSE_SECRET_KEY 时自动使用 langfuse.openai.AsyncOpenAI）
Kimi 思考模型特殊处理（kimi-k2.5 / kimi-k2.6）
OpenRouter 默认注入 HTTP-Referer 标头
八、记忆系统三层架构
┌─────────────────────────────────────────────────────────┐
│            第一层：会话记忆（Session）                   │
│  channel:chat_id → sessions/{key}.jsonl                 │
│  • 按需加载/写入，内存 LRU 缓存                          │
│  • get_history() 自动对齐 tool-call 边界，避免孤儿消息   │
└─────────────────────────────────────────────────────────┘
            ↓ Token 超出 context_window 时触发
┌─────────────────────────────────────────────────────────┐
│            第二层：整合记忆（Consolidator）              │
│  • tiktoken 精确 token 计数（cl100k_base 编码）          │
│  • 按用户轮次边界裁剪，LLM 摘要 → history.jsonl          │
│  • 摘要失败降级为 raw-dump，永不丢失数据                 │
│  • WeakValueDictionary 管理 per-session asyncio.Lock     │
└─────────────────────────────────────────────────────────┘
            ↓ 定时（默认每2小时）Dream 调度触发
┌─────────────────────────────────────────────────────────┐
│            第三层：长期记忆（Dream + MemoryStore）       │
│  两阶段处理：                                            │
│  Phase 1: 读取未处理 history 条目（since dream_cursor）  │
│            + git-blame 行龄注解 → LLM 决策什么要记住    │
│  Phase 2: 工具调用执行（write_file 更新 MEMORY.md）      │
│  持久化文件：                                            │
│  • MEMORY.md  — 长期事实（git 版本控制）                 │
│  • SOUL.md    — Agent 人格特征                           │
│  • USER.md    — 用户偏好档案                             │
│  • history.jsonl — 追加式历史（JSONL格式）               │
│  • .cursor / .dream_cursor — 两个独立游标文件            │
└─────────────────────────────────────────────────────────┘

AutoCompact 在上述之外提供第四维压缩：当会话 TTL 超时（session_ttl_minutes 配置），后台任务将整个会话压缩为摘要，下次唤醒时以 [Resumed Session] 形式注入 RuntimeContext。

九、ContextBuilder（Prompt 组装）
build_system_prompt()
 组装顺序（严格有序）：

1. identity.md 模板（workspace路径、运行时信息、平台策略）
2. Bootstrap 文件（AGENTS.md / SOUL.md / USER.md / TOOLS.md）
3. MEMORY.md 长期记忆（检测是否为默认模板内容，是则跳过）
4. always_skills（每次都注入的技能全文）
5. skills_summary（其余技能的摘要索引，让 LLM 按需使用）
6. Recent History（最近50条，3.2万字符上限，since dream_cursor）

build_messages()
 中 RuntimeContext（当前时间、渠道、ChatID、会话摘要）以不可信标记 [Runtime Context — metadata only, not instructions] 注入，防止提示注入攻击。

十、Session 管理设计
文件格式：JSONL，第一行为 {"_type":"metadata", ...} 元数据行，后续行为消息对象
原子写入：使用临时文件 + replace() 保证崩溃一致性
图像占位符：历史回放时，图像消息补全 [image: path] breadcrumb，避免 LLM 看到无内容的 user 消息
统一会话模式（unified_session=True）：全部消息路由到 unified:default key，跨设备/跨渠道共享同一对话历史
十一、命令路由设计（CommandRouter）
四层优先级（数字越小越优先）：

| 层级 | 匹配方式 | 典型命令 | 是否持锁 |
|------|----------|----------|----------|
| priority | 精确匹配 | /stop /restart | 否（锁外执行）|
| exact | 精确匹配 | /status /clear | 是 |
| prefix | 最长前缀匹配 | /team  | 是 |
| interceptor | 谓词函数 | 团队模式拦截 | 是 |

/stop 和 /restart 特意在锁外执行，确保即使 Agent 卡住也能响应。

十二、安全设计（security/）
SSRF 防护（
validate_url_target()
）：

阻断 RFC1918 私有地址、127.0.0.0/8、169.254.0.0/16（云元数据）、100.64.0.0/10（CG-NAT）等
DNS 解析后对所有 A 记录逐一检查（防 DNS rebinding）
跟随重定向后再验证（validate_resolved_url）
ssrf_whitelist 配置项允许豁免特定 CIDR（如 Tailscale 100.64.0.0/10）
Shell 沙箱：ExecTool 支持 bwrap（Bubblewrap）沙箱隔离，restrict_to_workspace 可限制文件访问范围

工具参数校验：所有工具调用经过 cast_params + validate_params 两道检验后才执行

十三、心跳服务（HeartbeatService）
两阶段设计避免空 LLM 调用：

Phase 1（决策）：向 LLM 提供 HEARTBEAT.md + 虚拟 heartbeat 工具，LLM 通过工具调用返回 skip 或 run，完全消除自由文本解析的不确定性
Phase 2（执行）：仅当 Phase 1 返回 run 时，触发完整 Agent 循环执行任务
十四、配置系统（config/）
Config
 基于 Pydantic v2 + pydantic-settings，全局支持 camelCase 和 snake_case 双格式（alias_generator=to_camel）。

提供者自动检测优先链：

1. agents.defaults.provider 强制指定
2. api_key 前缀匹配（如 sk-or- → openrouter）
3. api_base URL 关键字匹配
4. model 名称关键字匹配（由 ProviderSpec.keywords 驱动）

已内置支持 25+ 提供商（OpenAI、Anthropic、DeepSeek、Kimi、Gemini、Ollama、vLLM、LM Studio、VolcEngine、SiliconFlow 等），扩展只需在 PROVIDERS 列表添加一条 
ProviderSpec
，无需修改任何路由代码。

现在我已经收集了足够详细的信息，可以输出完整的详细设计说明了。

Nanobot 详细架构设计说明
一、整体架构图
┌─────────────────────────────────────────────────────────────────────────┐
│                          外部接入层 (Entry Layer)                        │
│                                                                         │
│  ┌──────────┐  ┌──────────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │  CLI     │  │ OpenAI-compat API│  │  Chat Apps   │  │  WebUI     │  │
│  │(commands)│  │ /v1/chat/compl.  │  │  14+ 渠道   │  │ React+Vite │  │
│  └────┬─────┘  └────────┬─────────┘  └──────┬───────┘  └─────┬──────┘  │
│       │                 │                    │                │          │
└───────┼─────────────────┼────────────────────┼────────────────┼──────────┘
        │                 │                    │                │
        ▼                 ▼                    ▼                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       消息总线层 (Message Bus Layer)                      │
│                                                                         │
│     InboundMessage ──►  MessageBus(asyncio.Queue)  ──► OutboundMessage  │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Agent 核心层 (Agent Core)                         │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐        │
│  │                       AgentLoop                             │        │
│  │   ┌──────────────┐  ┌─────────────┐  ┌────────────────┐   │        │
│  │   │ContextBuilder│  │CommandRouter│  │  AutoCompact   │   │        │
│  │   └──────────────┘  └─────────────┘  └────────────────┘   │        │
│  │   ┌──────────────┐  ┌─────────────┐  ┌────────────────┐   │        │
│  │   │  Consolidator│  │   Dream     │  │SubagentManager │   │        │
│  │   └──────────────┘  └─────────────┘  └────────────────┘   │        │
│  └──────────────────────────┬──────────────────────────────────┘        │
│                             │                                            │
│  ┌──────────────────────────▼──────────────────────────────────┐        │
│  │                      AgentRunner                            │        │
│  │   LLM迭代 → 工具执行 → 注入检测 → 上下文治理 → 检查点      │        │
│  └──────────────────────────┬──────────────────────────────────┘        │
└──────────────────────────────┼──────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
┌─────────────────┐  ┌─────────────────┐  ┌──────────────────────┐
│  LLM Provider层 │  │  ToolRegistry   │  │  Session/Memory层    │
│                 │  │                 │  │                      │
│ OpenAICompat    │  │ FileSystem      │  │ SessionManager       │
│ Anthropic       │  │ Shell/Exec      │  │ MemoryStore          │
│ Azure OpenAI    │  │ Web/Search      │  │ GitStore(dulwich)    │
│ GitHub Copilot  │  │ MCP Client      │  │ Consolidator         │
│ OpenAI Codex    │  │ Cron/Schedule   │  │ Dream(两阶段)        │
└─────────────────┘  └─────────────────┘  └──────────────────────┘

二、消息总线（bus/）— 核心解耦机制
设计目标
将所有外部渠道与 Agent 核心完全隔离，Channel 无需感知 Agent 实现细节。

数据结构
InboundMessage（入站）：

channel / sender_id / chat_id — 路由三要素
content + media: list[str] — 文本 + 多媒体 URL
metadata — 渠道私有扩展字段（如 _wants_stream）
session_key_override — 线程级会话绑定（如 Slack thread）
OutboundMessage（出站）：

buttons: list[list[str]] — 跨渠道统一按钮抽象
reply_to — 消息回复关联
关键机制：session_key
InboundMessage.session_key
 属性通过 session_key_override or f"{channel}:{chat_id}" 计算，实现了线程级会话隔离（如 Slack Thread、Discord Thread）的统一抽象。

三、Agent 核心层
3.1 AgentLoop — 主控引擎
AgentLoop
 是系统的中枢，主要职责：

初始化阶段（__init__）：

实例化 ContextBuilder、SessionManager、ToolRegistry、AgentRunner、SubagentManager
创建 Consolidator、AutoCompact、Dream — 三级记忆体系
调用 _register_default_tools() 注册全套内置工具
初始化 CommandRouter 并注册内置斜线命令
并发控制：

# 每 session 独立锁，防止同一会话并发处理
self._session_locks: dict[str, asyncio.Lock]

# 全局并发门（默认3，可通过NANOBOT_MAX_CONCURRENT_REQUESTS配置）
self._concurrency_gate: asyncio.Semaphore | None

# 活跃任务追踪（session_key → [asyncio.Task]）
self._active_tasks: dict[str, list[asyncio.Task]]

# 中途消息注入队列（session正在处理时，新消息进队列而非创建新任务）
self._pending_queues: dict[str, asyncio.Queue]

MCP 连接策略：懒加载 + 首次消息时建立，连接失败不中断主流程（下次重试）。

命令路由四级架构（
CommandRouter
）：

priority：优先队列外处理（如 /stop、/restart），不占会话锁
exact：精确匹配（如 /status、/clear）
prefix：最长前缀优先（如 /team ...）
interceptors：兜底谓词（如 Team 模式拦截）
3.2 AgentRunner — 工具执行循环
AgentRunner
 是无产品关注点的纯 LLM 循环引擎：

单次迭代流程：

上下文治理 → before_iteration hook → 调用 LLM →
  ├─ 有工具调用 → before_execute_tools → 并行/串行执行工具 → 
  │              emit_checkpoint → drain_injections → after_iteration → continue
  └─ 无工具调用 → finalize_content → 处理空响应/截断恢复 →
                  drain_injections → after_iteration → break

上下文治理（每轮执行前）：

messages_for_model = self._drop_orphan_tool_results(messages)    # 清除孤立 tool results
messages_for_model = self._backfill_missing_tool_results(...)    # 补填缺失结果
messages_for_model = self._microcompact(...)                     # 微型压缩大型工具结果
messages_for_model = self._apply_tool_result_budget(...)         # 按字符预算截断
messages_for_model = self._snip_history(...)                     # 按 token 预算截断历史

弹性机制：
| 场景 | 策略 |
|------|------|
| 空响应 | 最多重试 _MAX_EMPTY_RETRIES=2 次，然后触发 finalization retry |
| 输出截断（finish_reason=length） | 最多 _MAX_LENGTH_RECOVERIES=3 次续写 |
| 中途消息注入 | 最多 _MAX_INJECTIONS_PER_TURN=3，每轮最多 _MAX_INJECTION_CYCLES=5 个注入循环 |
| ask_user 中断 | 工具调用截断至 ask_user 之前，返回 stop_reason="ask_user" |

检查点机制（checkpoint_callback）：

每个工具执行阶段（awaiting_tools / tools_completed / final_response）都会触发检查点，支持断点续跑——进程重启后可从最近检查点恢复未完成任务。

3.3 AgentHook — 可组合生命周期钩子
AgentHook
 定义的生命周期：

before_iteration(context)
    → [LLM 调用]
    → on_stream(context, delta)       # 流式 token 回调
    → on_stream_end(context, resuming)# resuming=True 表示工具调用后继续
    → before_execute_tools(context)   # 工具调用前，负责进度提示/日志
    → after_iteration(context)        # 每轮结束，负责 token 用量上报
    → finalize_content(context, content) # 最终内容后处理（如 strip <think>）

CompositeHook
 实现扇出：

异步方法级别错误隔离（单个 hook 异常不影响其他 hook）
finalize_content 是管道（无错误隔离，bug 应浮现）
通过 _reraise 标志控制内置 Hook 异常穿透
_LoopHook（内置）额外负责：

流式输出中过滤 <think>...</think> 标签（支持 DeepSeek-R1、Kimi 等 CoT 模型）
工具提示格式化（tool_hint）到渠道进度回调
四、LLM Provider 层
4.1 统一接口
LLMProvider
 抽象基类包含：

完整的重试策略体系：区分 transient error（429 rate limit / 5xx / timeout / connection）与 non-retryable（quota exhausted / billing_limit）
_CHAT_RETRY_DELAYS = (1, 2, 4) — standard 模式指数退避
_PERSISTENT_MAX_DELAY = 60 — persistent 模式最大等待，支持长时间任务
_NON_RETRYABLE_429_ERROR_TOKENS — 细粒度识别不可重试的 429（账单超限）
LLMResponse
 携带：

reasoning_content — DeepSeek-R1、Kimi 思维链
thinking_blocks — Anthropic extended thinking（结构化）
should_execute_tools — 综合 has_tool_calls + finish_reason 的安全执行判断（防止 gateway 注入的虚假工具调用）
4.2 Provider 注册表驱动
ProviderSpec
 元数据驱动一切：

关键字匹配（keywords）+ 前缀检测（detect_by_key_prefix）+ BaseURL 检测（detect_by_base_keyword）三层自动识别
is_gateway：OpenRouter、AiHubMix 等聚合网关（路由任意模型）
is_local：vLLM、Ollama 等本地部署（免 API Key 校验）
is_oauth：GitHub Copilot、OpenAI Codex 等 OAuth 认证
model_overrides：按模型名注入特殊参数（如 Kimi K2 的 temperature: 1.0）
thinking_style：按 Provider 风格构建 thinking 参数（thinking_type/enable_thinking/reasoning_split）
支持的 Provider（28+）：OpenRouter、Anthropic、OpenAI、DeepSeek、Gemini、Moonshot/Kimi、MiniMax、VolcEngine、SiliconFlow、Qianfan、StepFun、AiHubMix、Azure OpenAI、GitHub Copilot、OpenAI Codex、Ollama、vLLM、LM Studio 等。

4.3 OpenAI-compat Provider 特性
OpenAICompatProvider
 核心能力：

Langfuse 可观测：检测 LANGFUSE_SECRET_KEY 自动替换 AsyncOpenAI 为 instrumented 版本
Responses API 回退：OpenAI Responses API → 回退到 Chat Completions（双模式）
Prompt Caching：自动为 Anthropic via OpenRouter 注入缓存标记
Kimi Thinking：特殊处理 kimi-k2.5/kimi-k2.6 的 thinking 参数
4.4 Anthropic Provider 特性
AnthropicProvider
 核心能力：

消息格式转换（OpenAI Messages → Anthropic Messages API）
Extended Thinking（结构化 thinking_blocks）
流式 Prompt Caching 优化（减少重复 token 计费）
五、渠道层
5.1 BaseChannel 规范
BaseChannel
 定义 4 个抽象方法：

| 方法 | 职责 |
|------|------|
| start() | 长驻异步任务，连接平台、监听消息 |
| stop() | 优雅关闭，释放资源 |
| send(msg) | 发送完整消息 |
| send_delta(chat_id, delta) | 发送流式增量（可选实现） |

流式支持判断（supports_streaming property）：需同时满足：

config 中 streaming: true
子类覆写了 send_delta（通过 type(self).send_delta is not BaseChannel.send_delta 检测）
权限控制：is_allowed() 实现白名单，支持 allowFrom: ["*"]（全放行）或具体 ID 列表，空列表拒绝所有。

音频转写：transcribe_audio() 内置 Groq / OpenAI Whisper 双后端，配置化切换。

5.2 ChannelManager
ChannelManager
 职责：

插件发现：pkgutil.scan + Python entry_points（nanobot.channels 组）支持外部 Channel 插件
出站路由：消费 bus.outbound 队列，按 msg.channel 路由到对应渠道实例
重试策略：指数退避 (1, 2, 4) 秒，最多 send_max_retries 次（默认 3 次）
WebUI 集成：自动检测 nanobot.web 包的 dist 目录，为 WebSocket 渠道提供静态文件服务
六、工具体系
6.1 工具基类设计
Tool
 抽象基类的关键属性：

| 属性 | 说明 |
|------|------|
| read_only | 无副作用，可安全并行 |
| concurrency_safe | read_only and not exclusive |
| exclusive | 即便开启并发也要单独运行 |

@tool_parameters 装饰器：将 JSON Schema 注入类级属性，每次访问返回深拷贝防止污染。

Schema
 提供完整 JSON Schema 验证：enum/minimum/maximum/minLength/maxLength/minItems/maxItems/required 字段，深度递归验证 object/array。

cast_params 类型容错：LLM 返回 "true"/"1"/"yes" 会被自动转换为 bool(True)，字符串数字转 int/float，保证参数类型安全。

6.2 内置工具说明
文件系统工具（filesystem.py）：

restrict_to_workspace + allowed_dir 双重路径约束
ReadFileTool 支持 extra_allowed_dirs（允许读取内置 Skills 目录）
EditFileTool 支持行级精准编辑
Shell 执行（shell.py）：

bwrap（Bubblewrap）沙箱支持（Linux）
allowed_env_keys 精确控制透传的环境变量
NANOBOT_... 系列内部变量自动过滤
MCP 客户端（mcp.py）：

stdio / SSE / streamableHttp 三种传输
Windows 下自动将 npx/npm 等 Shell Launcher 包装为 cmd.exe /d /c ...
瞬态连接错误（ClosedResourceError/BrokenPipeError 等）自动单次重试
enabled_tools 白名单过滤（支持原始 MCP 名 + mcp_<server>_<tool> 包装名）
Nullable union 类型自动解包（anyOf: [schema, {type: null}]）
网络安全（security/network.py）：

SSRF 防护：阻断私有/保留 IP（RFC 1918、169.254 链路本地、云元数据地址等）
configure_ssrf_whitelist 支持 Tailscale 等可信内网 CIDR 豁免
双重检查：DNS 解析前校验 + 重定向后 IP 复验
七、记忆系统（三层架构）
┌──────────────────────────────────────────────────────────┐
│                    记忆三层架构                           │
│                                                          │
│  Layer 1: 工作记忆（Session）                            │
│  ┌──────────────────────────────────────────┐           │
│  │ sessions/<key>.jsonl — 实时对话历史       │           │
│  │ 按 channel:chat_id 隔离                  │           │
│  │ 内存缓存 + 原子写入                       │           │
│  └──────────────────┬───────────────────────┘           │
│                     │ token 超预算时触发                  │
│                     ▼                                    │
│  Layer 2: 中期记忆（Consolidator）                       │
│  ┌──────────────────────────────────────────┐           │
│  │ memory/history.jsonl — LLM 摘要归档      │           │
│  │ 按 cursor 游标追踪已整合位置              │           │
│  │ 失败时降级为 raw archive                 │           │
│  └──────────────────┬───────────────────────┘           │
│                     │ Dream 定期处理                     │
│                     ▼                                    │
│  Layer 3: 长期记忆（Dream + GitStore）                   │
│  ┌──────────────────────────────────────────┐           │
│  │ memory/MEMORY.md — 长期知识库             │           │
│  │ SOUL.md — Agent 人格/价值观               │           │
│  │ USER.md — 用户档案                        │           │
│  │ dulwich Git 版本管理，支持 blame 年龄注解 │           │
│  └──────────────────────────────────────────┘           │
└──────────────────────────────────────────────────────────┘

7.1 Session 管理
SessionManager
 细节：

格式：JSONL，首行 metadata（含 last_consolidated 游标），后续每行一条消息
原子写入：先写临时文件再 rename，防止写入中断损坏会话
自动迁移：从旧 ~/.nanobot/sessions/ 迁移到 workspace 内 sessions/
图像占位符：历史回放时将 media 列表转为 [image: path] 文本，保持 LLM 上下文语义完整
Session.retain_recent_legal_suffix(max_messages) — 保留最近 N 条消息时严格对齐到合法的用户轮次边界（不切断工具调用对）。

7.2 Consolidator — 中期记忆整合
触发条件：当前 session prompt token 数超过 context_window_tokens - max_completion_tokens - 1024（安全缓冲）

整合流程：

评估 session prompt 大小（使用 tiktoken cl100k_base 或 provider 专属计数）
按用户轮次边界确定归档区间（pick_consolidation_boundary）
LLM 调用生成摘要（用 consolidator_archive.md 模板）
摘要写入 history.jsonl，更新 last_consolidated 游标
最多 5 轮迭代压缩至目标 50% 预算
LLM 失败时降级 raw dump（[RAW] N messages\n...）
7.3 Dream — 两阶段长期记忆
Phase 1（评估阶段）：

读取 history.jsonl 中 dream_cursor 之后的新条目
LLM 分析（附 Git blame 年龄注解 ← Nd），识别需要更新/删除/保留的记忆
输出结构化操作指令
Phase 2（执行阶段）：

AgentRunner 执行文件读写工具，更新 MEMORY.md
更新 dream_cursor 防止重复处理
支持发现并记录新 Skills（Dream learns discovered skills）
GitStore（基于 dulwich）：

SOUL.md、USER.md、memory/MEMORY.md 自动 Git 追踪
annotate_line_ages=True：Phase 1 传入 git blame 年龄，帮助 LLM 判断记忆时效
Dream 可 commit --amend 同次会话的重复修改
7.4 AutoCompact — 空闲会话压缩
AutoCompact
 定期检查空闲会话：

TTL 触发（session_ttl_minutes）
保留最近 8 条消息作为上下文锚（_RECENT_SUFFIX_MESSAGES=8）
归档部分调用 Consolidator.archive() 生成摘要
摘要以 "Inactive for N minutes. Previous conversation summary: ..." 格式在下次会话时注入
八、上下文构建（ContextBuilder）
ContextBuilder.build_system_prompt()
 按以下顺序拼接：

1. identity.md 渲染（包含 workspace路径/OS运行时/平台策略）
2. Bootstrap 文件（AGENTS.md / SOUL.md / USER.md / TOOLS.md）
3. MEMORY.md 长期记忆（模板内容不注入，避免"空记忆"噪声）
4. always_skills（始终激活的技能完整内容）
5. skills_summary（其他技能的摘要索引，按需激活）
6. Recent History（history.jsonl 中 dream_cursor 后的最新 50 条，上限 32000 chars）

build_messages() 中，运行时上下文（Runtime Context）包含当前时间/Channel/Chat ID，以 [Runtime Context — metadata only, not instructions] 标记包裹，与用户消息合并为同一 user role 消息（避免连续同角色消息被某些 Provider 拒绝）。

九、定时任务与心跳
9.1 CronService
CronService
 三种调度模式：

at：一次性绝对时间（毫秒时间戳）
every：循环间隔（毫秒）
cron：标准 cron 表达式（依赖 croniter，支持时区 ZoneInfo）
持久化：JSON 文件 + filelock（跨进程安全），进程重启后自动恢复待执行任务。

9.2 HeartbeatService
HeartbeatService
 两阶段设计：

Phase 1（决策）：读取 HEARTBEAT.md，通过虚拟工具调用（heartbeat tool，action: skip|run）让 LLM 判断是否有待处理任务，避免自由文本解析
Phase 2（执行）：仅当 Phase 1 返回 run 时触发完整 Agent Loop，节省 token 消耗
十、子 Agent 体系
SubagentManager
 设计要点：

每个子 Agent 独立 asyncio.Task，共享父 AgentRunner 和 LLMProvider
SubagentStatus 实时追踪 phase/iteration/tool_events/usage（6 个生命周期阶段）
session_tasks: dict[str, set[str]] 按会话维度管理，支持 cancel_by_session 批量取消
子 Agent 完成后通过 bus.publish_outbound 将结果路由回原会话的 pending_queue（mid-turn 注入机制）
十一、安全设计
| 安全层 | 机制 |
|--------|------|
| 渠道访问控制 | allow_from 白名单，空列表拒绝所有 |
| 文件系统隔离 | restrict_to_workspace + allowed_dir 双重路径约束 |
| 命令执行沙箱 | bwrap 容器化（Linux），allowed_env_keys 环境变量过滤 |
| SSRF 防护 | DNS 解析 + IP 白名单校验，覆盖 RFC1918/169.254/::1 等所有私有段 |
| 命令注入防御 | Shell 命令检测内部 URL（contains_internal_url） |
| 工具调用安全 | should_execute_tools 阻断 gateway 注入的虚假工具调用 |
| 记忆防污染 | strip_think 在持久化前过滤 CoT 泄露片段 |

十二、关键设计模式总结
| 模式 | 体现 |
|------|------|
| 发布/订阅 | MessageBus 双向队列解耦 Channel 与 Agent |
| 策略模式 | Provider 注册表 + 后端工厂，运行时选择实现 |
| 责任链 | CommandRouter 四级路由：priority→exact→prefix→interceptors |
| 管道/过滤器 | AgentRunner 上下文治理 5 步管道 |
| 组合模式 | CompositeHook 扇出，允许任意组合 Hook |
| 模板方法 | ContextBuilder.build_system_prompt 固定组装顺序 |
| 装饰器模式 | @tool_parameters 注入 JSON Schema 到工具类 |
| 游标模式 | last_consolidated + dream_cursor 双游标追踪记忆处理进度 |
| 检查点模式 | checkpoint_callback 支持长任务断点续跑 |

现在我已经收集了足够详细的信息，可以输出完整的设计说明。

Nanobot 详细架构设计说明
一、总体设计哲学
Nanobot 以 "小核心 + 可插拔扩展" 为核心原则：

Agent 循环极简化：AgentRunner 仅关注 LLM 迭代与工具执行，不耦合任何渠道或产品逻辑。
总线解耦：Channel 与 Agent 之间通过 MessageBus 的双队列通信，双方互不感知。
上下文即配置：记忆、技能、角色设定均以 Markdown 文本注入 System Prompt，而非硬编码逻辑。
可观测性内置：AgentHook 生命周期钩子贯穿整个执行路径，用于日志、进度推送、流式输出。
二、模块详细设计
2.1 消息总线（bus/）
职责：异步解耦 Channel 与 Agent 核心。

Channel → publish_inbound(InboundMessage) → asyncio.Queue[InboundMessage]
                                                      ↓
                                              AgentLoop.run() 消费
                                                      ↓
                                        publish_outbound(OutboundMessage)
                                                      ↓
ChannelManager → dispatch → Channel.send(OutboundMessage)

InboundMessage 携带：channel、sender_id、chat_id、content、media、session_key_override
OutboundMessage 携带：channel、chat_id、content、reply_to、media、buttons
会话键推导：session_key = session_key_override or f"{channel}:{chat_id}"，线程级隔离（如 Slack Thread、Telegram Thread）可覆盖此键。
2.2 Agent 核心层（agent/）
2.2.1 AgentLoop（agent/loop.py）
AgentLoop 是产品层主控，负责：

| 职责 | 实现细节 |
|------|---------|
| 消息分发 | 从 MessageBus 消费 InboundMessage，按 session_key 路由到对应任务 |
| 并发控制 | NANOBOT_MAX_CONCURRENT_REQUESTS（默认 3）的 asyncio.Semaphore 限流 |
| 会话锁 | 每 session_key 一把 asyncio.Lock，保证同一会话串行处理 |
| 中途注入 | 活跃任务期间新消息进入 _pending_queues[session_key]，在工具执行间隙注入（最多 3 次/轮） |
| 命令路由 | 优先级命令（/stop、/restart）在锁外直接分发；普通命令在锁内分发 |
| MCP 懒加载 | 首次消息时才连接 MCP servers，连接失败自动重试 |
| 工具上下文更新 | 每次工具执行前更新 message/spawn/cron 工具的 channel/chat_id 路由信息 |

并发架构：

run() ──┬── dispatch_priority(msg)       # 锁外：/stop、/restart
        ├── [acquire session_lock]
        │    ├── dispatch_command(msg)   # 锁内：其他命令
        │    └── spawn asyncio.Task      # 锁内：Agent 处理任务
        └── _active_tasks[session_key].append(task)

2.2.2 AgentRunner（agent/runner.py）
AgentRunner 是纯执行引擎，无产品逻辑，每次 run(spec) 执行一个完整的 ReAct 循环：

单轮执行流程：

for iteration in range(max_iterations):
    ① 上下文治理（Context Governance）
       - drop_orphan_tool_results()    # 删除孤立的 tool 回复
       - backfill_missing_tool_results() # 补全丢失的 tool 回复
       - microcompact()                # 压缩大型工具结果（保留最近10条）
       - apply_tool_result_budget()    # 按字符预算裁剪工具结果
       - snip_history()                # 按 token 预算截断历史
    ② hook.before_iteration()
    ③ _request_model() → LLMResponse
    ④ 分支判断
       - has tool_calls → 执行工具 → 注入 injection → continue
       - finish_reason=length → 追加恢复消息 → continue
       - empty content → 最多重试 2 次
       - 正常内容 → 写入 session → break
    ⑤ hook.after_iteration()

检查点机制（checkpoint_callback）：每个关键阶段（awaiting_tools/tools_completed/final_response）发射检查点，用于崩溃恢复。

外部注入机制（injection_callback）：工具执行完成后检查是否有新用户消息待注入，最多 5 个注入周期、每周期 3 条消息，防止无限循环。

2.2.3 ContextBuilder（agent/context.py）
System Prompt 组装顺序：

① agent/identity.md（含 workspace_path、runtime、platform_policy、channel）
② BOOTSTRAP_FILES：AGENTS.md / SOUL.md / USER.md / TOOLS.md（从 workspace 读取）
③ memory/MEMORY.md（长期记忆，若与模板相同则跳过）
④ always_skills（始终加载的技能）
⑤ skills_summary（其余可用技能列表）
⑥ Recent History（自上次 Dream 以来的 history.jsonl 条目，最近 50 条，32K 字符上限）

用户消息组装：

RuntimeContext（当前时间、Channel、ChatID、会话摘要）
         +
UserMessage（文本 + 可选 base64 图片块）
         ↓
合并为单个 user role 消息（避免部分 Provider 拒绝连续相同 role）

2.2.4 SubagentManager（agent/subagent.py）
子 Agent 通过 asyncio.create_task() 在后台运行，共享主 Agent 的 LLMProvider
每个任务分配 UUID task_id，通过 SubagentStatus 跟踪阶段（initializing/awaiting_tools/done/error）
子 Agent 完成后通过 MessageBus.publish_outbound() 把结果投递回原始会话
按 session_key 分组，cancel_by_session() 可批量取消某会话所有子 Agent
2.3 记忆系统（agent/memory.py）
三层分离架构：

┌─────────────────────────────────────────┐
│         短期：Session（会话历史）         │
│  sessions/{session_key}.jsonl（JSONL）   │
└──────────────────┬──────────────────────┘
                   │ 超出 token 预算时触发
                   ▼
┌─────────────────────────────────────────┐
│    中期：Consolidator（会话压缩）        │
│  LLM 摘要 → memory/history.jsonl        │
│  失败时 raw_archive() 保底              │
└──────────────────┬──────────────────────┘
                   │ Dream 定期处理
                   ▼
┌─────────────────────────────────────────┐
│     长期：Dream（记忆蒸馏）             │
│  memory/MEMORY.md（+Git 版本管理）      │
│  Dream Phase1: 分析 history.jsonl        │
│  Dream Phase2: 更新 MEMORY.md           │
└─────────────────────────────────────────┘

Consolidator 详细逻辑：

使用 tiktoken cl100k_base 估算 token 数
每轮检查：estimated_tokens >= context_window_tokens - max_completion_tokens - 1024
超出时找最近 user-turn 边界截断，调用 LLM 生成摘要（≤8000 字符）
摘要追加 history.jsonl，最多 5 轮循环直到 token 数降到预算一半以下
GitStore（utils/gitstore.py）：

基于 dulwich 实现纯 Python Git
跟踪文件：SOUL.md、USER.md、memory/MEMORY.md
提供 git-blame 行年龄注解（Dream Phase1 使用，帮助 LLM 识别过时记忆）
AutoCompact（agent/autocompact.py）：

空闲会话（超过 session_ttl_minutes）自动压缩
保留最近 8 条消息作为上下文锚点
压缩摘要存入 session.metadata["_last_summary"]，下次唤醒时作为 session_summary 注入
2.4 工具系统（agent/tools/）
2.4.1 Tool 基类设计
class Tool(ABC):
    name: str          # 工具名（LLM function call name）
    description: str   # 工具说明
    parameters: dict   # JSON Schema（由 @tool_parameters 装饰器注入）
    read_only: bool    # 无副作用，可并发
    exclusive: bool    # 独占执行，不与其他工具并行
    
    async def execute(**kwargs) -> str | list  # 实际执行
    def cast_params(params) -> dict           # schema 驱动类型转换
    def validate_params(params) -> list[str] # JSON Schema 验证
    def to_schema() -> dict                  # 输出 OpenAI function schema

@tool_parameters 装饰器将 JSON Schema 冻结在类上，每次 parameters 访问返回深拷贝，防止工具被 LLM 注入篡改 schema。

2.4.2 ToolRegistry 设计
注册/注销：动态增删工具，注销自动清除 _cached_definitions
定义缓存：builtin 工具排序在前（字母序）、MCP 工具（mcp_ 前缀）排序在后，结果缓存直到下次变更，保证 prompt cache 命中率
执行流程：prepare_call() → cast_params → validate_params → tool.execute() → 追加 [Analyze the error above and try a different approach.] 提示
2.4.3 MCP 集成（agent/tools/mcp.py）
支持 stdio（子进程）、sse（HTTP SSE）、streamableHttp 三种连接类型
Windows 下自动将 npx/npm 等 shell 启动器包装为 cmd /d /c
瞬态连接错误（ClosedResourceError、BrokenPipeError 等）自动重试一次
enabled_tools 白名单过滤，支持 raw MCP 名和 mcp_{server}_{tool} 两种格式
MCP 工具 schema 中的 nullable union 类型自动提取非 null 分支
2.5 渠道层（channels/）
2.5.1 BaseChannel 契约
class BaseChannel(ABC):
    async def start()           # 长运行监听任务
    async def stop()            # 清理资源
    async def send(msg)         # 发送完整消息（失败时 raise，由 ChannelManager 重试）
    async def send_delta(chat_id, delta)  # 流式分块（子类可选实现）
    
    @property
    def supports_streaming      # config.streaming=True AND 实现了 send_delta
    
    def is_allowed(sender_id)   # allow_from 白名单检查（空列表 → 拒绝所有）

2.5.2 ChannelManager
通过 pkgutil 扫描 + entry_points 发现渠道插件（内置 + 第三方均可）
出站分发：_dispatch_task 持续消费 outbound 队列，按 channel 字段路由到对应 Channel
重试策略：指数退避 1s/2s/4s（最多 send_max_retries 次，默认 3）
流式输出：OutboundMessage.metadata["_stream_delta"] 触发 send_delta()，_stream_end 触发流结束
2.6 Provider 层（providers/）
2.6.1 LLMProvider 基类
重试策略（_run_with_retry）：

standard 模式：固定 3 次重试，延迟 1s/2s/4s
persistent 模式：指数退避至 60s，相同错误最多 10 次，配合 retry_wait_callback 向用户推送等待进度
429 错误细分：

余额不足（insufficient_quota/billing_hard_limit_reached 等）→ 不重试
速率限制（rate_limit_exceeded/overloaded 等）→ 重试，优先使用 Retry-After 头
消息净化（_sanitize_empty_content）：

修复空 content 字符串（assistant+tool_calls → None，其他 → "(empty)"）
剥离 _meta 内部字段（base64 图片路径等）防止泄露给 Provider
2.6.2 Provider 注册表驱动
ProviderSpec 声明式元数据（30+ 内置 Provider）：

@dataclass(frozen=True)
class ProviderSpec:
    name: str                    # 配置字段名
    keywords: tuple[str, ...]    # 模型名匹配关键词
    backend: str                 # 实现类：openai_compat/anthropic/...
    detect_by_key_prefix: str    # API key 前缀匹配（如 "sk-or-" → OpenRouter）
    detect_by_base_keyword: str  # API base URL 匹配
    is_gateway: bool             # 路由网关（不依赖 keywords 匹配）
    model_overrides: tuple       # 特定模型参数覆盖

匹配优先级：强制指定 provider > key prefix > base URL keyword > model keywords > gateway fallback

2.7 会话管理（session/）
存储格式：每个会话一个 sessions/{safe_key}.jsonl 文件：

{"_type":"metadata","created_at":"...","updated_at":"...","last_consolidated":N,"metadata":{}}
{"role":"user","content":"...","timestamp":"..."}
{"role":"assistant","content":"...","tool_calls":[...]}
{"role":"tool","tool_call_id":"...","name":"...","content":"..."}

历史边界规则（get_history）：

从 last_consolidated 之后的消息开始
对齐到最近的 user 消息（避免从 assistant/tool 消息开始）
剥除前置孤立 tool result（find_legal_message_start）
图片路径转为 [image: path] 占位符
内存缓存：_cache: dict[str, Session] 进程内缓存，invalidate(key) 强制重新从磁盘加载（AutoCompact 归档后使用）

2.8 命令路由（command/）
四层分发优先级：

| 优先级 | 范围 | 示例 | 执行位置 |
|--------|------|------|----------|
| priority | 精确匹配 | /stop, /restart | 会话锁外，立即响应 |
| exact | 精确匹配 | /status, /history, /clear | 会话锁内 |
| prefix | 最长前缀匹配 | /team , /skill  | 会话锁内 |
| interceptors | 谓词列表 | team 模式拦截 | 会话锁内 |

/stop 取消该 session 所有 asyncio.Task 和子 Agent 任务；
/restart 通过 os.execv() 原地重启进程，启动后通过环境变量检测并通知原始频道。

2.9 定时任务（cron/）
三种调度类型：

at：一次性，at_ms 毫秒时间戳
every：固定间隔，every_ms 毫秒
cron：标准 cron 表达式 + IANA 时区（croniter + zoneinfo）
持久化：任务状态存入 cron_jobs.json，filelock 保证多实例安全。每次触发通过 on_execute 回调把任务内容注入 AgentLoop 执行。

2.10 心跳服务（heartbeat/）
两阶段设计：

Phase 1（决策）：定期读取 HEARTBEAT.md，通过虚拟工具调用（heartbeat(action="skip|run")）让 LLM 决策是否有待办任务，避免文本解析不稳定
Phase 2（执行）：仅当 Phase 1 返回 run 时，触发 on_execute 回调执行完整 Agent 循环
2.11 安全设计（security/）
SSRF 防护（security/network.py）：

请求前 DNS 解析，检查所有解析 IP 是否在内网 CIDR（10.x/172.x/192.168.x/169.254.x 等）
重定向后再次检查已解析 IP
ssrf_whitelist 允许特定 CIDR 豁免（如 Tailscale 100.64.0.0/10）
contains_internal_url() 扫描 Shell 命令中的 URL，防止命令注入内网访问
工作区沙盒：

restrict_to_workspace=True 限制所有文件工具只能访问 workspace 目录
ExecTool 支持 bwrap（Bubblewrap）容器沙盒
ExecTool 过滤敏感环境变量（仅透传 allowed_env_keys 白名单）
2.12 外部接口设计
OpenAI-Compatible API（api/server.py）：

POST /v1/chat/completions：支持 stream=true（SSE）和同步两种模式
POST /v1/files：文件上传（base64 data URL，大小上限保护）
固定路由到 api:default 会话，所有请求共享同一对话历史
SSE 格式：data: {"choices":[{"delta":{"content":"..."}}]}\n\n
Python SDK Facade（nanobot.py）：

bot = Nanobot.from_config()
result = await bot.run("Summarize this repo", hooks=[MyHook()])
# result.content: 最终回复文本

from_config() 完整初始化 Provider + AgentLoop，run() 每次使用独立 session_key，可通过 AgentHook 注入观测逻辑。

三、数据流总览
用户消息
    │
    ▼
[Channel] WeChat/Telegram/Discord...
    │  InboundMessage
    ▼
[MessageBus.inbound Queue]
    │
    ▼
[AgentLoop.run()]
    ├─ 命令路由（CommandRouter）
    │       └─ /stop /status 等内置命令
    └─ Agent 处理任务（asyncio.Task）
            ├─ SessionManager（加载历史）
            ├─ AutoCompact（空闲压缩）
            ├─ Consolidator（token 预算管理）
            ├─ ContextBuilder（组装 Prompt）
            └─ AgentRunner.run(spec)
                    ├─ 上下文治理
                    ├─ LLMProvider.chat()
                    ├─ ToolRegistry.execute()
                    │      ├─ filesystem/shell/web/search...
                    │      ├─ MCP 工具
                    │      └─ spawn/cron/message
                    └─ AgentHook（流式、进度、日志）
                            │
                            ▼
                    OutboundMessage
                            │
                    [MessageBus.outbound Queue]
                            │
                    [ChannelManager] 路由 + 重试
                            │
                    [Channel].send() / send_delta()
                            │
                         用户界面

四、关键设计权衡
| 决策 | 选择 | 权衡原因 |
|------|------|---------|
| 记忆存储 | 纯文件（Markdown + JSONL + Git） | 无数据库依赖，易于人工查看和编辑 |
| 并发模型 | asyncio + 每 session 一把锁 | 避免多线程竞争，串行保证会话一致性 |
| Provider 抽象 | 注册表元数据驱动 | 新增 Provider 零修改路由逻辑 |
| 技能扩展 | Markdown 上下文注入 | 无编排框架依赖，模型自行理解和执行 |
| Tool schema | 装饰器冻结 + 深拷贝 | 防止运行时 schema 被篡改，提升安全性 |
| 重试策略 | 两级（standard/persistent） | 交互场景快失败，长任务场景持久重试 |