# 用GLM-5构建Text-to-SQL智能体课程总结

> 对应原文 PDF：`D:\notebook\agent开发\langChain框架实战\第二课：用 GLM-5 构建 Text-to-SQL 智能体.pdf`
> 说明：在不改变原文含义的前提下做精简总结；代码示例保持完整，不删减。

本案例基于LangChain框架，以智谱GLM-5为大模型、Chinook音乐商店SQLite数据库为数据源，实现“自然语言→自动探索数据库→生成SQL→执行查询→自然语言回答”的端到端智能体，核心是让Agent像数据分析师一样自主推理、动态决策，而非固定流程执行。

## 一、项目核心概览
### 1. 核心目标
用户输入自然语言问题（如“加拿大有多少客户？”），Agent自动完成：`探索表结构→生成SQL→校验语法→执行查询→返回答案`，支持多表关联、聚合分析等复杂查询，且能自主重试错误SQL。

### 2. 技术栈选型
| 组件 | 核心作用 | 关键特性 |
|------|----------|----------|
| LangChain | Agent框架 | 编排LLM与工具交互，提供统一接口 |
| LangGraph | 底层运行时 | 驱动Agent的ReAct模式状态机循环 |
| ChatOpenAI | LLM接口 | 兼容OpenAI协议，接入智谱GLM-5 |
| SQLDatabaseToolkit | 数据库工具集 | 自动生成表查询、结构查看、SQL校验、执行工具 |
| SQLAlchemy | 数据库连接层 | 封装数据库交互，支持多类型数据库 |
| Chinook DB | 示例数据库 | 含11张表（客户、订单、音轨等），3500+条数据，适合多表查询 |

### 3. 项目结构（极简设计）
```
text-to-sql-agent/
├── agent.py          # 核心代码（约50行）
├── chinook.db        # SQLite示例数据库
├── .env              # API密钥配置（智谱GLM-5、LangSmith）
└── pyproject.toml    # 依赖配置
```

## 二、环境搭建（3步完成）
### 1. 安装依赖
```bash
# 克隆项目→创建虚拟环境→安装依赖
git clone https://github.com/kevinbfrank/text-to-sql-agent.git
cd text-to-sql-agent
uv venv --python 3.11 && source .venv/bin/activate
uv pip install -e .
```

### 2. 准备数据库（Chinook）
下载并初始化音乐商店数据库，含客户、订单、音轨等关联表：
```bash
# 下载SQL脚本→初始化数据库→清理脚本
curl -L -o Chinook_Sqlite.sql "https://raw.githubusercontent.com/lerocha/chinook-database/master/ChinookDatabase/DataSources/Chinook_Sqlite.sql"
sqlite3 chinook.db < Chinook_Sqlite.sql
rm Chinook_Sqlite.sql
```

### 3. 配置API密钥
复制模板文件并填入智谱GLM-5密钥（支持LangSmith调试，可选）：
```bash
cp .env.example .env
```
编辑`.env`：
```env
# 智谱GLM-5配置
ZHIPU_API_KEY=你的智谱API密钥
ZHIPU_BASE_URL=https://open.bigmodel.cn/api/paas/v4/

# LangSmith（可选，用于追踪推理）
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=你的LangSmith密钥
LANGSMITH_PROJECT="text2sql-agent"
```

## 三、核心代码逐行拆解（5大关键步骤）
### 1. 数据库连接（关键参数提升SQL生成准确性）
用`SQLDatabase`封装数据库连接，通过`sample_rows_in_table_info=3`返回表结构时附带3行样本数据，帮助LLM理解字段含义（如看到`Country`值为“Canada”，明确字段用途）：
```python
from langchain_community.utilities import SQLDatabase
db = SQLDatabase.from_uri(
    "sqlite:///chinook.db",
    sample_rows_in_table_info=3  # 附带样本数据，提升字段理解准确性
)
```

### 2. 初始化GLM-5模型（兼容OpenAI协议）
通过`ChatOpenAI`接口接入GLM-5，利用OpenAI兼容协议，换模型仅需修改`model`和`openai_api_base`，业务代码无需改动：
```python
from langchain_openai import ChatOpenAI
import os
from dotenv import load_dotenv

load_dotenv()  # 加载.env文件
model = ChatOpenAI(
    model="glm-5",  # 模型名称
    openai_api_key=os.getenv("ZHIPU_API_KEY"),
    openai_api_base=os.getenv("ZHIPU_BASE_URL"),
    temperature=0.3  # 低温度保证SQL生成的确定性（生产可设为0）
)
```

### 3. 自动生成数据库工具集（Toolkit模式）
`SQLDatabaseToolkit`自动打包4个核心工具，无需手动编写，实现“开箱即用”的数据库交互能力：
```python
from langchain_community.agent_toolkits import SQLDatabaseToolkit
toolkit = SQLDatabaseToolkit(db=db, llm=model)
tools = toolkit.get_tools()  # 生成4个工具
```
#### 工具集详情
| 工具名 | 核心作用 | Agent使用场景 |
|--------|----------|--------------|
| `sql_db_list_tables` | 列出所有表名 | 第一步：探索数据库有哪些可用表 |
| `sql_db_schema` | 查看指定表的DDL+样本数据 | 第二步：理解目标表结构和字段含义 |
| `sql_db_query_checker` | 校验SQL语法正确性 | 执行前：避免语法错误导致查询失败 |
| `sql_db_query` | 执行SQL并返回结果 | 最后一步：获取数据并生成回答 |

