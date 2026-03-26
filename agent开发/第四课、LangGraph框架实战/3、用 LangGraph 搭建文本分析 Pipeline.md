# LangGraph 文本分析 Pipeline 课程总结（实战进阶）

> 对应原文 PDF：`D:\notebook\agent开发\langGraph框架实战\第三课：手把手教你用 LangGraph 搭建文本分析 Pipeline.pdf`
> 说明：在不改变原文含义的前提下做精简总结；代码示例保持完整，不删减。

本课程以**多步骤文本分析 Pipeline**为案例，聚焦 LangGraph 在“结构化流程编排”与“动态路由”的核心应用，实现文本分类、实体提取、摘要生成、情感分析的自动化流转，支持条件分支优化流程效率，以下为精简总结。

## 一、核心目标
通过 LangGraph 搭建**可扩展、可分支的文本分析流水线**，解决传统线性流程“一刀切”的问题，实现：
1. 多维度文本分析（分类、实体、摘要、情感）；
2. 基于文本类型的动态路由（新闻/论文需提取实体，博客直接跳过）；
3. 节点解耦，支持灵活新增/删除分析步骤。

## 二、环境准备
### 1. 安装依赖
```bash
pip install langgraph langchain langchain-openai python-dotenv
```

### 2. 配置模型（OpenAI 兼容接口）
```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI

# 加载环境变量（.env 文件中配置 OPENAI_API_KEY）
load_dotenv()
os.environ["OPENAI_API_KEY"] = os.getenv('OPENAI_API_KEY')

# 初始化模型（temperature=0 保证分析结果稳定）
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
```

## 三、核心组件实现
### 1. 定义状态（结构化数据容器）
使用 `TypedDict` 明确定义流水线中流转的数据结构，包含输入文本和所有中间/最终结果字段，支持按需扩展：
```python
from typing import TypedDict, List

# 增强版状态（含情感分析）
class EnhancedState(TypedDict):
    text: str               # 输入原始文本
    classification: str      # 文本分类（News/Blog/Research/Other）
    entities: List[str]      # 提取的实体（人名/组织/地点）
    summary: str             # 文本摘要
    sentiment: str           # 情感分析（Positive/Negative/Neutral）
```

### 2. 定义分析节点（流水线工位）
每个节点为独立函数，接收 `State` 并返回更新后的字段，职责单一、解耦性强，核心逻辑为“Prompt 拼接→LLM 调用→结果返回”：

#### （1）文本分类节点
```python
from langchain_core.prompts import PromptTemplate
from langchain_core.messages import HumanMessage

def classification_node(state: EnhancedState):
    """将文本分类为 News/Blog/Research/Other"""
    prompt = PromptTemplate(
        input_variables=["text"],
        template="Classify the following text into one of the categories: News, Blog, Research, or Other.\n\nText:{text}\n\nCategory:"
    )
    message = HumanMessage(content=prompt.format(text=state["text"]))
    classification = llm.invoke([message]).content.strip()
    return {"classification": classification}
```

#### （2）实体提取节点
```python
def entity_extraction_node(state: EnhancedState):
    """提取文本中的人名、组织、地点，返回逗号分隔列表"""
    prompt = PromptTemplate(
        input_variables=["text"],
        template="Extract all the entities (Person, Organization, Location) from the following text. Provide the result as a comma-separated list.\n\nText:{text}\n\nEntities:"
    )
    message = HumanMessage(content=prompt.format(text=state["text"]))
    entities = llm.invoke([message]).content.strip().split(", ")
    return {"entities": entities}
```

#### （3）摘要生成节点
```python
def summarization_node(state: EnhancedState):
    """生成一句话摘要"""
    prompt = PromptTemplate(
        input_variables=["text"],
        template="Summarize the following text in one short sentence.\n\nText:{text}\n\nSummary:"
    )
    message = HumanMessage(content=prompt.format(text=state["text"]))
    summary = llm.invoke([message]).content.strip()
    return {"summary": summary}
```

#### （4）情感分析节点
```python
def sentiment_node(state: EnhancedState):
    """分析文本情感为 Positive/Negative/Neutral"""
    prompt = PromptTemplate(
        input_variables=["text"],
        template="Analyze the sentiment of the following text. Is it Positive, Negative, or Neutral?\n\nText:{text}\n\nSentiment:"
    )
    message = HumanMessage(content=prompt.format(text=state["text"]))
    sentiment = llm.invoke([message]).content.strip()
    return {"sentiment": sentiment}
```

### 3. 定义路由函数（条件分支逻辑）
根据文本分类结果动态决定流程走向，优化资源消耗（非新闻/论文跳过实体提取）：
```python
def route_after_classification(state: EnhancedState) -> bool:
    """路由规则：新闻/论文返回 True（走实体提取），其他返回 False（直接摘要）"""
    return state["classification"].lower() in ["news", "research"]
```

### 4. 组装 Pipeline（串联节点与路由）
通过 `StateGraph` 定义节点流转规则，支持普通边（无条件跳转）和条件边（动态分支）：
```python
from langgraph.graph import StateGraph, END

# 1. 创建状态图实例
workflow = StateGraph(EnhancedState)

# 2. 注册所有节点
workflow.add_node("classification_node", classification_node)       # 分类节点
workflow.add_node("entity_extraction", entity_extraction_node)     # 实体提取节点
workflow.add_node("summarization", summarization_node)             # 摘要节点
workflow.add_node("sentiment_analysis", sentiment_node)           # 情感分析节点

# 3. 定义流程入口
workflow.set_entry_point("classification_node")

# 4. 核心：条件边（分类后动态路由）
workflow.add_conditional_edges(
    start_node="classification_node",          # 起始节点
    condition=route_after_classification,      # 路由判断函数
    path_map={
        True: "entity_extraction",             # 新闻/论文 → 实体提取
        False: "summarization"                 # 其他类型 → 直接摘要
    }
)

# 5. 普通边（无条件流转）
workflow.add_edge("entity_extraction", "summarization")    # 实体提取 → 摘要
workflow.add_edge("summarization", "sentiment_analysis")   # 摘要 → 情感分析
workflow.add_edge("sentiment_analysis", END)               # 情感分析 → 流程结束

# 6. 编译生成可运行应用
app = workflow.compile()
```

