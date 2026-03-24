# LangChain核心组件-Middleware核心笔记
Middleware（中间件）是LangChain为Agent提供的**流程控制与安全保障组件**，通过在Agent运行循环的关键节点插入钩子（Hook），实现限流、容错、安全审批等控制逻辑，解决生产环境中Agent“失控、费钱、易宕机”的问题，是Agent从“能跑”到“稳跑”的关键。

## 一、核心认知：中间件的作用与运行循环
### 1. 核心定位
中间件是Agent的“安全气囊+流量控制器”，无需修改Agent核心逻辑，即可在关键节点注入自定义规则，支持：
- 成本控制（限制模型/工具调用次数）
- 容错兜底（模型宕机自动切换）
- 安全管控（关键操作人工审批）
- 流程优化（任务拆分、上下文裁剪）

### 2. Agent运行循环与中间件钩子
Agent的核心运行循环：`用户输入 → 模型推理 → 工具执行 → 模型推理 → ... → 最终回复`
中间件通过两类钩子嵌入循环：
| 钩子类型 | 核心节点 | 作用示例 |
|----------|----------|----------|
| 节点式（Node-style） | `before_agent`（Agent启动前）、`before_model`（模型调用前）、`after_model`（模型调用后）、`after_agent`（Agent结束后） | 模型调用前裁剪消息、Agent结束后记录日志 |
| 包装式（Wrap-style） | `wrap_model_call`（包裹模型调用）、`wrap_tool_call`（包裹工具调用） | 模型重试、工具限流、缓存复用 |

## 二、5个高频内置中间件（开箱即用）
### 1. Model Call Limit：限制模型调用次数（成本控制）
- 核心作用：防止Agent无限循环调用模型，导致token费用暴涨。
- 关键参数：
    - `run_limit`：单次交互最多调用模型次数；
    - `thread_limit`：单个会话（thread_id）累计最多调用次数；
    - `exit_behavior`：超限时行为（`continue`继续/`end`结束/`error`抛异常）。
- 示例：
```python
from langchain.agents.middleware import ModelCallLimitMiddleware
agent = create_agent(
    model="gpt-4o-mini",
    tools=tools,
    middleware=[ModelCallLimitMiddleware(run_limit=5, exit_behavior="error")]
)
```

### 2. Tool Call Limit：限制工具调用次数（安全防滥用）
- 核心作用：精细控制特定工具的调用频率，避免危险操作（如下单、转账）被滥用。
- 关键参数：
    - `tool_name`：指定限制的工具名；
    - `run_limit`：单次交互最多调用该工具次数；
    - `thread_limit`：单个会话累计最多调用次数。
- 示例（限制“下单”工具单次交互最多用1次）：
```python
middleware=[ToolCallLimitMiddleware(tool_name="purchase_item", run_limit=1)]
```

### 3. Model Fallback：模型宕机自动切换（容错兜底）
- 核心作用：主模型故障时，自动切换到备用模型，避免服务中断。
- 核心优势：支持**跨厂商切换**（如主模型Claude→备用GPT-4o→备用Gemini），不依赖单一供应商。
- 示例：
```python
middleware=[ModelFallbackMiddleware(fallbacks=["gpt-4o", "gemini-2.0-flash"])]
```

### 4. Human in the Loop：关键操作人工审批（安全管控）
- 核心作用：高危工具（发邮件、删数据、转账）调用前，触发人工审核，防止Agent误操作。
- 示例（发邮件前必须人工审批）：
```python
middleware=[HumanInTheLoopMiddleware(send_email=True, delete_data=True)]
```

### 5. ToDo List：复杂任务拆分与进度跟踪（流程优化）
- 核心作用：Agent处理复杂任务时，自动生成待办列表，按步骤执行并标记进度（`completed`/`in_progress`/`pending`）。
- 优势：Agent不易漏步骤，用户可实时查看进度，提升交互体验。
- 示例：
```python
middleware=[ToDoListMiddleware()]  # 自动新增write_todos工具
```

## 三、自定义中间件（满足个性化需求）
内置中间件不够用时，可通过“装饰器写法”（简单场景）或“类写法”（复杂场景）自定义，核心是通过钩子修改状态或控制流程。

