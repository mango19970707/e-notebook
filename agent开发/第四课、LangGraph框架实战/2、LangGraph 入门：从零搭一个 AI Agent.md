# LangGraph 入门课程补充总结（计算器 AI Agent 实战）
本课程以**计算器 Agent**为实操案例，聚焦 LangGraph 核心概念与极简入门流程，采用智谱 glm-5 模型（国产模型兼容 OpenAI 接口），核心是通过“节点+边+状态”实现循环计算逻辑，以下为精简总结。

## 一、核心价值
LangGraph 解决 LangChain Chain 的核心痛点：**支持“判断→执行→再判断→再执行”的循环逻辑**。例如多步计算任务（`(3+4)×2`），需先算加法、再算乘法，Chain 无法灵活实现这种动态流程，而 LangGraph 可通过“图结构”明确节点分工与路由规则。

## 二、环境准备
### 1. 安装依赖
```bash
pip install langchain langgraph langchain-openai
```

### 2. 初始化国产模型（智谱 glm-5）
智谱/深度求索/月之暗面等国产模型兼容 OpenAI 接口，仅需修改 `api_key` 和 `api_base`：
```python
from langchain_openai import ChatOpenAI

ZHIPU_API_KEY = "你的智谱API Key"  # 需注册智谱开放平台获取：https://open.bigmodel.cn
ZHIPU_BASE_URL = "https://open.bigmodel.cn/api/paas/v4/"

llm = ChatOpenAI(
    temperature=0,  # 计算任务设为0，保证结果稳定
    model="glm-5",
    openai_api_key=ZHIPU_API_KEY,
    openai_api_base=ZHIPU_BASE_URL,
)
```

## 三、核心组件实现
### 1. 定义工具（Agent 执行能力）
通过 `@tool` 装饰器将普通函数封装为 Agent 可调用的工具，**docstring 需清晰描述功能与参数**（模型依赖该描述判断是否调用）：
```python
from langchain_core.tools import tool

@tool
def add(a: int, b: int) -> int:
    """Adds a and b.
    Args:
        a: First int
        b: Second int
    """
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """Multiply a and b.
    Args:
        a: First int
        b: Second int
    """
    return a * b

@tool
def divide(a: int, b: int) -> float:
    """Divide a by b.
    Args:
        a: First int
        b: Second int
    """
    return a / b

# 工具绑定与映射
tools = [add, multiply, divide]
tools_by_name = {t.name: t for t in tools}  # 按工具名快速查找
llm_with_tools = llm.bind_tools(tools)  # 告诉模型可用工具
```

### 2. 定义状态（共享数据容器）
状态是全局共享的“数据存储”，所有节点可读写，此处存储消息列表（支持追加不覆盖）：
```python
from langchain_core.messages import AnyMessage
from typing_extensions import TypedDict, Annotated
import operator

class MessagesState(TypedDict):
    # Annotated+operator.add 表示消息追加，而非覆盖
    messages: Annotated[list[AnyMessage], operator.add]
```
- 简化方案：直接使用 LangGraph 内置状态 `from langgraph.graph import MessagesState`

### 3. 定义节点（执行单元）
图中包含两个核心节点：“思考节点”（LLM 决策）和“执行节点”（工具调用）。

#### （1）LLM 节点（思考决策）
接收当前状态的消息列表，让模型判断“直接回答”或“调用工具”：
```python
from langchain_core.messages import SystemMessage

def llm_call(state: MessagesState):
    """大模型决定：直接回答还是调工具"""
    return {
        "messages": [
            llm_with_tools.invoke(
                [SystemMessage(content="你是一个计算助手，使用提供的工具来完成计算任务。")]
                + state["messages"]  # 传入历史消息
            )
        ]
    }
```

#### （2）工具节点（执行工具）
解析 LLM 节点的 `tool_call` 指令，执行对应工具并返回结果（包装为 `ToolMessage`，通过 `tool_call_id` 关联请求与结果）：
```python
from langchain_core.messages import ToolMessage

def tool_node(state: MessagesState):
    """执行工具调用，返回结果"""
    result = []
    # 从最后一条 AI 消息中提取工具调用指令
    for tool_call in state["messages"][-1].tool_calls:
        tool = tools_by_name[tool_call["name"]]
        observation = tool.invoke(tool_call["args"])  # 执行工具
        # 包装结果，绑定 tool_call_id
        result.append(ToolMessage(content=str(observation), tool_call_id=tool_call["id"]))
    return {"messages": result}
```

### 4. 定义路由逻辑（边的规则）
条件边函数，决定 LLM 节点执行后“走工具节点”还是“结束流程”：
```python
from typing import Literal
from langgraph.graph import END

def should_continue(state: MessagesState) -> Literal["tool_node", "__end__"]:
    """判断：有工具调用则继续，否则结束"""
    last_message = state["messages"][-1]
    if last_message.tool_calls:  # 模型返回了 tool_call 指令
        return "tool_node"
    return END  # 无工具调用，流程结束
```

### 5. 组装图（串联组件）
通过 `StateGraph` 拼接节点与边，形成完整工作流：
```python
from langgraph.graph import StateGraph, START

# 1. 创建图实例
graph_builder = StateGraph(MessagesState)

# 2. 添加节点
graph_builder.add_node("llm_call", llm_call)  # 思考节点
graph_builder.add_node("tool_node", tool_node)  # 执行节点

# 3. 添加边（定义流程）
graph_builder.add_edge(START, "llm_call")  # 入口 → 思考节点
# 思考节点 → 条件边（工具节点 或 结束）
graph_builder.add_conditional_edges("llm_call", should_continue, ["tool_node", END])
graph_builder.add_edge("tool_node", "llm_call")  # 执行节点 → 思考节点（循环）

# 4. 编译生成 Agent
agent = graph_builder.compile()
```

