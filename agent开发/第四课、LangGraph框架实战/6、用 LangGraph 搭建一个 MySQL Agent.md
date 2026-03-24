# LangGraph搭建MySQL Agent课程总结
本课程讲解了基于LangChain+LangGraph搭建MySQL Agent的完整流程，该Agent可将自然语言问题转化为SQL、执行查询并返回自然语言结果，还具备SQL错误自修复能力，核心为ReAct模式，由Agent自主决定工具调用逻辑。

## 一、核心能力与架构
1. **Agent核心流程**：查询数据库结构→生成SQL→验证SQL安全性→执行SQL→自然语言返回结果，报错可自动修复
2. **核心工具**：共5个专属工具，Agent根据需求调用
    - get_database_schema：查询数据库/表结构
    - generate_sql_query：根据自然语言和库结构生成SQL
    - validate_sql_query：验证SQL安全性与语法
    - execute_sql_query：执行验证后的SQL
    - fix_sql_error：根据错误信息修复SQL

## 二、环境准备
### 1. 安装依赖
```
pip install langchain langchain-community langgraph
```
### 2. 数据库连接
使用LangChain的SQLDatabase连接，支持SQLite/MySQL，验证连接成功的标志为可打印表名。
```python
from langchain_community.utilities.sql_database import SQLDatabase
# SQLite连接
db = SQLDatabase.from_uri("sqlite:///db/employees.db")
# MySQL连接
# db = SQLDatabase.from_uri("mysql+pymysql://user:password@localhost/employees")

# 验证连接
print(db.get_usable_table_names())
# 成功输出：['departments', 'dept_emp', 'dept_manager', 'employees', 'salaries', 'titles']
```

## 三、核心工具编写
所有工具通过`@tool`装饰器转为LangChain工具，**docstring需清晰编写**，供Agent判断调用时机，工具内可嵌套LLM调用。
### 工具1：获取数据库结构
```python
from langchain.tools import tool

@tool
def get_database_schema(table_name: str = None):
    """Get database schema information for SQL query generation.
    Use this first to understand the table structure before creating query.

    """
    if table_name:
        tables = db.get_usable_table_names()
        if table_name.lower() in [t.lower() for t in tables]:
            return db.get_table_info([table_name])
        else:
            return f"Error: Table '{table_name}' not found. Available tables: {', '.join(tables)}"
    else:
        return db.get_table_info()
```
### 工具2：生成SQL查询
```python
from langchain_openai import ChatOpenAI

# 初始化LLM，支持Qwen、GPT、DeepSeek等
lm = ChatOpenAI(model="qwen3-32b", temperature=0)

@tool
def generate_sql_query(question: str, schema_info: str = None):
    """Generate SQL query from natural language question.
    Use this after getting the database schema.
    """
    schema_to_use = schema_info if schema_info else db.get_table_info()

    prompt = f"""Based on the database schema:
{schema_to_use}

Generate SQL query to answer this question: {question}

Rules:
- Use only SELECT statements (read-only, no write/update/delete)
- Include only existing columns and tables
- Add appropriate WHERE, GROUP BY, ORDER BY clauses as needed
- Limit results to 10 unless otherwise specified
- Use proper SQL syntax
- Return ONLY the SQL query, nothing else
"""
    response = lm.invoke(prompt)
    sql_query = response.content.strip()

    print(f"[Tool] Generated SQL query: {sql_query[:50]}...")
    return sql_query
```
### 工具3：验证SQL查询
```python
import re

@tool
def validate_sql_query(query: str):
    """Validate SQL query for safety checks and syntax before execution.
    Use this to check if a generated SQL query is safe to run.
    """
    # 清理 markdown 代码块
    clean_query = query.strip()
    clean_query = re.sub(r'```sql\s*', '', clean_query, flags=re.IGNORECASE)
    clean_query = re.sub(r'```\s*', '', clean_query)
    clean_query = clean_query.strip().rstrip(';')

    # 检查危险操作
    dangerous_keywords = ['INSERT', 'UPDATE', 'DELETE', 'DROP', 'ALTER', 'TRUNCATE', 'CREATE']
    for keyword in dangerous_keywords:
        if re.search(rf'\b{keyword}\b', clean_query, re.IGNORECASE):
            return f"Error: Only SELECT statements are allowed. Found '{keyword}' operation."
    # 检查是不是以 SELECT 开头
    if not clean_query.upper().startswith('SELECT'):
        return f"Error: Query must start with SELECT. Got: {clean_query[:30]}..."
    return clean_query