### 1. 装饰器写法（节点式钩子，简单场景）
示例：对话消息超过50条时，强制结束对话：
```python
from langchain.agents.middleware import before_model, AgentState
from langchain.messages import AIMessage

@before_model(can_jump_to=["end"])  # 声明可跳转目标
def check_message_limit(state: AgentState, runtime) -> dict | None:
    if len(state["messages"]) >= 50:
        return {
            "messages": [AIMessage("对话太长了，请开启新对话～")],
            "jump_to": "end"  # 直接结束Agent
        }
    return None  # 返回None表示继续执行
```

### 2. 类写法（包装式钩子，复杂场景）
示例：模型调用失败自动重试（最多3次）：
```python
from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse

class RetryMiddleware(AgentMiddleware):
    def __init__(self, max_retries=3):
        self.max_retries = max_retries

    def wrap_model_call(self, request: ModelRequest, handler) -> ModelResponse:
        for attempt in range(self.max_retries):
            try:
                return handler(request)  # 执行模型调用
            except Exception as e:
                if attempt == self.max_retries - 1:
                    raise  # 最后一次重试失败则抛异常
                print(f"模型调用失败，重试{attempt+1}/{self.max_retries}")

# 加入Agent
agent = create_agent(model, tools, middleware=[RetryMiddleware(max_retries=3)])
```

### 3. 实用自定义案例：动态切换模型（省成本）
对话短（≤10条消息）用便宜小模型，对话长（>10条）用高性能大模型：
```python
from langchain.agents.middleware import wrap_model_call

small_model = init_chat_model("gpt-4.1-mini")  # 便宜模型
big_model = init_chat_model("gpt-4.1")         # 高性能模型

@wrap_model_call
def dynamic_model(request: ModelRequest, handler):
    model = big_model if len(request.messages) > 10 else small_model
    return handler(request.override(model=model))  # 替换模型
```

## 四、关键进阶：多中间件执行顺序与流程跳转
### 1. 多中间件执行顺序（洋葱模型）
同时配置多个中间件时，执行逻辑遵循：
- `before`钩子：按中间件列表顺序执行（1→2→3）；
- `wrap`钩子：嵌套执行（1包裹2，2包裹3，3包裹核心逻辑）；
- `after`钩子：按中间件列表反序执行（3→2→1）。
- 建议：安全兜底（如ModelFallback）放最前，业务逻辑放最后。

### 2. 流程跳转（Agent Jumps）
中间件可通过`jump_to`改变Agent运行流程，需在装饰器中声明`can_jump_to`：
- 跳转目标：
    - `"end"`：直接结束Agent；
    - `"tools"`：跳到工具执行环节；
    - `"model"`：跳回模型重新推理。
- 示例（检测到敏感内容直接结束）：
```python
@after_model
@hook_config(can_jump_to=["end"])
def check_sensitive_content(state):
    if "敏感内容" in state["messages"][-1].content:
        return {"messages": [AIMessage("该请求无法处理")], "jump_to": "end"}
    return None
```

## 五、生产环境中间件组合推荐（按优先级排序）
根据实际场景筛选，核心原则：**安全兜底→成本控制→业务优化**：
```python
agent = create_agent(
    model="claude-sonnet-4-20250514",
    tools=tools,
    middleware=[
        # 1. 容错兜底：模型宕机不中断服务
        ModelFallbackMiddleware(fallbacks=["gpt-4o"]),
        # 2. 成本控制：限制模型调用次数
        ModelCallLimitMiddleware(run_limit=10, thread_limit=100),
        # 3. 工具安全：限制高危工具滥用
        ToolCallLimitMiddleware(tool_name="purchase", run_limit=1),
        # 4. 人工审批：关键操作防误触
        HumanInTheLoopMiddleware(send_email=True, delete_data=True),
        # 5. 上下文优化：防止消息超限
        SummarizationMiddleware(model="gpt-4.1-mini", trigger=("proportion", 0.7))
    ],
    checkpointer=PostgresSaver(DB_URI)  # 记忆持久化
)
```

## 核心总结
1. 中间件的核心价值是**无侵入式扩展Agent能力**，无需修改核心逻辑即可添加安全、成本、容错控制；
2. 内置中间件覆盖80%生产场景，优先直接使用（如ModelFallback、HumanInTheLoop）；
3. 自定义中间件支持灵活扩展，复杂场景用类写法（wrap钩子），简单场景用装饰器写法（node钩子）；
4. 多中间件配置需注意执行顺序，安全和容错类中间件放最外层，避免被其他中间件拦截；
5. 中间件是Agent落地生产的必备组件，合理组合可大幅提升系统稳定性和安全性。

是否需要我针对某个中间件（如自定义重试中间件）提供更详细的代码示例？