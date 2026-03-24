# 第六课：多 Agent 协作——共享记忆、跨 Agent 通信、Group Chat 内容总结
本课核心为基于Letta框架实现**多Agent协作**，解决独立Agent间的信息共享与通信问题，依托Memory Block全局唯一ID实现**跨Agent共享记忆**，并通过**工具通信**和**Group Chat群聊**两种核心方式实现Agent协作，以招聘场景为实战案例，完成“简历评估+自动写邮件”的多Agent自动化流程。

## 一、跨Agent共享记忆区段
利用Memory Block的**全局唯一ID**特性，创建独立于Agent的公共记忆块，挂载到多个Agent上，实现多Agent共享同一份记忆，且修改后所有挂载Agent实时同步，核心是通过`client.blocks.create()`创建独立块，而非与Agent绑定的`memory_blocks`。

### 核心操作代码
```python
from letta_client import Letta
client = Letta(base_url="http://localhost:8283")

# 定义共享的公司信息内容
company_description = (
    "公司名称:AgentOS｡"
    "主要业务:开发 AI 工具,帮助用户更轻松地创建和部署 LLM Agent｡"
)

# 创建独立的共享记忆块（全局唯一，可挂载到任意Agent）
company_block = client.blocks.create(
    value=company_description,
    label="company",
    limit=10000
)
print(f"Block ID: {company_block.id}")
```
**关键区别**：`client.blocks.create()`创建的块独立存在，可多Agent共享；`memory_blocks`创建的块与Agent一对一绑定，无法共享。

## 二、工具通信：一对一有明确调用关系的Agent协作
适用于**上下游清晰的请求-响应场景**（如评估Agent筛简历→外联Agent写邮件），通过Letta内置的`send_message_to_agent_and_wait_for_reply`工具实现跨Agent同步通信，一个Agent调用工具给另一个Agent发消息，等待回复后继续执行，搭配共享记忆块实现信息互通。

### 2.1 实战：招聘系统（评估Agent+外联Agent）
#### 步骤1：定义外联Agent的工具与Agent本身
```python
# 1. 定义写邮件工具
def draft_candidate_email(content: str):
    """
    Draft an email to reach out to a candidate.
    Args:
        content (str): Content of the email
    """
    return f"Here is a draft email: {content}"

# 注册工具
draft_email_tool = client.tools.upsert_from_function(func=draft_candidate_email)

# 2. 定义外联Agent人设
outreach_persona = (
    "你负责为公司撰写招聘邮件｡"
    "当收到候选人信息时,用 draft_candidate_email 工具写一封邮件草稿｡"
)

# 3. 创建外联Agent（挂载共享记忆块）
outreach_agent = client.agents.create(
    name="outreach_agent",
    memory_blocks=[{"label": "persona", "value": outreach_persona}],
    model="openai/gpt-4o-mini-2024-07-18",
    embedding="openai/text-embedding-ada-002",
    tools=[draft_email_tool.name],
    block_ids=[company_block.id]  # 挂载共享公司信息块
)
```

#### 步骤2：定义评估Agent的工具与Agent本身
```python
# 1. 定义拒绝候选人工具
def reject(candidate_name: str):
    """
    Reject a candidate.
    Args:
        candidate_name (str): The name of the candidate
    """
    return

# 注册工具
reject_tool = client.tools.upsert_from_function(func=reject)

# 2. 定义评估Agent人设（写入外联Agent ID，明确调用目标）
skills = "前端开发(React､TypeScript)或 LLM 相关的软件工程经验"
eval_persona = (
    f"你负责评估候选人｡"
    f"理想候选人应具备以下技能:{skills}｡"
    "不符合要求的候选人,用 reject 工具拒绝｡"
    f"优秀候选人,发送给 Agent ID {outreach_agent.id} 处理｡"
    "你必须对每个候选人做出判断——要么拒绝,要么转发给外联 Agent｡"
)

# 3. 创建评估Agent（内置通信工具+Tool Rules约束行为+挂载共享块）
eval_agent = client.agents.create(
    name="eval_agent",
    memory_blocks=[{"label": "persona", "value": eval_persona}],
    model="openai/gpt-4o-mini-2024-07-18",
    embedding="openai/text-embedding-ada-002",
    tool_ids=[reject_tool.id],
    tools=["send_message_to_agent_and_wait_for_reply"],  # 内置跨Agent通信工具
    include_base_tools=False,  # 关闭默认工具，仅保留必要工具
    block_ids=[company_block.id],  # 挂载共享公司信息块
    tool_rules=[  # 约束行为：调用通信工具后立即退出推理循环
        {
            "type": "exit_loop",
            "tool_name": "send_message_to_agent_and_wait_for_reply"
        }
    ]
)

# 验证评估Agent工具列表
print([tool.name for tool in eval_agent.tools])
```

