# LangGraph 进阶课程总结（客服邮件 Agent 实战）

> 对应原文 PDF：`D:\notebook\agent开发\langGraph框架实战\第五课：LangGraph进阶-搭一个客服邮件Agent.pdf`
> 说明：在不改变原文含义的前提下做精简总结；代码示例保持完整，不删减。

本课程以**客服邮件自动处理 Agent**为核心案例，融合 LangGraph 高级特性（`Command` 路由、`interrupt` 人机协作、多场景分支），实现邮件分类、智能路由、文档检索、回复生成、人工审核的全流程自动化，重点解决“业务规则落地”与“人机协同”问题，以下为精简总结。

## 一、核心需求
客服邮件 Agent 需覆盖多场景自动化处理，同时满足业务风控要求：
1. 功能覆盖：读取邮件→分类意图/紧急度→搜索知识库→生成回复→人工审核（涉钱/紧急邮件）→发送回复；
2. 场景适配：支持简单咨询、Bug 报告、账单问题、功能建议等 4 类核心场景；
3. 风控规则：账单相关或紧急程度为 `critical` 的邮件必须人工审核，避免资金风险。

## 二、环境准备
### 1. 安装依赖
```bash
pip install langgraph langchain langchain-openai python-dotenv
```

### 2. 初始化模型（智谱 glm-5）
```python
from langchain_openai import ChatOpenAI

ZHIPU_API_KEY = "你的智谱API Key"
ZHIPU_BASE_URL = "https://open.bigmodel.cn/api/paas/v4/"

llm = ChatOpenAI(
    temperature=0,  # 业务场景需稳定输出，避免随机性
    model="glm-5",
    openai_api_key=ZHIPU_API_KEY,
    openai_api_base=ZHIPU_BASE_URL,
)
```

## 三、核心设计与实现
### 1. 状态设计（存储原始数据，避免格式化耦合）
状态仅存储原始输入和中间结果，不存储格式化文本，确保各节点可灵活处理数据：
```python
from typing import TypedDict

# 邮件分类结果子结构
class EmailClassification(TypedDict):
    intent: str       # 意图：question/bug/billing/feature/complex
    urgency: str      # 紧急度：low/medium/high/critical
    topic: str        # 具体话题（如“密码重置”“重复扣款”）
    summary: str      # 邮件一句话总结

# 全局状态
class EmailAgentState(TypedDict):
    # 原始邮件数据
    email_content: str  # 邮件正文
    sender_email: str   # 发件人邮箱
    email_id: str       # 邮件唯一ID
    # 中间结果
    classification: EmailClassification | None  # 分类结果
    search_results: list[str] | None           # 知识库搜索结果（原始文档列表）
    draft_response: str | None                 # 回复草稿
```

### 2. 节点实现（核心：`Command` 路由 + 业务逻辑封装）
每个节点通过 `LangGraph Command` 实现“状态更新 + 下一步路由”，无需额外条件边函数，代码更集中。

#### （1）基础节点
- **读邮件节点**：解析邮件原始数据（实际项目对接邮件服务 API）
  ```python
  def read_email(state: EmailAgentState):
      print(f"📧 收到邮件:{state['email_content'][:50]}...")
      return {}  # 无需更新状态，直接流转
  ```

- **发送回复节点**：对接邮件发送服务（实际项目调用 SMTP/API）
  ```python
  def send_reply(state: EmailAgentState):
      print(f"📤 回复已发送给 {state['sender_email']}")
      return {}
  ```

