# agent.run() 方法工具调用判断逻辑分析

## 概述

`agent.run()` 方法通过以下流程判断是否要调用工具：

## 核心判断逻辑

### 1. 获取 LLM 响应（第 384 行）

```python
response = await self.llm.generate(messages=self.messages, tools=tool_list)
```

- LLM 客户端接收到可用的 `tools` 列表
- LLM 根据提示词和工具列表自行决定是否需要调用工具
- 返回的 `response` 包含 `tool_calls` 字段

### 2. 检查是否有工具调用（第 432 行）

```python
if not response.tool_calls:
    return response.content
```

**这是核心判断点**：
- 如果 `response.tool_calls` 为 `None` 或空列表 → 任务完成，直接返回文本响应
- 如果 `response.tool_calls` 包含内容 → 需要执行工具调用

### 3. 执行工具调用（第 446 行）

```python
for tool_call in response.tool_calls:
    function_name = tool_call.function.name
    arguments = tool_call.function.arguments
    
    # 检查工具是否存在
    if function_name not in self.tools:
        # 工具不存在，返回错误
    else:
        # 执行工具
        result = await tool.execute(**arguments)
```

## 工具定义传递流程

### 工具从 Agent 到 LLM 的传递

1. **Agent 初始化时**（第 60 行）
   ```python
   self.tools = {tool.name: tool for tool in tools}
   ```
   - 将工具列表转换为字典，以工具名为键

2. **每次循环准备工具列表**（第 377 行）
   ```python
   tool_list = list(self.tools.values())
   ```
   - 将工具字典转换回列表

3. **传递给 LLM 客户端**（第 384 行）
   ```python
   response = await self.llm.generate(messages=self.messages, tools=tool_list)
   ```

4. **LLM 客户端转换工具格式**（anthropic_client.py 第 83-112 行）
   ```python
   def _convert_tools(self, tools: list[Any]) -> list[dict[str, Any]]:
       result = []
       for tool in tools:
           if hasattr(tool, "to_schema"):
               result.append(tool.to_schema())
       return result
   ```
   - 将 Tool 对象转换为 LLM 可识别的 schema 格式（包含 name, description, input_schema）

## 工具调用判断的本质

**工具调用是由 LLM 决定的，而非 Agent 代码硬编码判断**

1. **LLM 的推理决策**：
   - LLM 接收系统提示词（包含任务描述）
   - LLM 接收可用工具列表（每个工具的 name, description, parameters）
   - LLM 根据用户问题和当前上下文，自主判断是否需要调用工具

2. **Agent 的响应逻辑**：
   - 检查 LLM 返回的 `tool_calls` 字段是否为空
   - 不为空 → 遍历执行所有工具调用
   - 为空 → 任务完成，返回 LLM 的文本响应

## 工具 Schema 格式

Tool 对象的 `to_schema()` 方法需要返回如下格式：

```python
{
    "name": "tool_name",
    "description": "Tool description",
    "input_schema": {
        "type": "object",
        "properties": {
            "arg1": {"type": "string", "description": "..."},
            "arg2": {"type": "integer", "description": "..."}
        },
        "required": ["arg1", "arg2"]
    }
}
```

## 循环执行流程

```
┌─────────────────────────────────────────────┐
│              开始执行 step                  │
├─────────────────────────────────────────────┤
│  1. 检查取消和 token 限制                      │
│  2. 准备工具列表 tool_list                   │
│  3. 调用 LLM: response = await llm.generate()│
│  4. 检查 response.tool_calls                │
│     ├─ 为空：返回文本，结束                  │
│     └─ 不为空：进入工具执行流程               │
│  5. 遍历执行每个 tool_call                  │
│  6. 将工具结果添加到消息历史                 │
│  7. step++，回到步骤 1                       │
└─────────────────────────────────────────────┘
```

## 关键设计要点

1. **LLM 自主决策**：工具调用决策完全由 LLM 根据上下文和可用工具自主判断

2. **工具列表动态传递**：每次 LLM 调用前都重新准备并传递完整的工具列表

3. **工具执行容错**：
   - 工具不存在时返回错误结果
   - 工具执行异常时捕获并转换为错误结果
   - 错误结果会反馈给 LLM，LLM 可以选择重试或改用其他方式

4. **多工具调用支持**：一次 LLM 响应可以包含多个工具调用，Agent 会顺序执行
