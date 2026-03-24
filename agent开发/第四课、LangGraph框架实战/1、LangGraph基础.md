# LangGraph 基础课程内容总结
本课程围绕 LangGraph 展开，从基础环境配置到高级应用如多智能体、RAG、网络搜索等进行了全面讲解，核心基于 Python 结合 LangChain 生态，以阿里百炼平台模型为实操基础，以下为精简总结。

## 一、快速入门
### 1. 环境配置
- **安装依赖**：可克隆仓库安装完整依赖，或直接安装核心包
  ```bash
  # 完整依赖
  cd dive-into-langgraph
  pip install -r requirements.txt
  # 核心包
  pip install langgraph langchain
  ```
- **导入依赖**：创建并配置 `.env` 文件，填入阿里百炼 `DASHSCOPE_API_KEY`
  ```bash
  cp .env.example .env
  ```
  ```python
  import os
  from dotenv import load_dotenv
  from langchain_openai import ChatOpenAI
  from langchain.agents import create_agent
  from langchain.chat_models import init_chat_model
  _ = load_dotenv()
  ```
- **加载 LLM**：两种方式加载阿里百炼 Qwen 模型（兼容 OpenAI API）
  ```python
  # 方法一：ChatOpenAI
  llm = ChatOpenAI(
      model="qwen3-coder-plus",
      api_key=os.getenv("DASHSCOPE_API_KEY"),
      base_url=os.getenv("DASHSCOPE_BASE_URL"),
  )
  # 方法二：init_chat_model
  llm = init_chat_model(
      model="qwen3-coder-plus",
      model_provider="openai",
      api_key=os.getenv("DASHSCOPE_API_KEY"),
      base_url=os.getenv("DASHSCOPE_BASE_URL"),
  )
  ```

### 2. 基础 Agent 开发
- **简单 ReAct Agent**
  ```python
  agent = create_agent(
      model=llm,
      system_prompt="You are a helpful assistant",
  )
  response = agent.invoke({'messages': '你好'})
  response['messages'][-1].content
  ```
- **带工具调用的 Agent**
  ```python
  def get_weather(city: str) -> str:
      """Get weather for a given city."""
      return f"It's always sunny in {city}!"
  tool_agent = create_agent(
      model=llm,
      tools=[get_weather],
      system_prompt="You are a helpful assistant",
  )
  response = tool_agent.invoke(
      {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
  )
  response['messages'][-1].content
  ```
- **工具权限控制（ToolRuntime）**
  ```python
  from typing import Literal, Any
  from pydantic import BaseModel
  from langchain.tools import tool, ToolRuntime
  class Context(BaseModel):
      authority: Literal["admin", "user"]
  @tool
  def math_add(runtime: ToolRuntime[Context, Any], a: int, b: int) -> int:
      """Add two numbers together."""
      authority = runtime.context.authority
      if authority != "admin":
          raise PermissionError("User does not have permission to add numbers")
      return a + b
  tool_agent = create_agent(
      model=llm,
      tools=[get_weather, math_add],
      system_prompt="You are a helpful assistant",
  )
  response = tool_agent.invoke(
      {"messages": [{"role": "user", "content": "请计算 8234783 + 94123832 =?"}]},
      config={"configurable": {"thread_id": "1"}},
      context=Context(authority="admin"),
  )
  for message in response['messages']:
      message.pretty_print()
  ```
- **结构化输出**
  ```python
  from pydantic import BaseModel, Field
  class CalcInfo(BaseModel):
      """Calculation information."""
      output: int = Field(description="The calculation result")
  structured_agent = create_agent(
      model=llm,
      tools=[get_weather, math_add],
      system_prompt="You are a helpful assistant",
      response_format=CalcInfo,
  )
  response = structured_agent.invoke(
      {"messages": [{"role": "user", "content": "请计算 8234783 + 94123832 =?"}]},
      config={"configurable": {"thread_id": "1"}},
      context=Context(authority="admin"),
  )
  response['structured_response']
  ```
- **流式输出**
  ```python
  agent = create_agent(
      model=llm,
      tools=[get_weather],
  )
  for chunk in agent.stream( 
      {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
      stream_mode="updates",
  ):
      for step, data in chunk.items():
          print(f"step: {step}")
          print(f"content: {data['messages'][-1].content_blocks}")
  ```

