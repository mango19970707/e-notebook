# LangChain核心组件-Message核心笔记
Message 是 LangChain 中对话管理的基础抽象。它把“谁说的、说了什么、附带什么信息”标准化，解决纯字符串输入无法表达角色、上下文、工具回执的问题，并提供跨模型统一格式，便于同一套代码适配不同模型。

## 一、Message核心认知
### 1. 本质定义
Message 是对话中的标准消息单元，用于承载来源、内容和元信息，支撑多轮对话、工具调用、多模态输入等能力。

### 2. 三大组成
| 组成部分 | 核心作用 | 示例 |
|----------|----------|------|
| 角色（Role） | 标识消息发送方 | 用户、AI、系统、工具 |
| 内容（Content） | 消息主体 | 文本、图片、PDF、音频、视频等 |
| 元数据（Metadata） | 辅助信息 | 消息ID、token用量、模型名、工具调用ID |

### 3. 核心优势
**格式统一、适配简单**：无论接 OpenAI、Gemini 还是开源模型，Message 结构保持一致，减少适配成本。

## 二、LangChain四大核心消息角色
四类角色共同构成 Agent 交互闭环，角色分工清晰是工具调用和多轮对话稳定运行的前提。

### 1. SystemMessage：规则与人设
- **作用**：在对话开始前定义角色、语气、边界、输出要求；
- **特点**：属于全局指令，不是普通问答内容；
- **影响**：是否设置高质量 SystemMessage，通常会明显影响回答质量。

### 2. HumanMessage：用户输入
- **作用**：承载用户问题与指令；
- **特点**：支持多模态（文本/图像/音频/PDF/视频等）；
- **定位**：模型主要响应对象。

### 3. AIMessage：模型输出
- **作用**：承载模型回复与工具调用计划；
- **常用字段**：
  - `content`：文本回复；
  - `usage_metadata`：token 用量（成本监控）；
  - `response_metadata`：模型名、停止原因等；
  - `tool_calls`：工具调用名称、参数、调用ID；
  - `id`：消息唯一标识；
- **扩展**：可手动构造 AIMessage 注入历史，用于定制流程或调试。

### 4. ToolMessage：工具回执
- **作用**：承载工具执行结果，回传给模型继续推理；
- **闭环流程**：HumanMessage → AIMessage（发起工具调用）→ 工具执行 → ToolMessage（返回结果）→ AI 最终回复；
- **关键约束**：`tool_call_id` 必须与 AIMessage 里的调用 ID 一致，否则调用链会断。

## 三、多轮对话：消息列表的有序管理
### 1. 核心方式
将不同角色的 Message 按时间顺序组成列表，传给 `model.invoke()`。模型依据**整段历史**生成回复，而不是只看最后一句。

### 2. 注意事项
1. **顺序即语义**：顺序错，理解就会错；
2. **可续写**：若末尾是未完成的 AIMessage，模型会沿该上下文继续生成；
3. **持续追加**：多轮场景下不断追加 Human/AI/Tool 消息即可。

### 3. 基础示例
```python
messages = [
    SystemMessage("你是一个金融专家"),
    HumanMessage("写一篇股市分析"),
    AIMessage("股市是一个买卖上市公司股票的场所..."), # 模型的半条回复
    HumanMessage("继续分析未来走势") # 新的用户提问
]
response = model.invoke(messages) # 模型基于全部上下文继续回复
```

## 四、多模态支持：Message的多格式承载
Message 支持文本、PDF、音频、视频、图片等多模态。实现方式是把 `content` 设为列表，每个元素包含 `type`、具体内容（url/base64/file_id）和可选 `mime_type`。

### 1. 典型场景
- **PDF 输入**：支持 URL、base64、厂商文件 ID；
- **视频输入**：支持 base64、厂商文件 ID。

### 2. 关键特点
1. **统一格式**：都按“类型+内容”描述；
2. **兼容性好**：可直接使用厂商原生文件 ID；
3. **可混合对话**：多模态消息可与纯文本消息混合。

## 五、核心开发要点总结
1. Agent 开发应使用 Message 封装输入输出，不再依赖纯字符串；
2. 四类角色分工明确，避免角色混用；
3. 工具调用链路里 `tool_call_id` 必须严格匹配；
4. 多轮对话靠“按顺序维护并追加消息列表”；
5. 通过 `usage_metadata` 做 token 与成本监控；
6. 多模态输入本质是 `content` 列表化。

Message 看起来简单，但它是 Agent、多轮对话、工具调用、多模态能力的共同底座。

## 代码补充（来自 PDF）
用于说明消息角色与对话流程的精简代码骨架。

### 示例 1：SystemMessage + HumanMessage
```python
from langchain.messages import SystemMessage, HumanMessage

messages = [
    SystemMessage("你是一名资深 Python 开发工程师。"),
    HumanMessage("我该如何创建一个 REST API？"),
]
response = model.invoke(messages)
```

### 示例 2：查看 AIMessage 元数据
```python
res = model.invoke("请用 3 个要点解释什么是机器学习")
print(res.content)
print(res.usage_metadata)
print(res.response_metadata)
print(res.tool_calls)
```

### 示例 3：工具调用闭环（AIMessage -> ToolMessage）
```python
from langchain.messages import AIMessage, ToolMessage

ai_message = AIMessage(content="", tool_calls=[{"id": "call_1", "name": "get_weather", "args": {"city": "上海"}}])
tool_message = ToolMessage(content="晴天，26C", tool_call_id="call_1")
final = model.invoke([ai_message, tool_message])
```
