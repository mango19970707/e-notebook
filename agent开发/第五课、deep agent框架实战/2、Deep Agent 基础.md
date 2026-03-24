# Deep Agent 基础课程总结
本课程为Deep Agent入门教程，基于LangGraph构建的`deepagents`独立库展开，讲解其核心定位、适用场景、快速上手流程及自定义能力（模型、工具、技能、记忆、沙箱），重点阐释了**规划、上下文管理、子Agent、长期记忆**四大核心能力的底层逻辑与实操方法，是搭建复杂多步骤任务Agent的基础指南。

## 一、Deep Agent核心概述
### 1. 定位与基础
`deepagents`是用于构建**复杂多步骤任务智能体**的独立Python库，基于LangGraph开发，灵感来自Claude Code、Deep Research等产品，内置规划、文件系统、子Agent生成、长期记忆能力，与LangChain生态无缝兼容。

### 2. 适用/不适用场景
| 适用场景（必用Deep Agent）| 不适用场景（选其他方案）|
|---------------------------------------------|---------------------------------------------|
| 复杂多步骤任务，需规划/任务分解              | 简单单步任务                                 |
| 需管理大量上下文，防止窗口溢出              | 无大量上下文处理需求                         |
| 需子Agent分工，实现上下文隔离                | 单Agent即可完成的简单逻辑                     |
| 需跨对话/跨线程保持长期记忆                  | 无持久化记忆需求，单次会话即可完成           |
| 不适用场景推荐方案：LangChain `create_agent`或自定义LangGraph工作流。|

### 3. 四大核心能力
1. **规划与任务分解**：内置`write_todos`工具，将复杂任务拆分为离散步骤，跟踪进度并随新信息调整计划；
2. **上下文管理**：提供`ls`/`read_file`/`write_file`/`edit_file`文件系统工具，将大上下文卸载到内存，避免窗口溢出；
3. **Subagent生成**：内置`task`工具，生成专门的子Agent实现上下文隔离，主Agent仅聚焦核心协调，子Agent处理专属子任务；
4. **长期记忆**：基于LangGraph的`Store`实现，支持跨线程/跨对话的持久化记忆，可保存/检索历史对话信息。

### 4. 与LangChain生态的关系
Deep Agent是LangChain生态的上层封装，底层依赖三大核心组件，实现全流程能力支撑：
- **LangGraph**：提供底层图执行、状态管理核心能力；
- **LangChain**：实现工具、大模型的无缝集成，自定义工具可直接复用；
- **LangSmith**：提供可观测性、任务评估、生产级部署能力，支持监控与扩缩容。

## 二、快速上手：5步搭建基础Deep Agent
以**调研类Agent**为例，实现互联网搜索+信息汇总能力，全程基于`tavily`搜索工具，默认使用Claude Sonnet 4.5模型。
### 步骤1：安装依赖（推荐`uv`，速度优于`pip`）
```bash
uv init
uv add deepagents tavily-python
uv sync
```
### 步骤2：配置API Key
默认模型为Anthropic Claude，需配置Anthropic和Tavily（搜索）密钥：
```bash
export ANTHROPIC_API_KEY="your-anthropic-api-key"
export TAVILY_API_KEY="your-tavily-api-key"
```
### 步骤3：创建自定义搜索工具
需包含清晰的**类型注解+docstring**，Deep Agent自动识别工具用途：
```python
import os
from typing import Literal
from tavily import TavilyClient
from deepagents import create_deep_agent

# 初始化Tavily客户端
tavily_client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])

# 定义互联网搜索工具
def internet_search(
    query: str,
    max_results: int = 5,
    topic: Literal["general", "news", "finance"] = "general",
    include_raw_content: bool = False,
):
    """Run a web search"""
    return tavily_client.search(
        query,
        max_results=max_results,
        include_raw_content=include_raw_content,
        topic=topic,
    )
```
### 步骤4：创建Deep Agent
通过`system_prompt`定义Agent角色，`tools`传入自定义工具，框架自动集成内置工具：
```python
# 系统提示词：定义为专业研究员，说明工具使用方式
research_instructions = """You are an expert researcher. Your job is to 
conduct thorough research and then write a polished report.
You have access to an internet search tool as your primary means of gathering 
information.
## `internet_search`
Use this to run an internet search for a given query. You can specify the max 
number of results to return, the topic, and whether raw content should be included.
"""

# 创建Deep Agent
agent = create_deep_agent(
    tools=[internet_search],
    system_prompt=research_instructions
)
```
### 步骤5：运行Agent并输出结果
传入用户问题，Agent**自动完成规划→搜索→上下文管理→报告生成**全流程：
```python
# 调用Agent，查询LangGraph相关信息
result = agent.invoke({"messages": [{"role": "user", "content": "What is langgraph?"}]})
# 打印最终回答
print(result["messages"][-1].content)
```
### Agent自动执行流程
无需手动干预，框架内置逻辑驱动Agent完成以下步骤：
1. 规划：调用`write_todos`拆分调研任务为具体步骤；
2. 调研：调用`internet_search`工具收集信息；
3. 上下文管理：通过`write_file`/`read_file`存储/读取长搜索结果；
4. 子Agent生成：复杂子任务自动派发给专属子Agent；
5. 汇总：将所有信息整理为连贯的报告并输出。

