# 从零搭建有记忆的Agent课程总结
本课程围绕Agent记忆系统展开，核心参考MemGPT论文与Letta框架思路，从“大模型无原生记忆”的痛点出发，讲解了记忆系统的设计逻辑，并用不到100行代码实现了基础版“会记事、能编辑记忆”的Agent，同时铺垫了进阶的两层记忆架构，为复杂Agent开发打下基础。

## 一、核心背景：大模型“失忆”的本质
1. **大模型无原生记忆**：每次API调用都是独立的，模型仅能访问当前上下文窗口内的信息，无法主动留存历史对话/信息，所谓“记忆”本质是开发者将历史数据手动塞入上下文。
2. **传统记忆方案的三大问题**：
    - 上下文窗口有限：超过长度后早期信息被截断，导致失忆；
    - 成本线性增长：每次请求携带完整历史记录，token消耗与对话轮次成正比；
    - 模型注意力分散：长上下文下，早期关键信息易被忽略（大海捞针问题）。

## 二、MemGPT核心思路：让LLM自主管理记忆
借鉴操作系统“虚拟内存”机制，核心转变是**从开发者手动管理上下文，转为让LLM自主决定上下文内容**：
1. 赋予Agent记忆管理工具（写入/搜索/更新/编排），让其自主决策“记什么、忘什么、何时调用”；
2. 类比操作系统：LLM如同“操作系统内核”，记忆如同“内存+硬盘”，Agent自主完成数据“换入换出”，让有限上下文窗口“感知无限记忆”。

## 三、Agent与聊天机器人的关键区别
| 维度         | 聊天机器人（Chatbot）                | Agent                              |
|--------------|-------------------------------------|------------------------------------|
| 工作模式     | 单轮LLM调用：用户消息→拼接历史→生成回复 | 多轮LLM调用（Agentic Loop）：用户消息→思考→工具调用→更新状态→生成回复 |
| 记忆管理     | 被动依赖开发者拼接历史              | 主动调用工具编辑/检索记忆          |
| 交互逻辑     | 一对一（一条消息→一个回复）          | 多步循环（一条消息→多次推理→一个回复） |

Agentic Loop（智能体循环）：Agent状态→编译为上下文→LLM推理→更新Agent状态→持续循环，直至完成任务。

## 四、实操：从零实现基础版有记忆的Agent
核心目标：让Agent能自动保存用户新信息，后续对话可复用，分三步实现。

### 1. 环境与依赖
- 依赖：`openai`库（需配置`OPENAI_API_KEY`）；
- 模型：选用`gpt-4o-mini`（支持工具调用）。

### 2. 完整代码实现
```python
import json
from openai import OpenAI

# ========== 初始化 ==========
client = OpenAI()  # 需提前配置OPENAI_API_KEY环境变量
MODEL = "gpt-4o-mini"

# 记忆存储：用字典模拟核心记忆（分用户信息human、Agent人设persona）
agent_memory = {"human": "", "persona": ""}

# 记忆保存工具：往指定记忆区域追加信息
def core_memory_save(section: str, memory: str):
    """保存关于用户或Agent自身的重要信息到长期记忆"""
    if section in agent_memory:
        if agent_memory[section]:
            agent_memory[section] += f"\n{memory}"  # 已有内容则追加
        else:
            agent_memory[section] = memory  # 无内容则直接赋值
    return f"已保存到 [{section}]: {memory}"

# ========== 工具定义（OpenAI Tool Calling格式） ==========
tools = [
    {
        "type": "function",
        "function": {
            "name": "core_memory_save",
            "description": "当从对话中了解到用户或自身的新重要信息时，调用此工具保存到长期记忆",
            "parameters": {
                "type": "object",
                "properties": {
                    "section": {
                        "type": "string",
                        "description": "记忆存储区域：'human'存储用户信息，'persona'存储Agent自身信息",
                        "enum": ["human", "persona"]
                    },
                    "memory": {
                        "type": "string",
                        "description": "需要保存的具体信息（如用户姓名、职业、偏好等）"
                    }
                },
                "required": ["section", "memory"]  # 必传参数
            }
        }
    }
]

# ========== 系统提示词模板（嵌入记忆） ==========
SYSTEM_PROMPT = """你是一个有自主记忆能力的智能助手。

<memory>
{memory}
</memory>

## 行为规范
1. 从对话中获取新的重要信息时，必须先调用core_memory_save工具保存
2. 保存记忆后，再给用户生成自然回复
3. 不重复调用相同工具（避免重复保存同一信息）
4. 优先使用记忆中的信息个性化对话，让用户感受到你记得他的情况
"""

# 编译记忆为字符串，嵌入系统提示词
def compile_memory():
    parts = []
    for section, content in agent_memory.items():
        if content:  # 只保留非空记忆
            parts.append(f"[{section}]: {content}")
    return "\n".join(parts) if parts else "(暂无记忆)"

# ========== Agentic Loop：实现“保存记忆+回复用户”循环 ==========
def agent_step(user_message: str):
    """执行一次完整的Agent交互循环：接收用户消息→处理记忆→生成回复"""
    # 初始化消息列表（系统提示词+用户消息）
    system_prompt = SYSTEM_PROMPT.format(memory=compile_memory())
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_message}
    ]

    while True:
        # 调用LLM，传入工具定义
        response = client.chat.completions.create(
            model=MODEL,
            messages=messages,
            tools=tools
        )
        msg = response.choices[0].message
        messages.append(msg)  # 记录LLM响应，维持对话上下文

        # 若不是工具调用，说明Agent决定直接回复用户，跳出循环
        if response.choices[0].finish_reason != "tool_calls":
            return msg.content

        # 执行工具调用（此处仅处理core_memory_save）
        for tool_call in msg.tool_calls:
            func_name = tool_call.function.name
            args = json.loads(tool_call.function.arguments)
            print(f" [工具调用] {func_name}({args})")

            # 执行记忆保存工具
            if func_name == "core_memory_save":
                tool_result = core_memory_save(**args)
            else:
                tool_result = f"未知工具：{func_name}"

            # 将工具执行结果追加到上下文，让LLM知道保存成功
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": tool_result
            })

            # 更新系统提示词中的记忆（避免后续步骤使用旧记忆）
            updated_system_prompt = SYSTEM_PROMPT.format(memory=compile_memory())
            messages[0] = {"role": "system", "content": updated_system_prompt}

# ========== 测试：验证记忆功能 ==========
if __name__ == "__main__":
    print("=== 第一轮对话 ===")
    reply = agent_step("你好，我叫南哥，是个做AI Agent的开发者。")
    print(f"Agent回复：{reply}")
    print(f"当前记忆：{agent_memory}\n")

    print("=== 第二轮对话 ===")
    reply = agent_step("我叫什么？我是做什么的？")
    print(f"Agent回复：{reply}")
```