### 4. System Prompt工程（定义Agent行为边界）
Prompt是Text-to-SQL Agent的核心调优点，需包含**安全约束、行为规范、质量保证、结果控制**四大维度，强制Agent按“探索→分析→生成→校验→执行”流程操作：
```python
SYSTEM_PROMPT = """
You are an agent designed to interact with a SQL database.
Given an input question, create a syntactically correct {dialect} query to run,
then look at the results of the query and return the answer. Unless the user
specifies a specific number of examples they wish to obtain, always limit your
query to at most {top_k} results.

You can order the results by a relevant column to return the most interesting
examples in the database. Never query for all the columns from a specific table,
only ask for the relevant columns given the question.

You MUST double check your query before executing it. If you get an error while
executing a query, rewrite the query and try again.

DO NOT make any DML statements (INSERT, UPDATE, DELETE, DROP etc.) to the
database.

To start you should ALWAYS look at the tables in the database to see what you
can query. Do NOT skip this step.

Then you should query the schema of the most relevant tables.
"""
```
- 动态参数：`{dialect}`（数据库方言，如sqlite）、`{top_k}`（最大返回结果数，设为5）；
- 关键约束：禁止DML操作、必须先探索表结构、只查必要列、SQL执行前自检、错误重试。

### 5. 组装并调用Agent（ReAct模式）
通过`create_agent`整合模型、工具、Prompt，底层基于LangGraph的ReAct模式（`Think→Act→Observe`循环），Agent自主决策每一步操作：
```python
from langchain.agents import create_agent

# 组装Agent
agent = create_agent(
    model,
    tools,
    system_prompt=SYSTEM_PROMPT.format(dialect=db.dialect, top_k=5)
)

# 调用Agent（输入自然语言问题）
result = agent.invoke({
    "messages": [{"role": "user", "content": "加拿大有多少客户?"}]
})

# 提取最终答案
print(result["messages"][-1].content)  # 输出：There are 8 customers from Canada.
```

## 四、Agent核心工作流程（以“加拿大有多少客户？”为例）
Agent并非一次性生成SQL，而是模拟人类数据分析师的思考过程，分5步完成：
1. **调用`sql_db_list_tables`**：获取数据库中11张表名，确定潜在相关表（如`Customer`）；
2. **调用`sql_db_schema("Customer")`**：查看`Customer`表结构+3行样本数据，识别`Country`字段；
3. **调用`sql_db_query_checker`**：校验SQL `SELECT COUNT(*) FROM Customer WHERE Country = 'Canada'` 语法正确性；
4. **调用`sql_db_query`**：执行SQL，返回结果`[(8,)]`；
5. **生成自然语言回答**：将查询结果转化为“加拿大有8位客户”。

## 五、关键特性与扩展场景
### 1. 支持复杂查询场景
- 聚合查询：`按国家统计收入`→多表关联（`Invoice`+`Customer`）+`SUM`聚合；
- 多表JOIN：`最畅销的5首歌`→`InvoiceLine`+`Track`关联+`GROUP BY`+`ORDER BY`；
- 员工绩效分析：`哪个员工创造的收入最多`→`Employee`+`Customer`+`Invoice`多表关联。

### 2. 可观测性（LangSmith追踪）
配置LangSmith后，可实时查看Agent的完整推理链路，方便调试：
- 工具调用顺序、输入输出；
- 每步LLM的Prompt和返回结果；
- Token用量、耗时分析；
- SQL错误重试过程。

### 3. 模型替换灵活性
通过OpenAI兼容协议，无需修改业务代码即可切换模型：
```python
# 切换为DeepSeek
model = ChatOpenAI(model="deepseek-chat", openai_api_base="https://api.deepseek.com/v1")
# 切换为本地Ollama部署的Qwen
model = ChatOpenAI(model="qwen2.5", openai_api_base="http://localhost:11434/v1")
```

## 六、核心概念与核心理念
### 1. Agent vs Chain（Text-to-SQL场景适配性）
| 维度 | Chain（固定流程） | Agent（自主决策） |
|------|------------------|------------------|
| 执行方式 | 按预设步骤执行 | 动态选择下一步操作 |
| 工具使用 | 固定调用或不使用 | 按需选择工具 |
| 错误处理 | 需预设逻辑 | 自主修改SQL重试 |
| 适用场景 | 简单查询（固定表+字段） | 复杂查询（需探索表结构、多表关联） |

### 2. 核心理念：Agent = LLM + Tools + Prompt
- LLM（GLM-5）：提供推理、决策能力（如判断查哪张表、如何写SQL）；
- Tools（SQLDatabaseToolkit）：提供行动能力（探索表、执行SQL）；
- Prompt：定义行为边界（安全约束、流程规范），避免Agent“失控”。

## 总结
本项目以极简代码（50行核心逻辑）实现了高实用性的Text-to-SQL智能体，关键亮点：
1. Toolkit模式降低开发成本，无需手动编写数据库工具；
2. System Prompt精准定义Agent行为，兼顾安全性和准确性；
3. 兼容OpenAI协议，模型替换灵活；
4. 支持复杂查询和自主重试，落地性强。