## 三、Deep Agent核心自定义能力
### 1. 自定义大模型
默认使用`claude-sonnet-4-5-20250929`，支持替换为**所有LangChain兼容的模型**，可通过`init_chat_model`快捷配置或直接传入LangChain模型类：
```python
from langchain.chat_models import init_chat_model
from deepagents import create_deep_agent

# 快捷配置：OpenAI GPT-5
model = init_chat_model(model="openai:gpt-5")
# 创建Agent并绑定自定义模型
agent = create_deep_agent(model=model, tools=[internet_search])
```

### 2. 自定义系统提示词
Deep Agent内置基于Claude Code的系统提示词（含内置工具使用说明），用户需根据**具体业务场景**追加自定义提示词，定义Agent角色、工作目标、工具使用规范：
```python
from deepagents import create_deep_agent

# 自定义科研调研提示词
research_instructions = """\
You are an expert scientific researcher. Your job is to conduct \
in-depth academic research, search for the latest papers, and then write a \
structured research report with clear citations.
"""
# 绑定自定义提示词
agent = create_deep_agent(
    system_prompt=research_instructions,
    tools=[internet_search]
)
```

### 3. 工具扩展
Deep Agent工具分为**内置工具**和**自定义工具**，两者可无缝结合，自定义工具为普通Python函数，内置工具覆盖规划、文件、子Agent核心能力：
- **内置工具**：`write_todos`（规划）、`ls`/`read_file`/`write_file`（文件管理）、`task`（子Agent生成）；
- **自定义工具**：用户根据需求开发（如搜索、查天气、查数据库），需保证**类型注解+docstring清晰**；
```python
# 直接在tools参数中追加自定义工具，框架自动集成
agent = create_deep_agent(
    tools=[internet_search, 自定义工具1, 自定义工具2],
    system_prompt=research_instructions
)
```

### 4. 技能（Skills）：解决Prompt爆炸与上下文丢失
#### 核心痛点
传统Agent将所有操作指南、业务流程塞进System Prompt，会导致**Prompt臃肿（推理成本高）**和**信息分散（上下文丢失）**，Skills通过**渐进式披露**解决该矛盾。

