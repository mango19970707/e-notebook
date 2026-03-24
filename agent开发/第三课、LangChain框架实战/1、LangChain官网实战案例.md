# LangChain官网实战案例核心笔记
LangChain官网提供4个典型实战案例，覆盖**文档检索、RAG智能体、数据库交互、语音交互**四大高频场景，核心是将LangChain核心组件（RAG、Agent、Tool、Middleware）落地到实际业务，以下是各案例的核心流程、实现要点与价值总结。

## 一、PDF语义搜索知识库（基础文档检索）
### 核心目标
实现对海量PDF文档（以Nike 2023年10-K报表为例）的**语义搜索**（理解问题意图而非关键词匹配），快速定位相关内容。

### 核心流程（5步完成）
#### 1. 环境准备
安装PDF处理与向量库依赖：
```bash
pip install langchain-community pypdf
```

#### 2. 文档加载（Load）
用`PyPDFLoader`读取PDF，转换为LangChain统一的`Document`对象（包含`page_content`内容和`metadata`元数据）：
```python
from langchain_community.document_loaders import PyPDFLoader
loader = PyPDFLoader("nke-10k-2023.pdf")
docs = loader.load()  # 输出107页文档（每页一个Document对象）
```

#### 3. 文档切割（Split）
用`RecursiveCharacterTextSplitter`拆分长文档，避免跨页信息丢失：
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
all_splits = text_splitter.split_documents(docs)  # 拆分后生成多个短文本块
```

#### 4. 向量化与存储（Embed & Store）
用OpenAI Embedding模型将文本块转为向量，存入内存向量库：
```python
from langchain_openai import OpenAIEmbeddings
from langchain_core.vectorstores import InMemoryVectorStore
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
vector_store = InMemoryVectorStore(embeddings)
vector_store.add_documents(all_splits)  # 向量入库
```

#### 5. 语义检索与进阶（Retrieve）
- 基础检索：按问题语义匹配相关文本块，返回来源页码：
  ```python
  results = vector_store.similarity_search("耐克在美国有多少个分销中心?", k=3)
  ```
- 进阶：转为`Retriever`对象，适配LangChain Chain/Agent：
  ```python
  retriever = vector_store.as_retriever(search_kwargs={"k": 1})
  response_docs = retriever.invoke("耐克是什么时候成立的?")
  ```

### 核心价值
- 解决“长PDF精准检索”问题，无需手动翻页；
- 语义搜索支持自然语言提问，比关键词匹配更灵活；
- 可扩展为企业知识库（如产品手册、合规文档检索）。

## 二、RAG Agent（智能检索增强Agent）
### 核心目标
解决传统RAG Chain的“盲目检索”问题，让Agent自主判断“是否需要检索、检索几次”，适配复杂多步问题。

### 核心流程（4步完成）
#### 1. 环境准备
```bash
pip install langchain langchain-text-splitters langchain-community bs4 langchain-openai
```

#### 2. 知识入库（索引阶段）
以Lilian Weng的《LLM Powered Autonomous Agents》博文为知识库，完成“加载→切割→向量化→存储”：
```python
import bs4
from langchain_community.document_loaders import WebBaseLoader
# 1. 加载网页正文
loader = WebBaseLoader(web_paths=("https://lilianweng.github.io/posts/2023-06-23-agent/",),
                       bs_kwargs={"parse_only": bs4.SoupStrainer(class_="post-content")})
docs = loader.load()
# 2. 切割+向量化+存储（同PDF案例逻辑）
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
all_splits = text_splitter.split_documents(docs)
vector_store = InMemoryVectorStore(OpenAIEmbeddings(model="text-embedding-3-large"))
vector_store.add_documents(all_splits)
```

#### 3. 定义检索工具（Tool）
将向量库封装为Agent可调用的工具：
```python
from langchain.tools import tool
@tool(response_format="content_and_artifact")
def retrieve_context(query: str):
    """从博客中检索相关信息回答问题"""
    retrieved_docs = vector_store.similarity_search(query, k=2)
    # 格式化结果（含来源+内容），供模型读取
    serialized = "\n\n".join(f"Source: {doc.metadata}\nContent: {doc.page_content}" for doc in retrieved_docs)
    return serialized, retrieved_docs  # 文本给模型，元数据供应用使用
