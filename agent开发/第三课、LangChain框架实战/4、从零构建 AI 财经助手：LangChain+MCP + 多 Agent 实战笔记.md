# 从零搭建AI金融研究员（进阶）课程总结

> 对应原文 PDF：`D:\notebook\agent开发\langChain框架实战\第四课：从零搭个能读年报、查实时行情的 AI 金融研究员（进阶）.pdf`
> 说明：在不改变原文含义的前提下做精简总结；代码示例保持完整，不删减。

本项目是一套**多模态+双数据源+规划能力**的高阶金融研究Agent，核心实现“读年报PDF+查实时行情+综合分析”全流程自动化，支持复杂问题拆解、精准数据检索与专业报告生成，技术栈覆盖MCP协议、多模态RAG、向量数据库优化等进阶能力。

## 一、项目核心能力与架构
### 1. 三大核心能力
- 读年报：解析上市公司10-K/10-Q年报PDF，提取文本、表格（资产负债表/利润表）、图表（趋势图/饼图）等关键财务数据；
- 查实时：通过MCP协议对接Yahoo Finance，获取实时股价、市值、涨跌幅、分析师评级等动态数据；
- 综合分析：面对复杂问题（如“对比微软Q2财报与当前股价表现”），自动拆解任务、制定执行计划、跨数据源整合信息，生成结构化分析报告。

### 2. 八层技术架构（从底到顶）
| 层级 | 核心组件 | 技术选型 | 核心职责 |
|------|----------|----------|----------|
| 实时数据层 | 实时行情接入 | MCP + Yahoo Finance | 提供9类实时金融工具（股价、财报、新闻等） |
| 数据处理层 | PDF解析+图表转换 | Docling（PDF提取）+ Gemini（图表转文字） | 统一文本/表格/图表为Markdown格式 |
| 存储层 | 向量数据库 | Qdrant（Docker本地部署） | 支持混合搜索、Metadata过滤、数据持久化 |
| 检索层 | 高精度检索 | Dense（Gemini Embedding）+ Sparse（BM25）+ Cross Encoder重排序 | 提升金融场景检索精度（关键词+语义双匹配） |
| Agent层 | 核心推理 | LangChain + LangGraph | 双数据源调度（RAG+MCP）、多模态数据处理 |
| 规划层 | 任务管理 | TodoListMiddleware + SummarizationMiddleware | 复杂任务规划、上下文窗口控制 |

### 3. 关键技术栈
| 类别 | 技术/工具 | 核心作用 |
|------|-----------|----------|
| 大模型 | Gemini 2.5 Flash | 多模态理解（图表转文字）、Agent推理、结构化输出 |
| 文档处理 | Docling（IBM开源） | 高精度提取PDF文本/表格/图片，支持Markdown导出 |
| 向量数据库 | Qdrant | 原生支持混合搜索、Metadata过滤、可视化Dashboard |
| 检索优化 | Gemini Embedding + BM25 + Cross Encoder | 语义匹配+关键词精确匹配+重排序，提升检索准确率 |
| 实时数据 | MCP协议 + Yahoo Finance | 标准化调用实时金融工具，支持跨应用复用 |
| Agent框架 | LangChain + LangGraph | 工具编排、状态管理、多中间件集成 |
| 存储持久化 | SQLite + Qdrant | 对话记忆持久化、向量数据本地存储 |

## 二、核心模块实现（进阶重点）
### 1. 年报PDF多模态处理（金融场景核心难点）
金融PDF含文本、表格、图表，需统一格式后才能检索，最终采用“图片转文字”方案（兼顾精度与易用性）：
#### （1）PDF提取：Docling替代传统工具
- 优势：表格结构完整保留（导出Markdown表格）、图片提取支持、复杂排版适配性强；
- 关键配置：过滤500x500以下小图（剔除Logo/装饰图标），保留有价值图表；
- 核心处理：
    - 文本→Markdown（保留标题层级、格式）；
    - 表格→Markdown表格（附带上方标题上下文，提升检索相关性）；
    - 图片→Gemini生成文字描述（含图表类型、精确数据、趋势分析）。

