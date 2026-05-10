动作 4：这才是 nanobot 真正的架构问题
可以记一笔——AgentLoop 在 _process_message 之前，没有做"prompt + maxTokens ≤ contextWindowTokens"的 pre-check 与强制压缩，导致刚加载完历史就直接报 400。这比"反复读同一文件"更致命，也更容易被忽视，因为它只在 session 累积到一定程度后才会复现。