#### 步骤3：运行测试（简历评估+自动写邮件）
```python
# 读取简历文件
resume = open("resumes/tony_stark.txt", "r").read()
print(resume[:200])

# 给评估Agent发送简历，流式获取结果
response = client.agents.messages.create_stream(
    agent_id=eval_agent.id,
    messages=[
        {
            "role": "user",
            "content": f"请评估这份简历:{resume}"
        }
    ]
)
for message in response:
    print_message(message)

# 查看外联Agent的执行日志
for message in client.agents.messages.list(agent_id=outreach_agent.id)[1:]:  # 跳过系统消息
    print_message(message)
```

### 2.2 共享记忆同步验证
修改其中一个Agent的共享块，所有挂载Agent自动同步，无需额外同步逻辑：
```python
# 让外联Agent更新公司名称（AgentOS→Letta）
response = client.agents.messages.create_stream(
    agent_id=outreach_agent.id,
    messages=[
        {
            "role": "user",
            "content": "公司已经从 AgentOS 改名为 Letta 了,更新一下｡"
        }
    ]
)
for message in response:
    print_message(message)

# 验证评估Agent的共享块是否同步更新
eval_company = client.agents.blocks.retrieve(
    agent_id=eval_agent.id,
    block_label="company"
)
print(f"评估 Agent 看到的公司信息: {eval_company.value}")

# 验证外联Agent的共享块
outreach_company = client.agents.blocks.retrieve(
    agent_id=outreach_agent.id,
    block_label="company"
)
print(f"外联 Agent 看到的公司信息: {outreach_company.value}")
```

### 2.3 核心特性
1. **同步通信**：`send_message_to_agent_and_wait_for_reply`为同步工具，调用方会等待被调用方回复后再继续；
2. **Tool Rules约束**：通过`exit_loop`规则防止Agent调用工具后继续无意义推理，让行为更可控；
3. **共享记忆实时同步**：修改共享块后，所有挂载Agent的记忆自动更新，依托数据库层面实现。

## 三、Group Chat：松散的多Agent群聊协作
适用于**无明确上下游的团队型协作场景**，Letta提供`Group`抽象，将多个Agent加入群聊，共享消息历史，默认采用**Round-Robin（轮流发言）** 模式，Agent根据共享的对话历史自主判断行为，无需跨Agent通信工具。

### 3.1 核心操作代码
#### 步骤1：定义多Agent消息打印函数（区分发言主体）
```python
def print_message_multiagent(message):
    if message.message_type == "reasoning_message":
        print(f" [{message.name}] 思考: {message.reasoning}")
    elif message.message_type == "assistant_message":
        print(f"🤖 [{message.name}] 回复: {message.content}")
    elif message.message_type == "tool_call_message":
        print(f"🔧 [{message.name}] 工具调用: {message.tool_call.name}")
        print(f" {message.tool_call.arguments}")
    elif message.message_type == "tool_return_message":
        print(f"🔧 [{message.name}] 工具返回: {message.tool_return}")
    elif message.message_type == "user_message":
        print(f"👤 用户: {message.content}")
    elif message.message_type == "usage_statistics":
        print(f"📊 统计: {message}")
    print("-----")
    return
```