#### （2）图表转文字：Prompt工程关键
```python
image_description_prompt = """
作为金融分析师，详细描述图表，必须包含：
1. 图表类型（柱状图/折线图等）；
2. 标题、所有精确数据点；
3. 趋势/对比关系；
4. 标签、时间范围、计量单位；
数字需精确，禁止使用“约”“大概”，格式用Markdown。
"""
```
- 效果：苹果营收柱状图→生成“iPhone 2022-2024年收入分别为2052亿/2006亿/2012亿美元”等精确描述。

### 2. 高精度检索系统（金融场景刚需）
金融数据需精准匹配（如“revenue growth”不能模糊匹配），采用“三步检索法”：
#### （1）混合搜索（Dense + Sparse）
- Dense Search：Gemini Embedding（3072维）做语义匹配（如“营收增长”=“revenue growth”）；
- Sparse Search：BM25算法做关键词精确匹配（如“Apple”“10-K”）；
- 融合方式：Qdrant原生支持RRF（Reciprocal Rank Fusion）算法，自动合并两类结果。

#### （2）动态Metadata过滤
从用户问题中提取过滤条件（公司名、年份、文档类型），缩小检索范围：
```python
# 定义结构化Schema
class ChunkMetadata(BaseModel):
    company_name: Optional[str]  # 公司名（小写）
    fiscal_year: Optional[int]   # 财年
    doc_type: Optional[str]      # 文档类型（10-k/10-q）

# 提取过滤条件
def extract_filters(query):
    structured_llm = llm.with_structured_output(ChunkMetadata)
    prompt = f"从查询中提取过滤条件，未提及字段返回None：{query}"
    return structured_llm.invoke(prompt).model_dump(exclude_none=True)
```

#### （3）Cross Encoder重排序
混合搜索获取Top-20结果后，用`BAAI/bge-reranker-base`模型重排，提升Top-1准确率：
```python
reranker = HuggingFaceCrossEncoder(model_name="BAAI/bge-reranker-base")
def rerank_results(query, documents):
    pairs = [(query, doc.page_content) for doc in documents]
    scores = reranker.score(pairs)
    return [doc for _, doc in sorted(zip(scores, documents), reverse=True)[:5]]
```

### 3. 双数据源Agent（RAG+MCP协同）
Agent同时具备“历史数据检索”和“实时数据查询”能力，通过System Prompt明确分工：
#### （1）工具封装
- `deep_rag_search`：包装检索系统，查询年报历史数据；
- `live_finance_researcher`：包装MCP工具，查询Yahoo Finance实时数据。

#### （2）System Prompt核心逻辑
```python
system_prompt = """
你是资深金融分析师，拥有两个数据源：
1. Deep RAG Search：查询公司年报（10-K）历史财务数据、表格、图表；
2. Live Finance Researcher：查询实时股价、市值、新闻、分析师评级。
规则：
- 历史数据/年报内容→用deep_rag_search；
- 实时数据/市场动态→用live_finance_researcher；
- 综合分析→双工具联用，数据需注明来源，金融数据用表格展示。
"""
```

#### （3）对话记忆持久化
用`SqliteSaver`将对话历史存入SQLite，重启程序不丢失上下文：
```python
conn = sqlite3.connect('financial_agent.db', check_same_thread=False)
checkpointer = SqliteSaver(conn=conn)
agent = create_agent(model, tools, system_prompt, checkpointer=checkpointer)
```

### 4. 任务规划与上下文控制（中间件增强）
#### （1）TodoListMiddleware：复杂任务规划
- 核心作用：自动生成执行计划，按步骤执行（简单问题不触发）；
- 示例：用户问“对比微软Q2财报与当前股价”，Agent生成3步计划：
    1. 检索微软Q2收入数据；
    2. 获取实时股价表现；
    3. 综合分析并输出报告；
- 自适应调整：若本地无微软年报，自动标记“步骤1失败”，切换至MCP工具查询。