#### 什么是Skills
Skills是**知识+操作的组合包**，核心为`SKILL.md`文件（含元数据+详细指令），可搭配专属脚本，存放在独立文件夹中，Agent仅**按需加载**，不匹配则不消耗Token：
```
skills/
└── arxiv_search/  # 技能独立文件夹
    ├── SKILL.md   # 核心：元数据+arxiv搜索步骤（必选）
    └── arxiv_tool.py  # 配套脚本（可选）
```
#### `SKILL.md`核心规范
必须包含**Frontmatter元数据**（Agent用于匹配需求），后接详细执行步骤，元数据是Agent判断是否激活技能的关键：
```markdown
# 元数据：Agent仅读取此部分做匹配，不加载全文
name: arxiv_search
description: 当用户询问最新的学术论文、研究进展或需要从arXiv检索信息时使用此技能。
allowed-tools: fetch_url, python_repl

---
# 详细执行指令：匹配成功后才加载
1. 根据用户问题构造arxiv搜索关键词，需包含研究方向+最新年份；
2. 使用python_repl运行arxiv_tool.py脚本执行搜索；
3. 筛选前3篇最相关的论文，提取标题、作者、摘要、链接；
4. 将结果整理为结构化格式，写入文件并汇总给用户。
```
#### Agent加载Skills的三步逻辑
1. **匹配**：用户提问后，Agent仅扫描所有Skills的`description`，匹配需求才激活对应技能；
2. **读取**：仅加载激活技能的`SKILL.md`全文，未激活技能不消耗任何Token；
3. **执行**：根据`SKILL.md`的详细指令，调用指定工具完成任务。
#### 技能加载实操（基于FilesystemBackend）
需配置`checkpointer`（记忆/中断必备），指定技能目录，支持配置人工审批（安全审查）：
```python
from deepagents import create_deep_agent
from deepagents.backends.filesystem import FilesystemBackend
from langgraph.checkpoint.memory import MemorySaver

# 1. 初始化checkpointer（内存版，开发用）
checkpointer = MemorySaver()

# 2. 创建Agent，加载本地skills目录
agent = create_deep_agent(
    backend=FilesystemBackend(root_dir="./my_project"),  # 项目根目录
    skills=["./skills/"],  # 技能目录路径
    interrupt_on={"write_file": True},  # 写文件前需人工审批，安全管控
    checkpointer=checkpointer
)

# 3. 调用Agent，自动匹配并加载arxiv_search技能
result = agent.invoke(
    {"messages": [{"role": "user", "content": "帮我搜一下关于 DeepSeek-R1 的最新论文"}]},
    config={"configurable": {"thread_id": "Class_03"}}
)
```
#### Skills vs 记忆（Memory）：核心区别
| 特性         | Skills（技能）| Memory（记忆/AGENTS.md）|
|--------------|---------------------------------------------|---------------------------------------------|
| 加载时机     | 按需加载（渐进式披露，匹配才加载）| 始终加载（Agent启动即注入上下文）|
| 存放位置     | 独立文件夹中的`SKILL.md`                    | 项目根目录的`AGENTS.md`                     |
| 适用场景     | 复杂任务流程、长文档查询、专业领域操作       | 用户偏好、项目编码规范、全局固定上下文       |
| Token消耗    | 低（不匹配不消耗，仅加载匹配技能）| 高（始终占用固定Token份额）|

### 5. 长期记忆（Long-term Memory）：跨线程/跨对话持久化
#### 基础问题
Deep Agent默认文件系统基于`StateBackend`，仅对**单个线程有效**，会话结束后文件丢失，无法实现跨对话记忆。

#### 解决方案：CompositeBackend混合存储
通过`CompositeBackend`将**不同路径路由到不同后端**，实现**临时存储+持久存储**的混合管理：
- 默认路径：绑定`StateBackend`，为临时存储，线程结束即丢失；
- 专属路径（如`/memories/`）：绑定`StoreBackend`，为持久存储，跨线程/跨对话可访问。
#### 长期记忆配置实操
需配合`LangGraph Store`（开发用`InMemoryStore`，生产用`PostgresStore`）和`checkpointer`：
```python
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore
from langgraph.checkpoint.memory import MemorySaver

# 1. 初始化必要组件
checkpointer = MemorySaver()
store = InMemoryStore()  # 内存版Store，开发用

# 2. 定义混合后端：/memories/路径持久化，其余临时
def make_backend(runtime):
    return CompositeBackend(
        default=StateBackend(runtime),  # 全局默认：临时存储
        routes={"/memories/": StoreBackend(runtime)}  # 专属路径：持久存储
    )

# 3. 创建带长期记忆的Agent
agent = create_deep_agent(
    store=store,  # StoreBackend必须绑定store
    backend=make_backend,
    checkpointer=checkpointer
)
```
#### 长期记忆核心使用场景
1. **跨线程读取**：不同会话（不同thread_id）可读写`/memories/`下的文件，实现信息共享；
2. **存储用户偏好**：将用户习惯保存到`/memories/user_preferences.txt`，后续对话自动读取；
3. **Agent自我演进**：将用户反馈保存到`/memories/instructions.txt`，Agent启动时读取，持续优化行为；
4. **积累项目知识库**：将项目信息保存到`/memories/project_notes.txt`，跨对话复用项目上下文。
#### 生产级持久化：PostgresStore
开发用`InMemoryStore`重启后数据丢失，**生产环境必须使用持久化Store**，如`PostgresStore`：
```python
from langgraph.store.postgres import PostgresStore
import os

# 初始化PostgresStore，从环境变量读取数据库连接串
store_ctx = PostgresStore.from_conn_string(os.environ["DATABASE_URL"])
store = store_ctx.__enter__()
store.setup()  # 初始化数据库表

# 创建Agent时绑定PostgresStore即可
agent = create_deep_agent(
    store=store,
    backend=make_backend,
    checkpointer=checkpointer
)
```

