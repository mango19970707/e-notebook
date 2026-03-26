# Deep Agents（开源版Claude Code）课程总结

> 对应原文 PDF：`D:\notebook\agent开发\deep agent框架实战\第一课： Deep Agents，_开源版 Claude Code_.pdf`
> 说明：在不改变原文含义的前提下做精简总结；代码示例保持完整，不删减。

本课程讲解了LangChain推出的**Deep Agents**框架（基于`deepagents==0.4.3`，要求Python3.11+），该框架是Claude Code核心能力的开源抽象实现，主打**模型无绑定**，解决了传统LangChain ReAct Agent在复杂任务中**上下文爆炸、不会规划、不会分工**的痛点，让Agent具备“先想再做、拆分任务、分工协作”的深层能力，是搭建生产级复杂任务Agent的核心框架。

## 一、Deep Agents核心定位与解决的问题
1. **核心定位**：非绑定任何大模型的通用Agent框架，堪称“任何模型都能用的Claude Code内核”，底层基于LangChain+LangGraph，沉淀了Agent开发的最佳实践。
2. **传统ReAct Agent的三大痛点**
    - **上下文爆炸**：多步工具调用的结果塞满上下文，模型“遗忘”早期信息，进入“呆滞区”；
    - **无规划能力**：接收到复杂任务后直接闷头执行，不会拆分步骤、制定执行计划；
    - **无分工能力**：单Agent承担所有任务，上下文混杂不同类型信息，效率极低。
3. **核心价值**：通过**四层脚手架能力**，将“浅层”ReAct循环Agent升级为“深层”智能Agent，无需提升模型能力，仅通过框架设计实现任务规划、分工、上下文管理的质变。

## 二、Deep Agents的四大核心能力（四根柱子）
这四大能力是LangChain团队从Claude Code、Deep Research等优秀Agent产品中提炼的通用设计，也是Deep Agents“深”的核心体现：
### 1. 详细的系统提示词
并非简单的三五行指令，而是包含**工具使用、文件管理、任务拆分、中间结果存储**的完整“上岗培训手册”，用户仅需在基础上追加专属业务指令，定义Agent的核心定位。
### 2. 规划工具（`write_todos`）
Agent接收任务后**先制定待办清单**，再按清单逐步执行，清单全程保留在上下文中，让Agent清晰知道“当前执行步骤、剩余任务”，避免跑偏，本质是一种高效的上下文工程。
### 3. 子Agent（Sub-agents）
主Agent可将子任务派发给专属子Agent，**每个子Agent有独立上下文**，完成后将结果通过文件系统同步给主Agent，从根本解决上下文爆炸问题，类似“项目经理派活给专业员工”。
### 4. 虚拟文件系统
Agent具备**读写/编辑/查看文件**的能力，将文件系统作为“外部记忆”和“共享工作区”：长搜索结果写入文件、子Agent结果写入文件、笔记写入文件，仅在需要时读取，让上下文窗口（工作记忆）只保留核心信息，契合人类“脑子记不住就写下来”的思维模式。

## 三、快速上手：5分钟跑通第一个Deep Agent
### 1. 安装依赖
支持`pip`/`uv`（推荐，速度更快）两种方式，自动集成LangChain/LangGraph：
```bash
# pip安装
pip install deepagents tavily-python
# uv安装（推荐）
uv init my-deep-agent
cd my-deep-agent
uv add deepagents tavily-python
uv sync
```
### 2. 配置API Key
默认使用Claude Sonnet 4.5，需Anthropic Key；Tavily为搜索API（免费版每月1000次搜索）：
```bash
export ANTHROPIC_API_KEY="你的-anthropic-key"
export TAVILY_API_KEY="你的-tavily-key"
```
### 3. 核心运行代码（30行实现调研Agent）
实现“调研LangGraph与LangChain的关系”的核心功能，包含自定义搜索工具、系统提示词、Agent创建与运行：
```python
import os
from typing import Literal
from tavily import TavilyClient
from deepagents import create_deep_agent

# 初始化搜索客户端
tavily_client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])

# 定义自定义搜索工具（type hints+docstring必须写清）
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

# 系统提示词：定义Agent为专业研究员
research_instructions = """你是一个专业的研究员｡
你的任务是深入调研用户提出的问题,然后写一份结构清晰的研究报告｡
你可以使用 internet_search 工具来搜索网上的信息｡
请用中文输出｡
"""

# 创建Deep Agent
agent = create_deep_agent(
    tools=[internet_search],
    system_prompt=research_instructions
)

# 运行Agent
result = agent.invoke({
    "messages": [{"role": "user", "content": "LangGraph 是什么?跟 LangChain 是什么关系?"}]
})

# 打印结果
print(result["messages"][-1].content)
```
### 4. 核心运行效果
Agent会**自发完成规划→搜索→文件写入→结果汇总**，而非简单的工具调用循环：先通过`write_todos`制定调研清单，再逐次调用搜索工具，将结果写入虚拟文件，最后读取文件整理成结构化报告。

