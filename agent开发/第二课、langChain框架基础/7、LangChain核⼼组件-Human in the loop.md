# LangChain核心组件-Human in the Loop（HITL）核心笔记
Human in the Loop（HITL，人工介入）是LangChain中保障Agent**生产环境安全**的关键组件，核心是在Agent执行高风险工具（如发邮件、删数据、转账）前，插入“人工确认”环节，避免模型幻觉或误操作导致不可逆损失，是Agent从“功能可用”到“生产可靠”的必备保障。

## 一、核心原理：Agent的“暂停-审批-恢复”机制
### 1. 介入时机
HITL在Agent运行循环的**模型推理后、工具执行前**触发，精准拦截高风险操作：
`模型推理→决定调用高风险工具→[HITL拦截检查]→需审批则暂停→人工决策→恢复执行/重新推理`

### 2. 底层依赖：Checkpointer（状态存档）
HITL必须配合`Checkpointer`使用，核心原因：
- Agent暂停时，会将当前完整状态（对话历史、工具调用计划、上下文）存入`Checkpointer`；
- 人工审批后，Agent从存档点恢复运行，无需重复之前的推理步骤，类似游戏“存档读档”。
- 开发/测试用`InMemorySaver`，生产用数据库级存储（如PostgresSaver）。

## 二、基础用法：快速配置高风险工具拦截
### 1. 最简代码实现
```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver

# 定义工具（示例：发邮件、查联系人）
@tool
def send_email(recipient: str, subject: str, body: str) -> str:
    """发送邮件（高风险）"""
    return f"邮件已发送给{recipient}"

@tool
def search_contacts(name: str) -> str:
    """查询联系人（低风险）"""
    return f"联系人{name}的邮箱：xxx@example.com"

# 创建带HITL的Agent
agent = create_agent(
    model="gpt-4o",
    tools=[send_email, search_contacts],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": True,    # 高风险工具：拦截审批
                "search_contacts": False # 低风险工具：直接放行
            }
        )
    ],
    checkpointer=InMemorySaver()  # 必须配置，用于状态存档
)
```

### 2. `interrupt_on`核心配置
字典格式，key为工具名，value控制是否拦截及允许的决策类型：
| value类型 | 作用 | 示例 |
|----------|------|------|
| `True` | 拦截，允许所有决策（approve/edit/reject） | `"send_email": True` |
| `False` | 不拦截，自动放行 | `"search_contacts": False` |
| 字典 | 精细控制（允许的决策类型、描述信息） | `"delete_data": {"allowed_decisions": ["approve", "reject"], "description": "删除数据库记录，谨慎审批"}` |

## 三、三种人工决策类型（核心操作）
Agent被拦截后，人工可通过`Command(resume={...})`下达决策，支持三种核心操作：

### 1. Approve（批准）：按原计划执行
工具调用计划无问题时，批准后Agent直接执行工具。
```python
from langgraph.types import Command

# 同一thread_id，从存档点恢复
config = {"configurable": {"thread_id": "email-session-1"}}

# 人工批准执行
agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config
)
```

### 2. Edit（修改）：微调后执行
工具调用计划大体合理，但需修改参数（如邮件内容、收件人），修改后Agent执行修改后的操作。
```python
agent.invoke(
    Command(
        resume={
            "decisions": [
                {
                    "type": "edit",
                    "edited_action": {
                        "name": "send_email",  # 工具名（必须与原计划一致）
                        "args": {
                            "recipient": "alice@example.com",
                            "subject": "下周约咖啡",
                            "body": "Hi Alice，下周三下午3点老地方见～"  # 修改后的内容
                        }
                    }
                }
            ]
        }
    ),
    config=config
)
```
- 注意：仅做**小幅修改**，避免参数与模型之前的推理上下文冲突，导致后续操作异常。

### 3. Reject（拒绝）：重新生成
工具调用计划完全不合理（如语气错误、收件人错误），拒绝后需附带反馈信息，模型会重新推理并生成新的工具调用计划，再次触发HITL拦截审批。
```python
agent.invoke(
    Command(
        resume={
            "decisions": [
                {
                    "type": "reject",
                    "message": "语气太正式了，用轻松口语化的口吻重写邮件"  # 反馈给模型的修改建议
                }
            ]
        }
    ),
    config=config
)
```
- 流程闭环：`拒绝→模型重新推理→新工具调用计划→再次拦截审批→直至批准`。

## 四、精细配置：按需控制决策权限与提示
针对不同工具的风险等级，可通过字典格式精细化配置，避免过度审批或权限滥用：
```python
HumanInTheLoopMiddleware(
    interrupt_on={
        # 1. 允许所有决策（approve/edit/reject）
        "send_email": True,
        # 2. 仅允许批准/拒绝，禁止手动修改（如SQL执行，避免上下文冲突）
        "execute_sql": {"allowed_decisions": ["approve", "reject"]},
        # 3. 自定义审批提示，方便前端展示
        "delete_record": {
            "allowed_decisions": ["approve", "reject"],
            "description": "即将删除数据库核心记录，请确认操作合法性！"
        },
        # 4. 无需审批
        "search_web": False
    },
    description_prefix="以下操作需人工审批："  # 全局审批提示前缀
)
```

