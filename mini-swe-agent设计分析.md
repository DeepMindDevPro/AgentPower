## 本地启动命令
mini -c mini_textbased.yaml -c qwen_textbased.yaml -y -t "分析/Users/gechunfa1/Documents/ai-code/Mini-Agent/examples/09_chat_bot_qwen_local.py这个类的功能"


## mini-swe-agent 源码快速阅读指南

### 阅读顺序（6个文件，约600行核心代码）

按**数据流**从入口到核心，依次阅读：

---

#### 第1步：入口 — `run/mini.py`（110行）

只需看 `main()` 函数，核心就3行：
```python
model = get_model(config=...)      # 创建模型
env = get_environment(config=...)  # 创建环境
agent = get_agent(model, env, ...) # 创建代理并运行
```
其余全是 Typer CLI 参数解析和配置合并逻辑。

---

#### 第2步：协议定义 — `__init__.py`（93行）

3个 Protocol 接口是整个项目的契约，看懂这3个就懂了60%：

| Protocol | 核心方法 | 作用 |
|---|---|---|
| `Model` | `query()` / `format_message()` / `format_observation_messages()` | 查询LM、格式化消息 |
| `Environment` | `execute()` | 执行命令、返回结果 |
| `Agent` | `run()` / `save()` | 驱动循环、保存轨迹 |

---

#### 第3步：核心循环 — `agents/default.py`（156行）

**这是最重要的文件**，整个 Agent 的运转逻辑：

```
run()           → 渲染模板 → 循环 step() 直到退出
  step()        → query() + execute_actions()
    query()     → 检查限制 → model.query() → 计费
    execute_actions() → env.execute() → model.format_observation_messages()
```

关键设计：**异常驱动控制流** — `InterruptAgentFlow` 子类（Submitted/LimitsExceeded/FormatError）通过异常中断循环，消息中 `role="exit"` 表示终止。

---

#### 第4步：交互扩展 — `agents/interactive.py`（184行）

在 `DefaultAgent` 上增加3种模式：
- **human**：覆盖 `query()`，用户直接输入命令
- **confirm**：覆盖 `execute_actions()`，执行前确认
- **yolo**：直接执行（等同 DefaultAgent）

---

#### 第5步：模型实现（二选一）

- `models/litellm_model.py`（148行）— **tool call 模式**，核心是 `_query()` 传 `tools=[BASH_TOOL]`，`_parse_actions()` 解析 tool_calls
- `models/litellm_textbased_model.py`（46行）— **文本解析模式**，`_query()` 不传 tools，`_parse_actions()` 用正则从文本提取命令

两者区别仅在 `_query()` 和 `_parse_actions()` 两个方法。

---

#### 第6步：环境实现 — `environments/local.py`（80行）

最简单的环境，核心就是 `subprocess.run()` + `_check_finished()` 检测提交信号。Docker 环境同理，只是命令在容器内执行。

---

### 可跳过的部分

| 模块 | 原因 |
|---|---|
| `models/openrouter_*` / `portkey_*` / `requesty_*` | 特定提供商变体，逻辑同 litellm |
| `environments/singularity.py` / `extra/*` | 特定环境变体，逻辑同 local/docker |
| `run/benchmarks/` | SWE-bench 评测，与核心逻辑无关 |
| `run/utilities/inspector.py` | TUI 检查器，独立功能 |
| `config/*.yaml` | 配置文件，需要时查阅 |

### 一句话总结架构

**Agent 循环调用 Model 拿命令 → Environment 执行命令 → 结果回传 Model 格式化 → 再问 Model，直到抛出 Submitted 异常退出。**



# mini-swe-agent 核心执行流程与详细架构设计

## 一、整体架构

```
┌─────────────────────────────────────────────────────┐
│                      CLI 入口                        │
│               run/mini.py (Typer)                    │
│         解析参数 → 合并配置 → 组装三组件              │
└──────────┬──────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│                    Agent (驱动层)                      │
│  ┌─────────────────┐  ┌─────────────────────────┐    │
│  │  DefaultAgent   │←─│  InteractiveAgent       │    │
│  │  run()/step()   │  │  +human/confirm/yolo    │    │
│  └────┬───────┬────┘  └─────────────────────────┘    │
│       │       │                                      │
└───────┼───────┼──────────────────────────────────────┘
        │       │
   ┌────┘       └─────┐
   ▼                  ▼
┌────────────┐  ┌──────────────┐
│   Model    │  │  Environment │
│ (决策层)    │  │  (执行层)     │
│            │  │              │
│ Litellm    │  │ Local        │
│ Textbased  │  │ Docker       │
│ Response   │  │ Singularity  │
│ OpenRouter │  │ Bubblewrap   │
│ Portkey    │  │ ...          │
└────────────┘  └──────────────┘
```

