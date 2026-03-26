# LangChain快速入门-旅行助手核心笔记
LangChain是**开源Agent开发框架**，支持Python/TypeScript，可快速集成大模型与工具，轻松切换模型避免供应商锁定，本次以**智谱GLM-4**为大模型、**旅行助手**为实战场景，从基础入门到完整Web应用实现，拆解LangChain核心开发流程。

## 一、LangChain基础入门：核心环境与最简Agent实现
### 1. 环境准备
- **安装依赖**：支持`pip`/`uv`两种方式，需安装核心库+大模型厂商集成库（如智谱/OpenAI/Anthropic）
- **API Key配置**：从智谱控制台获取，通过环境变量设置`ZHIPU_API_KEY`
```bash
# 核心安装命令
uv venv && source .venv/bin/activate
pip install -U langchain langchain-openai langchain-anthropic
export ZHIPU_API_KEY="你的智谱API密钥"
```

### 2. 最简Agent开发四步走
#### （1）创建LLM实例
配置大模型参数（温度、模型名、API地址），对接智谱GLM-4
#### （2）定义工具
用`@tool`装饰器创建原子工具，**docstring是关键**（Agent通过它理解工具用途）
#### （3）创建Agent
通过`create_agent`整合LLM与工具集
#### （4）运行Agent
传入用户问题，通过`invoke`调用，可查看完整消息历史追踪工具调用过程

### 3. 实战效果
Agent可自主理解任务逻辑，**串联调用多工具**（如先`add`计算和，再`multiply`做乘积），并汇总结果生成自然语言回答，全程无需人工干预。

## 二、集成真实API工具：对接外部服务实现实用能力
基于基础Agent，集成**心知天气API**（需HMAC-SHA1签名验证）和**本地时间工具**，实现真实场景的信息查询，核心步骤：
### 1. 第三方API适配
- 处理API签名：按服务商要求生成签名（如心知天气的`ts+uid+sig`组合）
- 封装工具函数：用`@tool`装饰器包装API调用逻辑，统一返回格式（自然语言文本）
### 2. 工具组合与调用
将自定义工具（`get_weather`/`get_current_time`）加入工具集，Agent可**智能判断并调用对应工具**，同时对结果做二次加工（如天气查询后添加保暖建议）。
### 3. 核心亮点
实现**从“模拟工具”到“真实工具”**的跨越，让Agent具备对接外部真实数据源/服务的能力，落地实际业务场景。

## 三、Agent记忆与多轮对话：解决上下文丢失问题
### 1. 记忆的核心价值
解决基础Agent“单次对话独立、无上下文”的问题，让Agent能理解用户后续提问的指代（如先问北京天气，再问“那上海呢？”，Agent能识别仍为天气查询）。

### 2. LangGraph记忆机制实现
#### （1）基础内存记忆（开发/测试用）
通过`InMemorySaver`创建内存检查点，创建Agent时传入`checkpointer=memory`，实现对话历史内存存储。
#### （2）唯一对话标识`thread_id`
用`thread_id`区分不同对话，**同一`thread_id`共享上下文**，不同`thread_id`相互独立，通过配置参数传入：
```python
config = {"configurable": {"thread_id": "user-001"}}
agent.invoke(消息, config)
```
#### （3）生产环境持久化记忆
替换`InMemorySaver`为**数据库级检查点**（如`PostgresSaver`），将对话历史存入PostgreSQL，避免程序重启后丢失。

### 3. 实战效果
Agent可跨轮次记住关键信息，甚至基于历史信息做对比分析（如先查北京/上海天气，再问“哪个城市更冷？”，Agent能直接基于历史数据回答）。

## 四、处理复杂任务：多工具协作与工作流设计
### 1. 简单任务vs复杂任务
- **简单任务**：单工具调用即可完成（如查单个城市天气）
- **复杂任务**：需**多工具串联+步骤规划+结果汇总**（如北京到上海3天旅行规划）

