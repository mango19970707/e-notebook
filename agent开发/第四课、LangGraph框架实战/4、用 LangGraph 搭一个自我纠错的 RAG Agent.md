# LangGraph 进阶课程总结（会“自我纠错”的 RAG Agent）

> 对应原文 PDF：`D:\notebook\agent开发\langGraph框架实战\第四课：手把手教你用 LangGraph 搭一个会_自我纠错_的 RAG Agent.pdf`
> 说明：在不改变原文含义的前提下做精简总结；代码示例保持完整，不删减。

本课程聚焦 **Agentic RAG** 实战，解决普通 RAG “检索结果无关仍硬生成回答”的痛点，通过 LangGraph 实现“决策→检索→评估→改写→重试”的闭环流程，核心基于智谱 glm-5 模型，以下为原文核心内容的精简总结。

## 一、核心目标
普通 RAG 流程为“用户提问→检索→生成回答”，缺乏对检索结果的质量判断；本 Agent 新增 **文档相关性评估** 和 **问题改写** 步骤，实现：
1.  Agent 自主决策：判断问题是否需要检索（技术相关需检索，闲聊/简单数学直接回答）；
2.  检索结果质检：评估文档与问题的相关性，仅使用相关文档生成回答；
3.  自我纠错：检索结果无关时，改写问题重新检索，避免无效回答。

## 二、环境准备
### 1. 安装依赖
```bash
pip install langchain langgraph langchain-openai langchain-community langchain-text-splitters beautifulsoup4
```

### 2. 初始化模型（智谱 glm-5）
智谱模型兼容 OpenAI 接口，需配置 API Key 和基础 URL：
```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings

ZHIPU_API_KEY = "你的智谱API Key"  # 智谱开放平台注册获取：https://open.bigmodel.cn
ZHIPU_BASE_URL = "https://open.bigmodel.cn/api/paas/v4/"

# 初始化 LLM（glm-5）
llm = ChatOpenAI(
    temperature=0,  # 保证输出稳定，避免随机性
    model="glm-5",
    openai_api_key=ZHIPU_API_KEY,
    openai_api_base=ZHIPU_BASE_URL,
)

# 初始化 Embedding（智谱 embedding-3）
zhipu_embeddings = OpenAIEmbeddings(
    model="embedding-3",
    openai_api_key=ZHIPU_API_KEY,
    openai_api_base=ZHIPU_BASE_URL,
)
```

## 三、知识库构建
以 Lilian Weng 的 LLM Agent 英文博客为数据源，流程为“加载网页→文本分块→向量化→存入向量库”：
```python
import bs4
from langchain_community.document_loaders import WebBaseLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.vectorstores import InMemoryVectorStore

# 1. 加载网页（仅解析正文）
loader = WebBaseLoader(
    web_paths=("https://lilianweng.github.io/posts/2023-06-23-agent/",),
    bs_kwargs={"parse_only": bs4.SoupStrainer(
        class_=("post-title", "post-header", "post-content")
    )},
)
docs = loader.load()

# 2. 文本分块（避免语义断裂）
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,  # 每块最多 1000 字符
    chunk_overlap=200,  # 相邻块重叠 200 字符
)
all_splits = text_splitter.split_documents(docs)

# 3. 向量化 + 存储（内存向量库，仅用于测试）
vector_store = InMemoryVectorStore(zhipu_embeddings)
vector_store.add_documents(all_splits)
print(f"索引了 {len(all_splits)} 个文档块")
```

## 四、核心组件实现
### 1. 包装检索工具
将向量库检索功能封装为 Agent 可调用的工具，返回结构化结果（文本+原始文档）：
```python
from langchain.tools import tool

@tool(response_format="content_and_artifact")
def retrieve(query: str):
    """Search blog posts for relevant information to answer user questions."""
    # 每次检索返回 5 个最相似文档（提升命中率）
    retrieved_docs = vector_store.similarity_search(query, k=5)
    # 序列化文档（供模型读取）
    serialized = "\n\n".join(
        f"Source: {doc.metadata}\nContent: {doc.page_content}"
        for doc in retrieved_docs
    )
    return serialized, retrieved_docs  # content_and_artifact 格式：文本+原始文档
```
- 关键细节：工具 docstring 需用英文（模型对英文 function calling 理解更优）；`k=5` 适配智谱 embedding-3 的语义匹配精度。