## 四、核心基础功能
### 1. 模型无绑定：灵活切换大模型
支持**provider:model**快捷格式，或直接传入LangChain模型类（可自定义参数），兼容所有LangChain支持的模型（OpenAI/Azure/Google/Gemini/本地模型等）：
```python
# 快捷格式：OpenAI
agent = create_deep_agent(
    model="openai:gpt-5.2",
    tools=[internet_search],
    system_prompt=research_instructions
)

# 快捷格式：Google Gemini
agent = create_deep_agent(
    model="google_genai:gemini-2.5-flash-lite",
    tools=[internet_search],
    system_prompt=research_instructions
)

# LangChain模型类：自定义参数（推荐）
from langchain_openai import ChatOpenAI
model = ChatOpenAI(model="gpt-4.1", temperature=0.3)
agent = create_deep_agent(
    model=model,
    tools=[internet_search],
    system_prompt=research_instructions
)
```
**注意**：模型推理能力直接影响效果，建议最低使用**GPT-4.1/Claude Sonnet 4.5**级别，小模型会出现“跳过规划、乱调用工具”的问题，退化为浅层Agent。

### 2. 自定义工具：Agent的“手和脚”
- 自定义工具为**普通Python函数**，需包含**清晰的type hints和docstring**（Agent通过这两者识别工具用途，模糊描述会导致工具被忽略）；
- 工具与Deep Agents内置工具（`write_todos`/`read_file`/`ls`等）并列，Agent自动选择调用；
```python
# 自定义查天气/查数据库工具
def get_weather(city: str) -> str:
    """获取某个城市的实时天气，返回天气状况和温度"""
    return f"{city}今天晴,25度"

def query_database(sql: str) -> str:
    """执行SQL查询，仅支持SELECT语句，返回查询结果"""
    return "查询结果: 员工总数300024人"

# 集成自定义工具
agent = create_deep_agent(
    tools=[internet_search, get_weather, query_database],
    system_prompt="你是一个全能助手,可以搜索､查天气､查数据库｡"
)
```

### 3. 流式输出：可视化Agent执行过程
支持流式输出每一步的**规划、工具调用、子Agent状态、文件操作**，解决“傻等结果、无法调试”的问题，可实时发现Agent死循环/无效操作，调试效率大幅提升：
```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    tools=[internet_search],
    system_prompt=research_instructions
)

# 流式输出核心代码
for event in agent.stream(
    {"messages": [{"role": "user", "content": "调研一下 MCP 协议的最新进展"}]},
    stream_mode="updates"
):
    for node_name, node_output in event.items():
        if "messages" in node_output:
            for msg in node_output["messages"]:
                if hasattr(msg, "content") and msg.content:
                    print(f"[{node_name}] {msg.content[:200]}")
                if hasattr(msg, "tool_calls") and msg.tool_calls:
                    for tc in msg.tool_calls:
                        print(f"[工具调用] {tc['name']}: {tc['args']}")
```
**输出效果**：可清晰看到Agent的思考和执行步骤，类似“坐在旁边看Agent干活”，思路不对可随时打断。