### 2. 复杂任务核心设计：定制化工具集
围绕旅行规划场景，设计协同工具集，每个工具聚焦单一能力，为Agent提供完整的任务支撑：
| 工具名 | 核心功能 | 任务用途 |
|--------|----------|----------|
| `get_weather` | 查询城市实时天气 | 穿衣/出行建议 |
| `get_current_time` | 获取当前时间 | 确定出发时间参考 |
| `search_attractions` | 搜索城市热门景点 | 游玩路线规划 |
| `estimate_travel_time` | 估算城市间交通时间 | 交通方式选择 |
| `create_plan` | 生成旅行计划 | 汇总所有信息输出最终建议 |

### 3. Agent核心能力：自主任务分解与执行
无需开发者硬编码步骤，Agent能**自动分析用户需求→拆解为子任务→按逻辑调用对应工具→汇总所有结果→生成结构化答案**（如旅行规划会依次查两地天气、推荐景点、估算交通，最终输出3天详细行程）。

## 五、构建完整Web应用：将Agent打包为可落地产品
整合前四节知识，基于**Flask+LangChain+前端HTML/CSS/JS**，构建可直接访问的**旅行规划助手Web应用**，实现从“开发调试”到“产品落地”的闭环。

### 1. 技术栈与应用架构
- **前端**：HTML/CSS/JS实现交互界面（对话窗口、输入框、工具调用记录展示）
- **后端**：Flask提供API接口（`/chat`处理对话、`/new_chat`创建新对话）
- **核心**：LangChain Agent负责智能推理、工具调用、多轮记忆
- **通信**：前后端通过REST API交互，传递用户消息、`thread_id`、Agent回复、工具调用记录

### 2. 项目核心结构
```
lesson05/
├── agent.py       # 核心Agent定义（工具+LLM+记忆）
├── app.py         # Flask后端API
├── templates/
│   └── index.html # 前端交互页面
├── static/
│   ├── style.css  # 前端样式
│   └── 静态资源
├── requirements.txt # 项目依赖
└── run.sh         # 启动脚本
```

### 3. 核心功能亮点
1. **完整交互体验**：浏览器端直接对话，支持发送问题、创建新对话
2. **多轮记忆保留**：同一对话自动维护上下文，无需重复说明
3. **工具调用透明**：展示Agent调用的工具及返回结果，让推理过程可追溯
4. **结构化结果输出**：复杂任务（如旅行规划）返回格式化答案（分天气、景点、交通、行程）
5. **异常处理**：对空消息、网络错误、API调用失败做友好提示

### 4. 启动与访问
```bash
cd lesson05 && ./run.sh
# 浏览器访问 http://localhost:8080
```

## 六、课程核心总结
本次LangChain快速入门以**旅行助手**为实战载体，完成了从**基础环境搭建→模拟工具开发→真实API集成→多轮对话实现→复杂任务处理→完整Web应用构建**的全流程开发，核心掌握：
1. LangChain框架的核心使用方式：LLM创建、工具定义、Agent构建
2. 工具开发的关键：`@tool`装饰器、docstring规范、第三方API适配
3. 多轮对话的核心：`Checkpointer`记忆机制、`thread_id`对话标识
4. 复杂任务的设计：场景化工具集、Agent自主任务分解能力
5. Agent产品化：Flask后端封装、前后端交互、记忆持久化
6. 核心思想：**LangChain让Agent开发“低代码化”**，无需关注底层推理，只需聚焦工具设计和业务场景，快速实现智能体落地。

## 代码补充（来自 PDF）
用于快速练习的精简代码骨架。

### 示例 1：基础旅行助手
```python
from langchain.agents import create_agent
from langchain.tools import tool

@tool
def get_weather(city: str) -> str:
    """返回城市天气概览。"""
    return f"{city}: 晴"

agent = create_agent("openai:gpt-4.1-mini", tools=[get_weather])
print(agent.invoke({"messages": [{"role": "user", "content": "帮我查一下上海今天的天气"}]}))
```

### 示例 2：流式输出
```python
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "帮我做一个北京三日游计划"}]},
    stream_mode="values",
):
    print(chunk["messages"][-1].content)
```
