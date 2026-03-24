# LangChain核心组件-RAG核心笔记
RAG（Retrieval-Augmented Generation，检索增强生成）是LangChain中解决大模型“知识有限、上下文不足”的核心组件，通过“检索外部知识库+生成回答”的模式，让Agent具备查询最新/私有数据的能力，本质是给模型做“开卷考试”，弥补预训练知识过时、上下文窗口有限的硬伤。

## 一、RAG核心价值与整体流程
### 1. 核心解决的问题
- 知识过时：模型预训练数据有截止日期，无法回答最新信息（如公司新产品、近期事件）；
- 上下文有限：无法将海量文档（如企业知识库、长篇PDF）完整塞进模型上下文窗口；
- 私有数据访问：无需重新训练模型，即可让Agent使用未公开的私有文档（如内部手册）。

### 2. 整体流程：从知识库构建到回答生成
#### （1）知识库构建（索引阶段）：4步完成“数据→可检索向量”
```
原始文档 → 文档加载（Load）→ 文本分割（Split）→ 向量化（Embed）→ 存入向量数据库（Store）
```
#### （2）回答生成（检索+生成阶段）：3种架构按需选择
```
用户问题 → 检索知识库（获取相关片段）→ 模型结合检索结果生成回答
```

## 二、知识库构建：四大核心组件拆解
### 1. 文档加载器（Document Loaders）：统一数据输入
- 核心作用：将不同来源的原始数据（网页、PDF、Notion、数据库等）加载为LangChain统一的`Document`对象；
- 优势：内置160+种加载器，覆盖绝大多数常见数据源，无需手动适配格式；
- 示例（加载网页内容，仅保留正文）：
```python
import bs4
from langchain_community.document_loaders import WebBaseLoader

# 筛选网页中需要的内容（标题、正文）
bs4_strainer = bs4.SoupStrainer(class_=("post-title", "post-content"))
loader = WebBaseLoader(
    web_paths=("https://lilianweng.github.io/posts/2023-06-23-agent/",),
    bs_kwargs={"parse_only": bs4_strainer}
)
docs = loader.load()  # 输出LangChain Document对象列表
```

### 2. 文本分割器（Text Splitters）：拆分长文档
- 核心问题：长文档（如4万字文章）无法直接输入模型，且易导致关键信息被截断；
- 核心组件：`RecursiveCharacterTextSplitter`（最常用），按自然分隔符（换行、句号）递归拆分，保证语义完整；
- 关键参数：
    - `chunk_size`：每个文本块的最大字符数（如1000）；
    - `chunk_overlap`：相邻块的重叠字符数（如200），避免跨块信息丢失；
- 示例：
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,  # 每块最多1000字符
    chunk_overlap=200,  # 重叠200字符
    add_start_index=True  # 记录原始文档中的起始位置
)
all_splits = text_splitter.split_documents(docs)  # 拆分后的文本块列表
```

### 3. 向量化模型（Embedding Models）：语义转向量
- 核心作用：将文本块转换为数值向量（Embedding），语义相近的文本向量距离更近，支撑“语义搜索”（而非关键词匹配）；
- 常用模型：OpenAI的`text-embedding-3-large`、开源的`Sentence-BERT`等；
- 示例：
```python
from langchain_openai import OpenAIEmbeddings
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")  # 初始化向量化模型
```

### 4. 向量数据库（Vector Stores）：存储与检索向量
- 核心作用：专门优化向量的存储和相似度搜索，快速找到与用户问题语义相关的文本块；
- 支持类型：40+种，包括：
    - 测试用：`InMemoryVectorStore`（内存存储，重启丢失）；
    - 生产用：FAISS（本地部署）、Chroma、Pinecone（云端）、PGVector（PostgreSQL扩展）；
- 示例（向量化+存储一步完成）：
```python
from langchain_core.vectorstores import InMemoryVectorStore

vector_store = InMemoryVectorStore(embeddings)  # 初始化向量库
vector_store.add_documents(all_splits)  # 文本块向量化并存入
```

## 三、3种RAG架构：按需选择灵活度与效率
### 1. Agentic RAG：最灵活（Agent自主决策检索）
#### 核心逻辑
将“知识库检索”包装为工具，交给Agent，由Agent自主判断：**是否需要检索、检索什么关键词、检索几次**，无需固定流程。

#### 实现步骤
（1）定义检索工具（包装向量库搜索逻辑）：
```python
from langchain.tools import tool

@tool(response_format="content_and_artifact")
def retrieve_context(query: str):
    """检索知识库相关信息，用于回答问题"""
    # 语义搜索，返回top2相关文本块
    retrieved_docs = vector_store.similarity_search(query, k=2)
    # 格式化结果（给模型看的文本 + 原始文档元数据）
    formatted_text = "\n\n".join(f"来源：{doc.metadata}\n内容：{doc.page_content}" for doc in retrieved_docs)
    return formatted_text, retrieved_docs  # 文本给模型，元数据供应用使用