### 4. 文件后端（Backends）：定义虚拟文件系统的存储位置
Deep Agents的虚拟文件系统通过**后端**定义实际存储位置，默认为`StateBackend`（内存临时存储），提供5种后端，适配不同场景：

| 后端                | 核心特点                     | 适用场景                     |
|---------------------|------------------------------|------------------------------|
| StateBackend（默认） | 存在LangGraph状态，临时存储   | 一次性任务，无需持久化       |
| FilesystemBackend   | 读写本地磁盘真实文件         | 需要Agent操作本地文件       |
| LocalShellBackend   | 本地文件+支持Shell命令执行   | DevOps场景，需跑终端命令     |
| StoreBackend        | 跨会话持久化存储             | 需要保留Agent的历史文件     |
| CompositeBackend    | 混合路由，不同路径用不同后端 | 复杂场景，区分临时/持久化文件 |

**本地文件后端使用示例**（限制根目录，避免Agent乱操作）：
```python
from deepagents.backends import FilesystemBackend
agent = create_deep_agent(
    backend=FilesystemBackend(root_dir="/Users/me/workspace"),
    tools=[internet_search],
    system_prompt="你是一个研究助手,调研结果写到 output/ 目录下｡"
)
```

## 五、进阶核心功能
### 1. 子Agent：分工协作，解决上下文爆炸
#### 核心优势
- 主Agent派活，子Agent专注单一任务，**独立上下文**不干扰；
- 子Agent完成后销毁，主Agent仅读取其文件结果，**token消耗降低30%左右**；
- 报告质量更高，搜索更有针对性，避免单Agent上下文混杂。
#### 实现代码（搜索+写作双生子Agent）
子Agent通过**字典定义**，包含`name/description/system_prompt/tools/model`，主Agent通过`subagents`参数集成：
```python
# 搜索专家子Agent：专注信息收集
research_subagent = {
    "name": "research-agent",
    "description": "专门负责深度搜索和信息收集",
    "system_prompt": "你是一个搜索专家,擅长从互联网上找到准确､全面的技术信息｡",
    "tools": [internet_search],
    "model": "openai:gpt-4.1", # 子Agent可使用与主Agent不同的模型
}

# 写作专家子Agent：专注报告整理
writer_subagent = {
    "name": "writer-agent",
    "description": "专门负责将调研信息整理成结构清晰的技术报告",
    "system_prompt": "你是一个技术写作专家,擅长把复杂信息写成通俗易懂的中文文章｡",
    "tools": [],
}

# 主Agent：协调子Agent工作
agent = create_deep_agent(
    model="claude-sonnet-4-5-20250929",
    subagents=[research_subagent, writer_subagent],
    system_prompt="你是高级研究经理，协调子Agent完成调研并输出最终报告"
)
```

### 2. 沙箱（Sandboxes）：隔离Agent执行环境
避免Agent执行代码/命令时搞乱本地环境，支持**Modal/Runloop/Daytona**三种沙箱，Agent在**远程隔离环境**中创建文件、装依赖、跑代码，与本地环境完全隔离，是生产环境的必备能力：
```python
import modal
from langchain_anthropic import ChatAnthropic
from deepagents import create_deep_agent
from langchain_modal import ModalSandbox

# 初始化Modal沙箱
app = modal.App.lookup("my-agent-sandbox")
modal_sandbox = modal.Sandbox.create(app=app)
backend = ModalSandbox(sandbox=modal_sandbox)

# 创建带沙箱的Agent
agent = create_deep_agent(
    model=ChatAnthropic(model="claude-sonnet-4-20250514"),
    system_prompt="你是一个 Python 编程助手｡可以写代码并在沙箱中运行｡",
    backend=backend,
)

# 运行并最终销毁沙箱
try:
    result = agent.invoke({
        "messages": [{"role": "user", "content": "写一个爬虫抓取 Hacker News 首页标题,然后运行它"}]
    })
finally:
    modal_sandbox.terminate() # 用完必须销毁，避免资源浪费
```

