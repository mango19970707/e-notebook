# LangChain核心组件-MCP核心笔记
MCP（Model Context Protocol，模型上下文协议）是LangChain中实现**工具标准化共享**的核心组件，本质是“工具界的USB接口”——通过统一协议定义工具的暴露与调用方式，让工具一次开发即可被所有支持MCP的应用（Claude Desktop、Cursor、自定义Agent等）调用，解决工具重复开发的痛点。

## 一、核心认知：MCP的核心角色与价值
### 1. 核心角色
- **MCP Server**：工具提供者，负责暴露工具（如计算、天气查询），支持本地/远程部署；
- **MCP Client**：工具调用者（如LangChain Agent），负责连接多个MCP Server，统一获取并使用工具；
- **协议价值**：打破工具的应用绑定，实现“一次开发、多端复用”，将工具开发与应用开发解耦。

### 2. 关键依赖库
| 库名 | 核心作用 |
|------|----------|
| `fastmcp` | 快速创建MCP Server，语法类似FastAPI，开发高效 |
| `langchain-mcp-adapters` | LangChain官方适配器，实现Agent与MCP Server的连接 |
| `langgraph` | 构建支持MCP工具调用的Agent |
| `python-dotenv` | 管理环境变量（如API Key） |

## 二、MCP Server：工具的标准化暴露
MCP Server负责将工具按协议暴露，支持两种通信方式（transport），开发语法与Web框架（Flask/FastAPI）类似，核心是`@mcp.tool()`装饰器+工具函数+docstring（模型理解工具的关键）。

### 1. 本地通信：stdio模式（开发/测试用）
- 通信方式：通过命令行标准输入输出（stdin/stdout）传输JSON消息，无需网络配置；
- 适用场景：本地开发、测试，工具仅在本地应用调用；
- 示例：数学工具Server（`math_server.py`）
```python
from fastmcp import FastMCP

# 初始化MCP Server，指定名称
mcp = FastMCP("Math")

# 注册工具（@mcp.tool()装饰器）
@mcp.tool()
def add(a: int, b: int) -> int:
    """两个数相加"""  # docstring必须有，模型靠它识别工具用途
    return a + b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """两个数相乘"""
    return a * b

if __name__ == "__main__":
    # 启动Server，指定通信方式为stdio
    mcp.run(transport="stdio")
```

### 2. 远程通信：HTTP模式（生产部署用）
- 通信方式：启动HTTP服务，Client通过HTTP请求调用工具，支持远程访问；
- 适用场景：生产环境部署，工具供多个远程应用调用；
- 示例：天气工具Server（`weather_server.py`）
```python
from fastmcp import FastMCP

mcp = FastMCP("Weather")

# 支持异步函数（真实场景对接API常用）
@mcp.tool()
async def get_weather(location: str) -> str:
    """获取指定城市的天气信息"""
    # 真实场景替换为第三方天气API调用
    return f"{location}今天晴,气温25°C,适合出门遛弯"

if __name__ == "__main__":
    # 启动HTTP服务（默认端口8000）
    mcp.run(transport="streamable-http")
```

### 3. 两种通信方式对比
| 通信方式 | 核心特点 | 适用场景 | 启动后状态 |
|----------|----------|----------|------------|
| stdio | 无网络依赖、配置简单 | 本地开发/测试 | 无端口/URL，作为子进程被Client启动 |
| streamable-http | 支持远程调用、多应用共享 | 生产部署 | 启动HTTP服务（默认http://127.0.0.1:8000） |

## 三、MCP Client：多Server的统一调用
MCP Client负责连接多个不同通信方式的MCP Server，将工具统一整合后供Agent使用，无需关注Server的部署方式（本地/远程）。

### 1. 核心实现步骤
#### （1）创建Client并配置Server连接
```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI

async def main():
    # 初始化MCP Client，配置多个Server
    client = MultiServerMCPClient({
        # 1. 连接本地stdio模式的Math Server
        "math": {
            "transport": "stdio",
            "command": "python",
            "args": ["/绝对路径/math_server.py"]  # 需指定Server脚本绝对路径
        },
        # 2. 连接远程HTTP模式的Weather Server
        "weather": {
            "transport": "http",
            "url": "http://localhost:8000/mcp"  # HTTP Server的MCP接口地址
        }
    })

    # 2. 获取所有Server的工具（自动整合）
    tools = await client.get_tools()

    # 3. 创建Agent并传入整合后的工具
    llm = ChatOpenAI(model="gpt-4o", temperature=0.1)
    agent = create_agent(llm, tools)

    # 4. 调用Agent使用MCP工具
    # 测试数学工具：(3+5)×12
    math_result = await agent.ainvoke({
        "messages": [{"role": "user", "content": "(3 + 5) × 12 等于多少？"}]
    })
    print("数学结果：", math_result["messages"][-1].content)

    # 测试天气工具：查询纽约天气
    weather_result = await agent.ainvoke({
        "messages": [{"role": "user", "content": "纽约现在天气怎么样？"}]
    })
    print("天气结果：", weather_result["messages"][-1].content)

if __name__ == "__main__":
    asyncio.run(main())
```

#### （2）运行流程
1. 先启动HTTP模式的Weather Server：`python weather_server.py`（stdio模式的Math Server无需手动启动，Client会自动作为子进程启动）；
2. 运行Client脚本：`python client.py`；
3. Agent自动识别工具用途，按需调用对应Server的工具，返回结果。

### 2. 核心亮点
- 多Server统一整合：Client可同时连接stdio/HTTP模式的多个Server，工具使用无感知；
- 无需修改Agent：Agent调用MCP工具的方式与本地工具完全一致，无需适配；
- 扩展性强：新增工具只需开发新的MCP Server，Client配置后即可使用，无需修改Agent逻辑。

## 核心总结
1. MCP的核心价值是**工具标准化与复用**，解决“一个工具多应用重复开发”的痛点；
2. MCP Server开发简单：用`fastmcp`库，`@mcp.tool()`装饰器+工具函数+docstring即可，支持本地（stdio）和远程（HTTP）部署；
3. MCP Client统一调用：整合多个Server的工具，Agent无需关注工具部署位置，调用方式与本地工具一致；
4. 适用场景：需多应用共享工具、工具需独立部署维护、避免重复开发的场景（如企业内部工具库、第三方工具服务）。

是否需要我针对“真实天气API对接+MCP Server部署”提供更详细的实战代码？

## 代码补充（来自 PDF）
用于说明 MCP 工具集成的精简代码骨架。

### 示例 1：加载 MCP 工具
```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient(
    {
        "filesystem": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-filesystem", "D:/data"],
            "transport": "stdio",
        }
    }
)
tools = client.get_tools()
```

### 示例 2：使用 MCP 工具创建 Agent
```python
agent = create_agent("openai:gpt-4.1-mini", tools=tools)
result = agent.invoke({"messages": [{"role": "user", "content": "请列出 data 目录下的所有文件"}]})
```