#### （2）核心业务节点
##### ① 分类意图节点（路由核心）
分析邮件意图和紧急度，根据意图动态指定下一步流程：
```python
from langgraph.types import Command
import json

def classify_intent(state: EmailAgentState) -> Command:
    """分类邮件并路由：question/feature/billing→搜索文档，bug→创建工单，其他→直接生成回复"""
    prompt = (
        "分析下面这封客户邮件，返回 JSON 格式的分类结果。\n"
        "字段：intent(question/bug/billing/feature/complex)、urgency(low/medium/high/critical)、topic(话题)、summary(总结)\n"
        "只返回 JSON，不要其他内容。\n\n"
        f"邮件：{state['email_content']}\n发件人：{state['sender_email']}"
    )
    response = llm.invoke([("human", prompt)])

    # 解析 JSON（处理智谱模型可能的 markdown 代码块包裹）
    try:
        text = response.content.strip()
        if text.startswith("```"):
            text = text.split("```")[1].lstrip("json")  # 去除 ```json 标记
        classification = json.loads(text.strip())
    except (json.JSONDecodeError, IndexError):
        # 解析失败降级处理
        classification = {
            "intent": "complex", "urgency": "medium",
            "topic": "unknown", "summary": state["email_content"][:100]
        }

    # 路由逻辑：根据意图指定下一步
    intent = classification.get("intent", "complex")
    goto = {
        "question": "search_documentation",
        "feature": "search_documentation",
        "billing": "search_documentation",
        "bug": "bug_tracking"
    }.get(intent, "draft_response")

    print(f"📋 分类结果：intent={intent}, urgency={classification.get('urgency')}")
    return Command(update={"classification": classification}, goto=goto)
```

##### ② 搜索文档节点（知识库对接）
根据邮件话题搜索相关文档（实际项目对接向量数据库/知识库）：
```python
def search_documentation(state: EmailAgentState) -> Command:
    """模拟知识库搜索，返回相关文档"""
    classification = state.get("classification", {})
    topic = classification.get("topic", "").lower()
    email_content = state.get("email_content", "").lower()

    # 模拟知识库（实际替换为向量数据库检索）
    mock_docs = {
        "password": ["重置密码：设置 → 安全 → 修改密码", "密码要求：至少12位，包含大小写和数字"],
        "billing": ["退款流程：提交工单后3-5个工作日处理", "重复扣款：系统自动检测并退还多余费用"],
        "dark": ["深色模式已在开发计划中", "预计下个版本发布"],
    }

    # 匹配相关文档
    search_results = []
    for key, docs in mock_docs.items():
        if key in topic or key in email_content:
            search_results.extend(docs)
    search_results = list(dict.fromkeys(search_results))  # 去重
    search_results = search_results if search_results else ["暂未找到相关文档"]

    print(f"🔍 搜索到 {len(search_results)} 条相关文档")
    return Command(update={"search_results": search_results}, goto="draft_response")
```

##### ③ Bug 追踪节点（工单系统对接）
Bug 类邮件自动创建工单（实际项目对接 Jira/Linear 等工具）：
```python
def bug_tracking(state: EmailAgentState) -> Command:
    """创建 Bug 工单，返回工单信息"""
    email_id = state.get("email_id", "000")
    ticket_id = f"BUG-{email_id[-3:]}"  # 从邮件ID生成工单号
    print(f"🐛 创建了 Bug 工单：{ticket_id}")
    return Command(
        update={"search_results": [f"已创建 Bug 工单 {ticket_id}，技术团队会尽快处理"]},
        goto="draft_response"
    )
```

##### ④ 生成回复节点（风控规则落地）
生成专业回复，并根据“涉钱/紧急度”判断是否需要人工审核：
```python
def draft_response(state: EmailAgentState) -> Command:
    """生成回复草稿，涉钱/紧急邮件触发人工审核"""
    classification = state.get("classification", {})
    search_results = state.get("search_results", [])

    # 格式化知识库内容（节点内动态格式化，不存储到状态）
    context = "参考资料：\n" + "\n".join([f"- {doc}" for doc in search_results]) if search_results else ""

    # 生成回复的 Prompt
    prompt = (
        f"为以下客户邮件写回复（专业友好，中文直接输出）：\n"
        f"邮件内容：{state['email_content']}\n"
        f"邮件类型：{classification.get('intent', '未知')}\n"
        f"紧急程度：{classification.get('urgency', '中')}\n"
        f"{context}"
    )
    draft = llm.invoke([("human", prompt)]).content

    # 风控规则：涉钱（billing）或极紧急（critical）需人工审核
    needs_review = (
        classification.get("urgency") == "critical"
        or classification.get("intent") == "billing"
    )
    goto = "human_review" if needs_review else "send_reply"
    print(f"✍️ 回复草稿已生成，{'需要人工审核' if needs_review else '直接发送'}")

    return Command(update={"draft_response": draft}, goto=goto)