### 3. 人工审批（Human-in-the-Loop）：给Agent套“安全绳”
对**删文件、发邮件、执行Shell**等高危操作，配置**人工审批后才能执行**，Agent调用对应工具时会暂停，等待用户**同意/编辑/拒绝**，生产环境必备。
#### 核心注意点
- 必须配置`checkpointer`（状态保存器），用于Agent暂停时保存状态、审批后恢复；
- 开发用`MemorySaver`（内存），生产用Redis/数据库版checkpointer。
#### 实现代码
```python
from langchain.tools import tool
from deepagents import create_deep_agent
from langgraph.checkpoint.memory import MemorySaver

# 定义高危/普通工具
@tool
def delete_file(path: str) -> str: """删除指定文件｡"""
    return f"已删除 {path}"
@tool
def read_file(path: str) -> str: """读取指定文件｡"""
    return f"{path} 的内容..."
@tool
def send_email(to: str, subject: str, body: str) -> str: """发送邮件｡"""
    return f"邮件已发送给 {to}"

# 配置checkpointer
checkpointer = MemorySaver()

# 创建带人工审批的Agent
agent = create_deep_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[delete_file, read_file, send_email],
    interrupt_on={
        "delete_file": True, # 需审批：同意/编辑/拒绝
        "read_file": False,  # 无需审批，直接执行
        "send_email": {"allowed_decisions": ["approve", "reject"]}, # 仅允许同意/拒绝
    },
    checkpointer=checkpointer # 必须配置，否则无法暂停
)
```

### 4. Skill系统：给Agent加“专业技能”
- **Skill**：比工具更高层的**专业任务执行知识**（如“如何做技术调研”），工具是底层能力，Skill是“怎么用好工具完成任务”；
- 用**Markdown文件**定义，Agent**按需加载**（渐进式披露），不一开始塞入上下文，节省token；
- 加新能力**无需改代码**，仅需编写Markdown的Skill文件，适配快速迭代。
#### 实现代码
```python
from deepagents import create_deep_agent
from deepagents.backends.utils import create_file_data
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()

# 定义技术调研Skill：Markdown格式，包含流程/注意事项
skill_content = """# 技术调研 Skill
## 调研流程
1. 先明确调研目标和关键问题
2. 从官方文档入手,不要一上来就搜博客
3. 至少对比 3 个信息源
4. 记录每个信息源的可信度
5. 最终报告包含:概述､核心特点､优缺点､适用场景､参考链接
## 注意事项
- 标注信息的时效性(技术文章半年易过时)
- 区分"官方说法"和"社区评价"
- 数据要具体:star 数､下载量､最近更新时间
"""

# 将Skill文件放入虚拟文件系统
skills_files = {
    "/skills/tech-research/SKILL.md": create_file_data(skill_content)
}

# 创建带Skill的Agent
agent = create_deep_agent(
    skills=["./skills/"],
    checkpointer=checkpointer,
)

# 运行：Agent自动加载匹配的Skill
result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "调研一下 Deep Agents 这个框架"}],
        "files": skills_files
    },
    config={"configurable": {"thread_id": "12345"}},
)
```

### 5. 长期记忆（Memory）：跨会话的经验积累
通过**AGENTS.md** Markdown文件实现Agent的长期记忆，存储**用户偏好、已知上下文、业务规则**等，Agent启动时自动读取，解决普通Agent“每次对话从零开始”的问题；配合`StoreBackend`可实现**跨会话持久化记忆**。
#### 实现代码
```python
from deepagents import create_deep_agent
from deepagents.backends.utils import create_file_data
from langgraph.checkpoint.memory import MemorySaver

# 定义记忆内容：用户偏好+已知上下文
memory_content = """# 用户偏好
- 喜欢简洁直白的表达,不要太正式
- 输出用中文
- 代码示例优先用 Python
- 调研报告不超过 2000 字
# 已知上下文
- 用户是全栈开发者,熟悉 Python/TypeScript
- 关注 AI Agent 开发,用过 LangChain/CrewAI
"""

checkpointer = MemorySaver()

# 创建带长期记忆的Agent
agent = create_deep_agent(
    memory=["/AGENTS.md"],
    checkpointer=checkpointer,
)

# 运行：Agent读取记忆并按偏好输出
result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "帮我看看 LangGraph 有啥新功能"}],
        "files": {"/AGENTS.md": create_file_data(memory_content)},
    },
    config={"configurable": {"thread_id": "session-001"}},
)
```

