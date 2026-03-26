# Vanna实现自然语言查询数据库课程总结

> 对应原文 PDF：`D:\notebook\agent开发\langGraph框架实战\第七课：用大白话问数据库，Vanna 让不会 SQL 的人也能做数据分析.pdf`
> 说明：在不改变原文含义的前提下做精简总结；代码示例保持完整，不删减。

本课程讲解了开源Python框架Vanna的核心能力与实操部署流程，该框架基于RAG实现**自然语言转SQL**，支持自动执行查询、可视化结果，还能学习业务知识并记忆查询，适配无SQL基础的使用者，且部署简单、兼容多模型和多数据库。

## 一、Vanna核心介绍
1. **定位**：开源Python框架（MIT协议，GitHub 22.5k Star，最新v2.0.2），核心能力是将自然语言转化为SQL查询，自动执行并以表格/图表展示结果。
2. **核心原理**：RAG（检索增强生成），支持先“教”其数据库结构、业务字段含义、常用查询，学习后可精准生成SQL，且**越用越智能**，成功查询会被记忆复用。
3. **核心优势**：区别于普通AI写SQL的弊端，Vanna熟悉业务与库结构，生成的SQL可执行性高，且流程透明、结果直观。

## 二、实操部署：智谱GLM-5对接Vanna（SQLite版）
### 1. 安装依赖
```bash
pip install 'vanna[openai,fastapi]'
```
### 2. 完整运行代码
```python
from vanna import Agent
from vanna.core.registry import ToolRegistry
from vanna.core.user import UserResolver, User, RequestContext
from vanna.tools import RunSqlTool, VisualizeDataTool
from vanna.tools.agent_memory import (
    SaveQuestionToolArgsTool,
    SearchSavedCorrectToolUsesTool,
    SaveTextMemoryTool
)
from vanna.servers.fastapi import VannaFastAPIServer
from vanna.integrations.openai import OpenAILlmService
from vanna.integrations.sqlite import SqliteRunner
from vanna.integrations.local.agent_memory import DemoAgentMemory

# 配置智谱 GLM-5(走 OpenAI 兼容接口)
llm = OpenAILlmService(
    model="glm-5",
    api_key="你的智谱API密钥",
    base_url="https://open.bigmodel.cn/api/paas/v4/"
)
# 连接 SQLite 数据库
db_tool = RunSqlTool(
    sql_runner=SqliteRunner(database_path="./your_database.db")
)
# 配置记忆（内存级，重启丢失）
agent_memory = DemoAgentMemory(max_items=1000)

# 配置用户认证：通过cookie邮箱判断权限
class SimpleUserResolver(UserResolver):
    async def resolve_user(self, request_context: RequestContext) -> User:
        user_email = request_context.get_cookie('vanna_email') or 'guest@example.com'
        group = 'admin' if user_email == 'admin@example.com' else 'user'
        return User(id=user_email, email=user_email, group_memberships=[group])

# 注册工具并分配权限
tools = ToolRegistry()
tools.register_local_tool(db_tool, access_groups=['admin', 'user'])
tools.register_local_tool(SaveQuestionToolArgsTool(), access_groups=['admin'])
tools.register_local_tool(SearchSavedCorrectToolUsesTool(), access_groups=['admin', 'user'])
tools.register_local_tool(SaveTextMemoryTool(), access_groups=['admin', 'user'])
tools.register_local_tool(VisualizeDataTool(), access_groups=['admin', 'user'])

# 创建 Agent 并启动FastAPI服务
agent = Agent(
    llm_service=llm,
    tool_registry=tools,
    user_resolver=SimpleUserResolver(),
    agent_memory=agent_memory
)

server = VannaFastAPIServer(agent)
server.run()  # 访问 http://localhost:8000 使用自带Web界面
```
### 3. 实际运行效果
以查询“薪资最高的员工是谁？”为例，Vanna流式执行流程：
1. 调用`search_saved_correct_tool_uses`检索历史记忆（首次无结果）；
2. 调用`run_sql`自动生成并执行SQL，返回结果；
3. 调用`save_question_tool_args`将本次问答存入记忆，供后续复用；
4. 最终以**表格+文字总结**展示结果，全程步骤透明。