## 四、运行与测试
### 1. 简单计算（单步）
```python
from langchain_core.messages import HumanMessage

# 调用 Agent
response = agent.invoke({"messages": [HumanMessage(content="3 加 4 等于多少？")]})

# 打印结果
for msg in response["messages"]:
    msg.pretty_print()
```
**流程**：用户提问 → LLM 决策调用 `add` → 工具返回 7 → LLM 生成最终回答。

### 2. 多步计算（循环逻辑）
```python
# 复杂任务：100 ÷ 5 + 30 × 3
response = agent.invoke({"messages": [HumanMessage(content="100 除以 5，再加上 30，最后乘以 3")]})

for msg in response["messages"]:
    msg.pretty_print()
```
**流程**：LLM 先调用 `divide`（100÷5=20）→ 回到 LLM 决策 → 调用 `add`（20+30=50）→ 回到 LLM 决策 → 调用 `multiply`（50×3=150）→ 回到 LLM 生成最终结果。

## 五、完整代码
```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import AnyMessage, SystemMessage, HumanMessage, ToolMessage
from typing_extensions import TypedDict, Annotated
from typing import Literal
from langgraph.graph import StateGraph, START, END
import operator

# ========== 1. 初始化模型 ==========
ZHIPU_API_KEY = "你的智谱API Key"
ZHIPU_BASE_URL = "https://open.bigmodel.cn/api/paas/v4/"

llm = ChatOpenAI(
    temperature=0,
    model="glm-5",
    openai_api_key=ZHIPU_API_KEY,
    openai_api_base=ZHIPU_BASE_URL,
)

# ========== 2. 定义工具 ==========
@tool
def add(a: int, b: int) -> int:
    """Adds a and b.
    Args:
        a: First int
        b: Second int
    """
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """Multiply a and b.
    Args:
        a: First int
        b: Second int
    """
    return a * b

@tool
def divide(a: int, b: int) -> float:
    """Divide a by b.
    Args:
        a: First int
        b: Second int
    """
    return a / b

tools = [add, multiply, divide]
tools_by_name = {t.name: t for t in tools}
llm_with_tools = llm.bind_tools(tools)

# ========== 3. 定义状态 ==========
class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]

# ========== 4. 定义节点 ==========
def llm_call(state: MessagesState):
    """大模型决定：直接回答还是调工具"""
    return {
        "messages": [
            llm_with_tools.invoke(
                [SystemMessage(content="你是一个计算助手，使用提供的工具来完成计算任务。")]
                + state["messages"]
            )
        ]
    }

def tool_node(state: MessagesState):
    """执行工具调用"""
    result = []
    for tool_call in state["messages"][-1].tool_calls:
        t = tools_by_name[tool_call["name"]]
        observation = t.invoke(tool_call["args"])
        result.append(ToolMessage(content=str(observation), tool_call_id=tool_call["id"]))
    return {"messages": result}

# ========== 5. 定义路由 ==========
def should_continue(state: MessagesState) -> Literal["tool_node", "__end__"]:
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tool_node"
    return END

# ========== 6. 组装图 ==========
graph_builder = StateGraph(MessagesState)
graph_builder.add_node("llm_call", llm_call)
graph_builder.add_node("tool_node", tool_node)
graph_builder.add_edge(START, "llm_call")
graph_builder.add_conditional_edges("llm_call", should_continue, ["tool_node", END])
graph_builder.add_edge("tool_node", "llm_call")
agent = graph_builder.compile()

# ========== 7. 测试 ==========
response = agent.invoke({"messages": [HumanMessage(content="先算 3 加 4，再把结果乘以 2")]})
for msg in response["messages"]:
    msg.pretty_print()
```

## 六、关键概念与常见问题
### 1. 核心概念梳理
| 组件         | 作用                                  |
|--------------|---------------------------------------|
| State（状态）| 共享数据容器，存储全局信息（如消息列表）|
| Node（节点） | 执行单元，如 LLM 决策节点、工具执行节点|
| Edge（边）   | 节点间的路径，普通边（无条件）或条件边（按规则跳转）|
| 条件边       | 控制流程方向的 if-else 逻辑（如是否调用工具）|

### 2. 常见问题解答
- **Q：为何不用 LangChain 的 AgentExecutor？**  
  A：AgentExecutor 循环逻辑写死，灵活性差；LangGraph 支持自定义节点/边，适配复杂流程。
- **Q：状态能否添加其他字段？**  
  A：可以，例如记录 LLM 调用次数：`class MyState(TypedDict): messages: ...; llm_call_count: int`。
- **Q：其他国产模型能否替代智谱？**  
  A：支持（如 deepseek-chat、moonshot-v1），只需替换 `api_key` 和 `api_base`（需模型支持 OpenAI 兼容的 function calling）。
- **Q：模型不爱调用工具怎么办？**  
  A：在 System Prompt 中强调“必须使用工具完成计算任务”，或更换 tool calling 能力更强的模型（如 glm-5 优于 glm-4-flash）。