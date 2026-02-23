# Agent Framework 架构设计

## 总体架构

三层架构，依赖方向单向向下。上层通过闭包和回调注入关注点，下层不依赖上层的任何类型。

```
Layer 3: Application (chat_routes.py)
  │  构建 hook 闭包 (credit消费, 消息持久化)
  │  构建 ProviderResolver 闭包 (信用检查→选provider)
  │  Session load/save, rate limiting, rollback
  │
  ▼
Layer 2: Agent Runtime (app/agent/)
  │  AgentState (request-scoped, 无session概念)
  │  run_agent_turn(resolve_provider, state, tools, transform, hooks)
  │  通过 hooks 回调应用层，自身不依赖任何DB/Service类型
  │
  ▼
Layer 1: LLM Abstraction (app/llm/)
  │  provider.complete(messages, tools, ...)
  │  返回统一的 LLMResponse + AssistantMessage
```

---

## Layer 1: LLM Abstraction (`app/llm/`)

统一的、provider 无关的 LLM 接口层。所有 provider 内部完成格式转换，调用方永远不感知 Anthropic/OpenAI/Ollama 的 API 差异。

### 模块一览

| 模块 | 文件 | 职责 |
|------|------|------|
| **类型系统** | `types.py` | `TextBlock`, `ToolUseBlock`, `ToolResultBlock` 内容块；`UserMessage`, `AssistantMessage` 消息类型；`Usage`, `StopReason`, `ToolDefinition`, `LLMResponse` |
| **流式事件** | `events.py` | `AssistantMessageEvent` 联合类型：`TextStart/Delta/End`, `ToolCallStart/Delta/End`, `UsageEvent`, `DoneEvent`, `ErrorEvent` |
| **Provider 协议** | `provider.py` | `LLMProvider` ABC，定义 `complete()` + `stream()` + `health_check()` 接口，每个实现携带 `provider_name` 和 `model_id` |
| **Provider 实现** | `providers/claude.py` | Claude provider（AsyncAnthropic SDK） |
| | `providers/ollama.py` | Ollama provider（httpx，自动生成 tool UUID） |
| | `providers/openai_compat.py` | OpenAI 兼容 provider（AsyncOpenAI，含 JSON repair 逻辑） |
| **Provider 工厂** | `factory.py` | `create_provider()` 创建实例，`get_provider()` 带单例缓存 |
| **错误体系** | `errors.py` | `LLMError` → `RateLimitError`（429）/ `OverloadedError`（529），携带 `is_retryable` 标志 |
| **模型注册表** | `models.py` | `ModelInfo` 元数据（cost/capability），按 model ID 查询 |

### 核心设计特征

- 每条 `AssistantMessage` 携带 `model` + `provider` 元数据，支持 **Context Handoff**（跨 provider 回放历史）
- 所有格式转换封装在 provider 内部，调用方只操作统一的 `Message[]`
- 错误类型携带 `is_retryable` 标志，上层可据此决定重试策略

### 关键类型

```python
# 内容块
TextBlock(text: str)
ToolUseBlock(id: str, name: str, input: dict)
ToolResultBlock(tool_use_id: str, content: str, is_error: bool)
ContentBlock = TextBlock | ToolUseBlock | ToolResultBlock

# 消息
UserMessage(content: str | list[ContentBlock])
AssistantMessage(
    content: list[ContentBlock],
    stop_reason: StopReason,
    model: str,       # 哪个模型生成的（Context Handoff 用）
    provider: str,    # 哪个 provider（Context Handoff 用）
    usage: Usage,     # 本条消息的 token 用量
)

# Provider 协议
class LLMProvider(ABC):
    provider_name: str
    model_id: str
    async def complete(messages, *, system_prompt, tools, max_tokens, temperature) -> LLMResponse
    async def stream(messages, *, system_prompt, tools, max_tokens, temperature) -> AsyncIterator[AssistantMessageEvent]
    async def health_check() -> bool
```