### 2. 定义图节点（5 个核心节点）
使用 LangGraph 内置 `MessagesState` 存储消息列表，所有节点共享状态：

#### （1）Agent 决策节点（generate_query_or_respond）
判断问题是否需要检索，需检索则生成英文查询词（知识库为英文文档）：
```python
from langchain_core.messages import SystemMessage
from langgraph.graph import MessagesState

def generate_query_or_respond(state: MessagesState):
    """Agent 决策：技术相关问题调用检索工具（英文查询），其他直接回答"""
    llm_with_tools = llm.bind_tools([retrieve])  # 绑定检索工具
    system_prompt = SystemMessage(
        content="你是博客问答助手。涉及 AI Agent、LLM、任务分解、记忆机制等技术话题时，必须用 retrieve 工具搜索；与技术无关的问题直接回答。重要：调用工具时 query 必须为英文。"
    )
    response = llm_with_tools.invoke([system_prompt] + state["messages"])
    # 日志输出决策结果
    if response.tool_calls:
        print(f"--- Agent 决定搜索: {response.tool_calls[0]['args']['query']} ---")
    else:
        print("--- Agent 决定直接回答 ---")
    return {"messages": [response]}
```

#### （2）检索节点（retrieve_node）
使用 LangGraph 内置 `ToolNode` 执行检索，自动解析 tool call 并返回结果：
```python
from langgraph.prebuilt import ToolNode
retrieve_node = ToolNode([retrieve])  # 自动执行检索工具
```

#### （3）文档评估节点（grade_documents）
评估检索文档与问题的相关性，仅保留相关文档：
```python
def grade_documents(state: MessagesState):
    """评估文档相关性，相关返回 yes，无关返回 no，仅保留原始问题"""
    messages = state["messages"]
    question = messages[0].content  # 用户原始问题
    # 提取检索结果
    tool_content = ""
    for msg in reversed(messages):
        if hasattr(msg, "type") and msg.type == "tool":
            tool_content = msg.content
            break
    # 生成评估提示词
    grading_prompt = (
        "你是相关性评分员。文档包含问题相关关键词或语义则回复 yes，完全无关则回复 no。只输出 yes 或 no，不解释。\n\n"
        f"用户问题: {question}\n\n"
        f"检索到的文档: {tool_content[:2000]}"  # 截取前 2000 字符避免超长
    )
    response = llm.invoke([("human", grading_prompt)])
    score = response.content.strip().lower()
    # 处理结果：相关则保留所有消息，无关则仅保留原始问题
    if "yes" in score:
        print("--- 评估结果: 文档相关 ---")
        return {"messages": state["messages"]}
    else:
        print("--- 评估结果: 文档不相关 ---")
        return {"messages": [messages[0]]}  # 清空检索结果，仅留原始问题
```

#### （4）问题改写节点（rewrite_question）
检索结果无关时，优化问题表述以提升检索效果：
```python
def rewrite_question(state: MessagesState):
    """改写问题，使其更适合语义检索"""
    question = state["messages"][0].content
    response = llm.invoke(
        [SystemMessage(content="优化问题使其更适合语义搜索，仅输出改写后的问题，不解释。")]
        + [("human", question)]
    )
    rewritten = response.content
    print(f"--- 问题改写: {question} ￫ {rewritten} ---")
    return {"messages": [("human", rewritten)]}
```

#### （5）生成回答节点（generate_answer）
基于相关检索文档生成简洁准确的中文回答：
```python
def generate_answer(state: MessagesState):
    """基于检索到的相关文档生成中文回答"""
    messages = state["messages"]
    question = messages[0].content
    # 提取检索到的相关文档
    docs_content = ""
    for msg in reversed(messages):
        if hasattr(msg, "type") and msg.type == "tool":
            docs_content = msg.content
            break
    # 生成回答
    response = llm.invoke(
        [SystemMessage(content="根据参考文档回答问题，中文输出，简洁准确。")]
        + [("human", f"问题: {question}\n\n参考文档:\n{docs_content}")]
    )
    return {"messages": [response]}
```

