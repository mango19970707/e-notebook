# LangChain核心组件-概述核心笔记
本课围绕 7 个核心组件：Model、Tools、System Prompt、Streaming、Memory、Middleware、Structured Output。目标是先明确每个组件负责什么，再给出可直接复用的完整代码。

## 一、回顾
前五课已经完成 Agent 入门闭环：基础概念、工具调用、真实 API、记忆、多步任务与 Web 应用。本课开始系统拆解 LangChain Agent 的核心组件。

## 二、Model（模型）
模型是 Agent 的推理引擎。常见做法分为静态模型和动态模型切换。

### 1）静态模型
```python
from langchain.agents import create_agent

agent = create_agent("openai:gpt-5", tools=tools)
```

```python
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI

model = ChatOpenAI(
    model="gpt-5",
    temperature=0.1,
    max_tokens=1000,
    timeout=30,
    # ... (other params)
)

agent = create_agent(model, tools=tools)
```

### 2）动态模型切换
```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse

basic_model = ChatOpenAI(model="gpt-4.1-mini")
advanced_model = ChatOpenAI(model="gpt-4.1")

@wrap_model_call
def dynamic_model_selection(request: ModelRequest, handler) -> ModelResponse:
    """Choose model based on conversation complexity."""
    message_count = len(request.state["messages"])
    if message_count > 10:
        # Use an advanced model for longer conversations
        model = advanced_model
    else:
        model = basic_model

    return handler(request.override(model=model))

agent = create_agent(
    model=basic_model,  # Default model
    tools=tools,
    middleware=[dynamic_model_selection],
)
```

## 三、Tools（工具）
工具是 Agent 与外部能力交互的执行层，支持静态注册、动态控制、错误兜底。

### 1）静态工具
```python
from langchain.tools import tool
from langchain.agents import create_agent

@tool
def search(query: str) -> str:
    """Search for information."""
    return f"Results for: {query}"

@tool
def get_weather(location: str) -> str:
    """Get weather information for a location."""
    return f"Weather in {location}: Sunny, 72°F"

agent = create_agent(model, tools=[search, get_weather])
```

### 2）动态工具：过滤预注册工具
```python
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from typing import Callable

@wrap_model_call
def filter_tools(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    """Filter tools based on user permissions."""
    user_role = request.runtime.context.user_role

    if user_role == "admin":
        # Admins get all tools
        tools = request.tools
    else:
        # Regular users get read-only tools
        tools = [t for t in request.tools if t.name.startswith("read_")]

    return handler(request.override(tools=tools))

agent = create_agent(
    model="gpt-4o",
    tools=[read_data, write_data, delete_data],  # All tools pre-registered
    middleware=[filter_tools],
)
```

### 3）动态工具：运行时注册并执行
```python
from langchain.tools import tool
from langchain.agents import create_agent
from langchain.agents.middleware import AgentMiddleware, ModelRequest
from langchain.agents.middleware.types import ToolCallRequest

# A tool that will be added dynamically at runtime
@tool
def calculate_tip(bill_amount: float, tip_percentage: float = 20.0) -> str:
    """Calculate the tip amount for a bill."""
    tip = bill_amount * (tip_percentage / 100)
    return f"Tip: ${tip:.2f}, Total: ${bill_amount + tip:.2f}"

class DynamicToolMiddleware(AgentMiddleware):
    """Middleware that registers and handles dynamic tools."""

    def wrap_model_call(self, request: ModelRequest, handler):
        # Add dynamic tool to the request
        # This could be loaded from an MCP server, database, etc.
        updated = request.override(tools=[*request.tools, calculate_tip])
        return handler(updated)

    def wrap_tool_call(self, request: ToolCallRequest, handler):
        # Handle execution of the dynamic tool
        if request.tool_call["name"] == "calculate_tip":
            return handler(request.override(tool=calculate_tip))
        return handler(request)

agent = create_agent(
    model="gpt-4o",
    tools=[get_weather],  # Only static tools registered here
    middleware=[DynamicToolMiddleware()],
)

# The agent can now use both get_weather AND calculate_tip
result = agent.invoke({
    "messages": [{"role": "user", "content": "Calculate a 20% tip on $85"}]
})
```

### 4）工具错误处理
```python
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_tool_call
from langchain.messages import ToolMessage

@wrap_tool_call
def handle_tool_errors(request, handler):
    """Handle tool execution errors with custom messages."""
    try:
        return handler(request)
    except Exception as e:
        # Return a custom error message to the model
        return ToolMessage(
            content=f"Tool error: Please check your input and try again. ({str(e)})",
            tool_call_id=request.tool_call["id"],
        )

agent = create_agent(
    model="gpt-4.1",
    tools=[search, get_weather],
    middleware=[handle_tool_errors],
)
```

```python
[
    ...,
    ToolMessage(
        content="Tool error: Please check your input and try again. (division by zero)",
        tool_call_id="...",
    ),
    ...,
]
```

## 四、System Prompt（系统提示词）
系统提示词用于约束角色、语气、输出边界。可静态定义，也可按上下文动态生成。

### 1）静态 System Prompt
```python
agent = create_agent(
    model,
    tools,
    system_prompt="你是一个专业的旅行顾问，使用中文回答问题，回答要简洁。",
)
```

### 2）SystemMessage 高级配置（含缓存控制）
```python
from langchain.agents import create_agent
from langchain.messages import SystemMessage, HumanMessage

literary_agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    system_prompt=SystemMessage(
        content=[
            {
                "type": "text",
                "text": "You are an AI assistant tasked with analyzing literary works.",
            },
            {
                "type": "text",
                "text": "<the entire contents of 'Pride and Prejudice'>",
                "cache_control": {"type": "ephemeral"},
            },
        ]
    ),
)

result = literary_agent.invoke(
    {"messages": [HumanMessage("Analyze the major themes in 'Pride and Prejudice'.")]}
)

# cache_control 字段 {"type": "ephemeral"} 告诉 Anthropic 缓存该内容块，
# 降低重复请求同一系统提示词时的延迟和成本。
```