项目采用 **Agent-Model-Environment (AME)** 三层架构，核心接口通过 Python Protocol 定义，支持鸭子类型扩展。

---

## 二、核心执行流程（完整生命周期）

```
mini -y -t "fix bug"
  │
  ▼
[1] CLI 参数解析 (run/mini.py:main)
  │  读取 -c 配置文件 → YAML + CLI参数 → recursive_merge() 合并
  │  环境变量(.env) → dotenv.load_dotenv() 加载
  │
  ▼
[2] 组件初始化
  │  get_model(config)        → 根据model_class动态导入模型类
  │  get_environment(config)  → 根据environment_class动态导入环境类
  │  get_agent(model,env,config) → 根据agent_class动态导入代理类
  │
  ▼
[3] Agent.run(task) 启动
  │
  ├─ 渲染 system_template (Jinja2) → system message
  │   变量来源: config + env.get_template_vars() + model.get_template_vars()
  │
  ├─ 渲染 instance_template (Jinja2) → user message
  │   变量包含: task, n_model_calls, model_cost, 系统信息等
  │
  └─ 进入 while True 主循环 ──────────────────────────────┐
     │                                                    │
     ▼                                                    │
  [4] step() = query() + execute_actions()               │
     │                                                    │
     ├─ query()                                           │
     │  ├─ 检查 step_limit / cost_limit                   │
     │  │  超限 → raise LimitsExceeded                    │
     │  │                                                 │
     │  ├─ model.query(messages)                          │
     │  │  ├─ _prepare_messages_for_api()                 │
     │  │  │  ├─ 移除 extra 字段                          │
     │  │  │  ├─ 重排 Anthropic thinking blocks           │
     │  │  │  └─ 设置 cache_control                       │
     │  │  │                                              │
     │  │  ├─ litellm.completion() 或 litellm.completion(tools=[BASH_TOOL])
     │  │  │  └─ 带retry: tenacity最多10次, 指数退避      │
     │  │  │                                              │
     │  │  ├─ _calculate_cost() → 更新 GLOBAL_MODEL_STATS │
     │  │  │                                              │
     │  │  └─ _parse_actions(response)                    │
     │  │     ├─ [tool call模式] 解析tool_calls → [{command, tool_call_id}]
     │  │     └─ [text模式]   正则匹配content → [{command}]
     │  │        匹配失败 → raise FormatError             │
     │  │                                                 │
     │  ├─ 累加 cost, n_calls                            │
     │  └─ add_messages(lm_response) → messages列表       │
     │                                                    │
     ├─ execute_actions(lm_response)                      │
     │  │                                                 │
     │  ├─ 从 message["extra"]["actions"] 提取命令列表     │
     │  │                                                 │
     │  ├─ env.execute(action) 对每个命令执行              │
     │  │  ├─ subprocess.run(command, shell=True)          │
     │  │  ├─ 返回 {output, returncode, exception_info}   │
     │  │  └─ 检测 "COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT" │
     │  │     → raise Submitted                           │
     │  │                                                 │
     │  └─ model.format_observation_messages()            │
     │     ├─ [tool call模式] role="tool", tool_call_id   │
     │     └─ [text模式]   role="user"                    │
     │     ├─ observation_template (Jinja2) 渲染输出       │
     │     │  (截断超长输出 >10000字符)                    │
     │     └─ add_messages(observations) → messages列表    │
     │                                                    │
     ▼                                                    │
  [5] 异常处理                                           │
     ├─ InterruptAgentFlow → add_messages(*e.messages)    │
     │  ├─ Submitted        → role="exit" 退出            │
     │  ├─ LimitsExceeded   → role="exit" 退出            │
     │  ├─ UserInterruption → role="user" 继续(追加消息)  │
     │  └─ FormatError      → role="user" 继续(追加纠正)  │
     ├─ 其他Exception → handle_uncaught_exception() + raise
     │                                                    │
     ├─ finally: save(output_path) 每步保存轨迹           │
     │                                                    │
     ├─ 检查 messages[-1]["role"] == "exit"               │
     │  是 → break 退出循环                               │
     │  否 ──────────────────────────────────────────────→┘
     │
     ▼
  [6] 返回 messages[-1]["extra"] → {exit_status, submission}
```