### 6. 沙箱（Sandboxes）：Agent的安全隔离环境
#### 核心矛盾
Agent拥有`python_repl`/Shell执行能力时，若不加约束，可能执行**删库、窃取密钥、恶意请求**等危险操作，沙箱在Agent与物理主机之间建立**防火墙**，实现计算隔离。

#### 沙箱的核心定位
Deep Agent中沙箱是一种**特殊的Backend**，在普通文件Backend基础上，新增**`execute`超级工具**，支持Agent在隔离环境中运行Shell命令/代码，且所有操作在**外部云环境**（Modal/Daytona/Runloop）执行，不影响本地主机。
- 自带标准文件工具：`ls`/`read_file`/`write_file`等；
- 新增核心工具：`execute`（运行Shell命令/安装依赖/执行代码）；
- 计算隔离：所有操作在远程云环境执行，本地主机无风险。

#### 三大主流沙箱供应商
Deep Agent深度集成，各有适配场景：
1. **Modal**：适合AI/ML工作负载，支持为Agent分配GPU，适配模型训练/推理场景；
2. **Daytona**：启动速度极快，适配Web开发、轻量代码执行场景；
3. **Runloop**：提供“一次性开发盒”，用完即焚，安全性拉满，适配高风险操作场景。

#### 沙箱实操（以Daytona为例）
三步实现沙箱配置，核心为**绑定沙箱Backend**，使用后必须关闭沙箱释放资源：
```python
from daytona import Daytona
from langchain_anthropic import ChatAnthropic
from deepagents import create_deep_agent
from langchain_daytona import DaytonaSandbox

# 1. 创建Daytona沙箱实例（远程隔离环境）
sandbox_instance = Daytona().create()

# 2. 将沙箱转为Deep Agent的Backend（核心步骤）
backend = DaytonaSandbox(sandbox=sandbox_instance)

# 3. 创建带沙箱的Agent，绑定backend
agent = create_deep_agent(
    model=ChatAnthropic(model="claude-3-5-sonnet"),
    system_prompt="你是一个拥有沙箱权限的 Python 编程助手｡",
    backend=backend  # 绑定沙箱Backend，Agent获得execute工具
)

# 4. 调用Agent：写代码+执行单元测试，所有操作在沙箱中运行
agent.invoke({"messages": [{"role": "user", "content": "写一个计算斐波那契数列的函数并运行单元测试"}]})

# 5. 释放资源：使用后必须关闭沙箱
sandbox_instance.stop()
```
#### 沙箱文件管理：两个平面
沙箱与本地主机的文件交互通过**上传/下载**实现，分为两个独立视角，不可混淆：
1. **Agent视角（工具平面）**：Agent通过`read_file`/`write_file`读写**沙箱内部**的文件，无法直接访问本地主机；
2. **开发者视角（传输平面）**：通过`upload_files`/`download_files`在**本地主机与沙箱**之间搬运文件；
```python
# 运行前：将本地代码文件上传到沙箱内部
backend.upload_files([("/src/logic.py", b"print('Hello Sandbox')")])

# 运行后：将沙箱内生成的报告下载到本地主机
results = backend.download_files(["/output/report.pdf"])
```
#### 沙箱核心安全警示
**绝对不要在沙箱内存储任何密钥/敏感信息**！
- 沙箱内Agent拥有**最高权限**，若被Prompt注入控制，可通过`print(os.environ)`窃取沙箱内的密钥；
- **正确做法**：将密钥留在本地主机，封装为**自定义工具**，Agent调用工具时在**本地侧读取密钥**，处理完结果后仅将返回值传回沙箱，全程不将密钥传入沙箱。

## 四、核心总结
1. Deep Agent是LangGraph上层封装，专为**复杂多步骤任务**设计，解决传统ReAct Agent的规划缺失、上下文爆炸、无分工能力等痛点；
2. 四大核心能力（规划、上下文管理、子Agent、长期记忆）是其核心优势，内置工具覆盖基础能力，支持自定义工具/模型/提示词扩展；
3. Skills通过**渐进式披露**解决Prompt爆炸问题，按需加载不浪费Token，是实现Agent专业能力的核心方式；
4. 长期记忆基于`CompositeBackend+LangGraph Store`实现，支持跨线程/跨对话持久化，开发与生产需区分Store类型；
5. 沙箱是生产环境必备能力，通过**远程隔离环境**实现Agent的安全操作，核心原则是**敏感信息不进沙箱**；
6. Deep Agent与LangChain生态深度兼容，可通过LangSmith实现部署、监控、评估，是搭建生产级复杂Agent的优选框架。