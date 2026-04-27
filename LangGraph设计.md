一、详细技术架构
1. 分层架构图
┌─────────────────────────────────────────────────────────────┐
│                     用户 API 层                              │
│   StateGraph / entrypoint / @task / create_react_agent      │
├─────────────────────────────────────────────────────────────┤
│                    图编译层 (Graph Layer)                    │
│   StateGraph.compile() → CompiledStateGraph (Pregel)        │
│   StateNode / BranchSpec / ChannelWrite / ChannelRead       │
├─────────────────────────────────────────────────────────────┤
│                  Pregel 执行引擎层                           │
│   Pregel.invoke() / stream()                                │
│   ├── SyncPregelLoop / AsyncPregelLoop                      │
│   ├── PregelRunner (并发任务调度)                           │
│   └── _algo: prepare_next_tasks / apply_writes             │
├─────────────────────────────────────────────────────────────┤
│               Channel 通信层 (状态通信总线)                  │
│   LastValue / Topic / BinaryOperatorAggregate               │
│   EphemeralValue / NamedBarrierValue / UntrackedValue       │
├─────────────────────────────────────────────────────────────┤
│               Checkpoint / 持久化层                          │
│   BaseCheckpointSaver → Memory / Postgres / SQLite         │
│   BaseStore (长期记忆) / BaseCache (节点缓存)               │
├─────────────────────────────────────────────────────────────┤
│                    序列化层 (Serde)                          │
│   JsonPlusSerializer / MsgPackSerializer / EncryptedSerde   │
└─────────────────────────────────────────────────────────────┘

2. Pregel 引擎核心对象关系
Pregel
 ├── nodes: dict[str, PregelNode]      # 节点集合 (actors)
 ├── channels: dict[str, BaseChannel]  # 通道集合 (状态总线)
 ├── checkpointer: BaseCheckpointSaver # 检查点持久化
 ├── store: BaseStore                  # 跨会话长期记忆
 ├── cache: BaseCache                  # 节点结果缓存
 └── trigger_to_nodes: Mapping         # channel→nodes 触发映射

PregelNode (Actor)
 ├── channels: str | list[str]         # 订阅的 channel
 ├── triggers: list[str]               # 触发该节点的 channel
 ├── bound: Runnable                   # 实际执行的可运行对象
 ├── writers: list[ChannelWrite]       # 写回的 channel
 ├── retry_policy: list[RetryPolicy]  # 重试策略
 └── cache_policy: CachePolicy        # 缓存策略

PregelLoop (执行上下文)
 ├── step: int                         # 当前超步编号
 ├── tasks: dict[str, PregelExecutableTask]  # 当前步的任务
 ├── channels: Mapping[str, BaseChannel]    # 当前 channel 快照
 ├── checkpoint: Checkpoint            # 当前检查点
 ├── status: Literal[input/pending/done/interrupt_before/interrupt_after]
 └── stream: StreamProtocol           # 流式输出协议

PregelExecutableTask
 ├── id: str                           # 任务唯一 ID (xxhash)
 ├── name: str                         # 节点名称
 ├── input: Any                        # 节点输入
 ├── proc: Runnable                    # 节点处理逻辑
 ├── writes: list[tuple]               # 写操作结果
 ├── triggers: list[str]               # 触发该任务的 channel
 └── path: tuple                       # 任务路径(用于子图追踪)

3. Channel 类型与职责
| Channel 类型 | 更新语义 | 典型用途 |
|---|---|---|
| LastValue | 最后一次写入覆盖 | 普通状态字段 |
| Topic | 多写累积/去重 | Send 消息队列、fanout |
| BinaryOperatorAggregate | reduce(op, updates) | 消息列表追加(add_messages) |
| EphemeralValue | 每步清空，单次读取 | 临时传递值 |
| NamedBarrierValue | 等待所有订阅者写入 | 并发 fanout 聚合 |
| UntrackedValue | 不持久化到 checkpoint | 大对象/临时运行时数据 |

