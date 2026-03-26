# 第五课：Agentic RAG——让 Agent 自己决定什么时候搜、搜什么 内容总结

> 对应原文 PDF：`D:\notebook\agent开发\agent专项强化记忆\第五课：Agentic RAG——让 Agent 自己决定什么时候搜、搜什么.pdf`
> 说明：在不改变原文含义的前提下做精简总结；代码示例保持完整，不删减。

本课核心为基于Letta框架实现**Agentic RAG**，解决Agent接入外部数据的问题，区别于传统硬编码RAG，让Agent自主决策搜索时机、关键词及结果使用方式，同时实现**静态文档批量导入归档记忆**和**自定义工具连接动态外部数据库**两种外部数据接入方案。

## 一、Agentic RAG与传统RAG的核心区别
传统RAG由开发者硬编码搜索逻辑，Agentic RAG将搜索作为Agent的工具，由LLM推理自主决策，核心差异体现在三点：
| 对比维度 | 传统RAG | Agentic RAG |
| ---- | ---- | ---- |
| 搜索时机 | 每次提问强制搜索，无判断 | Agent自主判断是否需要搜索，无需求则直接回复 |
| 搜索关键词 | 直接使用用户原始问题 | Agent自主改写/优化关键词，按需调整 |
| 结果使用 | 所有结果直接塞入prompt，无筛选 | Agent查看结果后判断是否足够，不足则换关键词重搜，还可将关键信息存入核心记忆 |

**执行流程差异**
- 传统RAG：用户提问→代码强制搜向量库→结果塞prompt→LLM回答
- Agentic RAG：用户提问→Agent思考是否搜索→需要则调用`archival_memory_search`→查看结果→判断是否足够→足够则回复，不足则重搜

Letta中Agent原生自带`archival_memory_search`工具，只需向归档记忆中导入数据，即可实现Agentic RAG能力。

## 二、用DataSource将PDF批量导入Agent归档记忆
Letta的**DataSource**专为静态数据（文档、手册、知识库）设计，可将文件自动切片、向量化后挂载到Agent，数据同步至Agent归档记忆，实现Agent自主语义搜索。核心流程：创建Source→上传文件→自动切片+向量化→挂载到Agent→Agent可搜索。

### 2.1 核心操作代码
```python
# 1. 初始化客户端
from letta_client import Letta
client = Letta(base_url="http://localhost:8283")

# 2. 创建Source（指定embedding模型，一个Source仅能使用一种）
source = client.sources.create(
    name="employee_handbook",
    embedding="openai/text-embedding-3-small"
)
print(f"Source ID: {source.id}")

# 3. 上传PDF文件（异步操作，返回job对象）
job = client.sources.files.upload(
    source_id=source.id,
    file=open("handbook.pdf", "rb")
)
print(f"任务状态: {job.status}")

# 4. 轮询查看任务状态，等待切片+向量化完成
import time
while job.status != 'completed':
    job = client.jobs.retrieve(job.id)
    print(f"状态: {job.status}")
    time.sleep(1)
print("上传完成!")
print(f"切片数: {job.metadata.get('num_passages')}")
print(f"文档数: {job.metadata.get('num_documents')}")

# 5. 查看切片后的文本内容
passages = client.sources.passages.list(source_id=source.id)
print(f"总共 {len(passages)} 个切片")
for p in passages[:2]:
    print(f"\n--- 切片 ---")
    print(p.text[:100] + "...")

# 6. 创建Agent（embedding模型必须与Source一致，否则向量搜索失效）
agent_state = client.agents.create(
    memory_blocks=[
        {
            "label": "human",
            "value": "用户名字:南哥"
        },
        {
            "label": "persona",
            "value": "你是一个友好的助手,帮助用户查询公司规章制度"
        }
    ],
    model="openai/gpt-4o-mini-2024-07-18",
    embedding="openai/text-embedding-3-small" # 与Source的embedding一致
)

# 7. 将Source挂载到Agent，数据同步至归档记忆
client.agents.sources.attach(
    agent_id=agent_state.id,
    source_id=source.id
)

# 8. 验证挂载结果和归档记忆条数
sources = client.agents.sources.list(agent_id=agent_state.id)
print(f"挂载的数据源: {[s.name for s in sources]}")
passages = client.agents.passages.list(agent_id=agent_state.id)
print(f"归档记忆条数: {len(passages)}")

# 9. 测试Agent的自主搜索能力
response = client.agents.messages.create(
    agent_id=agent_state.id,
    messages=[
        {
            "role": "user",
            "content": "我们公司的年假政策是什么?帮我查一下｡"
        }
    ]
)
for message in response.messages:
    print_message(message)
```

### 2.2 关键注意点
1. Source的embedding模型需与挂载的Agent一致，否则向量维度不匹配，搜索失效；
2. 文件上传为异步操作，需轮询job状态，等待切片和向量化完成；
3. 挂载后Source的切片会**复制**到Agent归档记忆，Agent通过原生工具实现自主语义搜索。