### 6. 结构化输出：固定格式返回结果
通过**Pydantic模型**定义Agent的输出格式，让Agent返回结构化数据（而非自由文本），无需从文本中解析，适合API服务、数据分析等场景，前端可直接对接。
#### 实现代码
```python
from pydantic import BaseModel, Field
from deepagents import create_deep_agent

# 定义Pydantic模型：竞品分析报告格式
class CompetitorAnalysis(BaseModel):
    """竞品分析报告"""
    name: str = Field(description="产品名称")
    category: str = Field(description="产品类别")
    strengths: list[str] = Field(description="优势列表")
    weaknesses: list[str] = Field(description="劣势列表")
    pricing: str = Field(description="定价策略")
    recommendation: str = Field(description="一句话推荐建议")

# 创建带结构化输出的Agent
agent = create_deep_agent(
    response_format=CompetitorAnalysis,
    tools=[internet_search]
)

# 运行：返回Pydantic对象
result = agent.invoke({
    "messages": [{"role": "user", "content": "分析一下 Cursor 这个产品"}]
})

# 提取结构化结果
analysis = result["structured_response"]
print(analysis.name) # 直接取字段：Cursor
print(analysis.strengths) # 直接取列表：["AI 代码补全准确率高", ...]
```

### 7. 中间件（Middleware）：Agent的“中间层”
Deep Agents内置多种中间件，实现**工具调用日志、上下文自动压缩、Prompt缓存、工具调用修复**等能力，也支持自定义中间件，核心用于**增强Agent能力、监控Agent行为**。
#### 内置核心中间件
- `TodoListMiddleware`：提供`write_todos`规划工具；
- `FilesystemMiddleware`：提供文件读写能力；
- `SubAgentMiddleware`：提供子Agent派发能力；
- `SummarizationMiddleware`：上下文过长时**自动压缩早期内容**，避免窗口溢出；
- `AnthropicPromptCachingMiddleware`：Claude模型的Prompt缓存，节省token成本。
#### 自定义中间件示例（工具调用日志）
```python
from langchain.agents.middleware import wrap_tool_call
from deepagents import create_deep_agent

# 自定义日志中间件：记录工具调用次数和名称
call_count = [0]
@wrap_tool_call
def log_tool_calls(request, handler):
    call_count[0] += 1
    tool_name = request.name if hasattr(request, 'name') else str(request)
    print(f"[日志] 第 {call_count[0]} 次工具调用: {tool_name}")
    result = handler(request)
    print(f"[日志] 工具调用完成")
    return result

# 集成自定义中间件
agent = create_deep_agent(
    tools=[internet_search],
    middleware=[log_tool_calls],
)
```
**注意**：自定义中间件**不可修改self属性**（会引发并发竞态问题），需将状态存入`graph state`。