---

## 三、消息流（Message 数据结构）

整个系统通过 `messages: list[dict]` 串联，每条消息结构：

```python
# 系统消息
{"role": "system", "content": "渲染后的system prompt", "extra": {}}

# 用户消息（任务）
{"role": "user", "content": "渲染后的instance prompt", "extra": {}}

# LM响应
{"role": "assistant", "content": "思考文本", "tool_calls": [...],
 "extra": {"actions": [{"command": "ls -la"}], "cost": 0.003, "response": ...}}

# 观测消息 (tool call模式)
{"role": "tool", "tool_call_id": "xxx", "content": "渲染后的执行结果", "extra": {...}}

# 观测消息 (text模式)
{"role": "user", "content": "渲染后的执行结果", "extra": {...}}

# 退出消息
{"role": "exit", "content": "提交内容", "extra": {"exit_status": "Submitted", "submission": "..."}}
```

关键：`extra` 字段仅在 Agent 内部使用，发送给 LM 前会被 `_prepare_messages_for_api()` 移除。

---

## 四、异常驱动的控制流

```python
InterruptAgentFlow (基类, 携带messages)
├── Submitted        # 环境检测到提交信号 → role="exit" → 循环结束
├── LimitsExceeded   # Agent检查超限     → role="exit" → 循环结束
├── FormatError      # 模型输出格式错误   → role="user" → 追加纠正消息, 继续循环
└── UserInterruption # 用户中断          → role="user" → 追加中断消息, 继续循环
```

设计优势：将控制流（退出/重试）和数据流（消息）统一到异常机制中，避免在 step() 中混入复杂的状态判断。

---

## 五、配置系统（三层合并）

```
YAML配置文件 (mini.yaml / mini_textbased.yaml)
     │
     ▼ recursive_merge()
CLI参数覆盖 (-c model.model_kwargs.timeout=120)
     │
     ▼ recursive_merge()
环境变量 (.env → MSWEA_MODEL_NAME / OPENAI_API_KEY / ...)
```

配置分三节：`agent:` / `model:` / `environment:`，分别传给对应组件。`UNSET` 哨兵值用于跳过未设置的参数。

---

## 六、两种动作解析模式对比

|  | Tool Call 模式 | Text 模式 |
|---|---|---|
| **模型类** | LitellmModel | LitellmTextbasedModel |
| **_query()** | 传 `tools=[BASH_TOOL]` | 不传 tools |
| **_parse_actions()** | 解析 `tool_calls[].function.arguments` | 正则匹配 `` ```mswea_bash_command ``` `` |
| **观测消息role** | `"tool"` + `tool_call_id` | `"user"` |
| **适用模型** | GPT/Claude/Gemini 等支持function calling的模型 | 不支持tool call的本地/开源模型 |
| **错误处理** | 无tool call → FormatError | 正则不匹配 → FormatError |

---

## 七、InteractiveAgent 扩展点

```
DefaultAgent
  │
  ├── query()           ← 覆盖: human模式下用户直接输入命令
  │                       confirm/yolo模式下包裹 console.status()
  │
  ├── step()            ← 覆盖: 捕获 KeyboardInterrupt → 用户中断
  │
  ├── execute_actions() ← 覆盖: confirm模式下执行前确认
  │                       确认拒绝 → UserInterruption
  │                       Submitted → 检查是否提交新任务
  │
  └── add_messages()    ← 覆盖: 打印消息到终端(rich格式化)
```

---

## 八、环境执行与提交机制

所有环境的 `execute()` 统一返回：

```python
{"output": "命令标准输出", "returncode": 0, "exception_info": ""}
```

提交机制：当输出首行为 `COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT` 且 returncode=0 时，`_check_finished()` 抛出 `Submitted` 异常，后续内容作为 submission。

---

## 九、全局状态追踪

