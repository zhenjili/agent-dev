# LangChain 压缩对话历史：数据流设计指南

在 LangChain 里，"压缩对话历史"的核心思路是把以下两部分**分开管理**：

- **给 LLM**：只给「摘要 + 最近 N 轮（原文）+ 本轮输入 + 必要的工具结果」
- **给前端 UI**：通常展示「完整原始对话（全量 message history）」；摘要一般不直接展示（除非你做 debug/"记忆面板"）

---

## 1. 多轮对话里，LangChain 到底"把什么给 LLM"？

以最常见的 **Summary + Window**（旧内容摘要 + 最近 N 轮原文）为例，每一轮调用 LLM 时，prompt 组成通常是：

| 部分 | 说明 |
|------|------|
| **System** | 角色/规则/工具说明（稳定 prefix，适合 Claude prompt caching） |
| **Summary（摘要）** | 把很久以前的对话压成一段短文本 |
| **Recent window（最近 N 轮原文）** | 比如最近 6~12 条 message（含 user/assistant） |
| **Current user input** | 本轮用户输入 |
| **Tool outputs（可选）** | 工具结果（通常精炼后写入） |

---

## 2. 常见 Memory 策略

| 策略 | 说明 | 典型类名 |
|------|------|----------|
| **Window memory** | 只保留最近 N 轮 | `ConversationBufferWindowMemory` |
| **Summary memory** | 一直维护一个摘要 | `ConversationSummaryMemory` |
| **Summary + Window** | 摘要 + 最近 N 轮（推荐） | `ConversationSummaryBufferMemory` |
| **RunnableWithMessageHistory** | 按 `thread_id` 取历史、写回历史，多会话封装 | `RunnableWithMessageHistory` |

> 你完全可以不用现成 memory 类，自己实现一个"summary + window"存储层，效果更可控（尤其是你还要做 async tool call、message queue）。

---

## 3. 把什么展示给前端用户界面？

### UI 常见展示策略

| 内容类型 | 是否展示 | 展示方式 |
|----------|----------|----------|
| user/assistant 自然语言消息 | 展示 | 正常聊天气泡 |
| tool call / tool result | 可选 | 折叠卡片（debug）或隐藏，只展示"我已查到结果：xxx" |
| agent scratchpad / chain-of-thought | 不展示 | — |
| Execution state（执行状态） | 内部维护 | tool futures、pending actions、队列位置、checkpoint |

### 当 async tool result 回来时

1. 更新 Execution state
2. 把 tool result 写入 transcript（给 UI）
3. 决定是否写入 context pack（给 LLM）：通常会写"精炼版结果"，大 payload 先压缩再写

---

## 4. 核心数据结构（Thread State）

每个 `thread_id` 存三样东西：

```python
state = {
    "transcript": [...],  # 全量原始消息 → 给 UI
    "summary":    "...",  # 历史摘要     → 给 LLM context
    "window":     [...],  # 最近 N 轮    → 给 LLM context
}
```

---

## 5. 迷你伪代码（核心思想）

```python
# 每个 thread_id 存三样
state = {
    "transcript": [... full messages ...],  # UI
    "summary": "....",                      # LLM context
    "window": [... last N messages ...],    # LLM context
}

def build_llm_messages(thread_id, user_input):
    s = load_state(thread_id)
    return [
        {"role": "system",  "content": SYSTEM_PREFIX},
        {"role": "system",  "content": f"Conversation summary:\n{s['summary']}"},
        *s["window"],
        {"role": "user",    "content": user_input},
    ]

def after_turn(thread_id, new_messages):
    s = load_state(thread_id)

    # 1) UI 全量保存
    s["transcript"].extend(new_messages)

    # 2) 更新 LLM summary（用一个 summarizer LLM）
    old = s["window"][:-N]           # 超出窗口的旧消息
    if old:
        s["summary"] = summarize(
            existing_summary=s["summary"],
            messages=old
        )

    # 3) 更新 window（只保留最近 N 轮）
    s["window"] = s["window"][-N:]

    save_state(thread_id, s)
```

---

## 6. 可扩展的设计问题

如果你告诉我以下信息，可以把上面这个流程改成更贴近你现有架构的版本：

- [ ] **使用的框架**：LangChain Python 还是 LangChain.js？
- [ ] **UI 展示偏好**：tool call 细节是折叠卡片还是完全隐藏？
- [ ] **需要的产出**：
  - Thread state schema（JSON）
  - Async tool result 插队时的状态机
  - Summary 的更新时机（每轮 / 每 K 轮 / 达到 token 上限才更新）