```
（2）创建Agent并调用：
```python
agent = create_agent(model, tools=[retrieve_context], system_prompt="可调用检索工具获取信息")
# 复杂问题（需多轮检索）
result = agent.invoke({"messages": [{"role": "user", "content": "任务分解的标准方法是什么？再查其常见扩展"}]})
```

#### 核心优势与适用场景
- 优势：灵活应对复杂问题（多轮检索、动态调整关键词），无需检索时不浪费资源；
- 劣势：至少两次模型调用（生成关键词+生成回答），延迟较高；
- 适用场景：多轮对话、研究型任务、复杂问答（如需要交叉验证信息）。

### 2. 2-Step RAG：最快（固定流程，一次模型调用）
#### 核心逻辑
流程固定为“检索→生成”，无需Agent决策：中间件在模型调用前自动检索，将结果注入上下文，模型直接基于检索结果回答，仅需一次模型调用。

#### 实现步骤（用`dynamic_prompt`中间件）：
```python
from langchain.agents.middleware import dynamic_prompt, ModelRequest

@dynamic_prompt
def inject_context(request: ModelRequest) -> str:
    # 提取用户最新问题
    user_query = request.state["messages"][-1].content
    # 自动检索相关文本块
    retrieved_docs = vector_store.similarity_search(user_query)
    context = "\n\n".join(doc.page_content for doc in retrieved_docs)
    # 注入system prompt
    return f"基于以下上下文回答问题：\n{context}"

# 无需工具，中间件自动完成检索
agent = create_agent(model, tools=[], middleware=[inject_context])
```

#### 核心优势与适用场景
- 优势：延迟低（一次模型调用）、流程可预测、实现简单；
- 劣势：每次都检索（即使问题无需外部信息），可能浪费资源；
- 适用场景：简单FAQ、文档问答、延迟敏感场景（如实时客服）。

### 3. Hybrid RAG：最可靠（加质量校验）
#### 核心逻辑
结合前两种架构的优点，增加“查询增强、检索验证、回答验证”三步校验：
1. 查询增强：改写模糊问题（如“什么是CoT？”→“Chain of Thought的定义”），提升检索精度；
2. 检索验证：判断检索结果是否相关，不相关则重新生成关键词检索；
3. 回答验证：检查生成的回答是否完全基于检索结果，避免幻觉。
- 适用场景：对准确性要求极高的领域（医疗、法律、金融），需用LangGraph做精细流程编排。

## 四、实战：完整RAG Agent实现（40行核心代码）
```python
# 1. 知识库构建（索引阶段）
import bs4
from langchain_community.document_loaders import WebBaseLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_core.vectorstores import InMemoryVectorStore

# 加载网页文档
loader = WebBaseLoader(web_paths=("https://lilianweng.github.io/posts/2023-06-23-agent/",),
                       bs_kwargs={"parse_only": bs4.SoupStrainer(class_=("post-title", "post-content"))})
docs = loader.load()

# 分割文本块
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
splits = text_splitter.split_documents(docs)

# 向量化并存储
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
vector_store = InMemoryVectorStore(embeddings)
vector_store.add_documents(splits)

# 2. 检索+生成阶段
from langchain.tools import tool
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model

# 定义检索工具
@tool
def search_blog(query: str) -> str:
    """搜索博客内容回答问题"""
    results = vector_store.similarity_search(query, k=2)
    return "\n\n".join(f"来源：{doc.metadata}\n内容：{doc.page_content}" for doc in results)

# 创建Agent
model = init_chat_model("gpt-4.1")
agent = create_agent(model, tools=[search_blog], system_prompt="用搜索工具查找信息，中文简洁回答")

# 提问并获取结果
result = agent.invoke({"messages": [{"role": "user", "content": "LLM Agent的核心组件有哪些？"}]})
print(result["messages"][-1].content)
```

## 核心总结
1. RAG的核心是“检索增强”：无需重新训练模型，即可让Agent使用外部/私有/最新数据；
2. 知识库构建是基础：4个组件（加载→分割→向量化→存储）缺一不可，`RecursiveCharacterTextSplitter`和`InMemoryVectorStore`是开发测试的首选；
3. 架构选择看需求：
    - 简单快用→2-Step RAG；
    - 复杂灵活→Agentic RAG；
    - 高精度场景→Hybrid RAG；
4. 关键优化点：文本分割的`chunk_overlap`（避免信息丢失）、检索工具的`k值`（控制返回文本块数量，平衡精度与效率）。

是否需要我针对“生产环境向量库（如PGVector）部署+RAG优化”提供更详细的代码示例？