```

##### ⑤ 人工审核节点（人机协作核心）
通过 `interrupt()` 暂停流程，等待人工审批后恢复，需配合 `checkpointer` 保存状态：
```python
from langgraph.types import interrupt

def human_review(state: EmailAgentState) -> Command:
    """暂停流程，等待人工审核回复草稿"""
    classification = state.get("classification", {})
    # 暂停流程，向人工展示关键信息
    human_decision = interrupt({
        "email_id": state.get("email_id"),
        "original_email": state.get("email_content"),
        "draft_response": state.get("draft_response"),
        "urgency": classification.get("urgency"),
        "action": "请审核回复：approved=True 表示通过，可传入 edited_response 修改草稿"
    })

    # 人工审批后继续流程
    if human_decision.get("approved"):
        # 优先使用人工修改后的回复，无修改则用AI草稿
        edited_draft = human_decision.get("edited_response", state.get("draft_response", ""))
        print("✅ 人工审核通过")
        return Command(update={"draft_response": edited_draft}, goto="send_reply")
    else:
        print("❌ 人工拒绝，由人工自行处理")
        return Command(update={}, goto="__end__")  # 终止流程
```

### 3. 组装图（`Command` 路由简化边配置）
因节点内部通过 `Command` 指定路由，仅需配置固定边，图结构简洁：
```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

# 1. 创建状态图
graph_builder = StateGraph(EmailAgentState)

# 2. 添加所有节点
graph_builder.add_node("read_email", read_email)
graph_builder.add_node("classify_intent", classify_intent)
graph_builder.add_node("search_documentation", search_documentation)
graph_builder.add_node("bug_tracking", bug_tracking)
graph_builder.add_node("draft_response", draft_response)
graph_builder.add_node("human_review", human_review)
graph_builder.add_node("send_reply", send_reply)

# 3. 配置固定边（路由逻辑在节点内通过 Command 实现）
graph_builder.add_edge(START, "read_email")          # 入口→读邮件
graph_builder.add_edge("read_email", "classify_intent")  # 读邮件→分类意图
graph_builder.add_edge("send_reply", END)            # 发送回复→流程结束

# 4. 配置 checkpointer（支持 interrupt() 状态保存）
memory = MemorySaver()
app = graph_builder.compile(checkpointer=memory)
```

## 四、测试场景与效果
### 1. 场景 1：简单咨询（全自动流程）
```python
# 测试“密码重置”咨询
result = app.invoke(
    {
        "email_content": "你好，请问怎么重置密码？谢谢",
        "sender_email": "zhangsan@example.com",
        "email_id": "email_001"
    },
    config={"configurable": {"thread_id": "test_001"}}
)
```
**流程**：读邮件→分类为 `question`→搜索密码相关文档→生成回复→直接发送  
**输出**：自动返回密码重置步骤，无需人工干预。

### 2. 场景 2：账单问题（人工审核流程）
```python
# 测试“重复扣款”投诉（涉钱需审核）
result = app.invoke(
    {
        "email_content": "我被扣了两次订阅费！这很紧急，请立刻处理！",
        "sender_email": "lisi@example.com",
        "email_id": "email_002"
    },
    config={"configurable": {"thread_id": "test_002"}}
)
# 流程暂停，人工审核后恢复
final_result = app.invoke(
    Command(resume={
        "approved": True,
        "edited_response": "非常抱歉给你造成不便！重复扣款将在3个工作日内退还，如有问题请随时联系我们。"
    }),
    config={"configurable": {"thread_id": "test_002"}}  # 需与之前的 thread_id 一致
)
```
**流程**：读邮件→分类为 `billing`→搜索退款文档→生成草稿→触发人工审核→审核通过→发送回复  
**关键**：`thread_id` 一致才能恢复暂停的流程。

### 3. 场景 3：Bug 报告（自动创建工单）
```python
# 测试“PDF导出崩溃”Bug报告
result = app.invoke(
    {
        "email_content": "导出PDF的时候页面直接崩溃了，每次都这样",
        "sender_email": "wangwu@example.com",
        "email_id": "email_003"
    },
    config={"configurable": {"thread_id": "test_003"}}
)
```
**流程**：读邮件→分类为 `bug`→创建工单（BUG-003）→生成回复→直接发送  
**输出**：回复包含工单号，用户可追踪进度。

## 五、完整代码
```python
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict
import json

