# 从零构建AI财经助手：LangChain+MCP+多Agent实战笔记
本项目基于LangChain、MCP协议与多Agent架构，打造了一款具备股票分析、财务解读、财经资讯检索功能的智能助手，核心亮点是**Agent自主决策工具调用**、**工具跨应用共享**、**多专家协作**与**流式交互体验**，技术栈轻量化且落地性强。

## 一、项目核心概览
### 1. 核心能力
- 大盘走势查询与分析（上证指数、深证成指等）；
- 个股基本面分析（价格、市值、市盈率等）；
- 财务报表解读（利润表、资产负债表、现金流）；
- 财经新闻检索与市场情绪分析；
- 多股横向对比与分析师评级参考。

### 2. 技术架构（三层架构设计）
| 层级 | 核心组件 | 技术选型 | 核心职责 |
|------|----------|----------|----------|
| 前端层 | 交互界面 | 原生HTML/CSS/JS | 暗色卡片式UI、Markdown渲染、SSE流式接收、快捷提问按钮 |
| 服务层 | 逻辑中枢 | FastAPI + LangChain + LangGraph | Agent推理循环、MCP Client连接、SSE流式响应、多Agent编排 |
| 工具层 | 能力支撑 | MCP Server + 数据源 | 封装财经工具、对接yfinance/DuckDuckGo、按MCP协议暴露工具 |

### 3. 关键技术栈
| 类别 | 技术/工具 | 核心作用 |
|------|-----------|----------|
| 大模型 | 智谱GLM-5 | 兼容OpenAI API，可无缝替换为其他模型 |
| Agent框架 | LangChain + LangGraph | 工具定义、ReAct模式推理、多Agent状态图编排 |
| 工具协议 | MCP（Model Context Protocol） | 工具标准化封装，支持跨应用共享（如Claude Desktop） |
| Web服务 | FastAPI + Uvicorn | 异步高性能接口、SSE流式通信 |
| 数据源 | yfinance + DuckDuckGo | 股票数据（价格、财报）、免费财经新闻检索 |
| 部署工具 | Docker + Makefile | 容器化部署、快捷命令执行 |

### 4. 项目结构
```
caijing/
├── finance_agent.py        # 核心Agent（工具定义+Prompt+Agent创建）
├── api/
│   └── server_with_mcp.py  # FastAPI服务（MCP Client+SSE流式）
├── agents/
│   ├── mcp_server.py       # MCP Server（工具标准化暴露）
│   ├── research_agent.py    # 研究型Agent（新闻/情绪收集）
│   ├── analysis_agent.py    # 分析型Agent（财务/数据解读）
│   └── multi_agent_system.py# 多Agent协调器（LangGraph状态图）
├── static/
│   ├── index.html          # 前端页面
│   ├── style.css           # 暗色主题样式
│   └── app.js             # 前端交互逻辑（SSE+Markdown渲染）
└── pyproject.toml          # 依赖配置
```

## 二、核心模块实现
### 1. 财经工具定义（8个核心工具）
基于`@tool`装饰器定义原子工具，每个工具聚焦单一功能，通过docstring让LLM理解调用场景，返回JSON格式数据便于解析：
| 工具名 | 数据源 | 核心功能 |
|--------|--------|----------|
| `get_stock_info` | yfinance | 股票基本信息（价格、市值、市盈率等） |
| `get_stock_history` | yfinance | 历史价格数据与趋势统计 |
| `get_financial_statement` | yfinance | 财务报表（利润表/资产负债表/现金流） |
| `get_stock_news` | yfinance | 个股相关新闻 |
| `get_recommendations` | yfinance | 分析师评级与目标价 |
| `compare_stocks` | yfinance | 多股横向对比（基本面/价格） |
| `search_financial_news` | DuckDuckGo | 全网财经新闻检索 |
| `think` | 无 | Agent思考反思（梳理分析思路，无实际数据源） |