```

#### 4. 构建并运行Agent
```python
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI
model = ChatOpenAI(model="gpt-4o")
# 系统提示词：告知Agent可调用检索工具
system_prompt = "你可使用检索工具获取LLM Agent相关最新信息，基于检索结果回答"
agent = create_agent(model, tools=[retrieve_context], system_prompt=system_prompt)

# 测试多步问题（需两次检索）
query = "什么是任务分解(Task Decomposition)?查到后列举常见扩展"
for step in agent.stream({"messages": [{"role": "user", "content": query}]}, stream_mode="values"):
    step["messages"][-1].pretty_print()
```

### 核心亮点
- 自主性：Agent自主判断检索时机（简单问题不检索，复杂问题多轮检索）；
- 多步推理：支持“先查定义→再查扩展”的多轮检索，适配复杂需求；
- 资源高效：避免无效检索，节省Token和时间。

## 三、SQL数据库Agent（数据库自然语言交互）
### 核心目标
让非技术用户通过自然语言（如“哪个流派的单曲平均时长最长？”）查询SQL数据库，Agent自动生成SQL、校验语法、执行查询并返回结果。

### 核心流程（5步完成）
#### 1. 环境与数据准备
- 安装依赖：`pip install langchain langgraph langchain-community langchain-openai`
- 连接数据库：使用Chinook音乐商店SQLite数据库，查看表结构：
  ```python
  from langchain_community.utilities import SQLDatabase
  db = SQLDatabase.from_uri("sqlite:///Chinook.db")
  print(db.get_usable_table_names())  # 查看所有表名
  ```

#### 2. 生成数据库工具集（Toolkit）
使用`SQLDatabaseToolkit`自动生成数据库操作工具（无需手动编写）：
```python
from langchain_community.agent_toolkits import SQLDatabaseToolkit
model = ChatOpenAI(model="gpt-4o")
toolkit = SQLDatabaseToolkit(db=db, llm=model)
tools = toolkit.get_tools()  # 包含查表、查结构、SQL校验、执行查询4类工具
```

#### 3. 构建ReAct模式Agent
通过System Prompt规范Agent行为（避免危险操作）：
```python
from langchain.agents import create_agent
system_prompt = """
你是SQL专家，按以下步骤操作：
1. 先查看数据库表名；
2. 查询相关表的结构；
3. 生成SQL（最多返回5条结果）；
4. 执行前用sql_db_query_checker校验语法；
5. 严禁执行INSERT/UPDATE/DELETE等DML语句。
"""
agent = create_agent(model, tools, system_prompt=system_prompt)
```

#### 4. 安全防护：Human in the Loop
添加人工审批中间件，拦截SQL执行操作，避免删库/慢查询风险：
```python
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver
agent = create_agent(
    model, tools, system_prompt=system_prompt,
    middleware=[HumanInTheLoopMiddleware(interrupt_on={"sql_db_query": True})],
    checkpointer=InMemorySaver()  # 存储状态，支持暂停/恢复
)
```

#### 5. 运行Agent并审批
```python
config = {"configurable": {"thread_id": "sql-agent-1"}}
query = "哪个流派的单曲平均时长最长？"
# 流式运行，触发SQL时暂停等待审批
for event in agent.stream({"messages": [{"role": "user", "content": query}]}, config):
    if "__interrupt__" in event:
        print("需审批SQL：", event["__interrupt__"][0].value["action_requests"][0]["description"])
        # 人工批准后继续执行
        from langgraph.types import Command
        agent.stream(Command(resume={"decisions": [{"type": "approve"}]}), config)
