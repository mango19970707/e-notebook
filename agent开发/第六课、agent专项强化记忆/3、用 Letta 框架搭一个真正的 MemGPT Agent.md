# 用Letta框架搭建MemGPT Agent课程总结
本课程聚焦MemGPT论文作者开源的Letta框架实操，讲解如何快速搭建具备**三层完整记忆（核心/回忆/归档）、自主记忆编辑、状态持久化**的生产级Agent，无需手动实现MemGPT架构细节，通过框架封装的API即可跑通全流程记忆管理功能。

## 一、Letta框架核心定位与优势
### 1. 核心定位
Letta是**有状态的MemGPT风格Agent框架**，将MemGPT的三层记忆、心跳机制、上下文编排等设计完全工程化，以“Server+Client”架构提供服务，Agent状态全量持久化到数据库，无需开发者关注底层存储与序列化。

### 2. 核心优势（区别于LangChain/CrewAI）
| 特性                | Letta框架                          | 传统Agent框架                      |
|---------------------|-----------------------------------|-----------------------------------|
| 状态管理            | 自动持久化（数据库存储）           | 内存变量存储，进程关闭即丢失       |
| 跨会话能力          | 天生支持（重启/跨客户端共享状态）  | 需手动实现持久化逻辑               |
| MemGPT架构适配      | 原生支持三层记忆、心跳机制         | 需手动搭建记忆系统                 |
| 部署形态            | 独立Server（本地/云端），支持多客户端 | 脚本式运行，单进程单实例           |
| 记忆工具            | 内置全套MemGPT记忆管理工具         | 需手动定义工具                     |

## 二、环境搭建
### 1. 安装依赖与启动Server
```bash
# 安装Letta框架（含Server+核心依赖）
pip install letta
# 启动本地Letta Server（默认监听http://localhost:8283）
letta server
# 安装客户端（用于与Server交互）
pip install letta-client
```
### 2. 环境配置
需提前设置`OPENAI_API_KEY`环境变量（Letta默认使用OpenAI模型，支持自定义模型）。

## 三、核心实操：搭建完整记忆Agent
### 1. 连接Server并创建Agent
通过Client API创建Agent，初始化核心记忆（`human`用户信息/`persona`Agent人设）、模型与嵌入模型：
```python
from letta_client import Letta

# 连接本地Letta Server
client = Letta(base_url="http://localhost:8283")

# 创建MemGPT风格Agent
agent_state = client.agents.create(
    name="my_first_agent",
    # 初始化核心记忆块（支持多区段，可设字符上限）
    memory_blocks=[
        {
            "label": "human",  # 存储用户信息
            "value": "用户名字:南哥",
            "limit": 10000  # 该记忆块的字符上限
        },
        {
            "label": "persona",  # 存储Agent人设
            "value": "你是一个友好的AI助手，说话简洁有趣",
            "limit": 5000
        }
    ],
    model="openai/gpt-4o-mini-2024-07-18",  # 推理模型
    embedding="openai/text-embedding-3-small"  # 归档记忆搜索用嵌入模型
)

print(f"Agent创建成功！ID: {agent_state.id}")
```

### 2. 与Agent对话（支持结构化消息输出）
Letta返回结构化消息列表，包含Agent的思考过程、工具调用、回复内容，可直观看到Agent内部逻辑：
```python
# 辅助函数：格式化输出消息
def print_message(message):
    if message.message_type == "reasoning_message":
        print(f"🧠 思考: {message.reasoning}")
    elif message.message_type == "assistant_message":
        print(f"🤖 回复: {message.content}")
    elif message.message_type == "tool_call_message":
        print(f"🔧 工具调用: {message.tool_call.name}")
        print(f" 参数: {message.tool_call.arguments}")
    elif message.message_type == "tool_return_message":
        print(f"🔧 工具返回: {message.tool_return}")
    elif message.message_type == "user_message":
        print(f"👤 用户: {message.content}")

# 发送用户消息
response = client.agents.messages.create(
    agent_id=agent_state.id,
    messages=[{"role": "user", "content": "在吗?最近怎么样?"}]
)

# 打印对话流程
for message in response.messages:
    print_message(message)

# 查看消耗与步数
print(f"\n输入token: {response.usage.prompt_tokens}")
print(f"输出token: {response.usage.completion_tokens}")
print(f"Agent步数: {response.usage.step_count}")
```
#### 输出示例
```
🧠 思考: 用户打了个招呼，语气挺随意的｡我看看记忆里有什么信息...用户叫南哥｡用轻松的口吻回复吧｡
🤖 回复: 在呢南哥!一切都挺好的，你呢?有什么想聊的?

输入token: 1847
输出token: 89
Agent步数: 1
```