4. Checkpoint 数据结构
Checkpoint (TypedDict)
 ├── v: int                     # 格式版本 (当前=4)
 ├── id: str                    # UUID6，单调递增，可排序
 ├── ts: str                    # ISO8601 时间戳
 ├── channel_values: dict       # 各 channel 的快照值
 ├── channel_versions: dict     # channel 版本号（用于变更检测）
 └── versions_seen: dict        # 各节点已处理的 channel 版本
                                # 用于触发判断（避免重复执行）

二、核心调用流程分析
流程一：graph.invoke(input) 完整调用链
用户调用 Pregel.invoke(input, config)
    │
    ▼
1. 初始化 SyncPregelLoop(input, config, checkpointer, ...)
    │   ├── 从 checkpointer 加载历史 checkpoint (get_tuple)
    │   ├── channels_from_checkpoint() 还原 channel 状态
    │   └── 设置 is_replaying / is_nested 等标志位
    │
    ▼
2. PregelLoop._first()  [仅首步]
    │   ├── 判断是否"续跑"(is_resuming)
    │   │   ├── input=None → resume after interrupt
    │   │   └── input=Command(resume=...) → 恢复中断
    │   ├── map_input() → 将 input 映射为 channel writes
    │   ├── apply_writes() → 写入 input channel，更新版本号
    │   └── _put_checkpoint(source="input") → 保存输入检查点
    │
    ▼
3. [循环] PregelLoop.tick()  — Pregel 超步 (Superstep)
    │   ├── prepare_next_tasks()  — Plan 阶段
    │   │   ├── 遍历 nodes，检查 triggers vs channel_versions
    │   │   ├── 跳过 versions_seen 已处理的 channel
    │   │   └── 返回 dict[task_id, PregelExecutableTask]
    │   ├── should_interrupt() — 检查 interrupt_before
    │   └── 若 tasks 非空返回 True，否则 status="done"
    │
   ▼
4. PregelRunner.tick(tasks)  — Execute 阶段
    │   ├── 单任务: 直接 run_with_retry(task)
    │   ├── 多任务: ThreadPoolExecutor 并发提交
    │   │   ├── 每个 task 在独立线程执行
    │   │   ├── concurrent.futures.wait(FIRST_COMPLETED)
    │   │   └── 任务完成即 yield (支持流式输出)
    │   └── 每个 task 完成后调用 PregelRunner.commit()
    │       └── loop.put_writes(task_id, writes)
    │           ├── 写入 checkpoint_pending_writes
    │           └── 异步持久化到 checkpointer (put_writes)
    │
    ▼
5. PregelLoop.after_tick()  — Update 阶段
    │   ├── apply_writes(checkpoint, channels, tasks)
    │   │   ├── 更新 channel.update(values) 
    │   │   ├── 递增 channel_versions[chan]
    │   │   ├── consume() 消耗 EphemeralValue
    │   │   └── 返回 updated_channels set
    │   ├── _emit("values", ...) → 输出 stream values
    │   ├── _put_checkpoint(source="loop") → 异步保存超步快照
    │   ├── should_interrupt() — 检查 interrupt_after
    │   └── step += 1
    │
    ▼
6. 重复步骤 3-5，直到:
    │   ├── tasks 为空 → status="done"
    │   ├── 触发 GraphInterrupt → status="interrupt_before/after"
    │   ├── step > stop → status="out_of_steps" (GraphRecursionError)
    │   └── 任意 task 抛出异常 → 向上冒泡
    │
    ▼
7. 结束
    ├── _suppress_interrupt() 处理中断 (非嵌套图抑制 GraphInterrupt)
    ├── durability=="exit" → 统一持久化所有 pending_writes
    └── 返回 read_channels(channels, output_keys)

流程二：Human-in-the-loop 中断与恢复
执行中 interrupt() 调用
    │
    ▼
task 内 raise GraphInterrupt([Interrupt(value=...)])
    │
    ▼
