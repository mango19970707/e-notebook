# 手把手教你用LangChain写ReAct Agent核心笔记
本文从底层原理到生产级实现，拆解ReAct Agent的构建逻辑，对比传统ReAct Prompt方案与现代Tool Calling方案，帮助理解Agent工具调用的核心机制，掌握可落地的Agent开发方法。

## 一、ReAct Agent核心认知
### 1. 本质定义
ReAct（Reasoning + Acting）是Agent的核心工作模式，核心逻辑为**“思考→行动→观察→再思考”** 的循环，直到Agent生成最终答案，本质是“让模型自主决策工具使用，基于工具结果迭代优化”。

### 2. 核心流程（伪代码）
```python
while True:
    # 1. 思考：基于问题和历史记录，决定是否调用工具
    prompt = 构建包含工具/历史的提示词
    response = LLM生成响应
    # 2. 行动：解析响应，执行工具或返回答案
    if 响应包含工具调用:
        result = 执行对应工具
        更新历史记录（思考+行动+结果）
    else:
        return 最终答案
```

## 二、方案一：传统ReAct Prompt实现（理解底层）
通过手动编写Prompt模板引导LLM按固定格式输出，适合理解原理，需处理格式解析、幻觉防控等问题。

### 1. 核心步骤
#### （1）定义工具
用`@tool`装饰器封装原子工具，**docstring是LLM判断工具用途的关键**：
```python
from langchain.tools import tool

@tool
def get_text_length(text: str) -> int:
    """返回文本的字符数（输入为字符串）"""
    return len(text.strip())
```

#### （2）编写ReAct Prompt模板
需明确工具列表、输出格式（Thought/Action/Action Input/Observation），强制LLM按规则输出：
```python
template = """
Answer the following questions as best as you can. You have access to the following tools:
{tools}

Use the following format:
Question: 待回答的问题
Thought: 思考该做什么（必须有）
Action: 工具名（仅从{tool_names}中选）
Action Input: 工具输入参数
Observation: 工具执行结果
...（可重复Thought/Action/Action Input/Observation）
Thought: 我现在知道最终答案了
Final Answer: 原始问题的最终答案

Begin!
Question: {input}
Thought: {agent_scratchpad}
"""
```

#### （3）配置LLM（关键：stop token防幻觉）
添加`stop=["\nObservation"]`，避免LLM编造工具执行结果：
```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model="gpt-3.5-turbo",
    temperature=0,
    stop=["\nObservation"]  # 输出到Action Input后停止，避免伪造结果
)
```

#### （4）组装Agent与解析输出
用PromptTemplate+LLM构建管道，通过`ReActSingleInputOutputParser`解析LLM文本输出：
```python
from langchain.prompts import PromptTemplate
from langchain.agents.output_parsers import ReActSingleInputOutputParser
from langchain.utilities.render_text_description import render_text_description

# 1. 渲染工具描述
tools = [get_text_length]
tool_desc = render_text_description(tools)
tool_names = ", ".join([t.name for t in tools])

# 2. 组装Prompt+LLM
prompt = PromptTemplate.from_template(template).partial(tools=tool_desc, tool_names=tool_names)
agent = prompt | llm

# 3. 解析器：将文本响应转为结构化数据（AgentAction/AgentFinish）
parser = ReActSingleInputOutputParser()
```

#### （5）运行循环（核心）
维护历史记录（`intermediate_steps`），迭代执行“思考→行动→观察”：
```python
from langchain.agents import AgentAction, AgentFinish
from langchain.utilities import format_log_to_str

intermediate_steps = []  # 存储（AgentAction, Observation）
user_query = "dog这个单词有多少个字符？"

while True:
    # 1. 填充历史思考记录
    agent_scratchpad = format_log_to_str(intermediate_steps)
    # 2. 调用Agent生成响应
    response = agent.invoke({"input": user_query, "agent_scratchpad": agent_scratchpad})
    # 3. 解析响应
    parsed = parser.parse(response)
    # 4. 判断是否结束
    if isinstance(parsed, AgentFinish):
        print("最终答案：", parsed.return_values["output"])
        break
    # 5. 执行工具并更新历史
    if isinstance(parsed, AgentAction):
        tool = next(t for t in tools if t.name == parsed.tool)
        observation = tool.run(parsed.tool_input)
        intermediate_steps.append((parsed, observation))
        print(f"执行工具：{tool.name}，结果：{observation}")
```

### 2. 核心痛点
- 稳定性差：依赖LLM严格按格式输出，轻微格式偏差（如多空格）会导致解析失败；
- 易幻觉：需手动添加stop token，否则LLM可能编造Observation；
- 代码繁琐：需手动处理历史记录、格式渲染、工具匹配。

## 三、方案二：现代Tool Calling实现（生产级）
利用模型原生工具调用能力（如OpenAI/Claude的Tool Calling），无需手动编写格式模板，稳定性和效率大幅提升，是当前主流方案。

### 1. 核心优化点
- 模型原生支持：工具描述、参数格式由模型内部处理，无需Prompt引导；
- 结构化输出：LLM直接返回JSON格式的工具调用信息，无需正则解析；
- 支持并行调用：可一次调用多个工具，效率更高；
- 无幻觉风险：工具执行结果由代码传入，模型不会编造。

