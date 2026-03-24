# LangSmith实现Agent可观测性课程总结
本课程讲解了LangSmith（LangChain团队出品的可观测性平台）的核心使用方法，通过搭建**维基百科研究助手Agent**，演示如何为Agent接入LangSmith实现全流程追踪，解决Agent运行“黑盒”问题，让其决策、执行、消耗等过程完全透明，同时提供了完整的实操代码和使用技巧。

## 一、LangSmith核心介绍
1. **定位**：LangChain官方可观测性平台，为LangGraph/LangChain搭建的Agent提供**全流程追踪能力**，类比为Agent的“行车记录仪”。
2. **核心价值**：打破Agent黑盒，无需修改业务代码，即可自动采集运行数据，解决**调试难、性能瓶颈定位难、成本核算难**等问题。
3. **核心追踪能力**：
    - 记录每次LLM调用的完整Prompt和返回结果
    - 采集每个工具的输入、输出及执行耗时
    - 追踪图节点的执行顺序、数据流和状态变化
    - 统计token用量并估算调用费用
4. **使用优势**：仅需配置**两行环境变量**即可开启，无需改动Agent业务代码，接入成本极低。

## 二、准备工作
1. **环境要求**：Python 3.9+
2. **必备密钥**：智谱/OpenAI API Key、LangSmith API Key（LangSmith账号免费注册：https://smith.langchain.com）
3. **安装依赖**：
```bash
pip install langchain-core langchain-openai langgraph langsmith requests
```

## 三、LangSmith快速开启
通过配置环境变量启动追踪，核心代码如下，**所有Agent运行数据会自动上报至LangSmith平台**：
```python
import os
# 开启LangSmith追踪V2版本
os.environ["LANGCHAIN_TRACING_V2"] = "true"
# 定义项目名，方便在LangSmith面板分类查看
os.environ["LANGCHAIN_PROJECT"] = "langsmith-demo"
# 配置个人LangSmith API Key
os.environ["LANGCHAIN_API_KEY"] = "你的 LangSmith API Key"
```

## 四、实操：搭建可追踪的维基百科研究助手Agent
本次搭建的Agent为**三节点线性流程**，核心功能是判断用户问题是否需要搜索维基百科，再综合信息生成回答，全程可被LangSmith追踪。

