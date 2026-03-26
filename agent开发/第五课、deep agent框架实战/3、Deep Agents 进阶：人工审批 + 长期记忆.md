# Deep Agents进阶：人工审批+长期记忆课程总结

> 对应原文 PDF：`D:\notebook\agent开发\deep agent框架实战\第三课：Deep Agents 进阶：人工审批 + 长期记忆.pdf`
> 说明：在不改变原文含义的前提下做精简总结；代码示例保持完整，不删减。

本课程聚焦Deep Agents的两大核心进阶功能——**人工审批（Human-in-the-Loop）** 和**长期记忆（Longterm Memory）**，基于`deepagents==0.4.3`（Python3.11+）展开实操讲解，解决Agent操作安全性与跨会话记忆丢失问题，是生产级Agent开发的关键能力补充。

## 一、人工审批（Human-in-the-Loop）：给Agent加“安全绳”
### 1. 核心价值
解决Agent高危操作（删文件、发邮件、执行SQL等）的安全风险，避免误删数据、恶意操作等问题，通过“Agent暂停→人工确认→恢复执行”的流程，确保操作可控，提升用户信任度。

### 2. 核心配置与实操（三步搞定）
#### 步骤1：定义工具+配置审批规则
通过`interrupt_on`参数指定需审批的工具及允许的操作，**必须配置`checkpointer`**（用于保存Agent暂停状态，否则无法恢复）：
```python
from langchain.tools import tool
from deepagents import create_deep_agent
from langgraph.checkpoint.memory import MemorySaver

# 定义工具（删文件、读文件、发邮件）
@tool
def delete_file(path: str) -> str:
    """删除指定文件｡"""
    return f"已删除 {path}"
@tool
def read_file(path: str) -> str:
    """读取指定文件｡"""
    return f"{path} 的内容..."
@tool
def send_email(to: str, subject: str, body: str) -> str:
    """发送邮件｡"""
    return f"邮件已发送给 {to}"

# 初始化checkpointer（必传）
checkpointer = MemorySaver()

# 创建Agent，配置审批规则
agent = create_deep_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[delete_file, read_file, send_email],
    interrupt_on={
        "delete_file": True,  # 开启审批，默认允许"同意/编辑/拒绝"
        "read_file": False,   # 不审批，直接执行
        "send_email": {"allowed_decisions": ["approve", "reject"]}  # 仅允许同意/拒绝
    },
    checkpointer=checkpointer  # 保存暂停状态，核心必填
)
```
#### `interrupt_on`参数规则
| value类型 | 含义 | 允许操作 |
|-----------|------|----------|
| True | 开启审批 | 默认：approve（同意）、edit（编辑）、reject（拒绝） |
| False | 不审批 | 直接执行工具 |
| 字典`{"allowed_decisions": [...]}` | 自定义审批操作 | 按需指定允许的操作（如仅同意/拒绝） |

#### 步骤2：检测Agent中断并获取审批请求
Agent调用需审批的工具时会自动暂停，返回中断信号，需解析待审批的工具及参数：
```python
import uuid
from langgraph.types import Command

# 生成唯一thread_id，确保会话一致性
config = {"configurable": {"thread_id": str(uuid.uuid4())}}

# 触发Agent执行需审批的操作（删除temp.txt）
result = agent.invoke(
    {"messages": [{"role": "user", "content": "删除 temp.txt 文件"}]},
    config=config
)

# 检测是否触发中断
if result.get("__interrupt__"):
    interrupts = result["__interrupt__"][0].value
    action_requests = interrupts["action_requests"]  # 待审批的工具列表
    review_configs = interrupts["review_configs"]    # 各工具的审批规则
    
    # 映射工具与审批规则，便于查看
    config_map = {cfg["action_name"]: cfg for cfg in review_configs}
    for action in action_requests:
        review_config = config_map[action["name"]]
        print(f"工具: {action['name']}")
        print(f"参数: {action['args']}")
        print(f"可选操作: {review_config['allowed_decisions']}")
```
#### 步骤3：提交审批结果，恢复Agent执行
需使用**同一`config`（同一thread_id）** 提交审批结果，否则Agent无法找到暂停状态：
```python
# 1. 同意执行（approve）
decisions = [{"type": "approve"}]

# 2. 拒绝执行（reject）
# decisions = [{"type": "reject"}]

# 3. 编辑参数后执行（edit）- 如修改邮件收件人
# decisions = [{
#     "type": "edit",
#     "edited_action": {
#         "name": "send_email",  # 必须指定工具名
#         "args": {
#             "to": "team@company.com",  # 修改后的参数
#             "subject": "周报汇总",
#             "body": "详见附件｡"
#         }
#     }
# }]

# 提交审批结果，恢复Agent执行
result = agent.invoke(
    Command(resume={"decisions": decisions}),
    config=config  # 必须与触发中断时的config一致
)

# 打印最终结果
print(result["messages"][-1].content)
```

