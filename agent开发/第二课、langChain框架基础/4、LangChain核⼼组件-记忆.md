# LangChain核心组件-记忆核心笔记
LangChain的「记忆」组件解决Agent“健忘”问题，分为**短期记忆（当前对话上下文）** 和**长期记忆（跨对话持久化信息）** 两类，核心通过`Checkpointer`（短期）和`Store`（长期）实现，同时提供上下文超限解决方案，是支撑多轮对话和个性化Agent的关键。

## 一、记忆的核心分类与定位
| 记忆类型 | 核心作用 | 生命周期 | 实现组件 | 典型场景 |
|----------|----------|----------|----------|----------|
| 短期记忆 | 维护当前对话的上下文（用户输入、AI回复、工具调用记录） | 对话内有效（程序重启/换会话ID失效） | `Checkpointer` | 多轮对话中理解指代（如“刚才的方案再改改”） |
| 长期记忆 | 存储跨对话的用户信息（偏好、属性、历史数据） | 永久有效（跨会话/跨程序重启） | `Store` | 新对话仍记得用户爱好（如“我喜欢简洁回答”） |

## 二、短期记忆：Checkpointer的使用（核心基础）
### 1. 核心原理
通过`Checkpointer`（检查点存储器）记录对话历史，本质是**按会话ID（thread_id）累积消息列表**，每次调用模型时，将完整历史消息传入，实现“上下文记忆”。

### 2. 基础实现三步法
#### （1）创建Checkpointer
开发/测试用`InMemorySaver`（内存存储），生产用数据库存储（如PostgreSQL）。
#### （2）创建带记忆的Agent
创建时传入`checkpointer`参数，启用记忆功能。
#### （3）指定thread_id调用
必须通过`thread_id`标识会话，**同一thread_id共享上下文**，不同thread_id相互隔离。
```python
# 1. 创建检查点存储器
from langchain.graph.checkpoint.memory import InMemorySaver
checkpoint = InMemorySaver()

# 2. 创建带记忆的Agent
agent = create_agent(model, tools, checkpointer=checkpoint)

# 3. 带thread_id调用（同一会话共享记忆）
config = {"configurable": {"thread_id": "session-001"}}
# 第一轮：告知信息
agent.invoke({"messages": "我叫南哥"}, config)
# 第二轮：验证记忆（AI能回答“你叫南哥”）
agent.invoke({"messages": "我叫什么名字？"}, config)
```

### 3. 关键细节
- **消息自动累积**：每次调用会将新消息（用户输入、AI回复、ToolMessage）追加到历史列表，模型基于全量列表推理；
- **消息格式兼容**：可直接传字符串（LangChain自动转为`HumanMessage`），建议显式封装为`HumanMessage`提升可读性；
- **会话隔离**：更换`thread_id`即开启新对话，原记忆完全隔离。

## 三、上下文超限解决方案（短期记忆优化）
短期记忆的消息列表会持续增长，导致token消耗激增、超出模型上下文窗口，LangChain提供3种核心解法：

### 1. 裁剪消息（Trim）：简单粗暴省token
通过`@before_model`中间件，只保留最近N条消息，丢弃早期对话内容，适合对历史信息要求低的场景。
```python
@before_model
def trim_messages(state):
    messages = state["messages"]
    if len(messages) > 4:
        # 只保留最近4条消息
        return {"messages": messages[-4:]}
    return None

# 创建Agent时加入中间件
agent = create_agent(model, tools, checkpointer=InMemorySaver(), middleware=[trim_messages])
```

### 2. 消息摘要（Summarize）：优雅平衡token与信息
用廉价小模型将旧消息压缩为摘要，保留核心信息，同时减少token消耗，是推荐方案。
```python
from langchain.agents.middleware import SummarizationMiddleware

middleware = [
    SummarizationMiddleware(
        model="gpt-4.1-mini",  # 轻量模型做摘要
        trigger=("tokens", 4000),  # token超4000触发摘要
        keep=("messages", 20)  # 保留最近20条原消息
    )
]

agent = create_agent(model, tools, checkpointer=InMemorySaver(), middleware=middleware)
```

### 3. 删除特定消息：精准清理
通过`@after_model`中间件，在模型回复后删除指定消息（如最早2条），实现精细化内存管理。

## 四、生产环境记忆持久化
`InMemorySaver`仅适用于开发/测试，程序重启后记忆丢失，生产环境需用**数据库级Checkpointer**，LangGraph支持PostgreSQL、SQLite、Azure Cosmos DB等，以PostgreSQL为例：
```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://user:pass@localhost:5432/mydb"
# 连接数据库并自动建表
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()
    agent = create_agent(model, tools, checkpointer=checkpointer)
```

## 五、长期记忆：Store的使用（跨对话持久化）
### 1. 核心原理
`Store`是**键值对存储**，支持命名空间（多级分层），用于存储用户长期信息（爱好、属性、历史偏好），本质是Agent的“个人数据库”。