PregelRunner.commit(task, exc=GraphInterrupt)
    ├── put_writes(task_id, [(INTERRUPT, [interrupt_obj])])
    └── 其他并发 task 被取消
    │
    ▼
PregelLoop._suppress_interrupt()
    ├── 保存当前 checkpoint (含 INTERRUPT writes)
    ├── _emit("values") 输出最终状态
    └── 抑制异常，status="interrupt_before/after"
    │
    ▼
用户调用 graph.invoke(Command(resume=value), config)
    │
    ▼
PregelLoop._first()
    ├── 识别 is_resuming=True
    ├── map_command(cmd) → (NULL_TASK_ID, RESUME, value)
    ├── put_writes(NULL_TASK_ID, [(RESUME, value)])
    ├── 更新 versions_seen[INTERRUPT] = current_versions
    └── 重新触发被中断的节点继续执行

流程三：StateGraph 编译为 Pregel
StateGraph.compile()
    │
    ├── 1. 遍历所有 add_node() 注册的节点
    │   └── StateNode → PregelNode(bound, channels, triggers, writers)
    │
    ├── 2. 遍历所有 add_edge() / add_conditional_edges()
    │   ├── 普通边 → ChannelWriteEntry(next_node_channel)
    │   └── 条件边 → BranchSpec(condition_fn, then_map)
    │       └── 编译为 ChannelRead → condition → ChannelWrite
    │
    ├── 3. 根据 StateSchema 生成 channels
    │   ├── Annotated[list, add_messages] → BinaryOperatorAggregate
    │   ├── 普通字段 → LastValue
    │   └── RemainingSteps → 托管值 (ManagedValue)
    │
    ├── 4. validate_graph() 检查孤立节点/无效边
    │
    └── 5. 返回 CompiledStateGraph(Pregel)
        ├── input_channels = "__start__"
        ├── output_channels = "__end__"
        └── stream_channels = all non-reserved channels

流程四：流式输出 (stream_mode)
StreamProtocol: (checkpoint_ns, mode, value) → queue

支持模式:
  "values"    → 每个超步完成后输出完整 channel 状态
  "updates"   → 每个节点完成后输出其写入的 delta
  "messages"  → LLM token 级别流（通过 callback handler）
  "checkpoints" → 每次 checkpoint 保存时输出元数据
  "debug"     → tasks + checkpoints 的详细调试信息
  "custom"    → 用户通过 StreamWriter 手动推送

DuplexStream: 同时写入多个 StreamProtocol（父图+子图共享流）

流程五：重试机制 (RetryPolicy)
run_with_retry(task, retry_policy)
    │
    ├── 执行 task.proc.invoke(input, config)
    │
    ├── 捕获异常 → retry_on(exc) 判断是否需要重试
    │   ├── 匹配 exception_types 白名单
    │   └── 排除 GraphBubbleUp / GraphInterrupt (不重试)
    │
    ├── 指数退避: min(max_wait, initial * (backoff_factor ** attempt))
    │   └── 加 jitter 避免惊群效应
    │
    └── 超过 max_attempts → 重新抛出原始异常

三、关键设计决策总结
| 设计点 | 实现策略 |
|---|---|
| 超步同步 | 同一步内 channel 写入对其他 task 不可见（BSP 模型） |
| 版本号追踪 | channel_versions 递增，versions_seen 去重，避免重复触发 |
| 异步持久化 | checkpoint 保存通过 submit() 异步后台执行，不阻塞执行 |
| 子图命名空间 | checkpoint_ns = "parent:task_id" 树状 namespace 隔离 |
| 时间旅行 | 传入 checkpoint_id → 加载历史快照 → fork 分支执行 |
| 任务ID计算 | xxhash(node_name + triggers + step) 确定性哈希，幂等重放 |
| 中断恢复 | INTERRUPT/RESUME 作为特殊 channel 写，随 checkpoint 持久化 |