## 四、运行与测试
### 1. 测试新闻类文本（走实体提取流程）
```python
news_text = """
OpenAI released the GPT-4 model with enhanced performance on academic
and professional tasks. It's seen as a major breakthrough in alignment
and reasoning capabilities.
"""
result = app.invoke({"text": news_text})

# 输出结果
print("Classification:", result["classification"])  # 输出：News
print("Entities:", result["entities"])              # 输出：['OpenAI', 'GPT-4']
print("Summary:", result["summary"])                # 输出：OpenAI released the GPT-4 model...
print("Sentiment:", result["sentiment"])            # 输出：Positive
```

### 2. 测试博客类文本（跳过实体提取流程）
```python
blog_text = """
Here's what I learned from a week of meditating in silence.
No phones, no talking—just me, my breath, and some deep realizations.
"""
result = app.invoke({"text": blog_text})

# 输出结果
print("Classification:", result["classification"])  # 输出：Blog
print("Entities:", result.get("entities", "Skipped"))# 输出：Skipped（未执行实体提取）
print("Summary:", result["summary"])                # 输出：The author shares insights from a week of silent meditation...
print("Sentiment:", result["sentiment"])            # 输出：Neutral
```

## 五、完整代码
```python
import os
from typing import TypedDict, List
from dotenv import load_dotenv
from langgraph.graph import StateGraph, END
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage

# ========== 1. 环境配置 ==========
load_dotenv()
os.environ["OPENAI_API_KEY"] = os.getenv('OPENAI_API_KEY')

# ========== 2. 定义状态 ==========
class EnhancedState(TypedDict):
    text: str
    classification: str
    entities: List[str]
    summary: str
    sentiment: str

# ========== 3. 初始化模型 ==========
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# ========== 4. 定义节点函数 ==========
def classification_node(state: EnhancedState):
    prompt = PromptTemplate(
        input_variables=["text"],
        template="Classify the following text into one of the categories: News, Blog, Research, or Other.\n\nText:{text}\n\nCategory:"
    )
    message = HumanMessage(content=prompt.format(text=state["text"]))
    return {"classification": llm.invoke([message]).content.strip()}

def entity_extraction_node(state: EnhancedState):
    prompt = PromptTemplate(
        input_variables=["text"],
        template="Extract all the entities (Person, Organization, Location) from the following text. Provide the result as a comma-separated list.\n\nText:{text}\n\nEntities:"
    )
    message = HumanMessage(content=prompt.format(text=state["text"]))
    return {"entities": llm.invoke([message]).content.strip().split(", ")}

def summarization_node(state: EnhancedState):
    prompt = PromptTemplate(
        input_variables=["text"],
        template="Summarize the following text in one short sentence.\n\nText:{text}\n\nSummary:"
    )
    message = HumanMessage(content=prompt.format(text=state["text"]))
    return {"summary": llm.invoke([message]).content.strip()}

def sentiment_node(state: EnhancedState):
    prompt = PromptTemplate(
        input_variables=["text"],
        template="Analyze the sentiment of the following text. Is it Positive, Negative, or Neutral?\n\nText:{text}\n\nSentiment:"
    )
    message = HumanMessage(content=prompt.format(text=state["text"]))
    return {"sentiment": llm.invoke([message]).content.strip()}

# ========== 5. 定义路由函数 ==========
def route_after_classification(state: EnhancedState) -> bool:
    return state["classification"].lower() in ["news", "research"]

# ========== 6. 组装 Pipeline ==========
workflow = StateGraph(EnhancedState)
workflow.add_node("classification_node", classification_node)
workflow.add_node("entity_extraction", entity_extraction_node)
workflow.add_node("summarization", summarization_node)
workflow.add_node("sentiment_analysis", sentiment_node)

workflow.set_entry_point("classification_node")
workflow.add_conditional_edges(
    "classification_node",
    route_after_classification,
    {True: "entity_extraction", False: "summarization"}
)
workflow.add_edge("entity_extraction", "summarization")
workflow.add_edge("summarization", "sentiment_analysis")
workflow.add_edge("sentiment_analysis", END)

app = workflow.compile()

# ========== 7. 测试 ==========
if __name__ == "__main__":
    test_text = "OpenAI has announced the GPT-4 model, which exhibits human-level performance on professional benchmarks."
    result = app.invoke({"text": test_text})
    print("=== 文本分析结果 ===")
    for key, value in result.items():
        print(f"{key}: {value}")
```

## 六、核心亮点与扩展方向
### 1. 核心亮点
| 特性         | 价值                                  |
|--------------|---------------------------------------|
| 节点解耦     | 新增/修改分析步骤仅需调整单个函数，无需改动整体流程 |
| 条件路由     | 按文本类型动态跳过非必要节点，节省 Token 和时间 |
| 结构化状态   | 数据流转清晰，便于调试和后续扩展字段        |
| 可视化支持   | 可通过 `app.get_graph().draw_mermaid_png()` 查看流程拓扑 |