## 二、状态图（StateGraph）
替代 Agent 实现**可控工作流**，通过节点、边、条件边定义流程，示例为天气查询工作流：
```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START, END
from langchain.messages import HumanMessage, SystemMessage
from langchain.tools import tool
from langgraph.prebuilt import ToolNode
from langchain_core.runnables import RunnableConfig
_ = load_dotenv()

# 加载模型
llm = ChatOpenAI(
    model="qwen3-coder-plus",
    temperature=0.7,
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url=os.getenv("DASHSCOPE_BASE_URL"),
)

# 工具函数
@tool
def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

# 工具节点
tools = [get_weather]
tool_node = ToolNode(tools)

# 助手节点
def assistant(state: MessagesState, config: RunnableConfig):
    system_prompt = 'You are a helpful assistant that can check weather.'
    all_messages = [SystemMessage(system_prompt)] + state['messages']
    model = llm.bind_tools(tools)
    return {'messages': [model.invoke(all_messages)]}

# 条件边
def should_continue(state: MessagesState, config: RunnableConfig):
    messages = state['messages']
    last_message = messages[-1]
    if last_message.tool_calls:
        return 'continue'
    return 'end'

# 构建状态图
builder = StateGraph(MessagesState)
builder.add_node('assistant', assistant)
builder.add_node('tool', tool_node)
builder.add_edge(START, 'assistant')
builder.add_conditional_edges(
    'assistant',
    should_continue,
    {
        'continue': 'tool',
        'end': END,
    },
)
builder.add_edge('tool', 'assistant')
my_graph = builder.compile(name='my-graph')

# 调用
response = my_graph.invoke({'messages': [HumanMessage(content='上海天气怎么样?')]})
for message in response['messages']:
    message.pretty_print()
```
- 支持**有向无环图（DAG）** 和**有向循环图（DCG）**，示例为DCG（助手与工具节点循环调用）。

## 三、中间件（Middleware）
通过**装饰器**实现工作流拓展，为钩子函数，可控制模型/工具调用、处理消息、风控等，核心装饰器及示例如下：
### 核心装饰器
| 装饰器          | 作用                     |
|-----------------|--------------------------|
| @before_agent   | Agent执行前执行逻辑      |
| @after_agent    | Agent执行后执行逻辑      |
| @before_model   | 模型调用前执行逻辑       |
| @after_model    | 模型接收响应后执行逻辑   |
| @wrap_model_call| 控制模型调用过程         |
| @wrap_tool_call | 控制工具调用过程         |
| @dynamic_prompt | 动态生成系统提示词       |
| @hook_config    | 配置钩子行为             |

### 典型示例
1. **预算控制（@wrap_model_call）**：对话超过阈值切换低费率模型
   ```python
   import os
   from dotenv import load_dotenv
   from langchain_openai import ChatOpenAI
   from langchain.agents import create_agent
   from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
   from langchain.messages import HumanMessage
   from langgraph.graph import MessagesState
   _ = load_dotenv()

   # 高低费率模型
   basic_model = ChatOpenAI(
       api_key=os.getenv("DASHSCOPE_API_KEY"),
       base_url=os.getenv("DASHSCOPE_BASE_URL"),
       model="qwen3-coder-plus",
   )
   advanced_model = ChatOpenAI(
       api_key=os.getenv("DASHSCOPE_API_KEY"),
       base_url=os.getenv("DASHSCOPE_BASE_URL"),
       model="qwen3-max",
   )

   @wrap_model_call
   def dynamic_model_selection(request: ModelRequest, handler) -> ModelResponse:
       message_count = len(request.state["messages"])
       if message_count > 5:
           model = basic_model
       else:
           model = advanced_model
       request.model = model
       print(f"message_count: {message_count}")
       print(f"model_name: {model.model_name}")
       return handler(request)

   agent = create_agent(
       model=advanced_model,
       middleware=[dynamic_model_selection]
   )

   # 测试
   state: MessagesState = {"messages": []}
   items = ['汽车', '飞机', '摩托车', '自行车']
   for idx, i in enumerate(items):
       print(f"\n=== Round {idx+1} ===")
       state["messages"] += [HumanMessage(content=f"{i}有几个轮子,请简单回答")]
       result = agent.invoke(state)
       state["messages"] = result["messages"]
       print(f'content: {result["messages"][-1].content}')
   ```