---

## Layer 2: Agent Runtime (`app/agent/`)

无状态的 agent 执行引擎。不感知 DB/Session/Credit，通过 hooks 和 resolver 回调与应用层交互。

### 模块一览

| 模块 | 文件 | 职责 |
|------|------|------|
| **消息类型** | `types.py` | `AgentMessageType`（7种）；`AgentMessage` 是 LLM Message 的超集；`AgentEvent` 联合类型（8种事件） |
| **状态容器** | `state.py` | `AgentState` — 纯数据容器，持有 `messages[]`, `steering_queue[]`, `followup_queue[]`, turn 计数, usage 累计 |
| **Agent Loop** | `loop.py` | `run_agent_turn()` — 核心循环，编排全部执行流程 |
| **工具注册表** | `tool_registry.py` | `ToolRegistry` — Server/Client 工具分类与调度 |
| **消息转换器** | `converters.py` | AgentMessage ↔ LLM Message ↔ DB Schema 三方桥接 |
| **上下文变换** | `context_transform.py` | 可插拔的消息压缩钩子 |
| **生命周期钩子** | `hooks.py` | `TurnHooks` — 应用层注入的回调（credit 消费、消息持久化、错误通知） |

### 四大设计能力

#### 1. Context Handoff — 动态 Provider 切换

`run_agent_turn()` 接收 `ProviderResolver`（而非固定 provider），每轮调用一次。支持 mid-loop 切换（如优先积分耗尽后 Sonnet→Haiku）。

```python
ProviderResolver = Callable[[], Awaitable[LLMProvider]]

# 应用层构建闭包
async def _resolve_provider() -> LLMProvider:
    balance = await entitlement_service.get_credit_balance_v2(user_id)
    if balance.get("priority", 0) >= 1:
        return get_provider(model="claude-sonnet-4-20250514")
    else:
        return get_provider(model="claude-haiku-4-20250514")
```

每条 `AgentMessage` 记录了产生它的 `model` 和 `provider`，`AgentProviderSwitchEvent` 会在切换发生时触发。

#### 2. Message Queue — 用户消息注入

两种队列，解决 agent loop 运行期间用户发送新消息的场景：

| 队列 | 时机 | 行为 |
|------|------|------|
| `steering_queue` | 工具执行之间检查 | 跳过剩余工具，注入新意图，触发新 LLM 调用 |
| `followup_queue` | loop 自然结束时检查 | 追加消息，自动开始新轮次 |

```python
# AgentState 提供队列操作
state.enqueue_steering(msg)    # 外部线程注入
state.enqueue_followup(msg)    # 外部线程注入
state.drain_steering() -> list  # loop 内部消费
state.drain_followup() -> list  # loop 内部消费
```

#### 3. Context Management — 可插拔上下文变换

每次 LLM 调用前，`ContextTransform` 回调可以压缩/裁剪/注入消息历史：

```python
ContextTransform = Callable[[list[Message]], Awaitable[list[Message]]]

transform = create_context_transform(
    resolve_provider=resolve_provider,  # 可用便宜模型做摘要
    max_context_tokens=32768,
    threshold_ratio=0.80,
)
```

包装了已有的 `context_manager.py` 逻辑（分割点检测、tool_use/tool_result 配对保护、CJK token 估算）。

#### 4. Message Split — AgentMessage vs LLM Message 分离

`AgentMessage` 是应用层超集，`convert_to_llm()` 过滤为 LLM 可理解的子集：

| AgentMessageType | 发送给 LLM？ | 用途 |
|------------------|-------------|------|
| `USER` | ✅ | 用户消息 |
| `ASSISTANT` | ✅ | LLM 响应 |
| `TOOL_RESULT` | ✅ | 工具执行结果 |
| `SUMMARY` | ✅ (作为 UserMessage) | 压缩后的历史摘要 |
| `CONTEXT_NOTE` | ❌ | 注入上下文（如「已切换到标准队列」） |
| `SYSTEM_EVENT` | ❌ | 积分警告、限速通知 |
| `TOOL_LOG` | ❌ | 服务端工具执行日志 |