### 2. 完整实现代码（仅70行）
```python
from dotenv import load_dotenv
from langchain_core.messages import HumanMessage, ToolMessage
from langchain.tools import tool, BaseTool
from langchain_openai import ChatOpenAI
from typing import List

# 1. 加载环境变量（API Key）
load_dotenv()

# 2. 定义工具（与传统方案一致）
@tool
def get_text_length(text: str) -> int:
    """返回文本的字符数（输入为字符串）"""
    text = text.strip("'\"\\n")
    return len(text)

# 3. 工具查找辅助函数
def find_tool_by_name(tools: List[BaseTool], tool_name: str) -> BaseTool:
    for tool in tools:
        if tool.name == tool_name:
            return tool
    raise ValueError(f"未找到工具：{tool_name}")

# 4. 配置LLM并绑定工具
tools = [get_text_length]
llm = ChatOpenAI(temperature=0)
llm_with_tools = llm.bind_tools(tools)  # 一键绑定工具，无需手动渲染

# 5. 运行循环（简化版）
messages = [HumanMessage(content="What is the length of the word: DOG")]

while True:
    # 1. 调用LLM（自动携带工具信息）
    ai_message = llm_with_tools.invoke(messages)
    # 2. 提取工具调用（结构化数据，无需解析）
    tool_calls = getattr(ai_message, "tool_calls", [])
    
    if tool_calls:
        # 3. 执行工具并记录结果
        messages.append(ai_message)  # 保存LLM的工具调用决策
        for tool_call in tool_calls:
            tool_name = tool_call["name"]
            tool_args = tool_call["args"]
            tool_call_id = tool_call["id"]
            # 执行工具
            tool = find_tool_by_name(tools, tool_name)
            observation = tool.invoke(tool_args)
            print(f"执行工具：{tool_name}，结果：{observation}")
            # 用ToolMessage封装结果（需匹配tool_call_id）
            messages.append(ToolMessage(str(observation), tool_call_id=tool_call_id))
    else:
        # 4. 无工具调用→输出最终答案
        print("最终答案：", ai_message.content)
        break
```

### 3. 关键改进细节
#### （1）`bind_tools`替代Prompt模板
自动将工具的名称、参数、docstring按模型要求格式传入，无需手动拼接工具描述，跨模型兼容性强。

#### （2）`messages`列表替代历史记录
直接用消息列表维护完整上下文（用户问题→LLM工具调用→ToolMessage结果），LLM自动理解迭代过程，无需手动格式化`agent_scratchpad`。

#### （3）`tool_calls`结构化输出
LLM返回的`ai_message.tool_calls`是列表，每个元素包含：
- `name`：工具名；
- `args`：参数字典；
- `id`：调用唯一ID（用于多工具并行时匹配结果）；
  无需正则解析，直接通过键值对获取信息。

#### （4）`ToolMessage`绑定调用ID
工具执行结果需封装为`ToolMessage`，并传入`tool_call_id`，确保模型能对应到具体的工具调用请求（支持并行调用场景）。

## 四、两种方案核心对比
| 维度 | ReAct Prompt（传统） | Tool Calling（现代） |
|------|----------------------|----------------------|
| 工具选择依据 | Prompt引导LLM | 模型原生能力 |
| 输出解析方式 | 正则解析文本格式 | 直接读取JSON结构 |
| 稳定性 | 低（依赖格式严格匹配） | 高（模型原生约束） |
| 代码量 | 多（需处理模板/解析/历史） | 少（框架封装核心逻辑） |
| Token消耗 | 高（需传入完整格式模板） | 低（仅传入工具元信息） |
| 并行调用 | 不支持 | 支持 |
| 可解释性 | 高（能看到Thought过程） | 低（推理过程封装在模型内） |
| 适用场景 | 模型不支持Tool Calling、需审计推理过程 | 生产环境、追求稳定性和效率 |

## 五、关键开发技巧与调试
### 1. 调试利器：Callback回调
通过`BaseCallbackHandler`打印LLM的输入（Prompt）和输出（响应），方便定位问题：
```python
from langchain_core.callbacks import BaseCallbackHandler
from langchain_core.outputs import LLMResult

class AgentCallbackHandler(BaseCallbackHandler):
    def on_llm_start(self, serialized, prompts, **kwargs):
        print(f"LLM输入：\n{prompts[0]}\n")
    def on_llm_end(self, response: LLMResult, **kwargs):
        print(f"LLM输出：\n{response.generations[0][0].text}\n")

# 配置LLM时添加回调
llm = ChatOpenAI(temperature=0, callbacks=[AgentCallbackHandler()])
```

### 2. 跨模型切换（LangChain统一抽象）
无需修改业务代码，仅需替换LLM导入和实例化，适配不同厂商模型：
```python
# OpenAI → Anthropic Claude
from langchain_anthropic import ChatAnthropic
llm = ChatAnthropic(model="claude-sonnet-4-20250514")
llm_with_tools = llm.bind_tools(tools)  # 后续逻辑完全不变
```

### 3. 工具设计关键原则
- docstring清晰：明确工具用途、参数格式、返回值，帮助模型精准判断是否调用；
- 原子化设计：一个工具聚焦一个功能（如“查天气”“算长度”），避免复杂逻辑；
- 参数规范：明确参数类型（如str/int），减少模型参数传递错误。

## 六、核心总结
1. ReAct Agent的核心是“思考-行动-观察”循环，两种实现方案分别对应“原理理解”和“生产落地”；
2. 优先选择Tool Calling方案：稳定性高、代码简洁、支持并行，是当前Agent开发的主流；
3. ReAct Prompt仅适用于：模型不支持Tool Calling（如部分开源小模型）、需审计推理过程的场景；
4. 关键认知：Agent开发的核心是“工具设计+上下文管理+决策逻辑”，LangChain已封装大部分底层细节，无需重复造轮子。

是否需要我针对“多工具并行调用”或“开源模型ReAct Prompt适配”提供更详细的代码示例？当前文件内容过长，豆包只阅读了前 48%。