2. **消息截断（@before_model）**：限制上下文长度，保留关键消息
3. **敏感词过滤（@before_agent）**：拦截含敏感词的用户请求
4. **PII检测**：识别/屏蔽用户输入中的隐私信息（文件路径、姓名、IP等）
5. **内置中间件**：`SummarizationMiddleware`（对话摘要）、`ModelFallbackMiddleware`（模型降级）等

## 四、人机交互（Human-in-the-loop, HITL）
通过`HumanInTheLoopMiddleware`实现**人工审批**，Agent执行工具前中断，获取人类反馈后继续，支持`approve/edit/reject`等审批类型：
```python
import os
import uuid
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langchain.tools import tool
from langgraph.checkpoint.memory import InMemorySaver
_ = load_dotenv()

llm = ChatOpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url=os.getenv("DASHSCOPE_BASE_URL"),
    model="qwen3-coder-plus",
)

# 工具
@tool
def get_weather(city: str) -> str: return f"It's always sunny in {city}!"
@tool
def add_numbers(a: float, b: float) -> float: return a + b
@tool
def calculate_bmi(weight_kg: float, height_m: float) -> float:
    if height_m <= 0 or weight_kg <= 0: raise ValueError("参数需大于0")
    return weight_kg / (height_m ** 2)

# 带HITL的Agent
tool_agent = create_agent(
    model=llm,
    tools=[get_weather, add_numbers, calculate_bmi],
    middleware=[
        HumanInTheLoopMiddleware( 
            interrupt_on={
                "get_weather": False,
                "add_numbers": True,
                "calculate_bmi": {"allowed_decisions": ["approve", "reject"]},
            },
            description_prefix="Tool execution pending approval",
        ),
    ],
    checkpointer=InMemorySaver(),
    system_prompt="You are a helpful assistant",
)

# 运行（触发中断）
config = {'configurable': {'thread_id': str(uuid.uuid4())}}
result = tool_agent.invoke(
    {"messages": [{"role": "user", "content": "我身高180cm,体重180斤,我的BMI是多少"}]},
    config=config,
)

# 人工审批后恢复
result = tool_agent.invoke(
    {"command": {"resume": {"decisions": [{"type": "approve"}]}}}, 
    config=config
)
result['messages'][-1].content
```
- 检查点推荐**持久化存储**：`SqliteSaver`/`PostgresSaver`/`RedisSaver`（生产环境），示例用`InMemorySaver`。

## 五、记忆（Memory）
分**短期记忆**和**长期记忆**，StateGraph原生`messages`满足基础记忆，复杂场景需专用记忆模块：
### 1. 短期记忆（MemorySaver）
用于临时保存Agent/工作流状态，支持故障恢复、会话续聊，示例：
```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

# 内存版（测试）
checkpointer = InMemorySaver()
# SQLite持久化版（生产）
checkpointer = SqliteSaver(sqlite3.connect("short-memory.db", check_same_thread=False))

# 工作流中使用
builder = StateGraph(MessagesState)
builder.add_node('assistant', assistant)
builder.add_edge(START, 'assistant')
builder.add_edge('assistant', END)
graph = builder.compile(checkpointer=checkpointer)

# 测试续聊
graph.invoke({"messages": ["你好!我是派大星"]}, {"configurable": {"thread_id": "1"}})
graph.invoke({"messages": [{"role": "user", "content": "请问我是谁?"}]}, {"configurable": {"thread_id": "1"}})
```