### Agent Loop 执行流程

```python
async def run_agent_turn(
    resolve_provider: ProviderResolver,
    state: AgentState,
    tool_registry: ToolRegistry,
    context_transform: Optional[ContextTransform] = None,
    hooks: Optional[TurnHooks] = None,
) -> list[AgentEvent]
```

每轮执行：

```
1. resolve_provider()        → 动态选择 provider（Context Handoff）
2. convert_to_llm()          → AgentMessage[] 过滤为 Message[]（Message Split）
3. context_transform()       → 压缩/裁剪历史（Context Management）
4. provider.complete()       → 调用 LLM
5. _add_message(ASSISTANT)   → 存储响应 + on_message hook
6. on_turn_end hook          → 积分消费，返回 bool 决定是否继续
7. 检查 wants_tool_use       → 无工具调用则检查 followup_queue
8. 遍历 tool_uses:
   a. drain_steering()       → 检查 steering 中断（Message Queue）
   b. classify_tool()        → SERVER 则执行，CLIENT 则暂停返回
   c. _add_message(TOOL_RESULT) → 存储结果 + on_message hook
9. 回到步骤 1（下一轮）
```

### 生命周期钩子（TurnHooks）

应用层通过闭包注入，agent runtime 无需依赖 DB/Service 类型：

```python
@dataclass
class TurnHooks:
    on_turn_end: Optional[OnTurnEnd] = None    # (turn, usage, model, provider) → bool
    on_message: Optional[OnMessage] = None     # (AgentMessage) → None
    on_error: Optional[OnError] = None         # (error_str, is_retryable) → None
```

| 钩子 | 触发时机 | 用途 | 失败行为 |
|------|---------|------|---------|
| `on_turn_end` | LLM 响应后，工具执行前 | 积分消费，返回 False 停止 loop | 抛异常 → AgentErrorEvent，终止 loop |
| `on_message` | ASSISTANT 或 TOOL_RESULT 加入 state 时 | 消息实时持久化到 DB | 抛异常 → 日志警告，loop 继续（消息已在 state 中） |
| `on_error` | LLM 错误或 provider 解析失败时 | 错误通知/记录 | 抛异常 → 静默忽略（不遮蔽原始错误） |

### AgentEvent 事件类型

`run_agent_turn()` 返回事件列表，供应用层处理：

| 事件 | 含义 |
|------|------|
| `AgentTurnStartEvent` | 新轮次开始（含 turn_number） |
| `AgentTurnEndEvent` | 轮次结束（含 stop_reason, model, provider） |
| `AgentToolExecutionEvent` | 服务端工具执行完成（含 tool_name, result, duration_ms） |
| `AgentClientToolRequestEvent` | 需要客户端执行工具（loop 暂停，返回 tool_calls） |
| `AgentSteeringEvent` | Steering 中断发生（含 skipped_tools 列表） |
| `AgentProviderSwitchEvent` | Provider 发生切换（含 from_model, to_model） |
| `AgentDoneEvent` | Loop 正常结束（含 final_text, total_turns, total_usage） |
| `AgentErrorEvent` | 错误终止（含 error, is_retryable） |

### 消息转换器（Converters）

| 函数 | 方向 | 用途 |
|------|------|------|
| `convert_to_llm()` | AgentMessage[] → Message[] | 过滤 app-only 类型，发给 LLM |
| `schema_messages_to_agent()` | DB Schema → AgentMessage[] | 从数据库加载历史 |
| `agent_message_to_schema()` | AgentMessage → DB Schema | 存储到数据库 |
| `llm_response_to_chat_response()` | LLMResponse → ChatResponse | 返回给 Flutter 客户端 |
| `schema_tools_to_llm()` | Schema Tool[] → ToolDefinition[] | Flutter 工具定义转换 |
| `dicts_to_content_blocks()` | dict[] → ContentBlock[] | 原始字典转类型化内容块 |