### 3. 核心执行流程与效果
#### （1）执行流程
1. 用户发送新信息（如“我叫南哥，是AI Agent开发者”）；
2. Agent调用`core_memory_save`工具，将信息保存到`human`记忆区；
3. 工具执行结果反馈给LLM，Agent生成包含记忆的回复；
4. 后续对话（如“我叫什么？”），Agent从记忆中读取信息并回复。

#### （2）测试输出
```
=== 第一轮对话 ===
 [工具调用] core_memory_save({'section': 'human', 'memory': '名字：南哥，职业：AI Agent开发者'})
Agent回复：你好南哥！作为AI Agent开发者，想必你对智能体技术很有研究～ 有什么想聊的话题或者需要我帮忙的吗？
当前记忆：{'human': '名字：南哥，职业：AI Agent开发者', 'persona': ''}

=== 第二轮对话 ===
Agent回复：你叫南哥呀！你是一名AI Agent开发者，专注于智能体相关的开发工作，对吗？
```

## 五、MemGPT进阶：两层记忆架构
基础版仅实现“核心记忆”，完整MemGPT包含两层记忆，类比操作系统的“内存+硬盘”：

| 记忆层级       | 类比组件 | 存储位置 | 核心特点                     | 适用场景                     |
|----------------|----------|----------|------------------------------|------------------------------|
| 核心记忆（Core Memory） | RAM（内存） | 上下文窗口 | 容量小（几百token）、访问快、时刻可用 | 用户基本信息、Agent人设、当前任务关键信息 |
| 归档记忆（Archival Memory） | 硬盘 | 外部数据库（向量库） | 容量无限、需工具检索、延迟较高 | 大量历史对话、详细背景信息、非关键长期数据 |

### 两层记忆协同逻辑
1. Agent发现重要信息→优先存入核心记忆，非关键信息存入归档记忆；
2. 需要历史信息时→调用搜索工具从归档记忆中检索，加载到上下文窗口；
3. 核心记忆满时→将次要信息迁移到归档记忆，释放上下文空间。

## 六、基础版到生产级的差距
当前实现仅为最小可用版本，生产环境需补充5点能力：
1. 记忆编辑功能：新增`core_memory_replace`（替换）、`core_memory_delete`（删除），处理用户信息更新/纠错；
2. 归档记忆实现：集成向量数据库（如Chroma、Qdrant），支持语义搜索与大量信息存储；
3. 对话历史管理：自动总结长对话并存入归档记忆，仅保留近期对话在上下文；
4. 记忆结构化：用表格、知识图谱等结构化格式存储记忆（比纯文本更高效）；
5. 多Agent共享记忆：设计共享记忆库，支持不同Agent（如客服、技术支持）复用用户信息。

## 七、核心总结
1. Agent记忆的核心是“自主管理”：而非手动拼接历史，让LLM通过工具调用实现记忆的写入、检索、更新，是解决长对话失忆的关键；
2. 基础版代码验证了核心链路：仅需工具定义、Agentic Loop、记忆存储三个核心模块，即可实现“记事”能力；
3. 进阶方向明确：基于MemGPT两层记忆架构，补充归档记忆、结构化存储、多Agent共享等能力，可逐步升级为生产级Agent；
4. 适用场景：客服助手、私人助理、长期协作类Agent（需持续记住用户偏好、任务进度等）。

参考资料：
- MemGPT论文：《MemGPT: Towards LLMs as Operating Systems》（2023）
- Letta开源框架：https://github.com/letta-ai/letta