### 3. 查看Agent内部状态（透明化管理）
Letta支持直接查询Agent的核心记忆、工具列表、对话历史、归档记忆，便于调试与监控：
```python
# 1. 查看系统提示词（Letta自动生成，含MemGPT记忆管理逻辑）
print("系统提示词（前500字符）:", agent_state.system[:500])

# 2. 查看内置记忆管理工具（MemGPT标准工具集）
print("内置工具列表:", [t.name for t in agent_state.tools])

# 3. 查看核心记忆
print("\n核心记忆:")
for block in agent_state.memory:
    print(f"[{block.label}] (limit: {block.limit} chars)")
    print(f" 内容: {block.value}")

# 4. 查看对话历史
print("\n对话历史:")
for message in client.agents.messages.list(agent_id=agent_state.id):
    print_message(message)

# 5. 查看归档记忆（初始为空）
passages = client.agents.passages.list(agent_id=agent_state.id)
print(f"\n归档记忆条数: {len(passages)}")
for p in passages:
    print(f" - {p.text[:50]}...")
```
#### 内置工具列表（MemGPT标准工具）
```
['archival_memory_insert', 'archival_memory_search', 'conversation_search', 'core_memory_append', 'core_memory_replace', 'send_message']
```

### 4. 核心记忆自编辑（Agent自主更新记忆）
测试Agent是否能自主识别信息变化，调用`core_memory_replace`工具更新核心记忆：
```python
# 告知Agent名字变更
response = client.agents.messages.create(
    agent_id=agent_state.id,
    messages=[{"role": "user", "content": "其实我的名字叫小明，之前写错了"}]
)

# 打印更新流程
for message in response.messages:
    print_message(message)

# 验证核心记忆是否更新
human_block = client.agents.blocks.retrieve(
    agent_id=agent_state.id,
    block_label="human"
)
print(f"\n更新后的核心记忆: {human_block.value}")
```
#### 输出示例
```
🧠 思考: 用户说名字其实是小明，不是南哥｡我需要更新核心记忆里的信息｡
🔧 工具调用: core_memory_replace
 参数: {"label":"human","old_content":"用户名字:南哥","new_content":"用户名字:小明","request_heartbeat":true}
🔧 工具返回: 记忆已更新
🧠 思考: 好的，记忆已经更新了｡现在该回复用户了｡
🤖 回复: 好的小明，已经记下了!以后就叫你小明啦｡

更新后的核心记忆: 用户名字:小明
```

### 5. 归档记忆：存储非核心重要信息
归档记忆用于存储“无需常驻上下文、偶尔需要检索”的信息，支持Agent自主存储与开发者直接写入：
#### 方法1：Agent自主存入归档记忆
```python
# 让Agent存储项目技术栈信息到归档记忆
response = client.agents.messages.create(
    agent_id=agent_state.id,
    messages=[
        {"role": "user", "content": "帮我记一下:我的项目用的是Python 3.11 + FastAPI + PostgreSQL，部署在AWS上。把这个存到归档记忆里。"}
    ]
)

for message in response.messages:
    print_message(message)
```

#### 方法2：开发者直接写入归档记忆（预置知识库）
```python
# 直接插入公司规定、团队信息到归档记忆
client.agents.passages.create(
    agent_id=agent_state.id,
    text="公司规定:每年有15天带薪年假，工作满3年后增加到20天｡"
)
client.agents.passages.create(
    agent_id=agent_state.id,
    text="团队成员:张三(后端)､李四(前端)､王五(产品)"
)

# 验证归档记忆
passages = client.agents.passages.list(agent_id=agent_state.id)
print(f"\n归档记忆条数: {len(passages)}")
for p in passages:
    print(f" - {p.text[:50]}...")
```

### 6. 归档记忆搜索：Agent自主检索
测试Agent是否能自主判断并检索归档记忆中的信息：
```python
# 询问团队成员信息（存储在归档记忆中）
response = client.agents.messages.create(
    agent_id=agent_state.id,
    messages=[{"role": "user", "content": "我们团队都有谁来着?帮我查查"}]
)

for message in response.messages:
    print_message(message)
```
#### 输出示例
```
🧠 思考: 用户在问团队成员的信息｡核心记忆里没有这个信息，但归档记忆里可能有｡搜一下｡
🔧 工具调用: archival_memory_search
 参数: {"query":"团队成员","request_heartbeat":true}
🔧 工具返回: [结果] 团队成员:张三(后端)､李四(前端)､王五(产品)
🧠 思考: 找到了!归档记忆里有团队信息｡
🤖 回复: 你们团队有三个人:张三负责后端､李四负责前端､王五负责产品｡
```