### 2. 长期记忆（MemoryStore）
用于持久化存储业务信息（用户偏好、属性），支持**Embedding向量检索**，示例：
```python
from langgraph.store.memory import InMemoryStore
from openai import OpenAI

# Embedding生成函数
EMBED_DIM = 1024
client = OpenAI(
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url=os.getenv("DASHSCOPE_BASE_URL"),
)
def embed(texts: list[str]) -> list[list[float]]:
    response = client.embeddings.create(
        model="text-embedding-v4",
        input=texts,
        dimensions=EMBED_DIM,
    )
    return [item.embedding for item in response.data]

# 初始化存储
store = InMemoryStore(index={"embed": embed, "dims": EMBED_DIM})

# 写入
store.put(("users", ), "user_1", {"rules": ["User likes short, direct language"], "rule_id": "3"})
store.put(("users", ), "user_2", {"name": "John Smith", "language": "English"})

# 读取/检索
store.get(("users", ), "user_2")
store.search(("users", ), query="language preferences", filter={"rule_id": "3"})

# 工具中调用
from dataclasses import dataclass
from langchain.tools import tool, ToolRuntime
@dataclass
class Context: user_id: str
@tool
def get_user_info(runtime: ToolRuntime[Context]) -> str:
    user_id = runtime.context.user_id
    user_info = runtime.store.get(("users",), user_id) 
    return str(user_info.value) if user_info else "未知用户"

agent = create_agent(
    model=llm,
    tools=[get_user_info],
    store=store,
    context_schema=Context
)
agent.invoke(
    {"messages": [{"role": "user", "content": "查阅用户信息"}]},
    context=Context(user_id="user_2")
)
```

## 六、上下文工程（Context Engineering）
LangGraph 上下文分**模型/工具/生命周期上下文**，通过Schema定义（dataclasses/pydantic/TypedDict），核心能力为**动态修改上下文**，示例：
### 1. 动态修改系统提示词（@dynamic_prompt）
- 基于State（对话长度）、Store（用户偏好）、Runtime（用户角色）动态生成prompt
### 2. 动态注入消息（@wrap_model_call）
将本地文件、外部信息注入模型对话上下文
### 3. 工具中使用上下文
结合SqliteStore实现工具对持久化上下文的读取/写入
### 4. 上下文压缩（SummarizationMiddleware）
自动将旧消息替换为摘要，缓解上下文过长问题：
```python
from langchain.agents.middleware import SummarizationMiddleware
checkpointer = InMemorySaver()
agent = create_agent(
    model=llm,
    middleware=[
        SummarizationMiddleware(
            model=llm,
            max_tokens_before_summary=40,
            messages_to_keep=1,
        ),
    ],
)
```

## 七、MCP Server
**模型控制平面**，将工具封装为独立服务，支持多服务管理和远程调用，核心流程：
### 1. 开发MCP服务
封装天气/算数工具为Python包，通过`streamable-http`/`stdio`部署：
```python
# main.py 示例（天气MCP）
import asyncio
import os
from . import server
host = os.getenv('HOST', '127.0.0.1')
port = int(os.getenv('PORT', 8000))
def http():
    asyncio.run(server.mcp.run(transport="http", host=host, port=port, path="/mcp"))
if __name__ == "__main__":
    http()
```
启动命令：`python -m get_weather_mcp`

### 2. 管理MCP服务（supervisord）
进程守护，自动重启挂掉的MCP服务，配置文件`mcp_supervisor.conf`，核心命令：
```bash
pip install supervisor
supervisord -c ./mcp_supervisor.conf  # 启动
pkill -f supervisord  # 关闭
lsof -i :8000  # 检查端口
```

### 3. LangGraph中调用MCP
```python
from langchain_mcp_adapters.client import MultiServerMCPClient
import asyncio
async def mcp_agent():
    client = MultiServerMCPClient({
        "math": {
            "command": "python",
            "args": [os.path.abspath("./mcp_server/math_mcp/server.py")],
            "transport": "stdio",
        },
        "weather": {
            "url": "http://localhost:8000/mcp",
            "transport": "streamable_http",
        }
    })
    tools = await client.get_tools()
    agent = create_agent(llm, tools=tools)
    return agent
async def use_mcp(messages):
    agent = await mcp_agent()
    response = await agent.ainvoke(messages)
    return response
# 调用
messages = {"messages": [{"role": "user", "content": "福州天气怎么样?"}]}
response = await use_mcp(messages)
```