## 三、核心代码模块解析
### 1. 大模型配置
Vanna通过`OpenAILlmService`适配**所有支持OpenAI兼容接口的模型**（国产模型为主），更换模型仅需修改配置，业务代码无需变动；也支持专属模型服务类：
```python
# 对接Anthropic Claude示例
from vanna.integrations.anthropic import AnthropicLlmService
llm = AnthropicLlmService(
    model="claude-sonnet-4-5",
    api_key="你的Anthropic密钥"
)
# 对接本地Ollama模型示例：base_url指向本地接口
llm = OpenAILlmService(
    model="qwen2.5",
    api_key="none",
    base_url="http://localhost:11434/v1/"
)
```
### 2. 工具注册与权限控制
`ToolRegistry`是Vanna 2.0核心，所有Agent可用工具需在此注册，且通过`access_groups`指定**用户组权限**，实现权限隔离：
- 普通工具（如执行SQL、可视化）：开放给`admin`/`user`；
- 核心工具（如保存记忆）：仅开放给`admin`。
### 3. 用户认证
通过实现`UserResolver`的`resolve_user`方法自定义认证逻辑，示例为cookie邮箱判断，生产环境可替换为**JWT/OAuth/现有用户系统**，灵活适配企业需求。
### 4. 记忆系统
- 示例用`DemoAgentMemory`：内存级存储，零配置，适合测试，**重启数据丢失**；
- 生产环境可替换为ChromaDB/Qdrant实现持久化；
- 核心记忆工具形成**学习-检索-应用**闭环：
    1. `SearchSavedCorrectToolUsesTool`：检索相似历史查询；
    2. `SaveQuestionToolArgsTool`：保存成功的问答（问题+SQL）；
    3. `SaveTextMemoryTool`：保存业务文本知识（如“GMV对应字段total_amount”）。

## 四、Vanna 2.0核心亮点
1. **用户权限与行级安全**：内置用户身份感知，按用户组自动过滤SQL查询结果（如运营仅看部门数据），且自带审计日志，满足合规需求；
2. **自带Web界面**：FastAPI启动后直接生成聊天界面，支持流式输出、表格/图表展示、明暗主题、手机适配；也可通过`<vanna-chat>`组件嵌入自有页面：
   ```html
   <script src="https://img.vanna.ai/vanna-components.js"></script>
   <vanna-chat sse-endpoint="https://your-api.com/chat"></vanna-chat>
   ```
3. **自定义工具扩展**：支持开发自定义工具（如发邮件、调外部API、生成报告），像插件一样为Agent赋能，示例框架：
   ```python
   from vanna.core.tool import Tool, ToolContext, ToolResult
   from pydantic import BaseModel, Field

   # 定义工具入参
   class EmailArgs(BaseModel):
       recipient: str = Field(description="收件人")
       subject: str = Field(description="邮件主题")

   # 实现自定义工具
   class EmailTool(Tool[EmailArgs]):
       @property
       def name(self):
           return "send_email"

       def get_args_schema(self):
           return EmailArgs

       async def execute(self, context: ToolContext, args: EmailArgs):
           # 自定义发邮件逻辑
           return ToolResult(success=True, result_for_llm=f"邮件已发送到{args.recipient}")
   ```

## 五、兼容范围
### 1. 支持的数据库
覆盖主流关系型/数仓/时序数据库：PostgreSQL、MySQL、SQLite、Snowflake、BigQuery、Redshift、Oracle、SQL Server、DuckDB、ClickHouse、Hive、Presto等。
### 2. 支持的大模型
- 海外：OpenAI（GPT-4o/GPT-5）、Anthropic Claude、Google Gemini、Azure OpenAI、AWS Bedrock、Mistral；
- 国产：智谱GLM、DeepSeek、月之暗面Kimi（所有支持OpenAI兼容接口的模型）；
- 本地：Ollama部署的模型（如Qwen2.5），数据本地运行，满足高安全需求。

## 六、与同类Text-to-SQL工具对比
| 工具       | 定位                  | 优势                          | 劣势                          | 适用场景                  |
|------------|-----------------------|-------------------------------|-------------------------------|---------------------------|
| Vanna      | 开发者轻量级框架      | 开源免费、部署灵活、易集成、零前期成本 | 无内置语义层，复杂指标一致性一般 | 个人/小团队、快速搭建内部数据查询工具 |
| Wren AI    | 企业级BI平台          | 内置语义层，查询一致性强      | 前期搭建成本高，需专业数据团队 | 大公司、有标准化指标需求  |
| MindsDB    | 融合ML的Text-to-SQL   | 支持SQL内调ML模型做预测       | 上手门槛高，需ML知识          | 预测分析场景              |
| Text2SQL.ai| 纯云端服务            | 注册即用，操作最简单          | 闭源、仅支持OpenAI、无自定义能力 | 偶尔临时查询              |

## 七、快速试用方法
1. 依赖安装：`pip install 'vanna[openai,fastapi]'`；
2. 替换代码中**API密钥**（智谱密钥从open.bigmodel.cn注册获取，新用户有免费额度）和**数据库路径**；
3. 运行代码，访问`http://localhost:8000`即可使用；
4. 官方资源：
    - 文档：https://vanna.ai/docs/
    - GitHub：https://github.com/vanna-ai/vanna
    - 示例：GitHub仓库`notebooks`目录（支持Google Colab一键运行）；
5. 部署方式：除FastAPI外，还支持Flask、Streamlit、Slack等，可适配Jupyter Notebook、Slack机器人等场景。