---

## 工具注册表（ToolRegistry）

支持混合执行模式 — 同一次 LLM 调用的工具结果中，部分在后端执行，部分由客户端执行：

```python
class ToolExecutionMode(Enum):
    SERVER = "server"    # 后端执行（如 semantic_search_photos）
    CLIENT = "client"    # Flutter 执行（如 show_photos_to_user）

class ToolRegistry:
    register_server_tool(name, description, input_schema, handler)
    register_client_tool(name, description, input_schema)
    merge_client_tools(tools: list[ToolDefinition])  # 从 Flutter 请求合并
    classify_tool(name) -> ToolExecutionMode
    get_handler(name) -> callable | None
    get_all_tool_definitions() -> list[ToolDefinition]
    has_tools() -> bool
```

---

## AgentState 详细结构

```python
@dataclass
class AgentState:
    # 核心状态
    system_prompt: str
    messages: list[AgentMessage]           # 完整对话历史（app-layer 超集）
    max_turns: int = 10
    max_tokens: int = 4096
    temperature: float = 0.7

    # 运行时计数
    current_turn: int = 0
    total_usage: Usage = Usage(0, 0)

    # 消息队列（线程安全注入点）
    steering_queue: list[AgentMessage]     # 中断当前工具执行
    followup_queue: list[AgentMessage]     # loop 结束后追加

    # 方法
    add_message(msg) -> None
    add_usage(usage) -> None
    drain_steering() -> list[AgentMessage]
    drain_followup() -> list[AgentMessage]
```

**关键特征**：AgentState 是 request-scoped 的纯数据容器，每次 HTTP 请求从 DB 重建，请求结束后丢弃。DB 是 session 历史的唯一真实来源。

---

## 文件清单

### Layer 1: LLM Abstraction

```
backend/app/llm/
├── __init__.py              # 公共导出
├── types.py                 # Message, ContentBlock, ToolDefinition, LLMResponse, Usage
├── events.py                # AssistantMessageEvent 流式事件联合类型
├── models.py                # ModelInfo 模型元数据注册表
├── errors.py                # LLMError → RateLimitError / OverloadedError
├── provider.py              # LLMProvider ABC
├── factory.py               # Provider 工厂 + 单例缓存
└── providers/
    ├── __init__.py
    ├── claude.py            # Claude provider (AsyncAnthropic)
    ├── ollama.py            # Ollama provider (httpx)
    └── openai_compat.py     # OpenAI 兼容 provider (AsyncOpenAI)
```

### Layer 2: Agent Runtime

```
backend/app/agent/
├── __init__.py              # 公共导出
├── types.py                 # AgentMessageType, AgentMessage, AgentEvent 联合类型
├── state.py                 # AgentState 数据容器（含消息队列）
├── tool_registry.py         # ToolRegistry (Server/Client 分类)
├── loop.py                  # run_agent_turn() 核心循环
├── converters.py            # AgentMessage ↔ LLM Message ↔ DB Schema 桥接
├── context_transform.py     # 可插拔上下文压缩钩子
└── hooks.py                 # TurnHooks 生命周期回调
```

### 测试

```
backend/tests/
├── llm/
│   ├── test_types.py              # 消息构造、内容块辅助方法
│   ├── test_events.py             # 事件类型判别
│   ├── test_claude_provider.py    # Mock Anthropic 客户端
│   ├── test_ollama_provider.py    # Mock httpx
│   └── test_openai_compat.py      # Mock AsyncOpenAI, JSON repair
└── agent/
    ├── test_tool_registry.py      # 工具分类、handler 查找、merge
    ├── test_loop.py               # Agent loop 全场景测试 (含 TurnHooks 9 cases)
    ├── test_converters.py         # 三方转换往返测试
    └── test_context_transform.py  # 压缩钩子集成测试
```