```

### 核心价值
- 降低数据库使用门槛：非技术用户无需懂SQL即可查询数据；
- 安全可控：通过“语法校验+人工审批”双重防护，避免危险操作；
- 实战性强：可落地为企业内部数据查询助手（如销售数据、用户数据查询）。

## 四、Voice Agent（实时语音交互Agent）
### 核心目标
构建“听→想→说”的实时语音交互系统，支持语音提问、Agent推理、语音回复，延迟低且交互自然。

### 核心架构（三大模块串联）
| 模块 | 功能 | 技术选型 | 核心要求 |
|------|------|----------|----------|
| STT（Speech-to-Text） | 音频→文字（实时转录） | AssemblyAI | 低延迟，WebSocket流式传输 |
| Agent | 理解意图→调用工具→生成文字回复 | LangChain Agent | 回复简短口语化，支持流式输出 |
| TTS（Text-to-Speech） | 文字→音频（实时播放） | Cartesia/ElevenLabs | 极速流式，音质自然 |

### 核心实现步骤
#### 1. STT模块（实时转录音频）
```python
async def stt_stream(audio_stream):
    stt = AssemblyAISTT(sample_rate=16000)
    # 异步发送音频流
    async def send_audio():
        async for chunk in audio_stream:
            await stt.send_audio(chunk)
    asyncio.create_task(send_audio())
    # 实时接收转录文本
    async for event in stt.receive_events():
        yield event  # 一句话结束后触发Agent
```

#### 2. Agent模块（实时推理回复）
```python
from langchain.agents import create_agent
# 定义业务工具（如三明治店下单工具）
tools = [add_to_order, confirm_order]
agent = create_agent(
    model="anthropic:claude-3-5-haiku",  # 选择低延迟模型
    tools=tools,
    system_prompt="说话简洁口语化，协助用户完成三明治下单"
)

async def agent_stream(transcript_stream):
    async for event in transcript_stream:
        if event.type == "stt_output":
            # 流式生成回复（生成一个词就传递给TTS）
            stream = agent.astream({"messages": [HumanMessage(content=event.transcript)]}, stream_mode="messages")
            async for message, _ in stream:
                if message.text:
                    yield AgentChunkEvent(text=message.text)
```

#### 3. TTS模块（实时生成音频）
```python
async def tts_stream(agent_text_stream):
    tts = CartesiaTTS()  # 流式TTS
    async def process_text():
        async for event in agent_text_stream:
            if event.type == "agent_chunk":
                await tts.send_text(event.text)  # 实时发送文字给TTS
    asyncio.create_task(process_text())
    # 实时接收音频流并返回
    async for audio_chunk in tts.receive_audio():
        yield audio_chunk
```

#### 4. 串联上线（WebSocket接口）
```python
from langchain_core.runnables import RunnableGenerator
# 串联三大模块
voice_pipeline = (
    RunnableGenerator(stt_stream)  # 听
    | RunnableGenerator(agent_stream)  # 想
    | RunnableGenerator(tts_stream)  # 说
)

# WebSocket接口接收音频并返回语音
@app.websocket("/voice-chat")
async def handle_voice(websocket):
    await websocket.accept()
    output_stream = voice_pipeline.atransform(websocket_audio_iter())
    async for event in output_stream:
        if event.type == "tts_chunk":
            await websocket.send_bytes(event.audio)
```

### 核心亮点
- 低延迟：全流程流式处理，“一边说一边转写→一边推理→一边播放”；
- 自然交互：回复口语化，音质接近真人，远超传统TTS；
- 可扩展性强：可落地为语音助手（如智能客服、车载语音、家居语音控制）。

## 四大案例核心总结
| 案例 | 核心组件 | 核心解决问题 | 典型应用场景 |
|------|----------|--------------|--------------|
| PDF语义搜索 | RAG（加载→切割→向量化→存储） | 长文档精准检索 | 企业知识库、文档问答 |
| RAG Agent | Agent+Tool+RAG | 复杂问题多轮检索 | 研究型问答、多轮知识查询 |
| SQL数据库Agent | Agent+SQL Toolkit+HITL | 自然语言查询数据库 | 内部数据查询、业务报表生成 |
| Voice Agent | STT+Agent+TTS | 实时语音交互 | 智能客服、车载语音、家居控制 |

### 共性要点
1. 工具化思维：所有外部能力（检索、数据库操作、语音转换）均封装为Tool，Agent统一调用；
2. 安全优先：生产环境必加防护（如SQL Agent的HITL、Tool调用限流）；
3. 效率优化：通过流式处理、模型选型（低延迟模型）降低交互延迟；
4. 落地导向：案例均为可直接扩展的实战场景，而非纯演示。

是否需要我针对某个案例（如SQL数据库Agent）提供更详细的代码注释或环境配置指南？当前文件内容过长，豆包只阅读了前 96%。