## 三、用自定义工具连接外部动态数据库
针对**动态数据**（数据库、外部API，数据实时更新），通过编写**无状态自定义工具**实现Agent按需查询，工具接收参数、执行查询并返回结果，由Agent自主判断是否调用。

### 3.1 核心操作代码
```python
# 1. 定义查询外部数据库的无状态工具（字典模拟实际数据库，可替换为MySQL/MongoDB/API调用）
def query_birthday_db(name: str):
    """
    This tool queries an external database to
    lookup the birthday of someone given their name.

    Args:
        name (str): The name to look up

    Returns:
        birthday (str): The birthday in mm-dd-yyyy format
    """
    my_fake_data = {
        "bob": "03-06-1997",
        "sarah": "07-06-1993"
    }
    name = name.lower()
    if name not in my_fake_data:
        return None
    else:
        return my_fake_data[name]

# 2. 注册自定义工具
birthday_tool = client.tools.upsert_from_function(func=query_birthday_db)

# 3. 创建Agent，关联工具（persona中可添加工具提示，帮助Agent理解使用场景）
agent_state = client.agents.create(
    memory_blocks=[
        {
            "label": "human",
            "value": "用户名字:南哥"
        },
        {
            "label": "persona",
            "value": "你有一个生日数据库工具,可以帮用户查询生日信息｡"
        }
    ],
    model="openai/gpt-4o-mini-2024-07-18",
    embedding="openai/text-embedding-3-small",
    tool_ids=[birthday_tool.id]
)

# 4. 测试工具调用（存在数据）
response_stream = client.agents.messages.create_stream(
    agent_id=agent_state.id,
    messages=[
        {
            "role": "user",
            "content": "帮我查一下 Sarah 的生日是几号?"
        }
    ]
)
for chunk in response_stream:
    print_message(chunk)

# 5. 测试工具调用（无数据）
response_stream = client.agents.messages.create_stream(
    agent_id=agent_state.id,
    messages=[
        {
            "role": "user",
            "content": "小明的生日是什么时候?"
        }
    ]
)
for chunk in response_stream:
    print_message(chunk)

# 6. 实际场景：安全传递数据库密钥（通过环境变量，LLM不可见）
agent_state = client.agents.create(
    memory_blocks=[
        {
            "label": "human",
            "value": "用户名字:南哥"
        },
        {
            "label": "persona",
            "value": "你有一个生日数据库工具,可以帮用户查询生日信息｡"
        }
    ],
    model="openai/gpt-4o-mini-2024-07-18",
    embedding="openai/text-embedding-3-small",
    tool_ids=[birthday_tool.id],
    # 安全传递数据库配置，工具内通过os.environ读取
    tool_exec_environment_variables={
        "DB_HOST": "your-db-host.com",
        "DB_KEY": "your-secret-key"
    }
)
```

### 3.2 关键注意点
1. 该工具为**无状态工具**，无需声明`agent_state: "AgentState"`，仅接收业务参数并返回结果；
2. 实际场景中数据库密钥/API Key通过`tool_exec_environment_variables`传递，仅工具执行时可用，LLM无法查看，保证安全；
3. Agent可识别工具返回的`None`，并自然语言告知用户未查到数据。

## 四、DataSource与自定义工具的选型对比
两种外部数据接入方式可单独使用，也可同时搭配（如客服Agent：挂载产品手册+查询订单数据库），核心选型依据数据特性：
| 特性 | Data Source（文档导入） | 自定义工具（外部查询） |
| ---- | ---- | ---- |
| 适合场景 | 静态文档、知识库、产品手册 | 动态数据库、外部API、实时更新数据 |
| 数据更新方式 | 需重新上传文件+挂载，无法实时更新 | 实时查询，数据始终为最新状态 |
| 搜索/查询方式 | 向量语义搜索，适配自然语言查询 | 精确查询，逻辑由开发者自定义 |
| 数据存放位置 | 复制到Agent的归档记忆中 | 数据保存在外部，Agent按需查询 |
| 适合数据量 | 中等（几百页文档） | 无限制（数据存储在外部系统） |

## 五、Agent记忆系统系列课程全回顾
五课内容从基础理论到工程实战，逐步实现Agent从“无记忆”到“自主管理记忆+接入外部数据”的能力，完整技术路径：
1. 第一课：手搓自编辑记忆Agent，理解**记忆是可编辑的上下文**，基于tool calling实现基础记忆功能；
2. 第二课：拆解MemGPT完整架构，掌握**核心/回忆/归档三层记忆体系**，理解心跳机制、内心独白、上下文编排；
3. 第三课：基于Letta框架搭建MemGPT Agent，跑通**核心记忆编辑、归档记忆存取**的基础操作；
4. 第四课：实现**自定义记忆结构+有状态工具**，打破默认human/persona限制，实战任务队列记忆管理；
5. 第五课：落地**Agentic RAG**，掌握两种外部数据接入方案——DataSource导入静态文档、自定义工具连接动态数据库，让Agent自主决策搜索行为。

通过五课学习，可基于Letta框架开发适配实际业务场景的Agent，实现记忆自定义、外部数据灵活接入，让Agent从“玩具”升级为可落地的“产品级工具”。