## 五、实战案例：安全邮件助手（完整流程）
### 1. 场景目标
实现“查询联系人邮箱→撰写邮件→人工审批→发送”的安全流程，仅拦截“发送邮件”步骤。

### 2. 完整代码
```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langchain.tools import tool
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command

# 1. 定义工具
@tool
def get_user_email(user_name: str) -> str:
    """查询用户邮箱（低风险，自动放行）"""
    user_emails = {"alice": "alice@example.com", "bob": "bob@example.com"}
    return user_emails.get(user_name.lower(), "未找到该用户邮箱")

@tool
def send_email(recipient: str, subject: str, body: str) -> str:
    """发送邮件（高风险，需审批）"""
    return f"邮件已发送至{recipient}，主题：{subject}"

# 2. 创建Agent
agent = create_agent(
    model="gpt-4o",
    tools=[get_user_email, send_email],
    system_prompt="你是安全邮件助手，帮用户查询邮箱并撰写邮件",
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={"get_user_email": False, "send_email": True}
        )
    ],
    checkpointer=InMemorySaver()
)

# 3. 执行流程
config = {"configurable": {"thread_id": "safe-email-1"}}

# 第一步：用户发起请求（Agent自动查邮箱，然后触发发送拦截）
result = agent.invoke(
    {"messages": [{"role": "user", "content": "帮我给Alice发邮件，问她明天有空吃午饭吗"}]},
    config=config
)

# 第二步：查看Agent的邮件计划（从interrupt字段提取）
interrupt = result["__interrupt__"]
action = interrupt[0].value["action_requests"][0]
print(f"待审批：\n收件人：{action['arguments']['recipient']}\n主题：{action['arguments']['subject']}\n内容：{action['arguments']['body']}")

# 第三步：人工审批（示例：批准）
agent.invoke(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config
)
```

### 3. 核心亮点
- 低风险操作（查邮箱）自动放行，高风险操作（发邮件）精准拦截；
- 人工可清晰查看Agent的操作计划，避免“盲批”；
- 审批后Agent无缝恢复，流程连贯无重复。

## 六、高级场景：多工具拦截与流式输出兼容
### 1. 多个高风险工具同时拦截
若Agent一次调用多个需审批的工具，需按`interrupt`中`action_requests`的顺序，逐个给出决策：
```python
# 两个工具同时被拦截：send_email（1）、delete_data（2）
agent.invoke(
    Command(
        resume={
            "decisions": [
                {"type": "approve"},  # 第一个工具：批准
                {"type": "reject", "message": "暂不删除，需确认数据归属"}  # 第二个工具：拒绝
            ]
        }
    ),
    config=config
)
```

### 2. 流式输出+HITL（优化用户体验）
HITL可与流式输出兼容，实现“边推理边输出→暂停审批→批准后继续输出”的流畅体验：
```python
# 流式运行，直到遇到拦截
for mode, chunk in agent.stream(
    {"messages": [{"role": "user", "content": "帮我给Bob发邮件"}]},
    config=config,
    stream_mode=["updates", "messages"]
):
    if mode == "messages":
        token, _ = chunk
        if token.content:
            print(token.content, end="", flush=True)  # 实时输出模型推理内容
    elif mode == "updates" and "__interrupt__" in chunk:
        print(f"\n\n需审批：{chunk['__interrupt__']}")  # 拦截后提示审批

# 人工审批后，继续流式输出
for mode, chunk in agent.stream(
    Command(resume={"decisions": [{"type": "approve"}]}),
    config=config,
    stream_mode=["updates", "messages"]
):
    if mode == "messages":
        token, _ = chunk
        if token.content:
            print(token.content, end="", flush=True)  # 继续输出后续结果
```

## 核心总结
1. HITL的核心价值是**拦截不可逆高风险操作**，仅对有副作用的工具启用，避免过度审批影响效率；
2. 必须依赖`Checkpointer`实现状态存档与恢复，否则无法暂停后继续运行；
3. 支持三种决策类型：`approve`（直接执行）、`edit`（小幅修改）、`reject`（重新生成），覆盖不同审批场景；
4. 生产环境建议：对发邮件、删数据、转账、执行SQL等操作强制启用HITL，低风险只读操作直接放行；
5. 兼容多工具拦截和流式输出，可无缝融入实际项目的交互流程。

是否需要我针对“生产环境HITL+PostgresSaver持久化”提供更详细的代码示例？

## 代码补充（来自 PDF）
用于说明 Human in the Loop 控制流程的精简代码骨架。

### 示例 1：为高风险工具启用审批
```python
from langchain.agents.middleware import HumanInTheLoopMiddleware

agent = create_agent(
    model="openai:gpt-4.1-mini",
    tools=[send_email, delete_record],
    middleware=[HumanInTheLoopMiddleware(send_email=True, delete_record=True)],
)
```

### 示例 2：带 thread_id 调用
```python
result = agent.invoke(
    {"messages": [{"role": "user", "content": "给客户发送本周项目进展邮件"}]},
    config={"configurable": {"thread_id": "order-1001"}},
)
```