### 1. 完整核心代码
```python
"""
LangSmith 追踪教程:给 AI Agent 装个"行车记录仪"
使用智谱 GLM-5 模型
"""
import os
import time
import requests
from typing import TypedDict

from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage
from langchain_core.tools import tool
from langgraph.graph import StateGraph, END

# ========== 1. 配置 ==========
ZHIPU_API_KEY = "你的智谱 API Key"
ZHIPU_BASE_URL = "https://open.bigmodel.cn/api/paas/v4/"

# LangSmith 追踪
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_PROJECT"] = "langsmith-demo"
os.environ["LANGCHAIN_API_KEY"] = "你的 LangSmith API Key"

# 初始化LLM（智谱GLM-5，兼容OpenAI接口）
llm = ChatOpenAI(
    temperature=0,
    model="glm-5",
    openai_api_key=ZHIPU_API_KEY,
    openai_api_base=ZHIPU_BASE_URL,
)

# ========== 2. 定义Agent状态（结构化，含追踪专用字段reasoning） ==========
class AgentState(TypedDict):
    user_question: str    # 用户问题
    needs_search: bool    # 是否需要搜索
    search_result: str    # 搜索结果
    final_answer: str     # 最终回答
    reasoning: str        # 决策理由（专为LangSmith追踪设计，提升可读性）

# ========== 3. 定义工具：维基百科搜索 ==========
@tool
def wikipedia_search(query: str) -> str:
    """搜索维基百科获取信息｡"""
    try:
        search_url = "https://en.wikipedia.org/w/api.php"
        params = {
            "action": "query",
            "list": "search",
            "srsearch": query,
            "format": "json",
            "srlimit": 3,
        }
        resp = requests.get(search_url, params=params, timeout=10)
        if resp.status_code != 200:
            return f"搜索失败,状态码:{resp.status_code}"

        results = resp.json().get("query", {}).get("search", [])
        if not results:
            return f"没找到关于 '{query}' 的内容"

        title = results[0]["title"]
        summary_url = (
            f"https://en.wikipedia.org/api/rest_v1/page/summary/"
            f"{title.replace(' ', '_')}"
        )
        summary_resp = requests.get(summary_url, timeout=10)
        if summary_resp.status_code == 200:
            extract = summary_resp.json().get("extract", "无摘要")
            return f"关于 '{title}':{extract[:500]}"
        return f"找到了 '{title}',但获取摘要失败"
    except Exception as e:
        return f"搜索出错:{e}"

# ========== 4. 定义Agent三大节点 ==========
def decide_search_need(state: AgentState) -> AgentState:
    """判断是否需要搜索，设置决策理由"""
    prompt = f"""分析这个问题,判断是否需要搜索最新信息:
问题:"{state['user_question']}"
判断标准:
- 最近的新闻､实时数据､当前价格 ￫ 需要搜索
- 常识､历史知识､概念解释 ￫ 不需要搜索
只回复 SEARCH 或 DIRECT,然后换行写理由｡"""

    response = llm.invoke([
        SystemMessage(content="你是一个智能助手,帮助判断问题是否需要搜索｡"),
        HumanMessage(content=prompt),
    ])
    text = response.content.strip()
    lines = [l.strip() for l in text.split("\n") if l.strip()]
    decision = lines[0].upper() if lines else "DIRECT"
    reasoning = lines[1] if len(lines) > 1 else "无"

    needs_search = "SEARCH" in decision
    state["needs_search"] = needs_search
    state["reasoning"] = f"{'需要搜索' if needs_search else '直接回答'}｡理由:{reasoning}"
    print(f" 判断:{'需要搜索' if needs_search else '直接回答'} — {reasoning}")
    return state

def execute_search(state: AgentState) -> AgentState:
    """执行搜索，无需则跳过"""
    if not state["needs_search"]:
        print("⏭️ 跳过搜索")
        state["search_result"] = ""
        return state
    print(f"🔍 搜索中:{state['user_question']}")
    result = wikipedia_search.invoke({"query": state["user_question"]})
    state["search_result"] = result
    print(f"📄 搜索完成,返回 {len(result)} 字符")
    return state

def generate_response(state: AgentState) -> AgentState:
    """根据是否有搜索结果，生成最终回答"""
    question = state["user_question"]
    search_result = state.get("search_result", "")

    if state["needs_search"] and search_result and "搜索出错" not in search_result:
        prompt = f"""根据搜索结果回答用户问题,用中文回答｡
问题:{question}
搜索结果:{search_result}"""
        messages = [
            SystemMessage(content="你是一个知识渊博的研究助手｡"),
            HumanMessage(content=prompt),
        ]
    else:
        prompt = f"""用你已有的知识回答这个问题,用中文回答｡
问题:{question}"""
        messages = [
            SystemMessage(content="你是一个知识渊博的助手｡"),
            HumanMessage(content=prompt),
        ]

    response = llm.invoke(messages)
    state["final_answer"] = response.content
    print(f"💬 回答生成完成({len(response.content)} 字符)")
    return state

# ========== 5. 组装LangGraph图结构 ==========
builder = StateGraph(AgentState)
# 添加节点
builder.add_node("decide", decide_search_need)
builder.add_node("search", execute_search)
builder.add_node("respond", generate_response)
# 设置流程：decide → search → respond → 结束
builder.set_entry_point("decide")
builder.add_edge("decide", "search")
builder.add_edge("search", "respond")
builder.add_edge("respond", END)
# 编译Agent
agent = builder.compile()

# ========== 6. 测试函数（带元数据标签，方便LangSmith分类分析） ==========
def ask(question: str, test_type: str = "general"):
    print("=" * 55)
    print(f"❓ {question}")
    print("-" * 55)

    start = time.time()
    initial_state = {
        "user_question": question,
        "needs_search": False,
        "search_result": "",
        "final_answer": "",
        "reasoning": "",
    }
    # 配置元数据和标签，用于LangSmith中版本/类型区分
    config = {
        "metadata": {"test_type": test_type},
        "tags": ["tutorial", test_type],
    }
    result = agent.invoke(initial_state, config=config)
    elapsed = time.time() - start

    print(f"\n💡 回答:\n{result['final_answer'][:300]}")
    print(f"\n⏱️ 耗时:{elapsed:.1f}s | 搜索:{'是' if result['needs_search'] else '否'}")
    print()

# 执行测试
ask("法国的首都是哪里?", "direct_answer")
ask("2024年美国总统大选结果是什么?", "current_info")
ask("简单介绍一下人工智能", "factual_lookup")
```