```python
# 单次运行追踪 (Agent内)
self.cost      # 累计费用
self.n_calls   # 累计调用次数

# 全局追踪 (跨运行)
GLOBAL_MODEL_STATS  # 线程安全, 支持全局费用/调用限制
                    # MSWEA_GLOBAL_COST_LIMIT / MSWEA_GLOBAL_CALL_LIMIT
```

---

## 十、核心组件详解

### 10.1 Model 层

模型层负责与语言模型交互，核心接口：

| 方法 | 作用 |
|---|---|
| `query(messages)` | 发送消息列表给LM，返回包含actions的响应 |
| `format_message(**kwargs)` | 格式化单条消息（处理多模态等） |
| `format_observation_messages(message, outputs, template_vars)` | 将执行结果格式化为观测消息 |
| `get_template_vars()` | 提供模型相关的模板变量 |
| `serialize()` | 序列化模型状态用于保存轨迹 |

**模型实现继承关系：**

```
LitellmModel (基类, tool call模式)
├── LitellmTextbasedModel  (覆盖 _query 和 _parse_actions)
└── LitellmResponseModel   (覆盖 _query 和 _parse_actions)

OpenRouterModel (独立实现, 直接HTTP调用)
├── OpenRouterTextbasedModel
└── OpenRouterResponseModel

PortkeyModel (独立实现, 通过Portkey代理)
└── PortkeyResponseAPIModel

RequestyModel (独立实现, 直接HTTP调用)
RouletteModel (实验性, 随机选择其他模型)
DeterministicModel (测试用, 确定性输出)
```

### 10.2 Environment 层

环境层负责执行bash命令，核心接口：

| 方法 | 作用 |
|---|---|
| `execute(action, cwd)` | 执行命令，返回 {output, returncode, exception_info} |
| `get_template_vars()` | 提供环境相关的模板变量（系统信息、环境变量等） |
| `serialize()` | 序列化环境配置 |

**环境实现：**

| 环境 | 执行方式 | 适用场景 |
|---|---|---|
| LocalEnvironment | 本机 subprocess.run() | 本地开发 |
| DockerEnvironment | docker exec 在容器中执行 | 隔离执行 |
| SingularityEnvironment | singularity exec 在容器中执行 | HPC集群 |
| BubblewrapEnvironment | bwrap 沙箱执行 | 轻量隔离 |
| SwerexDockerEnvironment | 通过SWE-ReX在Docker中执行 | SWE-bench评测 |
| SwerexModalEnvironment | 通过SWE-ReX在Modal中执行 | 云端评测 |
| ContreeEnvironment | 通过Contree SDK执行 | 云端执行 |

### 10.3 Agent 层

代理层驱动整个循环，核心接口：

| 方法 | 作用 |
|---|---|
| `run(task)` | 主入口，运行直到任务完成 |
| `step()` | 单步：查询LM + 执行动作 |
| `query()` | 查询LM，检查限制 |
| `execute_actions(message)` | 执行消息中的动作 |
| `save(path)` | 保存轨迹到文件 |
| `serialize()` | 序列化完整状态 |

**Agent实现：**

```
DefaultAgent
└── InteractiveAgent
    ├── mode=human   → 用户直接输入命令
    ├── mode=confirm → LM命令需用户确认
    └── mode=yolo    → 自动执行无需确认
```

---

## 十一、插件注册机制

所有组件通过名称映射表 + 动态导入实现插件化：

```python
# models/__init__.py
_MODEL_CLASS_MAPPING = {
    "litellm": "minisweagent.models.litellm_model.LitellmModel",
    "litellm_textbased": "minisweagent.models.litellm_textbased_model.LitellmTextbasedModel",
    ...
}

# environments/__init__.py
_ENVIRONMENT_MAPPING = {
    "docker": "minisweagent.environments.docker.DockerEnvironment",
    "local": "minisweagent.environments.local.LocalEnvironment",
    ...
}

# agents/__init__.py
_AGENT_MAPPING = {
    "default": "minisweagent.agents.default.DefaultAgent",
    "interactive": "minisweagent.agents.interactive.InteractiveAgent",
}
```

扩展新组件只需：1) 实现Protocol接口 2) 添加映射条目（或用全路径类名引用）。

---

## 十二、一句话总结

**Agent 驱动 while 循环 → Model 生成命令 → Environment 执行命令 → 异常控制退出/重试 → 消息列表贯穿全局，extra 字段内部流转、发送前剥离。**