### 3. 批量审批：多工具同时触发审批
当Agent一次调用多个需审批的工具时，`action_requests`会包含所有待审批工具，`decisions`列表需按**工具顺序一一对应**提交结果：
```python
# 触发多个需审批的操作（删文件+发邮件）
result = agent.invoke(
    {"messages": [{"role": "user", "content": "删除 temp.txt，然后发邮件通知 admin@example.com"}]},
    config=config
)

if result.get("__interrupt__"):
    interrupts = result["__interrupt__"][0].value
    action_requests = interrupts["action_requests"]
    print(f"需审批工具数量: {len(action_requests)}")  # 输出：2
    
    # 按顺序提交审批结果（第一个同意删文件，第二个拒绝发邮件）
    decisions = [
        {"type": "approve"},  # 对应delete_file
        {"type": "reject"}   # 对应send_email
    ]
    
    # 恢复执行
    result = agent.invoke(Command(resume={"decisions": decisions}), config=config)
```

### 4. 子Agent的审批配置
子Agent可独立配置审批规则，与主Agent差异化管控（如主Agent读文件不审批，子Agent读文件需审批）：
```python
agent = create_deep_agent(
    tools=[delete_file, read_file],
    interrupt_on={
        "delete_file": True,
        "read_file": False,  # 主Agent读文件不审批
    },
    # 子Agent独立配置审批规则
    subagents=[{
        "name": "file-manager",
        "description": "负责文件管理操作",
        "system_prompt": "你是一个文件管理助手｡",
        "tools": [delete_file, read_file],
        "interrupt_on": {
            "delete_file": True,
            "read_file": True  # 子Agent读文件需审批
        }
    }],
    checkpointer=checkpointer
)
```

### 5. 自定义中断逻辑：`interrupt()`函数
支持在工具中直接调用`interrupt()`函数，实现复杂审批逻辑（如需要审批者补充信息）：
```python
from langgraph.types import interrupt

@tool(description="执行敏感操作前请求人工审批")
def request_approval(action_description: str) -> str:
    """用interrupt()暂停执行，等待人工审批"""
    approval = interrupt({
        "type": "approval_request",
        "action": action_description,
        "message": f"请审批: {action_description}",
    })
    
    if approval.get("approved"):
        return f"操作 '{action_description}' 已批准，继续执行..."
    else:
        reason = approval.get('reason', '未提供原因')
        return f"操作 '{action_description}' 被拒绝，原因: {reason}"

# 恢复执行时传入自定义审批结果
result = agent.invoke(
    Command(resume={"approved": True, "reason": "允许执行"}),
    config=config
)
```

### 6. 审批规则推荐配置（按风险分级）
```python
interrupt_on = {
    # 高风险（删/发/改）：允许同意/编辑/拒绝
    "delete_file": {"allowed_decisions": ["approve", "edit", "reject"]},
    "send_email": {"allowed_decisions": ["approve", "edit", "reject"]},
    # 中风险（写文件）：仅允许同意/拒绝，不允许编辑
    "write_file": {"allowed_decisions": ["approve", "reject"]},
    # 低风险（读/查）：不审批
    "read_file": False,
    "list_files": False
}
```

## 二、长期记忆（Longterm Memory）：让Agent记住过往
### 1. 核心价值
解决默认虚拟文件系统“会话结束即丢失”的问题，实现**跨会话、跨线程**的记忆持久化，让Agent记住用户偏好、项目背景、调研进度等信息，提升使用体验。

### 2. 核心原理
通过`CompositeBackend`实现**混合存储**：
- 临时区（默认路径）：绑定`StateBackend`，会话结束后文件丢失；
- 持久区（`/memories/`前缀路径）：绑定`StoreBackend`，跨会话/跨线程保留，重启不丢失。