### 2. Agent核心设计要点
1. **结构化状态**：新增`reasoning`字段，专门用于记录Agent决策理由，**提升LangSmith追踪记录的可读性**，无需额外打印调试。
2. **三节点线性流程**：`decide`（判断是否搜索）→`search`（执行搜索/跳过）→`respond`（生成回答），流程简单易追踪。
3. **工具实现**：基于维基百科API实现两步搜索（模糊搜索→精准摘要），包含异常处理，保证工具稳定性。
4. **测试配置**：为每次调用添加`metadata`（元数据）和`tags`（标签），方便在LangSmith中按**问题类型/版本**分类分析。

### 3. 测试结果
Agent可精准判断问题类型，常识题直接回答，时效性题执行搜索，测试结果示例：
- 法国的首都是哪里？→ 直接回答，耗时21.9s
- 2024年美国总统大选结果是什么？→ 需要搜索，耗时78.2s
- 简单介绍一下人工智能 → 直接回答，耗时51.3s

## 五、LangSmith平台核心追踪功能
打开LangSmith官网（https://smith.langchain.com），找到对应项目，可查看三大核心追踪视图，所有数据**自动采集、实时展示**：

### 1. 追踪列表
展示Agent每次完整运行的**概览数据**，包括：用户输入、总耗时、运行是否成功、token消耗，可快速定位**慢查询/失败运行**。

### 2. 单次追踪详情
点进单条记录，可查看**全流程细节**：
- 每个节点的执行耗时
- 节点间的状态（State）传递数据
- LLM调用的完整Prompt和返回内容
- 工具调用的输入、输出结果
  → 可直接看到“Agent为什么决定搜索/不搜索”，无需额外打印调试。

### 3. 图可视化&性能指标
- 左侧：直观展示Agent的图结构和实际执行路径
- 右侧：统计每个节点的**平均耗时、调用次数**，**性能瓶颈一目了然**（如LLM调用慢/工具调用慢）。

## 六、LangSmith实际应用价值
1. **调试决策逻辑**：Agent执行路径异常时，直接查看追踪记录的State变化和决策理由，无需加大量print语句。
2. **定位性能瓶颈**：自动统计各节点/工具/LLM的耗时，快速找到Agent运行的“慢节点”。
3. **核算调用成本**：自动采集每次运行的token用量，精准估算不同类型问题的调用成本。
4. **对比版本效果**：修改Prompt/模型/节点逻辑后，通过`metadata/tags`标记版本，在LangSmith中直接对比**耗时、成功率、token消耗**。
5. **定位系统性问题**：可过滤出所有**失败/报错**的运行记录，快速找到Agent的共性问题（如工具调用异常、LLM生成结果格式错误）。

## 七、LangSmith使用核心建议
1. **从开发初期开启**：仅需配置环境变量，无接入成本，避免出bug后无数据可查。
2. **善用metadata/tags**：为每次运行打标签（如Prompt版本、问题类型、用户类型），方便后续分析和版本对比。
3. **结构化Agent State**：新增类似`reasoning`的专用字段，提升追踪记录的可读性，降低调试成本。
4. **重点分析失败记录**：失败的追踪数据比成功记录更有价值，是定位Agent系统性问题的关键。
5. **无需修改业务代码**：所有追踪逻辑由LangSmith自动实现，无需在Agent代码中添加额外追踪逻辑，保证代码简洁。