### 2. 单Agent实现（ReAct模式）
#### （1）系统Prompt设计（核心控制器）
Prompt需明确**角色定位、工具使用策略、输出格式、边界约束**：
```python
SYSTEM_PROMPT = """你是专业财经研究分析师，擅长股票分析与财务解读。
- 工具策略：查股票先调用get_stock_info，需历史数据用get_stock_history，复杂分析前用think梳理思路；
- 输出格式：1-2句概括+引用块核心结论+表格关键数据+ emoji标题分析要点；
- 重要规则：仅回答财经问题，分析基于真实数据，投资建议需附风险提示。
"""
```

#### （2）Agent组装与运行
```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_agent

# 初始化LLM
llm = ChatOpenAI(
    model="glm-5",
    openai_api_key=ZHIPU_API_KEY,
    temperature=0.3  # 低温度保证分析稳定性
)

# 工具列表
tools = [get_stock_info, get_stock_history, ..., think]

# 创建ReAct Agent（LangGraph驱动）
agent = create_agent(llm, tools=tools, system_prompt=SYSTEM_PROMPT)

# 两种运行模式
def run_agent(query):  # 同步调用，一次性返回
    result = agent.invoke({"messages": [HumanMessage(content=query)]})
    return result["messages"][-1].content

def stream_agent(query):  # 流式调用，实时输出
    for msg, _ in agent.stream({"messages": [HumanMessage(content=query)]}, stream_mode="messages"):
        if msg.tool_calls:
            print(f"调用工具：{msg.tool_calls[0]['name']}")
        elif msg.content:
            print(msg.content, end="", flush=True)
```

### 3. MCP Server：工具跨应用共享
#### （1）核心价值
按MCP协议封装工具，实现“一次开发、多端复用”，支持Claude Desktop、Cursor等外部应用直接调用，无需重复开发。

#### （2）实现步骤
1. 定义纯Python工具实现函数（脱离LangChain依赖）；
2. 通过JSON Schema声明工具参数格式；
3. 处理工具调用（同步工具通过线程池避免阻塞）；
4. 以stdio模式启动Server：
```python
from mcp.server import Server
from mcp.server.stdio import stdio_server

server = Server("finance-agent-mcp")

# 注册工具列表
@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="get_stock_info",
            description="获取股票基本信息",
            inputSchema={"type": "object", "properties": {"ticker": {"type": "string"}},"required": ["ticker"]}
        )
    ]

# 处理工具调用
@server.call_tool()
async def call_tool(name, arguments):
    if name == "get_stock_info":
        result = get_stock_info_impl(arguments["ticker"])  # 纯函数实现
    return [TextContent(type="text", text=json.dumps(result))]

# 启动Server
async def serve():
    async with stdio_server() as (read, write):
        await server.run(read, write)
```

### 4. 多Agent协作系统（分工协作）
#### （1）核心思路
将单Agent拆分为“研究型Agent”和“分析型Agent”，通过LangGraph状态图编排流程，实现“专业的人干专业的事”：
- 研究型Agent：专注信息收集（新闻、市场情绪、分析师评级）；
- 分析型Agent：专注数据分析（基本面、财务报表、多股对比）；
- 协调器：通过关键词判断用户需求，路由至对应Agent，复杂需求则并行协作。

#### （2）LangGraph状态图编排
```python
from langgraph.graph import StateGraph, END

# 定义状态结构
class AgentState(TypedDict):
    messages: list[BaseMessage]
    query: str
    research_result: str
    analysis_result: str
    final_report: str

# 创建状态图
workflow = StateGraph(AgentState)
# 添加节点（路由、研究、分析、综合报告）
workflow.add_node("route", route_query)
workflow.add_node("research", research_node)
workflow.add_node("analysis", analysis_node)
workflow.add_node("synthesize", synthesize_node)
# 条件路由（根据查询关键词判断流程）
workflow.add_conditional_edges("route", route_decision, {
    "research_only": "research",
    "analysis_only": "analysis",
    "parallel": "research"  # 先研究后分析
})
# 编译运行
multi_agent = workflow.compile()
```