## 六、实战：搭建完整的调研Agent
整合**子Agent、流式输出、规划工具、文件读写**，实现“用户输入主题→Agent自动规划→子Agent搜索/分析→主Agent汇总输出报告”的完整流程：
```python
import os
from typing import Literal
from tavily import TavilyClient
from deepagents import create_deep_agent
from langgraph.checkpoint.memory import MemorySaver

# 1. 初始化搜索工具
tavily_client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])
def internet_search(
    query: str,max_results: int = 5,
    topic: Literal["general", "news", "finance"] = "general",
    include_raw_content: bool = False,
):
    """在互联网上搜索信息,返回相关网页的标题､摘要和链接"""
    return tavily_client.search(query,max_results=max_results,include_raw_content=include_raw_content,topic=topic)

# 2. 定义子Agent
search_subagent = { # 搜索子Agent：多关键词搜索+结果写入文件
    "name": "search-expert",
    "description": "负责执行搜索任务,从互联网上收集指定主题的信息",
    "system_prompt": """你是一个搜索专家｡收到搜索任务后:
1. 用多个不同角度的关键词搜索
2. 把有价值的搜索结果整理后写入指定文件
3. 每条信息标注来源链接
4. 不要编造信息,搜不到就说搜不到""",
    "tools": [internet_search],
}
analysis_subagent = { # 分析子Agent：读取文件+结构化分析
    "name": "analysis-expert",
    "description": "负责分析和整理信息,生成结构化的分析报告",
    "system_prompt": """你是一个分析专家｡从文件中读取搜索结果后:
1. 提取关键信息和数据点
2. 识别趋势和模式
3. 评估信息的可靠性
4. 将分析结果写入报告文件""",
    "tools": [],
}

# 3. 主Agent系统提示词：协调子Agent工作
main_prompt = """你是一个高级研究经理｡你的任务是协调搜索专家和分析专家,完成用户的调研任务｡
工作流程:
1. 收到任务后,先用 write_todos 制定调研计划
2. 将搜索任务分配给 search-expert
3. 搜索完成后,将分析任务分配给 analysis-expert
4. 审阅分析结果,整合成最终中文报告
注意事项:
- 确保至少搜索 3 个不同角度
- 报告要有具体数据支撑,不要空洞
- 标注所有信息来源
"""

# 4. 配置checkpointer+创建Agent
checkpointer = MemorySaver()
agent = create_deep_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[internet_search],
    subagents=[search_subagent, analysis_subagent],
    system_prompt=main_prompt,
    checkpointer=checkpointer,
)

# 5. 流式运行Agent
print("=" * 60)
print("调研 Agent 启动")
print("=" * 60)
for event in agent.stream(
    {
        "messages": [{"role": "user", "content": "帮我调研一下 2026 年最值得关注的 3 个 AI Agent 框架,对比它们的优缺点"}]
    },
    config={"configurable": {"thread_id": "research-001"}},
    stream_mode="updates"
):
    for node_name, node_output in event.items():
        if "messages" in node_output:
            for msg in node_output["messages"]:
                if hasattr(msg, "content") and msg.content:
                    print(f"\n[{node_name}] {msg.content[:300]}")
                if hasattr(msg, "tool_calls") and msg.tool_calls:
                    for tc in msg.tool_calls:
                        print(f"\n[工具] {tc['name']}")
```

## 七、同类框架对比：Deep Agents/Claude Agent SDK/OpenCode
结合实际使用感受，按**需求场景**选择框架，核心差异在于**模型绑定、定位、隐私性**：

| 需求场景                     | 推荐框架          | 核心原因                                                                 |
|------------------------------|-------------------|--------------------------------------------------------------------------|
| 做产品，需灵活切换模型       | Deep Agents       | 模型无绑定，同一套代码适配所有大模型，不被供应商绑死                     |
| 确定只用Claude，追求最佳体验 | Claude Agent SDK  | 针对Claude深度优化，权限管理/ MCP集成更原生，经大规模产品实战检验         |
| 个人使用，隐私优先/断网环境  | OpenCode          | 支持75+模型，兼容本地Ollama，数据不联网，无需API付费                     |
| 学习Agent架构，易上手        | Deep Agents       | 代码可读性好，基于LangGraph，生态完善，文档清晰                           |
| 生产级部署，需监控/回滚      | Deep Agents       | 基于LangGraph，可利用LangSmith部署/监控/检查点/时间旅行等生产级能力       |

## 八、跑分数据：脚手架比模型更重要
LangChain团队在Terminal Bench 2.0（Agent编码能力评测）的实验证明：
- **模型不变**（GPT-5.2-Codex），仅优化Deep Agents的脚手架（Harness），成绩从52.8%提升至66.5%，杀进Top5；
- 核心优化点：**自我验证**（写完代码强制检查）、**上下文注入**（提前注入目录结构）、**时间预算**（限制Agent无限迭代）；
- 对比：Claude Opus 4.6（更强模型）+ 早期脚手架，跑分仅59.6%，**不如优化后的GPT-5.2+Deep Agents**。
  **结论**：Agent的效果并非仅由模型决定，**脚手架的设计和最佳实践**同样关键，Deep Agents的核心价值就是沉淀这些脚手架能力。