# ========== 1. 模型初始化 ==========
ZHIPU_API_KEY = "你的智谱API Key"
ZHIPU_BASE_URL = "https://open.bigmodel.cn/api/paas/v4/"
llm = ChatOpenAI(
    temperature=0,
    model="glm-5",
    openai_api_key=ZHIPU_API_KEY,
    openai_api_base=ZHIPU_BASE_URL,
)

# ========== 2. 状态定义 ==========
class EmailClassification(TypedDict):
    intent: str
    urgency: str
    topic: str
    summary: str

class EmailAgentState(TypedDict):
    email_content: str
    sender_email: str
    email_id: str
    classification: EmailClassification | None
    search_results: list[str] | None
    draft_response: str | None

# ========== 3. 节点函数 ==========
def read_email(state: EmailAgentState):
    print(f"📧 收到邮件:{state['email_content'][:50]}...")
    return {}

def classify_intent(state: EmailAgentState) -> Command:
    prompt = (
        "分析下面这封客户邮件，返回 JSON 格式的分类结果。\n"
        "字段：intent(question/bug/billing/feature/complex)、urgency(low/medium/high/critical)、topic(话题)、summary(总结)\n"
        "只返回 JSON，不要其他内容。\n\n"
        f"邮件：{state['email_content']}\n发件人：{state['sender_email']}"
    )
    response = llm.invoke([("human", prompt)])
    try:
        text = response.content.strip()
        if text.startswith("```"):
            text = text.split("```")[1].lstrip("json")
        classification = json.loads(text.strip())
    except (json.JSONDecodeError, IndexError):
        classification = {"intent": "complex", "urgency": "medium",
                         "topic": "unknown", "summary": state["email_content"][:100]}
    intent = classification.get("intent", "complex")
    goto = {
        "question": "search_documentation",
        "feature": "search_documentation",
        "billing": "search_documentation",
        "bug": "bug_tracking"
    }.get(intent, "draft_response")
    print(f"📋 分类结果：intent={intent}, urgency={classification.get('urgency')}")
    return Command(update={"classification": classification}, goto=goto)

def search_documentation(state: EmailAgentState) -> Command:
    classification = state.get("classification", {})
    topic = classification.get("topic", "").lower()
    email_content = state.get("email_content", "").lower()
    mock_docs = {
        "password": ["重置密码：设置 → 安全 → 修改密码", "密码要求：至少12位，包含大小写和数字"],
        "billing": ["退款流程：提交工单后3-5个工作日处理", "重复扣款：系统自动检测并退还多余费用"],
        "dark": ["深色模式已在开发计划中", "预计下个版本发布"],
    }
    search_results = []
    for key, docs in mock_docs.items():
        if key in topic or key in email_content:
            search_results.extend(docs)
    search_results = list(dict.fromkeys(search_results)) or ["暂未找到相关文档"]
    print(f"🔍 搜索到 {len(search_results)} 条相关文档")
    return Command(update={"search_results": search_results}, goto="draft_response")

def bug_tracking(state: EmailAgentState) -> Command:
    email_id = state.get("email_id", "000")
    ticket_id = f"BUG-{email_id[-3:]}"
    print(f"🐛 创建了 Bug 工单：{ticket_id}")
    return Command(
        update={"search_results": [f"已创建 Bug 工单 {ticket_id}，技术团队会尽快处理"]},
        goto="draft_response"
    )