## 八、监督者模式（多智能体系统）
**Supervisor** 作为主智能体，拆解任务给子智能体/工具执行，汇总结果返回，两种实现方式：
### 1. Tool-calling 实现
将子智能体封装为工具，由主智能体调用：
```python
# 子智能体
subagent1 = create_agent(model=llm, tools=[add], name="subagent-1")
@tool("subagent-1", description="可以准确地计算两数相加")
def call_subagent1(query: str):
    result = subagent1.invoke({"messages": [{"role": "user", "content": query}]})
    return result["messages"][-1].content

# 主智能体
supervisor_agent = create_agent(
    model=llm,
    tools=[call_subagent1, call_subagent2, divide],
    system_prompt="提示:如遇两数相减仍可用两数相加工具实现,只需将一个数加上另一个数的负数",
)
# 测试复杂计算
supervisor_agent.invoke({"messages": [{"role": "user", "content": "计算 38462 + 378 / 49 * 83723 - 123 的结果"}]})
```

### 2. langgraph-supervisor 包实现
```python
pip install -U langgraph-supervisor
from langgraph_supervisor import create_supervisor

subagent3 = create_agent(model=llm, tools=[divide], name="subagent-3")
supervisor_graph = create_supervisor(
    [subagent1, subagent2, subagent3],
    model=llm,
    prompt="提示:如遇两数相减仍可用两数相加工具实现,只需将一个数加上另一个数的负数"
)
supervisor_app = supervisor_graph.compile()
supervisor_app.invoke({"messages": [{"role": "user", "content": "计算 38462 + 378 / 49 * 83723 - 123 的结果"}]})
```
- 多智能体适用场景：单智能体工具过多、上下文过大、任务需专业化分工。

## 九、并行（实际为Python并发，绕开GIL）
四种实现方式，核心为**节点并发**、**Map-reduce**、**子图并发**：
### 1. 节点并发
StateGraph中并行添加多个节点，由START同时触发：
```python
def node_a(state: MessagesState): # 模拟耗时任务
    import time
    time.sleep(2)
    return {'messages': [HumanMessage(content='节点a完成')]}
def node_b(state: MessagesState):
    import time
    time.sleep(4)
    return {'messages': [HumanMessage(content='节点b完成')]}

builder = StateGraph(MessagesState)
builder.add_node('node_a', node_a)
builder.add_node('node_b', node_b)
builder.add_edge(START, 'node_a')
builder.add_edge(START, 'node_b')
builder.add_edge('node_a', END)
builder.add_edge('node_b', END)
my_graph = builder.compile()
my_graph.invoke({'messages': [HumanMessage(content='执行并发节点')]})
```

### 2. Map-reduce
**先并发分发任务，再聚合结果**，示例为生成不同人设的回复并筛选最佳：
```python
from langgraph.types import Send
from pydantic import BaseModel
class Role(BaseModel): role: str
class Overall(TypedDict):
    roles: list[str]
    responses: Annotated[list, operator.add]
    best_response: str

# Map：分发角色
def continue_to_responses(state: Overall):
    return [Send("generate_response", {"role": r}) for r in state["roles"]]
# Map：生成回复
def generate_response(state: Role):
    prompt = f"女神不回消息,作为{state['role']}怎么回复?"
    return {"responses": [llm.invoke(prompt).content]}
# Reduce：筛选最佳
def best_response(state: Overall):
    prompt = f"从{state['responses']}中选最佳回复,返回索引"
    idx = llm.invoke(prompt).content
    return {"best_response": state["responses"][int(idx)]}

# 构建图
builder = StateGraph(Overall)
builder.add_node("generate_response", generate_response)
builder.add_node("best_response", best_response)
builder.add_conditional_edges(START, continue_to_responses, ["generate_response"])
builder.add_edge("generate_response", "best_response")
builder.add_edge("best_response", END)
graph = builder.compile()

# 调用
roles = ["男神", "舔狗", "霸道总裁"]
graph.invoke({"roles": roles})
```

### 3. 子图并发
将多个已编译的子图作为节点，在父图中并行调用，实现工作流复用。