### 3）动态 System Prompt
```python
from typing import TypedDict
from langchain.agents import create_agent
from langchain.agents.middleware import dynamic_prompt, ModelRequest

class Context(TypedDict):
    user_role: str

# @dynamic_prompt 装饰器创建中间件，该中间件根据模型请求生成系统提示词
@dynamic_prompt
def user_role_prompt(request: ModelRequest) -> str:
    """根据用户角色生成不同提示词"""
    user_role = request.runtime.context.get("user_role", "普通用户")
    if user_role == "专家":
        return "你是一个技术专家，提供详细的技术分析。"
    elif user_role == "新手":
        return "你是一个耐心的讲师，用简单易懂的语言解释。"
    return "你是一个友好的助手。"

agent = create_agent(
    model="gpt-4o",
    tools=tools,
    middleware=[user_role_prompt],
    context_schema=Context,
)

# 根据上下文动态设置系统提示词
result = agent.invoke(
    {"messages": [{"role": "user", "content": "解释机器学习"}]},
    context={"user_role": "专家"},
)
```

## 五、Streaming（流式输出）
流式输出用于实时展示 Agent 的思考进度和工具调用过程。

```python
for chunk in agent.stream(
    {
        "messages": [
            {"role": "user", "content": "Search for AI news and summarize the findings"}
        ]
    },
    stream_mode="values",
):
    # Each chunk contains the full state at that point
    latest_message = chunk["messages"][-1]

    if latest_message.content:
        print(f"Agent: {latest_message.content}")
    elif latest_message.tool_calls:
        print(f"Calling tools: {[tc['name'] for tc in latest_message.tool_calls]}")
```

```python
Calling tools: ['get_weather']
Agent: 北京今天天气晴朗，温度 23°C。建议穿厚外套...
```

## 六、Memory（记忆）
默认对话历史由 message state 维护。业务场景通常需要扩展自定义状态，并在每轮推理前注入状态信息。

```python
from typing import Any, Optional
from typing_extensions import TypedDict
from langchain.agents import AgentState, create_agent, tool
from langchain.agents.middleware import AgentMiddleware

# 1. 定义自定义状态：这就是我们的“记账本”
class FlightBookingState(AgentState):
    destination: Optional[str] = None
    departure_date: Optional[str] = None

# 2. 定义一个工具：当模型发现用户提到了目的地或日期，就更新状态
@tool
def save_flight_info(destination: str = None, departure_date: str = None):
    """Save flight destination or departure date."""
    # 模型通过这个工具将信息“写入”状态
    return f"Status updated: {destination=}, {departure_date=}"

# 3. 定义中间件：在每一轮开始前，把已知信息告诉模型
class FlightManagerMiddleware(AgentMiddleware):
    state_schema = FlightBookingState

    def before_model(self, state: FlightBookingState, runtime) -> dict[str, Any] | None:
        # 核心逻辑：自动在 Prompt 后加上当前“进度表”
        status_update = f"\n(System Note: Already know - Destination: {state['destination']}, Date: {state['departure_date']})"
        # 动态修改发送给模型的最后一条消息
        return {"messages": state["messages"] + [{"role": "system", "content": status_update}]}

# 4. 创建 Agent
agent = create_agent(
    model="gpt-4o",
    tools=[save_flight_info],
    middleware=[FlightManagerMiddleware()],
)

# 第一次对话
result1 = agent.invoke({
    "messages": [{"role": "user", "content": "I want to fly to Paris"}],
    "destination": "Paris",  # 初始状态
    "departure_date": None,
})

# 此时 result1 返回后，外部系统可以将状态持久化。
# 下一次对话时，虽然用户没提 Paris，但 Agent 依然记得。
```

## 七、Middleware（中间件）
中间件是统一扩展层，可在模型前后、工具前后注入逻辑。常见用途是消息裁剪、权限控制、动态模型选择、工具治理。

```python
from langchain.agents.middleware import before_model

@before_model
def trim_messages(state, runtime):
    """只保留最近 10 条消息"""
    messages = state["messages"]
    if len(messages) > 10:
        return {"messages": messages[-10:]}
    return None
```

## 八、Structured Output（结构化输出）
结构化输出让 Agent 直接返回可编程对象，减少后处理和解析成本。

### 1）ToolStrategy
```python
from pydantic import BaseModel
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy

class ContactInfo(BaseModel):
    name: str
    email: str
    phone: str

agent = create_agent(
    model="gpt-4o-mini",
    tools=tools,
    response_format=ToolStrategy(ContactInfo),
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "提取联系信息: 张三, zhang@example.com, 13800138000"}]
})

contact = result["structured_response"]
# ContactInfo(name='张三', email='zhang@example.com', phone='13800138000')
```

### 2）ProviderStrategy
```python
from langchain.agents.structured_output import ProviderStrategy

agent = create_agent(
    model="gpt-4o",
    response_format=ProviderStrategy(ContactInfo),
)
```

## 九、本课要点
1. 先用静态配置快速跑通，再按场景引入动态能力。
2. 动态能力统一放在中间件层，保持主流程清晰。
3. 工具治理要覆盖权限、注册与错误处理三个层面。
4. 结构化输出与流式输出分别解决“可编程”和“可观测”。
5. 记忆不是单独模块，而是状态 schema + 中间件协同。
