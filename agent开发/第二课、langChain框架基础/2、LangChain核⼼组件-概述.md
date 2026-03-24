# LangChain核心组件-概述核心笔记
本文聚焦LangChain开发的**七大核心组件**（Model、Tools、System Prompt、Streaming、Memory、Middleware、Structured Output），详解各组件的**静态/动态配置方式**与**扩展能力**，是实现灵活、高可用Agent开发的关键，所有配置均支持通过**中间件（Middleware）** 实现运行时动态调整。

## 一、Model：Agent的推理引擎
模型是Agent的核心推理层，支持**静态配置**（固定不变）和**动态切换**（按场景适配），可设置温度、最大token、超时时间等核心参数。
### 1. 静态模型（最常用）
创建Agent时一次性配置，全程使用同一模型，支持直接传模型标识或自定义配置后的模型实例。
```python
# 简洁写法 | 自定义配置写法
agent = create_agent("openai:gpt-5", tools=tools)
model = ChatOpenAI(model="gpt-5", temperature=0.1, max_tokens=1000)
agent = create_agent(model, tools=tools)
```
### 2. 动态模型（场景化适配）
通过`@wrap_model_call`装饰器创建中间件，**根据对话状态动态选择模型**（如长对话用高级模型、短对话用轻量模型），兼顾效果与成本。
- 核心逻辑：基于请求中的对话状态（如消息数量、任务复杂度）判断，重写模型配置后执行。

## 二、Tools：Agent的手脚
工具是Agent与外部交互的载体，支持**静态定义**、**动态调整**和**自定义错误处理**，**docstring是模型理解工具用途的关键**，必须清晰描述功能。
### 1. 静态工具（基础用法）
创建Agent时预先定义，全程可用，通过`@tool`装饰器实现，直接加入工具集即可。
### 2. 动态工具（两种实现方式）
#### （1）预注册工具过滤
已知所有可能的工具，**按运行时状态（如用户权限）动态筛选可用工具**（如管理员用全量工具、普通用户用只读工具），通过`wrap_model_call`重写工具集。
#### （2）运行时工具注册
未知所有工具，**从外部（MCP服务器/插件商店）动态注入工具**，需实现两个中间件钩子：
- `wrap_model_call`：将动态工具添加到请求的工具集中；
- `wrap_tool_call`：处理动态工具的实际执行逻辑。
### 3. 工具错误处理
通过`@wrap_tool_call`装饰器自定义异常处理，工具执行失败时返回**自定义ToolMessage**，让模型能识别错误并做出响应（而非直接崩溃）。

## 三、System Prompt：Agent的角色与行为定义
用于定义Agent的**角色、回答风格、规则**，支持**静态配置**和**动态生成**，可传字符串或`SystemMessage`对象（更灵活）。
### 1. 静态System Prompt
创建Agent时直接指定，适用于固定角色的Agent（如旅行顾问、技术助手）。
```python
agent = create_agent(model, tools, system_prompt="你是专业旅行顾问，中文简洁回答问题。")
```
### 2. 高级静态配置（`SystemMessage`）
支持**提供商专属功能**（如Anthropic的`cache_control`缓存），降低重复请求的延迟和成本，适合需传入大段固定内容（如书籍全文）的场景。
### 3. 动态System Prompt
通过`@dynamic_prompt`装饰器创建中间件，**根据运行时上下文（如用户角色）生成个性化提示**（如专家用户给技术细节、新手用户用通俗语言）。

## 四、Streaming：Agent的实时响应输出
实现Agent的**流式输出**，实时返回响应内容和工具调用过程，提升用户体验，核心用`agent.stream()`方法，指定`stream_mode="values"`。
- 输出效果：可实时打印「工具调用行为」和「逐步生成的回答内容」，而非等待全部结果。
```python
for chunk in agent.stream(请求参数, stream_mode="values"):
    latest_msg = chunk["messages"][-1]
    if latest_msg.tool_calls: # 打印工具调用
        print(f"调用工具: {[tc['name'] for tc in latest_msg.tool_calls]}")
    if latest_msg.content: # 打印实时回答
        print(f"Agent: {latest_msg.content}")
```

## 五、Memory：Agent的状态与上下文管理
默认通过`message state`维护对话历史，支持**自定义状态schema**，实现**非对话信息的持久化记忆**（如机票预订的目的地、出发日期），解决Agent“丢数据”问题。
### 1. 核心使用场景
需多轮收集信息的任务（如订机票、填表单），通过自定义状态存储关键信息，避免重复询问用户。
### 2. 实现步骤
1. 定义**自定义状态类**（继承`AgentState`），声明需要记忆的字段；
2. 定义工具，让模型能将信息**写入**自定义状态；
3. 创建中间件，在每轮推理前**将当前状态拼接至Prompt**，让模型感知已收集的信息；
4. 外部系统可将状态持久化，实现跨会话记忆。

## 六、Middleware：Agent的扩展核心
中间件是LangChain**最核心的扩展能力**，可在Agent执行的**任意阶段**插入自定义逻辑，实现模型、工具、提示、状态的动态调整，核心装饰器/钩子：
| 中间件钩子/装饰器 | 作用阶段 | 核心功能 |
|--------------------|----------|----------|
| `@wrap_model_call` | 模型调用前 | 动态切换模型、过滤/添加工具 |
| `@wrap_tool_call` | 工具调用前/后 | 处理动态工具执行、自定义错误 |
| `@dynamic_prompt`  | 系统提示生成时 | 动态生成System Prompt |
| `@before_model`    | 模型推理前 | 预处理对话状态（如消息裁剪） |
### 经典示例：消息裁剪
通过`@before_model`中间件，只保留最近N条对话消息，**控制上下文长度**，减少token消耗并提升推理效率。
```python
@before_model
def trim_messages(state, runtime):
    if len(state["messages"]) > 10:
        return {"messages": state["messages"][-10:]} # 只保留最近10条
```

## 七、Structured Output：Agent的结构化结果输出
让Agent返回**强类型的结构化数据**（如Pydantic模型），而非纯文本，方便后续程序处理，支持两种实现策略。
### 1. ToolStrategy（通用型）
基于**工具调用能力**生成结构化输出，**适配所有支持工具调用的模型**，兼容性强，无模型厂商限制。
### 2. ProviderStrategy（原生型）
使用模型厂商**原生的结构化输出能力**（如GPT-4o的JSON模式），**可靠性更高**，但依赖模型本身的支持，兼容性稍弱。
### 核心实现步骤
1. 定义Pydantic模型，声明结构化数据的字段；
2. 创建Agent时指定`response_format`为对应策略；
3. 调用Agent后，从`result["structured_response"]`获取强类型结果。

## 核心总结
LangChain的七大核心组件围绕**“静态配置够用，动态扩展灵活”** 设计，核心亮点：
1. **无侵入式扩展**：所有动态调整均通过**中间件**实现，无需修改Agent核心逻辑；
2. **分层解耦**：模型、工具、提示、记忆等组件相互独立，可单独配置和扩展；
3. **兼顾易用性与高级性**：基础开发用静态配置快速落地，复杂场景用动态配置适配需求；
4. **标准化与兼容性**：结构化输出、流式响应等功能提供通用方案，同时支持厂商原生能力，兼顾兼容与性能。

所有组件的最终目标是让开发者能**快速构建适配复杂业务场景的Agent**，从“固定逻辑的智能体”升级为“可动态调整、可扩展、高可用的智能系统”。