### 7. 完整场景演示：多轮对话+记忆协作
模拟项目管理场景，验证核心记忆与归档记忆的协同工作：
```python
# 1. 创建项目管理助手Agent
project_agent = client.agents.create(
    name="project_agent",
    memory_blocks=[
        {"label": "human", "value": "(暂无用户信息)", "limit": 5000},
        {"label": "persona", "value": "你是一个项目管理助手，帮用户跟踪项目进度"}
    ],
    model="openai/gpt-4o-mini-2024-07-18",
    embedding="openai/text-embedding-3-small"
)

# 2. 对话函数封装
def chat(agent_id, text):
    response = client.agents.messages.create(
        agent_id=agent_id,
        messages=[{"role": "user", "content": text}]
    )
    print(f"\n--- 用户: {text} ---")
    for msg in response.messages:
        print_message(msg)
    print(f"[steps: {response.usage.step_count}]")

# 3. 多轮对话测试
chat(project_agent.id, "我叫南哥，在做一个电商Agent项目")  # 自我介绍→核心记忆
chat(project_agent.id, "项目分三个模块:商品推荐Agent､客服Agent､订单管理Agent。目前商品推荐做了一半，其他两个还没开始。技术栈用的LangGraph + GPT-4o。")  # 补充详情→核心记忆+归档记忆
chat(project_agent.id, "我之前跟你说过项目的进度，帮我回忆一下各模块的状态?")  # 检索进度→归档记忆搜索
```
#### 关键输出（第三轮对话）
```
--- 用户: 帮我回忆一下各模块的状态? ---
🧠 思考: 用户在问项目进度｡核心记忆里只有概要，详细的在归档里｡搜一下｡
🔧 工具调用: archival_memory_search
 参数: {"query":"项目模块进度","request_heartbeat":true}
🔧 工具返回: 电商Agent项目详情:- 商品推荐Agent:进行中，完成约50%...
🧠 思考: 找到了｡整理一下告诉用户｡
🤖 回复: 根据你之前说的:
商品推荐Agent:做了大概一半
客服Agent:还没开始
订单管理Agent:还没开始
技术栈是LangGraph + GPT-4o｡
有新进展了吗?
[steps: 2]
```

## 四、实际使用注意事项（避坑指南）
1. **核心记忆字符上限（limit）**：默认5000字符，建议设为10000-20000，避免频繁溢出到归档记忆导致额外token消耗；
2. **系统提示词不随意修改**：Letta默认提示词包含MemGPT记忆管理核心逻辑，修改可能导致Agent不主动保存/搜索记忆，需定制行为可通过`persona`记忆块实现；
3. **嵌入模型影响搜索质量**：归档记忆依赖向量搜索，`text-embedding-3-small`够用，高准确率场景可换更大模型（如`text-embedding-3-large`）；
4. **Agent步数与延迟平衡**：每步对应一次LLM调用，步数越多延迟越高，对延迟敏感场景可选用更快的模型（如GPT-4o-mini）；
5. **清理冗余Agent**：开发调试时创建的Agent需手动删除，避免占用数据库资源：
```python
# 删除Agent
client.agents.delete(agent_id=project_agent.id)
```

## 五、Letta版与手搓版Agent对比
| 特性                | 手搓版（基础实现）                | Letta版（框架实现）                |
|---------------------|-----------------------------------|-----------------------------------|
| 核心记忆            | Python字典（内存存储）            | 数据库持久化Memory Block          |
| 归档记忆            | 无                                | 向量数据库+语义搜索               |
| 回忆记忆            | 无                                | 自动摘要+对话历史持久化           |
| 心跳机制            | 简单while循环                     | `request_heartbeat`参数精准控制    |
| 内心独白            | 无                                | `reasoning_message`结构化输出      |
| 状态持久化          | 无（进程关闭丢失）                | 全量状态数据库存储                |
| 上下文编排          | 手动拼接字符串                    | 框架自动按MemGPT规范编排          |
| 对话历史管理        | 无限增长，无溢出处理              | 自动摘要+溢出到回忆记忆           |

## 六、核心总结
1. Letta框架是MemGPT架构的工程化实现，无需手动搭建三层记忆、心跳机制，开箱即用生产级记忆Agent；
2. 核心优势是**状态持久化**与**MemGPT原生适配**，支持跨会话/跨客户端共享Agent状态，大幅降低开发成本；
3. 关键功能覆盖核心记忆自编辑、归档记忆存取与搜索、结构化对话流程，完全满足多轮对话、长期协作场景；
4. 适用场景：私人助理、项目管理助手、客服机器人等需长期记忆、自主决策的Agent产品；
5. 开发建议：优先使用框架内置工具与提示词，聚焦业务逻辑（如`persona`人设、归档记忆预置），无需重复造MemGPT架构轮子。

参考资料：
- Letta官方文档：https://docs.letta.com
- Letta开源仓库：https://github.com/letta-ai/letta当前文件内容过长，豆包只阅读了前 86%。