## 九、踩坑记录：开发必避的6个问题
1. **小模型带不动**：最低使用GPT-4.1/Claude Sonnet 4.5，小模型会跳过规划、乱调用工具，退化为浅层Agent；
2. **token消耗更高**：规划/子Agent/文件操作均消耗token，复杂任务比普通Agent高40%左右，可通过调小搜索结果数控制成本；
3. **虚拟文件路径问题**：`StateBackend`的路径必须以`/`开头（如`/output/report.md`），相对路径会创建文件失败；
4. **忘记配置checkpointer**：使用人工审批/长期记忆时必须配置，否则Agent暂停/记忆功能会报错，错误信息不明显；
5. **并发竞态问题**：自定义中间件不可修改`self`属性，需将状态存入`graph state`，否则多子Agent并发时会出bug；
6. **流式输出刷屏**：`stream`的event包含内部系统消息，需针对性过滤`node_name`和`message`，避免无效信息刷屏。

## 十、生产级部署
Deep Agents基于LangGraph，支持**LangSmith部署**（托管式，无需自己搞Docker/K8s，自带扩缩容/监控）和**自定义部署**（包在FastAPI/Flask中），以下为FastAPI最小化部署示例（含普通/流式接口）：
```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from deepagents import create_deep_agent
import json
from tavily import TavilyClient
import os

# 初始化FastAPI+搜索工具
app = FastAPI()
tavily_client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])
def internet_search(query: str):
    """Run a web search"""
    return tavily_client.search(query, max_results=3)

# 创建Agent
agent = create_deep_agent(
    tools=[internet_search],
    system_prompt="你是一个专业的研究助手,用中文简洁回答问题｡"
)

# 普通聊天接口
@app.post("/chat")
async def chat(message: str):
    result = agent.invoke({
        "messages": [{"role": "user", "content": message}]
    })
    return {"response": result["messages"][-1].content}

# 流式聊天接口
@app.post("/chat/stream")
async def chat_stream(message: str):
    def generate():
        for event in agent.stream(
            {"messages": [{"role": "user", "content": message}]},
            stream_mode="updates"
        ):
            yield json.dumps(event, default=str) + "\n"
    return StreamingResponse(generate(), media_type="text/event-stream")
```
**生产注意**：需追加**异常处理、用户认证、接口限流、日志监控**，上述为基础示例。

## 十一、框架总结与适用场景
### 1. Deep Agents核心优势
- 模型无绑定，灵活切换，不被供应商绑死；
- 解决传统Agent的三大痛点，支持规划/分工/外部记忆，适配复杂任务；
- 基于LangGraph，生态完善，支持LangSmith部署/监控，适合生产级开发；
- 能力可扩展：Skill/记忆/沙箱/人工审批，覆盖从开发到生产的全流程需求；
- 代码简洁，自定义能力强，工具/Skill/中间件均可灵活扩展。

### 2. 局限性
- 对模型推理能力有要求，小模型效果差；
- token消耗高于普通Agent；
- 生态尚在早期，中文社区教程/案例较少，问题需查英文文档/GitHub Issues。

### 3. 适用/不适用场景
| 适用场景                     | 不适用场景                     |
|------------------------------|--------------------------------|
| 生产级复杂任务Agent开发      | 快速搭demo/简单单步任务        |
| 需灵活切换大模型的产品开发   | 仅使用Claude，追求极致体验     |
| 需分工/规划/上下文管理的Agent | 个人轻量使用，隐私优先/断网环境 |
| 需生产级部署/监控/扩缩容     | 模型资源有限，仅能使用小模型   |

### 4. 官方资源
- GitHub地址：https://github.com/langchain-ai/deepagents
- 核心依赖：`deepagents==0.4.3`、Python3.11+、LangChain/LangGraph