def draft_response(state: EmailAgentState) -> Command:
    classification = state.get("classification", {})
    search_results = state.get("search_results", [])
    context = "参考资料：\n" + "\n".join([f"- {doc}" for doc in search_results]) if search_results else ""
    prompt = (
        f"为以下客户邮件写回复（专业友好，中文直接输出）：\n"
        f"邮件内容：{state['email_content']}\n"
        f"邮件类型：{classification.get('intent', '未知')}\n"
        f"紧急程度：{classification.get('urgency', '中')}\n"
        f"{context}"
    )
    draft = llm.invoke([("human", prompt)]).content
    needs_review = (
        classification.get("urgency") == "critical"
        or classification.get("intent") == "billing"
    )
    goto = "human_review" if needs_review else "send_reply"
    print(f"✍️ 回复草稿已生成，{'需要人工审核' if needs_review else '直接发送'}")
    return Command(update={"draft_response": draft}, goto=goto)

def human_review(state: EmailAgentState) -> Command:
    classification = state.get("classification", {})
    human_decision = interrupt({
        "email_id": state.get("email_id"),
        "original_email": state.get("email_content"),
        "draft_response": state.get("draft_response"),
        "urgency": classification.get("urgency"),
        "action": "请审核回复：approved=True 表示通过，可传入 edited_response 修改草稿"
    })
    if human_decision.get("approved"):
        edited_draft = human_decision.get("edited_response", state.get("draft_response", ""))
        print("✅ 人工审核通过")
        return Command(update={"draft_response": edited_draft}, goto="send_reply")
    else:
        print("❌ 人工拒绝，由人工自行处理")
        return Command(update={}, goto="__end__")

def send_reply(state: EmailAgentState):
    print(f"📤 回复已发送给 {state['sender_email']}")
    return {}

# ========== 4. 组装图 ==========
graph_builder = StateGraph(EmailAgentState)
graph_builder.add_node("read_email", read_email)
graph_builder.add_node("classify_intent", classify_intent)
graph_builder.add_node("search_documentation", search_documentation)
graph_builder.add_node("bug_tracking", bug_tracking)
graph_builder.add_node("draft_response", draft_response)
graph_builder.add_node("human_review", human_review)
graph_builder.add_node("send_reply", send_reply)

graph_builder.add_edge(START, "read_email")
graph_builder.add_edge("read_email", "classify_intent")
graph_builder.add_edge("send_reply", END)

memory = MemorySaver()
app = graph_builder.compile(checkpointer=memory)

# ========== 5. 测试 ==========
if __name__ == "__main__":
    # 测试简单咨询场景
    app.invoke(
        {
            "email_content": "你好，请问怎么重置密码？谢谢",
            "sender_email": "zhangsan@example.com",
            "email_id": "email_001"
        },
        config={"configurable": {"thread_id": "test_001"}}
    )
```

## 六、关键亮点与实战经验
### 1. 核心亮点
| 特性                | 价值                                  |
|---------------------|---------------------------------------|
| `Command` 路由      | 节点内同时更新状态+指定路由，无需额外条件边，代码简洁 |
| 状态存储原始数据    | 避免格式化耦合，各节点可灵活处理数据        |
| `interrupt()` 人机协作 | 支持流程暂停/恢复，配合 checkpointer 保存状态，适配人工审核场景 |
| 业务风控内置        | 涉钱/紧急邮件强制人工审核，符合实际业务规则    |

### 2. 实战避坑指南
1. **模型 JSON 输出不稳定**：需处理 markdown 代码块包裹（` ```json `），添加解析失败降级逻辑；
2. **人工审核必须配 checkpointer**：`interrupt()` 依赖 `MemorySaver` 等检查点保存状态，否则无法恢复；
3. **路由逻辑优先业务规则**：人工审核触发条件（如“涉钱”）需硬编码，不依赖模型分类结果，避免风险；
4. **恢复流程需一致 thread_id**：`Command(resume=...)` 必须使用与暂停时相同的 `thread_id`，否则找不到状态。