### 2. 基础操作：存/取数据
```python
from langgraph.store.memory import InMemoryStore

# 创建Store（生产可用数据库存储）
store = InMemoryStore()

# 存数据：命名空间 + 键 + 值（支持多级命名空间）
store.put(("users",), "user-888", {"hobby": "打篮球", "city": "北京"})

# 读数据：按命名空间+键查询
user_info = store.get(("users",), "user-888")
print(user_info.value)  # {'hobby': '打篮球', 'city': '北京'}
```

### 3. 让Agent读写长期记忆（核心实战）
需通过**工具封装**让Agent具备访问Store的能力，工具通过`ToolRuntime`获取Store和上下文（如用户ID），模型无需感知底层存储逻辑。

#### （1）读取长期记忆（工具）
```python
from langchain.tools import tool
from dataclasses import dataclass

# 定义上下文（传入用户ID）
@dataclass
class Context:
    user_id: str

# 读取用户信息的工具
@tool
def get_user_hobby(runtime: ToolRuntime[Context]) -> str:
    """查询用户的爱好"""
    store = runtime.store  # 访问Store
    user_id = runtime.context.user_id  # 获取用户ID
    info = store.get(("users",), user_id)
    return info.value.get("hobby", "暂无爱好") if info else "暂无爱好"

# 创建Agent并传入Store和上下文
agent = create_agent(
    model="gpt-4o",
    tools=[get_user_hobby],
    store=store,
    context_schema=Context
)

# 调用Agent（传入用户ID上下文）
agent.invoke(
    {"messages": "我的爱好是什么？"},
    context=Context(user_id="user-888")
)
```

#### （2）写入长期记忆（工具）
让Agent在对话中主动存储用户信息，实现记忆自主更新。
```python
from typing_extensions import TypedDict

# 定义用户偏好数据结构
class UserPref(TypedDict):
    favorite_food: str

# 保存偏好的工具
@tool
def save_user_food(pref: UserPref, runtime: ToolRuntime[Context]) -> str:
    """记住用户最喜欢的食物"""
    store.put(("users",), runtime.context.user_id, pref)
    return "已记住你的喜好"

# 调用Agent存储信息
agent.invoke(
    {"messages": "记住，我最喜欢吃麻辣烫"},
    context=Context(user_id="user-888")
)
```

## 六、高级扩展：自定义Agent状态
默认Agent状态仅含`messages`字段，可扩展自定义字段（如`user_id`、`preferences`），实现工具间数据共享。
### 1. 定义自定义状态
```python
from langchain.agents import AgentState

class CustomState(AgentState):
    user_id: str  # 自定义用户ID字段
    preferences: dict  # 自定义偏好字段
```

### 2. 创建Agent并使用
```python
agent = create_agent(
    model, tools,
    state_schema=CustomState,  # 传入自定义状态
    checkpointer=InMemorySaver()
)

# 调用时传入自定义状态字段
agent.invoke(
    {
        "messages": "你好",
        "user_id": "user-123",
        "preferences": {"theme": "dark"}
    },
    config={"configurable": {"thread_id": "session-002"}}
)
```

### 3. 工具中访问/更新自定义状态
```python
@tool
def get_user_id(runtime: ToolRuntime) -> str:
    # 读取自定义状态字段
    return f"当前用户ID：{runtime.state['user_id']}"

@tool
def update_preference(runtime: ToolRuntime) -> Command:
    # 更新自定义状态
    return Command(update={"preferences": {"theme": "light"}})
```

## 核心总结
1. 短期记忆靠`Checkpointer`，核心是`thread_id`+消息累积，解决当前对话上下文问题；
2. 长期记忆靠`Store`，核心是命名空间+键值存储，解决跨对话信息持久化；
3. 上下文超限用“裁剪/摘要/精准删除”，生产环境用数据库存储记忆；
4. 自定义状态可扩展Agent存储能力，工具通过`ToolRuntime`访问Store和状态，模型无需感知底层逻辑。

记忆组件的核心价值是让Agent从“单次对话工具”升级为“有上下文感知、能记住用户的智能体”，是实现个性化、高粘性Agent的基础。

## 代码补充（来自 PDF）
用于说明自定义记忆状态的精简代码骨架。

### 示例 1：自定义状态 Schema
```python
from typing import NotRequired
from langchain.agents import AgentState

class BookingState(AgentState):
    destination: NotRequired[str]
    depart_date: NotRequired[str]
```

### 示例 2：感知状态的中间件
```python
from langchain.agents.middleware import before_model
from langchain_core.messages import SystemMessage

@before_model
def inject_memory(state: BookingState, runtime):
    msg = SystemMessage(
        content=f"已知信息：destination={state.get('destination')}, depart_date={state.get('depart_date')}"
    )
    return {"messages": [msg, *state["messages"]]}
```

### 示例 3：持久化检查点
```python
from langgraph.checkpoint.memory import MemorySaver

agent = create_agent(
    model="openai:gpt-4.1-mini",
    tools=[],
    state_schema=BookingState,
    middleware=[inject_memory],
    checkpointer=MemorySaver(),
)
```