### 3. 长期记忆配置实操
#### 步骤1：配置混合后端+Store
开发环境用`InMemoryStore`（内存版，便捷但重启丢失），生产环境用`PostgresStore`（持久化存储）：
```python
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore
from langgraph.checkpoint.memory import MemorySaver

# 初始化checkpointer（必传）
checkpointer = MemorySaver()

# 定义混合后端：/memories/路径持久化，其余临时
def make_backend(runtime):
    return CompositeBackend(
        default=StateBackend(runtime),  # 临时存储（默认）
        routes={"/memories/": StoreBackend(runtime)}  # 持久存储
    )

# 创建带长期记忆的Agent
agent = create_deep_agent(
    store=InMemoryStore(),  # 开发用内存Store
    backend=make_backend,
    checkpointer=checkpointer
)
```

#### 步骤2：使用长期记忆（读写`/memories/`路径文件）
```python
# 1. 写入持久化文件（跨会话保留）
agent.invoke({
    "messages": [{"role": "user", "content": "把我的偏好存到 /memories/preferences.txt：喜欢简洁代码，输出用中文"}]
})

# 2. 写入临时文件（会话结束丢失）
agent.invoke({
    "messages": [{"role": "user", "content": "把草稿写到 /draft.txt"}]
})
```

#### 步骤3：跨会话读取记忆
不同线程（不同`thread_id`）可读取`/memories/`下的持久化文件：
```python
import uuid

# 对话1：存储用户偏好（线程1）
config1 = {"configurable": {"thread_id": str(uuid.uuid4())}}
agent.invoke(
    {"messages": [{"role": "user", "content": "我喜欢简洁的代码风格，输出用中文，存到 /memories/preferences.txt"}]},
    config=config1
)

# 对话2：读取用户偏好（线程2，全新会话）
config2 = {"configurable": {"thread_id": str(uuid.uuid4())}}
result = agent.invoke(
    {"messages": [{"role": "user", "content": "看看我之前说过什么偏好？"}],
    config=config2
)

print(result["messages"][-1].content)  # 输出：你喜欢简洁的代码风格，输出用中文
```

### 4. 四大实用场景
#### 场景1：记住用户偏好
```python
agent = create_deep_agent(
    store=InMemoryStore(),
    backend=make_backend,
    system_prompt="""用户告诉你的偏好和习惯，存到 /memories/user_preferences.txt，
    下次对话开始先读这个文件，按偏好响应。"""
)
```

#### 场景2：自我进化的指令
Agent根据用户反馈自动更新工作指令，越用越贴合需求：
```python
agent = create_deep_agent(
    store=InMemoryStore(),
    backend=make_backend,
    system_prompt="""你有一个指令文件 /memories/instructions.txt。
    每次对话开始先读这个文件；
    当用户给反馈（如“以后请总是XXX”），用edit_file工具更新该文件。"""
)
```

#### 场景3：多轮调研项目续存
保留调研进度，跨会话继续深入，无需从头开始：
```python
research_agent = create_deep_agent(
    store=InMemoryStore(),
    backend=make_backend,
    system_prompt="""你是研究助手，调研进度存到 /memories/research/ 目录：
    - /memories/research/sources.txt：信息源
    - /memories/research/notes.txt：关键笔记
    - /memories/research/report.md：报告草稿
    每次对话先读这些文件，接着上次进度继续。"""
)
```

#### 场景4：知识库积累
自动存储项目背景、技术栈等信息，后续对话直接复用：
```python
# 对话1：存储项目背景
agent.invoke({
    "messages": [{"role": "user", "content": "我们项目用React+Next.js，后端FastAPI，存到 /memories/project_context.txt"}]
})

# 对话2：复用背景生成代码
result = agent.invoke({
    "messages": [{"role": "user", "content": "帮我写个获取用户列表的API"}]
})
# Agent自动读取项目背景，输出FastAPI风格代码
```

### 5. 生产环境持久化：PostgresStore
`InMemoryStore`重启后数据丢失，生产环境需用`PostgresStore`（数据存PostgreSQL，持久化且支持多实例共享）：
```python
from langgraph.store.postgres import PostgresStore
import os

# 从环境变量读取数据库连接串
store_ctx = PostgresStore.from_conn_string(os.environ["DATABASE_URL"])
store = store_ctx.__enter__()
store.setup()  # 初始化数据库表

# 创建Agent时绑定PostgresStore
agent = create_deep_agent(
    store=store,
    backend=make_backend,
    checkpointer=checkpointer
)
```