## 十、RAG（检索增强生成）
解决大模型实时性/专业性不足问题，**检索知识库内容注入提示词**，核心流程：
### 1. 核心原理
- **向量检索**：将文本转为Embedding向量，通过**余弦相似度**召回相关内容
- **关键词检索（BM25）**：基于词频的精确匹配，中文需jieba分词
- **混合检索**：向量+关键词检索，通过RRF/大模型重排结果

### 2. 向量检索RAG实现（六步）
```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_community.document_loaders import WebBaseLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_community.embeddings import DashScopeEmbeddings
from langchain_core.vectorstores import InMemoryVectorStore
from langchain.tools import tool
from langchain.agents import create_agent
_ = load_dotenv()
llm = ChatOpenAI(model="qwen3-max", api_key=os.getenv("DASHSCOPE_API_KEY"), base_url=os.getenv("DASHSCOPE_BASE_URL"))

# 1. 加载文档
bs4_strainer = bs4.SoupStrainer(class_=("post"))
loader = WebBaseLoader(
    web_paths=("https://luochang212.github.io/posts/quick_bi_intro/",),
    bs_kwargs={"parse_only": bs4_strainer},
)
docs = loader.load()

# 2. 分割文档
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
all_splits = text_splitter.split_documents(docs)

# 3. 生成向量
embeddings = DashScopeEmbeddings()

# 4. 向量存储
vector_store = InMemoryVectorStore(embedding=embeddings)
vector_store.add_documents(documents=all_splits)

# 5. 创建检索工具
@tool(response_format="content_and_artifact")
def retrieve_context(query: str):
    retrieved_docs = vector_store.similarity_search(query, k=2)
    serialized = "\n\n".join([f"Source: {doc.metadata}\nContent: {doc.page_content}" for doc in retrieved_docs])
    return serialized, retrieved_docs

# 6. 创建Agent调用
agent = create_agent(
    llm,
    tools=[retrieve_context],
    system_prompt="Use the tool to retrieve context and answer questions."
)
agent.invoke({"messages": [{"role": "user", "content": "当前的 Agent 能力有哪些局限性?"}]})
```

### 3. RAG架构
- **2-Step RAG**：先检索，再生成
- **Agentic RAG**：Agent控制检索时机/方式
- **Hybrid RAG**：增加Query改写、结果相关性验证

## 十一、网络搜索
对接第三方搜索API，解决大模型训练数据滞后问题，三种实现方式：
### 1. 阿里DashScope（推荐，已有API_KEY）
```python
from dashscope import Generation
@tool
def dashscope_search(query: str) -> str:
    response = Generation.call(
        model='qwen3-max',
        prompt=query,
        enable_search=True,
        result_format='message'
    )
    if response.status_code == 200:
        return response.output.choices[0].message.content
    else:
        return f"Search failed: {response.status_code}, {response.message}"

agent = create_agent(
    model=llm,
    tools=[dashscope_search],
    system_prompt="回答前必须使用工具搜索互联网信息",
)
agent.invoke({"messages": [{"role": "user", "content": "告诉我今天的日期和最重要的新闻"}]})
```

### 2. Tavily（LangChain推荐，需申请API_KEY）
### 3. DDGS（免费，无需API_KEY，结果质量较低）

## 十二、Deep Agents
LangChain 出品的**深度研究智能体**，支持**主动联网搜索**、**多轮深度洞察**，适用于复杂信息查询：
```python
pip install deepagents dashscope
import dashscope
from deepagents import create_deep_agent
dashscope.api_key=os.getenv("DASHSCOPE_API_KEY")

# 搜索工具（复用dashscope_search）
@tool
def get_today_date() -> str:
    from datetime import datetime
    return datetime.now().strftime("%Y-%m-%d")

# 创建Deep Agent
research_instructions = """你是资深研究员，使用互联网搜索工具进行全面研究，撰写精炼报告"""
agent = create_deep_agent(
    model=llm,
    tools=[dashscope_search, get_today_date],
    system_prompt=research_instructions
)

# 测试（查询最新玻利维亚总统信息）
result = agent.invoke({"messages": [{"role": "user", "content": "新上任的玻利维亚总统是谁?请介绍一下这位总统｡"}]})
print(result["messages"][-1].content)
```
- 可主动多轮调用搜索工具，补充信息并生成结构化报告。