```
### 工具4：执行SQL查询
```python
@tool
def execute_sql_query(query: str):
    """Execute a validated SQL query against the database.
    Use this after generating and validating a SQL query.
    """
    # 先验证
    validated = validate_sql_query.invoke({"query": query})
    if validated.startswith("Error"):
        return f"Query: {query}\nValidation failed with error: {validated}"

    # 再执行
    result = db.run(validated)
    if result:
        return f"Query results:\n{result}"
    else:
        return "Query executed successfully but no results found."
```
### 工具5：修复SQL错误
```python
@tool
def fix_sql_error(original_query: str, error_message: str, question: str):
    """Fix a failed SQL query based on the error message.
    Use this when a SQL query execution fails.
    """
    prompt = f"""The following SQL query failed:
Query: {original_query}
Error: {error_message}
Original question: {question}

Database schema:
{db.get_table_info()}

Please fix the specific error mentioned.
Still answer the original question.
Use only valid table and column names from the schema.
Follow proper SQL syntax.
Return ONLY the corrected SQL query, nothing else.
"""
    response = lm.invoke(prompt)
    fixed_query = response.content.strip()

    print(f"[Tool] Generated fixed SQL query: {fixed_query[:50]}...")
    return fixed_query
```

## 四、搭建LangGraph Agent
核心为定义Agent状态、绑定工具到LLM、创建Agent节点与条件路由、组装LangGraph图结构，实现Agent与工具的循环调用。
### 1. 定义Agent状态
```python
from typing import TypedDict, Annotated
from operator import add

class AgentState(TypedDict):
    messages: Annotated[list, add]
```
### 2. 绑定工具到LLM
```python
tools = [get_database_schema, generate_sql_query, execute_sql_query, fix_sql_error]
llm_with_tools = lm.bind_tools(tools)
```
### 3. 创建Agent节点
```python
from langchain_core.messages import SystemMessage

def agent_node(state: AgentState):
    system_prompt = """You are an expert SQL analyst working with an employee database.
Your workflow for answering questions:
1. Get the database schema to understand the structure
2. Generate a SQL query based on the question
3. Execute the SQL query to get results
4. If a query fails, use the fix tool and try again

Rules:
- Always follow the workflow step by step
- If a query fails, use the fix tool and try again
- Provide clear and informative answers
- Be precise with table and column names
- Handle errors gracefully and try to fix them
- If you fail after 3 attempts, explain what went wrong
"""
    messages = [SystemMessage(content=system_prompt)] + state["messages"]
    response = llm_with_tools.invoke(messages)
    return {"messages": [response]}
```
### 4. 创建条件路由
```python
def should_continue(state: AgentState):
    last_message = state["messages"][-1]
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        print(f"[Router] Calling tools: {[tc['name'] for tc in last_message.tool_calls]}")
        return "tools"
    else:
        print("[Router] Agent is producing final answer")
        return "end"
```
### 5. 组装LangGraph
```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode

def create_sql_agent():
    builder = StateGraph(AgentState)

    # 添加节点
    builder.add_node("agent", agent_node)
    builder.add_node("tools", ToolNode(tools))

    # 添加边
    builder.add_edge("__start__", "agent") # 起点 → Agent
    builder.add_edge("tools", "agent") # 工具 → Agent(执行完工具回到Agent)
    # 添加条件边
    builder.add_conditional_edges("agent", should_continue, {"tools": "tools", "end": END})
    return builder.compile()

agent = create_sql_agent()
```
**图执行逻辑**：START→Agent→判断是否有工具调用→是→Tools→Agent（循环）→否→END

## 五、运行Agent
通过传入自然语言问题的HumanMessage，调用Agent并输出结果，Agent会自主完成工具调用与查询流程。
```python
from langchain_core.messages import HumanMessage

# 简单问题测试
result = agent.invoke({
    "messages": [HumanMessage(content="公司一共有多少员工?")]
})
print(result["messages"][-1].content)

# 复杂多表联查测试
result = agent.invoke({
    "messages": [HumanMessage(content="薪资最高的 5 个员工是谁?他们在哪个部门?")]
})
print(result["messages"][-1].content)
```

## 六、踩坑记录
1. LLM生成的SQL可能带markdown格式（```sql```），需在验证工具中用正则清理；
2. LangGraph中节点名称与条件路由返回值需完全一致，避免拼写不一致；
3. 工具的docstring需详细说明用途和调用时机，否则Agent会乱调工具；
4. 必须在代码层面做SQL验证，仅靠prompt限制危险操作不可靠。