### 3. 组装图（定义节点流转逻辑）
通过 `StateGraph` 串联节点，使用条件边控制流程方向：
```python
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import tools_condition

# 1. 创建状态图实例
graph_builder = StateGraph(MessagesState)

# 2. 添加所有节点
graph_builder.add_node("generate_query_or_respond", generate_query_or_respond)
graph_builder.add_node("retrieve", retrieve_node)
graph_builder.add_node("grade_documents", grade_documents)
graph_builder.add_node("rewrite_question", rewrite_question)
graph_builder.add_node("generate_answer", generate_answer)

# 3. 定义流转规则
# 入口 → Agent 决策
graph_builder.add_edge(START, "generate_query_or_respond")

# Agent 决策 → 条件路由：有 tool call 则检索，无则结束
graph_builder.add_conditional_edges(
    "generate_query_or_respond",
    tools_condition,
    {"tools": "retrieve", "END": END}
)

# 检索 → 文档评估
graph_builder.add_edge("retrieve", "grade_documents")

# 文档评估 → 条件路由：仅留原始问题则改写，否则生成回答
def route_after_grading(state: MessagesState):
    return "rewrite_question" if len(state["messages"]) == 1 else "generate_answer"
graph_builder.add_conditional_edges(
    "grade_documents",
    route_after_grading,
    {"rewrite_question": "rewrite_question", "generate_answer": "generate_answer"}
)

# 问题改写 → 回到 Agent 决策（重新检索）
graph_builder.add_edge("rewrite_question", "generate_query_or_respond")

# 生成回答 → 结束
graph_builder.add_edge("generate_answer", END)

# 4. 编译生成可运行 Agent
graph = graph_builder.compile()
```

## 五、测试场景与效果
### 1. 技术相关问题（需检索）
```python
response = graph.invoke({"messages": [("human", "Agent 有哪些类型的记忆?")]})
for msg in response["messages"]:
    msg.pretty_print()
```
**流程**：Agent 决策→生成英文查询词（`types of memory in AI agents`）→检索→评估相关→生成回答  
**输出**：基于文档返回 Agent 的记忆类型（感官记忆、短期记忆、长期记忆及子类）。

### 2. 非技术问题（直接回答）
```python
response = graph.invoke({"messages": [("human", "你好,1+1等于几?")]})
for msg in response["messages"]:
    msg.pretty_print()
```
**流程**：Agent 决策→直接回答  
**输出**：`你好!1+1=2｡`

### 3. 复杂对比问题（多轮检索）
```python
response = graph.invoke({"messages": [("human", "什么是 Chain of Thought prompting?它和 Tree of Thoughts 有什么区别?")]})
```
**流程**：Agent 决策→并行检索两个概念→评估相关→生成回答  
**输出**：分别解释两个概念并对比差异。

