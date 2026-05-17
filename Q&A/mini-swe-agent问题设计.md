# mini-swe-agent 核心问题

## 问题 1: Agent 的主循环是如何运作的？消息流从用户输入到最终提交经历了哪些关键步骤？

**学习要点：** 从 [`DefaultAgent.run()`](src/minisweagent/agents/default.py:68) 入手，理解 `run()` → `step()` → `query()` + `execute_actions()` 的循环机制。注意消息列表 `self.messages` 是完全线性的——系统消息、实例消息、LM 回复、环境观察依次追加，不存在分支或状态机。循环终止条件是最后一条消息的 `role == "exit"`。这是整个项目最核心的控制流。

---

## 问题 2: Model / Environment / Agent 三层协议（Protocol）是如何解耦的？它们之间通过什么接口通信？

**学习要点：** 阅读 [`__init__.py`](src/minisweagent/__init__.py:40) 中的三个 Protocol 定义（`Model`、`Environment`、`Agent`）。核心接口只有：
- **Model**: `query(messages)` → 返回带 `extra.actions` 的消息
- **Environment**: `execute(action)` → 返回带 `output`/`returncode` 的结果
- **Agent**: `run(task)` → 返回包含 `exit_status`/`submission` 的字典

这种鸭子类型设计意味着你可以自由替换 Model（litellm/openrouter/portkey/textbased/response）和 Environment（local/docker/singularity/bubblewrap/contree），而 Agent 完全不需要修改。

---

## 问题 3: LM 的输出是如何被解析成可执行动作的？三种解析策略（toolcall / text-based / response API）有何区别？

**学习要点：** 对比三套实现：
- **Tool calling** ([`actions_toolcall.py`](src/minisweagent/models/utils/actions_toolcall.py:25))：v2 推荐方式，LM 通过 OpenAI 风格的 function calling 返回 `bash` tool call
- **Text-based regex** ([`actions_text.py`](src/minisweagent/models/utils/actions_text.py:18))：v1 遗留方式，用正则从文本中提取 ` ```mswea_bash_command ``` ` 代码块
- **Response API** ([`actions_toolcall_response.py`](src/minisweagent/models/utils/actions_toolcall_response.py))：使用 OpenAI `/response` 端点

理解为什么"只给 LM 一个 bash 工具"是此项目的核心设计哲学——这让任何模型都能用，无需特殊的 tool-calling 能力。

---

## 问题 4: 任务完成与异常中断是如何通过异常体系实现的？`InterruptAgentFlow` 的设计精妙在哪里？

**学习要点：** 阅读 [`exceptions.py`](src/minisweagent/exceptions.py:1)，理解异常继承链：
- `InterruptAgentFlow` → 基类，携带 `messages` 参数
  - `Submitted` → 任务正常完成（`COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT` 触发）
  - `LimitsExceeded` → 步数/费用超限
  - `UserInterruption` → 用户中断/添加新任务
  - `FormatError` → LM 输出格式错误

关键在 [`DefaultAgent.run()`](src/minisweagent/agents/default.py:68) 中的 `try/except InterruptAgentFlow`——所有中断都被捕获，中断携带的消息被追加到历史中，然后循环继续。这是一种优雅的"非局部跳转"模式，让各种中断场景统一处理。

---

## 问题 5: 配置系统是如何通过递归合并实现灵活性的？YAML + 命令行参数是如何叠加生效的？

**学习要点：** 从 [`mini.py`](src/minisweagent/run/mini.py:55) 的 `config_spec` 参数入手，追踪：
1. `get_config_from_spec()` 可以接受 YAML 文件路径或 `key=value` 形式
2. 多个配置源通过 [`recursive_merge()`](src/minisweagent/utils/serialize.py:7) 递归合并，后者覆盖前者，`UNSET` 值被跳过
3. 默认配置 → YAML 文件 → 命令行参数，层层覆盖

这种设计让用户可以用 `-c mini.yaml -c model.model_kwargs.temperature=0` 这种方式灵活组合配置，而不需要修改任何文件。


## 问题 6: 项目是如何使用原生 bash 的？提示词、Tool Calling、subprocess 三者如何配合？

**学习要点：** bash 的使用是"提示词 + Tool Calling + subprocess.run"三者配合，而非单一机制，完整链路分四层：

1. **提示词层**（[`mini.yaml`](src/minisweagent/config/mini.yaml:1)）：`instance_template` 告诉 LM "You can execute bash commands"、"Your response MUST include AT LEAST ONE bash tool call"，以及通过 `echo COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT` 提交任务——这是在**约束 LM 的行为**，让它知道只能通过 bash 命令操作电脑。

2. **Tool Calling 层**（[`actions_toolcall.py`](src/minisweagent/models/utils/actions_toolcall.py:11)）：定义了唯一的 tool `BASH_TOOL`（name="bash", parameters={"command": string}），在 [`LitellmModel._query()`](src/minisweagent/models/litellm_model.py:63) 中通过 `litellm.completion(tools=[BASH_TOOL])` 传给 LM API。LM 返回的 tool call 中 `function.arguments == {"command": "ls -la"}`，这样就**结构化地提取出了要执行的命令字符串**。

3. **执行层**（[`LocalEnvironment.execute()`](src/minisweagent/environments/local.py:24)）：直接 `subprocess.run(command, shell=True, ...)` 真正执行 bash 命令。Docker 环境则是 `subprocess.run(["docker", "exec", ..., "bash", "-lc", command])`。关键设计：**每次 `subprocess.run` 都是独立进程**，不维护 shell 会话状态，所以提示词里特意提醒 "Directory or environment variable changes are not persistent"。

4. **任务提交检测**（[`_check_finished()`](src/minisweagent/environments/local.py:55)）：检查命令输出第一行是否为 `COMPLETE_TASK_AND_SUBMIT_FINAL_OUTPUT`，是则抛出 `Submitted` 异常终止 agent 循环。

理解这三层配合的关系：提示词让 LM 知道"只能用 bash"，tool calling 让 LM 结构化地输出命令，`subprocess.run(shell=True)` 是真正执行命令的地方。三者缺一不可。

---

**建议学习路径：** 先从问题 1 开始，画出 `run()` 的流程图；然后看问题 2 理解架构；问题 3 和 4 分别深入"动作解析"和"异常流"两个关键子系统；最后用问题 5 理解配置如何把一切串起来。