### 6. 外部读写记忆文件（可选）
通过`create_file_data`工具函数或LangSmith Store API，可从外部代码读写记忆文件：
```python
# 1. 外部写入记忆文件
from deepagents.backends.utils import create_file_data
file_data = create_file_data("喜欢简洁风格\n输出用中文")  # 生成标准文件格式
# 后续可通过Store API写入Store

# 2. LangSmith部署后，通过API读写（需去掉/memories/前缀）
from langgraph_sdk import get_client
client = get_client(url="<DEPLOYMENT_URL>")

# 读记忆文件
item = await client.store.get_item(
    (assistant_id, "filesystem"),
    "/preferences.txt"  # 无/memories/前缀
)

# 写记忆文件
await client.store.put_item(
    (assistant_id, "filesystem"),
    "/preferences.txt",
    {
        "content": ["喜欢简洁风格", "输出用中文"],
        "created_at": "2026-02-25T10:30:00Z",
        "modified_at": "2026-02-25T10:30:00Z"
    }
)
```

## 三、人工审批+长期记忆组合使用
将两个功能结合，实现“安全可控+记忆持久”的Agent，以文件管理Agent为例：
```python
from langchain.tools import tool
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore
from langgraph.checkpoint.memory import MemorySaver

# 定义工具
@tool
def delete_file(path: str) -> str:
    """删除指定文件｡"""
    return f"已删除 {path}"
@tool
def organize_files(directory: str) -> str:
    """整理指定目录下的文件｡"""
    return f"已整理 {directory}"

# 初始化依赖组件
checkpointer = MemorySaver()
def make_backend(runtime):
    return CompositeBackend(
        default=StateBackend(runtime),
        routes={"/memories/": StoreBackend(runtime)}
    )

# 组合功能的Agent
agent = create_deep_agent(
    model="claude-sonnet-4-5-20250929",
    tools=[delete_file, organize_files],
    # 人工审批：删文件需审批，整理文件无需
    interrupt_on={
        "delete_file": True,
        "organize_files": False
    },
    # 长期记忆：/memories/路径持久化
    backend=make_backend,
    store=InMemoryStore(),
    checkpointer=checkpointer,
    # 系统提示词：结合记忆与审批
    system_prompt="""你是文件管理助手｡
    1. 每次对话开始先读 /memories/preferences.txt 了解用户偏好，新偏好及时更新；
    2. 删除文件前自动暂停等用户审批，避免误删；
    3. 整理文件时优先遵循用户偏好（如跳过.env文件）。"""
)
```

## 四、核心踩坑记录
1. **`checkpointer`必传**：人工审批和长期记忆都依赖`checkpointer`保存状态，漏传会导致Agent中断后无法恢复、记忆丢失；
2. **路径前缀被“吃掉”**：Agent写入`/memories/preferences.txt`，Store中实际存储路径为`/preferences.txt`，外部API访问需去掉`/memories/`前缀；
3. **`config`一致性**：人工审批恢复执行时，必须使用与触发中断时相同的`config`（同一`thread_id`），否则找不到暂停状态；
4. **`InMemoryStore`的局限性**：开发用便捷，但程序重启/崩溃会丢失数据，建议开发阶段也用SQLite/本地文件Store；
5. **记忆文件大小控制**：Agent每次对话会读取记忆文件，需在`system_prompt`中限制文件长度（如“超过50行自动整理合并”），避免占用过多上下文；
6. **多Agent隔离**：StoreBackend以`(assistant_id, "filesystem")`为命名空间，多Agent实例的记忆相互隔离，多租户场景需注意区分`assistant_id`。

## 五、总结
1. 人工审批和长期记忆是生产级Agent的必备能力，分别解决“安全可控”和“体验连贯”两大核心问题；
2. 配置简洁，无需大幅修改业务代码，通过`interrupt_on`和`CompositeBackend`即可快速集成；
3. 组合使用可实现“既安全又智能”的Agent，适配文件管理、调研分析、项目协作等复杂场景；
4. 开发与生产环境需区分Store类型，生产环境优先使用`PostgresStore`确保数据持久化。

官方代码仓库：https://github.com/langchain-ai/deepagents