## 六、完整代码
```python
import bs4
from langchain_community.document_loaders import WebBaseLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_core.messages import SystemMessage
from langchain.tools import tool
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode, tools_condition

# ========== 1. 初始化智谱模型 ==========
ZHIPU_API_KEY = "你的智谱API Key"
ZHIPU_BASE_URL = "https://open.bigmodel.cn/api/paas/v4/"

llm = ChatOpenAI(
    temperature=0,
    model="glm-5",
    openai_api_key=ZHIPU_API_KEY,
    openai_api_base=ZHIPU_BASE_URL,
)

zhipu_embeddings = OpenAIEmbeddings(
    model="embedding-3",
    openai_api_key=ZHIPU_API_KEY,
    openai_api_base=ZHIPU_BASE_URL,
)

# ========== 2. 构建知识库 ==========
loader = WebBaseLoader(
    web_paths=("https://lilianweng.github.io/posts/2023-06-23-agent/",),
    bs_kwargs={"parse_only": bs4.SoupStrainer(
        class_=("post-title", "post-header", "post-content")
    )},
)
docs = loader.load()

text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
all_splits = text_splitter.split_documents(docs)

vector_store = InMemoryVectorStore(zhipu_embeddings)
vector_store.add_documents(all_splits)
print(f"索引了 {len(all_splits)} 个文档块")

# ========== 3. 检索工具 ==========
@tool(response_format="content_and_artifact")
def retrieve(query: str):
    """Search blog posts for relevant information to answer user questions."""
    retrieved_docs = vector_store.similarity_search(query, k=5)
    serialized = "\n\n".join(
        f"Source: {doc.metadata}\nContent: {doc.page_content}"
        for doc in retrieved_docs
    )
    return serialized, retrieved_docs

# ========== 4. 图节点 ==========
def generate_query_or_respond(state: MessagesState):
    llm_with_tools = llm.bind_tools([retrieve])
    system_prompt = SystemMessage(
        content="你是博客问答助手。涉及 AI Agent、LLM、任务分解、记忆机制等技术话题时，必须用 retrieve 工具搜索；与技术无关的问题直接回答。重要：调用工具时 query 必须为英文。"
    )
    response = llm_with_tools.invoke([system_prompt] + state["messages"])
    if response.tool_calls:
        print(f"--- Agent 决定搜索: {response.tool_calls[0]['args']['query']} ---")
    else:
        print("--- Agent 决定直接回答 ---")
    return {"messages": [response]}

retrieve_node = ToolNode([retrieve])

def grade_documents(state: MessagesState):
    messages = state["messages"]
    question = messages[0].content
    tool_content = ""
    for msg in reversed(messages):
        if hasattr(msg, "type") and msg.type == "tool":
            tool_content = msg.content
            break
    grading_prompt = (
        "你是相关性评分员。文档包含问题相关关键词或语义则回复 yes，完全无关则回复 no。只输出 yes 或 no，不解释。\n\n"
        f"用户问题: {question}\n\n"
        f"检索到的文档: {tool_content[:2000]}"
    )
    response = llm.invoke([("human", grading_prompt)])
    score = response.content.strip().lower()
    if "yes" in score:
        print("--- 评估结果: 文档相关 ---")
        return {"messages": state["messages"]}
    else:
        print("--- 评估结果: 文档不相关 ---")
        return {"messages": [messages[0]]}

def rewrite_question(state: MessagesState):
    question = state["messages"][0].content
    response = llm.invoke(
        [SystemMessage(content="优化问题使其更适合语义搜索，仅输出改写后的问题，不解释。")]
        + [("human", question)]
    )
    rewritten = response.content
    print(f"--- 问题改写: {question} ￫ {rewritten} ---")
    return {"messages": [("human", rewritten)]}

def generate_answer(state: MessagesState):
    messages = state["messages"]
    question = messages[0].content
    docs_content = ""
    for msg in reversed(messages):
        if hasattr(msg, "type") and msg.type == "tool":
            docs_content = msg.content
            break
    response = llm.invoke(
        [SystemMessage(content="根据参考文档回答问题，中文输出，简洁准确。")]
        + [("human", f"问题: {question}\n\n参考文档:\n{docs_content}")]
    )
    return {"messages": [response]}

# ========== 5. 组装图 ==========
graph_builder = StateGraph(MessagesState)
graph_builder.add_node("generate_query_or_respond", generate_query_or_respond)
graph_builder.add_node("retrieve", retrieve_node)
graph_builder.add_node("grade_documents", grade_documents)
graph_builder.add_node("rewrite_question", rewrite_question)
graph_builder.add_node("generate_answer", generate_answer)

graph_builder.add_edge(START, "generate_query_or_respond")
graph_builder.add_conditional_edges(
    "generate_query_or_respond",
    tools_condition,
    {"tools": "retrieve", "END": END}
)
graph_builder.add_edge("retrieve", "grade_documents")

def route_after_grading(state: MessagesState):
    return "rewrite_question" if len(state["messages"]) == 1 else "generate_answer"
graph_builder.add_conditional_edges(
    "grade_documents",
    route_after_grading,
    {"rewrite_question": "rewrite_question", "generate_answer": "generate_answer"}
)
graph_builder.add_edge("rewrite_question", "generate_query_or_respond")
graph_builder.add_edge("generate_answer", END)

graph = graph_builder.compile()

# ========== 6. 测试 ==========
response = graph.invoke({"messages": [("human", "Agent 有哪些类型的记忆?")]})
for msg in response["messages"]:
    msg.pretty_print()
```

## 七、关键坑点总结
1.  智谱模型结构化输出不稳定：避免使用 `with_structured_output`，改用文本匹配（如 yes/no 判断）更稳定；
2.  中英文检索匹配：英文知识库需用英文查询词，需在 system prompt 中明确要求；
3.  检索 k 值配置：k=5 比 k=2 命中率更高，适配智谱 embedding-3 的语义匹配精度；
4.  避免死循环：生产环境需添加重试上限（在 State 中增加 `retry_count` 字段，超过阈值则提示“无相关信息”）；
5.  工具绑定与结构化输出区分：`bind_tools` 用于工具调用，`with_structured_output` 用于强制格式输出，不可混淆。当前文件内容过长，豆包只阅读了前 87%。