### 5. Web服务与流式交互
基于FastAPI实现SSE（Server-Sent Events）流式通信，实时推送工具调用过程和分析结果，提升用户体验：
```python
from sse_starlette.sse import EventSourceResponse

@app.get("/api/chat/stream")
async def stream_chat(message: str):
    async def event_generator():
        async for msg, _ in agent.astream({"messages": [HumanMessage(content=message)]}, stream_mode="messages"):
            # 推送工具调用事件（前端展示思考过程）
            if msg.tool_calls:
                yield {"event": "tool_call", "data": json.dumps({"name": msg.tool_calls[0]["name"]})}
            # 推送文本输出事件（前端实时渲染）
            elif msg.content:
                yield {"event": "message", "data": msg.content}
        # 推送完成标志
        yield {"event": "done", "data": "[DONE]"}
    return EventSourceResponse(event_generator())
```

### 6. 前端实现（暗色卡片式UI）
#### （1）核心功能
- 顶部大盘行情滚动条（实时展示指数与涨跌）；
- 快捷提问按钮（一键查询“大盘走势”“分析茅台”）；
- Markdown富文本渲染（表格、引用块、标题自动美化）；
- 工具调用过程实时展示（让用户感知Agent“思考”过程）。

#### （2）关键交互逻辑
```javascript
// 流式接收并渲染
streamResponse(message) {
    const eventSource = new EventSource(`/api/chat/stream?message=${encodeURIComponent(message)}`);
    // 接收工具调用事件
    eventSource.addEventListener('tool_call', (e) => {
        const data = JSON.parse(e.data);
        this.addThinkingStep(`调用工具：${data.name}`); // 展示思考步骤
    });
    // 接收文本输出事件
    eventSource.addEventListener('message', (e) => {
        this.renderMarkdown(e.data); // 实时渲染Markdown
    });
}
```

## 三、运行与部署
### 1. 环境搭建
```bash
# 安装uv包管理器（推荐）
curl -LsSf https://astral.sh/uv/install.sh | sh
# 克隆项目并安装依赖
git clone <项目仓库> && cd caijing
uv sync  # 自动创建虚拟环境并安装依赖
```

### 2. 三种运行模式
```bash
# 1. Web模式（推荐，带前端界面）
make dev  # 开发模式（自动重载）
# 访问 http://localhost:8000

# 2. 命令行模式（快速测试）
make cli

# 3. 多Agent模式（测试协作功能）
make multi-agent
```

### 3. 部署建议
- 开发环境：本地运行，使用stdio模式MCP Server；
- 生产环境：Docker容器化部署，MCP Server改用HTTP模式，添加数据库持久化Agent状态。

## 四、核心亮点与扩展方向
### 1. 核心亮点
- 自主决策：Agent自动判断工具调用顺序，复杂问题先“思考”再行动；
- 跨应用复用：MCP协议让工具支持Claude Desktop等外部应用；
- 流式体验：SSE实时推送工具调用和分析结果，无等待感；
- 专业输出：标准化Markdown格式，数据可视化（表格+涨跌着色）。

### 2. 扩展方向
- 数据源扩展：接入Wind、Tushare等专业财经数据；
- 功能扩展：添加量化分析、投资组合优化工具；
- 安全增强：加入Human-in-the-Loop审批（如高风险投资建议）；
- 多语言支持：前端添加中英文切换，适配国际化需求。

## 总结
本项目以“实用化、可扩展”为核心，通过LangChain实现Agent自主推理，MCP协议实现工具共享，多Agent架构提升分析专业性，最终打造了一款兼顾易用性和专业性的AI财经助手。核心逻辑可复用至其他垂直领域（如医疗、法律），只需替换工具和Prompt即可快速落地。当前文件内容过长，豆包只阅读了前 68%。