#### （2）SummarizationMiddleware：上下文窗口控制
- 触发条件：消息数超过25条；
- 处理逻辑：压缩早期16条消息为1条摘要，保留最近9条+摘要（共10条）；
- 价值：避免长对话超出LLM上下文窗口，保证Agent持续运行。

### 5. MCP实时数据接入（标准化工具复用）
通过MCP协议对接Yahoo Finance MCP Server，实现“一次开发、多端复用”：
```python
from langchain_mcp_adapters import MultiServerMCPClient
async def get_mcp_tools():
    client = MultiServerMCPClient({
        "yahoo_finance": {
            "command": "uvx",
            "args": ["yahoo-finance-mcp-server"],
            "transport": "stdio"
        }
    })
    return await client.get_tools()  # 加载9类金融工具
```

## 三、核心流程与效果
### 1. 完整工作流程（以“对比微软Q2财报与当前股价”为例）
1. 用户提问→Agent调用`write_todos`生成3步计划；
2. 执行步骤1：调用`deep_rag_search`检索本地年报→未找到微软数据；
3. 自适应调整：标记步骤1失败，调用`live_finance_researcher`从Yahoo Finance获取Q2财务数据；
4. 执行步骤2：调用`live_finance_researcher`获取实时股价、市值、分析师评级；
5. 执行步骤3：综合数据，生成结构化报告（含业务线收入、估值指标、对比分析、投资建议）；
6. 中间件作用：过程中触发2次摘要，控制上下文窗口。

### 2. 关键效果对比
| 检索方案 | Top-1准确率（示例问题：苹果2024年Services收入） |
|----------|------------------------------------------------|
| 纯语义搜索（Dense） | 命中iPhone收入chunk，未精准匹配Services |
| 混合搜索（Dense+Sparse） | 命中Services相关chunk |
| 混合搜索+重排序 | 精准命中Services收入表格，准确率100% |

## 四、部署与运行
### 1. 环境搭建
```bash
# 1. 安装依赖
pip install langchain langgraph qdrant-client docling langchain-google-genai

# 2. 部署Qdrant（Docker）
docker-compose up -d  # 配置文件见文档

# 3. 配置API密钥（Gemini）
export GOOGLE_API_KEY="你的Gemini密钥"
```

### 2. 核心运行命令
```python
# 1. 数据入库（PDF→Markdown→向量库）
python scripts/ingest_data.py  # 批量处理年报PDF

# 2. 启动Agent
python scripts/financial_agent.py

# 3. 测试复杂问题
agent.invoke({
    "messages": [{"role": "user", "content": "对比微软Q2 2024财报与当前股价表现"}]
}, config={"configurable": {"thread_id": "research-1"}})
```

## 五、核心亮点与扩展方向
### 1. 核心亮点
- 多模态兼容：统一处理PDF文本/表格/图表，覆盖金融文档全类型数据；
- 检索精度高：混合搜索+重排序+Metadata过滤，适配金融场景精准查询需求；
- 双数据源协同：历史年报（RAG）+实时行情（MCP），支持综合分析；
- 自主规划能力：复杂问题自动拆步骤，异常情况自适应调整；
- 本地可部署：核心组件（Qdrant、Docling）支持本地运行，数据安全可控。

### 2. 扩展方向
- 数据源扩展：接入Wind、Tushare等专业金融数据，提升数据广度；
- 功能增强：添加量化分析工具（如估值模型、风险指标计算）；
- 安全加固：加入Human-in-the-Loop审批（高风险投资建议需人工确认）；
- 多语言支持：适配中英文年报，支持跨境金融分析；
- 前端优化：开发专业金融可视化界面（K线图、财务指标趋势图）。

## 总结
本项目通过“多模态处理→高精度检索→双数据源Agent→智能规划”四层设计，解决了金融研究中“年报解析难、实时数据获取难、复杂问题分析难”三大痛点，最终实现一个自动化、高精度、可落地的AI金融研究员。核心价值在于将高阶技术（多模态RAG、混合搜索、MCP协议）与金融场景深度结合，所有组件均采用开源/免费方案，可快速复用于其他垂直领域（如法律文档分析、医疗报告解读）。当前文件内容过长，豆包只阅读了前 51%。