#### 步骤2：重新创建无通信工具的Agent（仅保留业务工具+共享块）
```python
# 重新创建外联Agent（移除通信工具，仅保留写邮件工具）
outreach_agent = client.agents.create(
    name="outreach_agent",
    memory_blocks=[{"label": "persona", "value": outreach_persona}],
    model="openai/gpt-4o-mini-2024-07-18",
    embedding="openai/text-embedding-ada-002",
    tool_ids=[draft_email_tool.id],
    block_ids=[company_block.id]
)

# 重新创建评估Agent（移除通信工具，仅保留拒绝工具）
eval_agent = client.agents.create(
    name="eval_agent",
    memory_blocks=[{"label": "persona", "value": eval_persona}],
    model="openai/gpt-4o-mini-2024-07-18",
    embedding="openai/text-embedding-ada-002",
    tool_ids=[reject_tool.id],
    block_ids=[company_block.id]
)
```

#### 步骤3：创建Round-Robin群聊并加入Agent
```python
# 创建群聊，指定协作Agent
round_robin_group = client.groups.create(
    description="招聘团队:负责评估候选人和撰写外联邮件",
    agent_ids=[eval_agent.id, outreach_agent.id],
)

# 给群聊发送不合格简历，测试协作
resume = open("resumes/spongebob_squarepants.txt", "r").read()
response_stream = client.groups.messages.create_stream(
    group_id=round_robin_group.id,
    messages=[
        {"role": "user", "content": f"请评估这份简历:{resume}"}
    ]
)

# 打印群聊协作过程
for message in response_stream:
    print_message_multiagent(message)
```

### 3.2 工具通信与Group Chat对比
| 对比维度 | 工具通信 | Group Chat |
| ---- | ---- | ---- |
| 消息可见性 | 各Agent有独立的消息历史 | 所有Agent共享同一个消息历史 |
| 触发方式 | Agent A主动调用工具给B发消息，同步等待回复 | 按Round-Robin规则轮流发言，自主判断行为 |
| 控制精度 | 高，可精准控制调用时机和目标 | 低，协作更松散，依赖Agent自主推理 |
| 实现复杂度 | 中等，需配置通信工具+写入目标Agent ID | 低，仅需创建Group并加入Agent |
| 适合场景 | 明确的上下游请求-响应（如评估→写邮件） | 团队型讨论协作，无固定调用关系 |

## 四、多Agent协作的关键注意事项
1. **共享块写冲突**：Letta无锁机制，多Agent同时修改共享块会**后写入覆盖先写入**，并发写入场景需在应用层做互斥处理；
2. **同步通信的延迟问题**：`send_message_to_agent_and_wait_for_reply`为同步工具，被调用Agent推理过慢会导致调用方卡住，对延迟敏感场景可自定义异步通信工具；
3. **Group Chat的Agent数量限制**：Round-Robin模式下Agent过多会导致token消耗剧增、对话历史膨胀，建议3-4个以内，多Agent场景可**分层协作**（小组内群聊，小组间工具通信）；
4. **Tool Rules的重要性**：除`exit_loop`外，还可设置“调用A工具后必须调用B工具”“工具最多调用N次”等规则，是约束多Agent行为、防止无限循环的关键；
5. **底层思路通用**：Agent框架迭代快，但**自编辑记忆、分层存储、有状态工具、跨Agent通信**的核心设计思路不变，掌握后可适配任意框架。

## 五、Agent记忆系统系列课程最终回顾
六课内容从**单Agent无记忆**逐步进阶到**多Agent协作**，完成了Agent从“玩具”到“产品级协作工具”的全流程搭建，完整技术路径：
1. 第一课：手搓自编辑记忆Agent，理解**记忆是可编辑的上下文**；
2. 第二课：拆解MemGPT三层架构（核心/回忆/归档），掌握Agent记忆的**操作系统思路**；
3. 第三课：Letta框架实战，跑通单Agent的**核心记忆编辑、归档记忆存取**；
4. 第四课：实现**自定义记忆结构+有状态工具**，打破默认记忆结构限制；
5. 第五课：落地**Agentic RAG**，实现静态文档导入+动态数据库查询，让Agent自主接入外部数据；
6. 第六课：实现**多Agent协作**，通过共享记忆块实现信息互通，依托工具通信和Group Chat完成不同场景的协作。

通过六课学习，可基于Letta框架设计并实现适配实际业务的多Agent系统，覆盖记忆管理、外部数据接入、跨Agent协作全环节。