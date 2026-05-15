# Agent 面试题完整指南

## 目录

1. LLM 和 Agent 有什么区别？
2. Agent 和 Workflow 有什么区别？
3. Agent 有哪些工作模式？
4. Function Call 是什么？底层怎么实现？
5. MCP 是什么协议？解决什么问题？
6. Skills 是什么？和 Prompt 有什么区别？
7. Function Call、MCP、Skills 三者区别与协作？
8. A2A 协议是什么？和 MCP 的关系？
9. Agent 的记忆系统怎么设计？
10. Agent 的安全与可靠性如何保障？
11. RAG 和 Agent 是什么关系？
12. 大厂真实面试追问汇总

---

## 1. LLM 和 Agent 有什么区别？

### 核心区别

**LLM：** 无状态的文本生成器，只会"说"不会"做"
**Agent：** LLM + 工具 + 记忆 + 规划，在循环中自主完成目标

### LLM 的四大天花板

1. **只会说不会做** - 能告诉你怎么做，但自己不执行
2. **没有记忆** - 上下文窗口满了就"失忆"
3. **知识截止** - 训练数据有截止日期
4. **不会规划** - 只能线性回答，不会拆解任务

### 实际对比示例

**任务：** 查明天北京天气，如果下雨就取消日历里的跑步计划

| 角色 | 行为 |
|------|------|
| **LLM** | "您可以打开天气 App 查询..." |
| **Agent** | 1. 调用天气 API → 明天中雨<br>2. 调用日历 API → 找到跑步计划<br>3. 删除该计划<br>4. 回复："已为您取消" |

### Agent 的四大模块

1. **LLM（大脑）** - 理解意图、推理判断
2. **规划模块** - 任务拆解、步骤排序
3. **记忆模块** - 短期上下文与长期知识存储
4. **工具模块** - 调用外部 API、数据库等

---

## 2. Agent 和 Workflow 有什么区别？

### 核心区别

**Workflow：** 控制权在代码手里，流程预先定义
**Agent：** 控制权在 LLM 手里，自主决策路径

### 量化对比

| 维度 | Workflow | Agent |
|------|----------|-------|
| 控制者 | 代码/开发者 | LLM |
| Token 消耗 | 低（约1x） | 高（约4-8x） |
| 可预测性 | 高 | 低 |
| 灵活性 | 低 | 高 |
| 适合任务 | 固定流程 | 开放式目标 |
| 调试难度 | 容易 | 困难 |

### 生产实践：混合架构

大多数生产系统采用 **Workflow + Agent 混合架构**：
- Workflow 提供稳定骨架
- Agent 处理异常和复杂情况

---

## 3. Agent 有哪些工作模式？

### 模式一：ReAct（推理 + 行动）

**核心循环：** Thought（思考）→ Action（行动）→ Observation（观察）

**优点：**
- 透明可审计
- 灵活适应
- 通用性强

**缺点：**
- Token 消耗大
- 可能死循环
- 延迟高

**防止死循环的三个方法：**
1. 最大步数限制（通常 15 步）
2. 重复动作检测（连续 3 次相同调用则退出）
3. 超时控制

### 模式二：Plan-and-Execute（先规划再执行）

**核心思路：** 先生成完整计划，再按计划逐步执行

**优势：**
- Token 消耗约 20%（相比 ReAct）
- 逻辑更清晰
- 适合复杂任务

**解决方案：** 加入重新规划检查点，发现偏差时更新计划

### 模式三：Reflection（自我反思）

**核心思路：** Writer Agent 生成 → Reviewer Agent 审查 → 循环迭代

**适用场景：**
- 代码生成
- 法律文书
- 学术论文
- 创意写作

### 模式四：Multi-Agent（多智能体协作）

**架构：**
- **Orchestrator** - 协调分配任务
- **Research Agent** - 搜集资料
- **Coder Agent** - 编写代码
- **Reviewer Agent** - 审查检查

**主流框架：** LangGraph、CrewAI、OpenAI SDK、AutoGen

**重要提醒：** 不要过早引入 Multi-Agent，单 Agent 往往更稳定省钱

---

## 4. Function Call 是什么？底层怎么实现？

### 核心认知

**LLM 自己并不执行函数！** 它只输出"我想调用什么函数、传什么参数"，真正执行的是你的代码。

### Function Call 四步流程

**Step 1：定义工具** - 告诉 LLM 有哪些工具可用

```json
{
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "获取指定城市的实时天气信息",
      "parameters": {
        "type": "object",
        "properties": {
          "city": {"type": "string", "description": "城市名称"}
        },
        "required": ["city"]
      }
    }
  }]
}
```

**Step 2：LLM 生成调用指令** - 输出结构化 JSON

```json
{
  "tool_calls": [{
    "id": "call_abc123",
    "type": "function",
    "function": {
      "name": "get_weather",
      "arguments": "{\"city\": \"北京\"}"
    }
  }]
}
```

**Step 3：应用程序解析执行**

```python
def handle_tool_calls(tool_calls):
    for call in tool_calls:
        func_name = call.function.name
        args = json.loads(call.function.arguments)

        if func_name == "get_weather":
            result = weather_api.get(args["city"])

        return result
```

**Step 4：把结果传回 LLM，生成最终回答**

### Parallel Function Call（并行调用）

GPT-4o 和 Claude 3.5+ 支持一次返回多个工具调用，可并行执行：

**性能提升：**
- 串行：T = T1 + T2 + T3
- 并行：T = max(T1, T2, T3)

---

## 5. MCP 是什么协议？解决什么问题？

### 解决的根本问题：N × M 爆炸

**没有 MCP：**
- 每个 AI 应用要跟每个外部服务单独集成
- 10 个应用 × 20 个工具 = 200 套代码

**有了 MCP：**
- 变成 N + M 问题
- 10 + 20 = 30 套代码

### MCP 架构

1. **MCP Host** - AI 应用（Claude Desktop / Cursor）
2. **MCP Client** - 负责通信的"翻译官"
3. **MCP Server** - 暴露工具能力的服务端

### MCP 提供的三类资源

| 类型 | 说明 |
|------|------|
| **Tools** | 可执行的操作（发消息、查数据） |
| **Resources** | 可读的数据源（文档、代码库） |
| **Prompts** | 预设提示词模板 |

### 工具发现机制

**Agent 可以在运行时动态发现新能力，不需要重新部署代码。**

1. Agent 启动时扫描 MCP Server 列表
2. 发送 `tools/list` 请求
3. 接收工具列表并注入 LLM 上下文
4. 运行时自动匹配可用工具

### 安全性设计

1. **能力声明** - Server 明确声明提供的工具
2. **授权控制** - 敏感操作需人工确认
3. **审计追踪** - 所有调用都有日志

### 底层协议

- **本地通信：** stdio（标准输入输出）
- **远程通信：** HTTP + SSE
- **消息格式：** JSON-RPC 2.0

---

## 6. Skills 是什么？和 Prompt 有什么区别？

### Skills 解决什么问题？

给 Agent 领域专家经验：
- 代码审查标准
- SQL 最佳实践
- 品牌语气要求

### Skills vs System Prompt

|  | System Prompt | Skills |
|--|---------------|--------|
| 作用范围 | 全局，一直生效 | 按需激活 |
| 内容 | 通用行为规范 | 特定领域指导 |
| 激活方式 | 每次都加载 | 场景触发 |
| 可维护性 | 随功能增多变复杂 | 模块化独立 |

### Skill 文件结构

```yaml
---
name: Senior_Code_Reviewer
description: 代码审查技能
triggers:
  - "帮我 review 代码"
  - "code review"
allowed-tools:
  - read_file
  - run_linter
---

# 角色定位
资深后端架构师，10 年经验

# 审查维度
1. 安全性（SQL 注入、越权访问）
2. 性能（N+1 查询、资源泄漏）
3. 代码质量（单一职责、命名规范）

# 输出格式
Markdown 报告：总体评分 + 严重问题 + 优化建议
```

### 激活机制

```
用户输入 → 扫描 Skills → 匹配 triggers →
注入上下文 → 按 Skill 指导执行
```

---

## 7. Function Call、MCP、Skills 三者区别与协作

### 用"新员工入职"类比

- **Function Call** - 打电话的基础能力
- **MCP** - 公司统一的通讯录和电话系统
- **Skills** - 岗位培训手册

### 三维对比

|  | Function Call | MCP | Skills |
|--|---------------|-----|--------|
| 解决的问题 | LLM 如何调用函数 | 工具集成标准化 | 领域知识编码 |
| 运行位置 | 应用程序 | 外部 Server | Agent 上下文 |
| 技术本质 | API 协议 | 通信标准 | 提示词扩展 |
| 外部调用 | 有 | 有 | 无 |

### 一句话总结

**Skills 决定「怎么想」→ MCP 决定「用什么」→ Function Call 决定「怎么调」**

### 完整协作流程

用户："帮我审查 agent.py"

1. **Skills 匹配** → 加载 Code_Review Skill
2. **Agent 规划** → "需要读文件、运行 linter"
3. **MCP 工具发现** → 发现 filesystem 和 linter Server
4. **Function Call 执行** → `read_file("agent.py")`
5. **再次 Function Call** → `run_linter("agent.py")`
6. **LLM 综合分析** → 输出标准审查报告

---

## 8. A2A 协议是什么？和 MCP 的关系？

### 为什么需要 A2A？

**MCP 解决了 Agent ↔ 工具的连接，但没有解决 Agent ↔ Agent 的连接。**

### A2A 核心概念

**① Agent Card（智能体名片）**
- JSON 描述文件
- 包含名称、能力、端点、认证方式

**② Task（任务）**
- 标准化任务对象
- 生命周期：CREATED → PROCESSING → COMPLETED / FAILED

**③ Message & Artifact**
- Message：过程沟通
- Artifact：最终成果（文档/代码/数据）

### A2A 通信流程

编排 Agent 收到"写 AI Agent 竞品分析报告"：

1. 查询 Agent Registry，找到 Research Agent 和 Writer Agent
2. 读取 Agent Card，了解能力
3. 委托 Research Agent：搜集框架信息
4. Research Agent 用 MCP 调工具完成，返回调研报告
5. 委托 Writer Agent：基于调研写报告
6. 汇总结果返回用户

### MCP vs A2A

|  | MCP | A2A |
|--|-----|-----|
| 连接对象 | Agent ↔ 工具 | Agent ↔ Agent |
| 通信模式 | 请求-响应 | 任务委托 |
| 状态管理 | 无状态 | 有状态（任务生命周期） |
| 发现机制 | tools/list | Agent Registry |

---

## 9. Agent 的记忆系统怎么设计？

### 三层记忆架构

**① 短期记忆（Working Memory）**
- 当前对话上下文
- 存储：LLM 上下文窗口
- 生命周期：单次会话

**② 长期记忆（Long-term Memory）**
- 历史对话、用户偏好
- 存储：向量数据库（如 Pinecone、Weaviate）
- 检索：语义相似度搜索

**③ 程序性记忆（Procedural Memory）**
- 工作流程、操作规范
- 存储：Skills 文件
- 激活：场景触发

### 记忆检索策略

1. **相关性检索** - 基于语义相似度
2. **时间衰减** - 近期记忆权重更高
3. **重要性评分** - 关键信息优先召回

### 记忆压缩技术

- **摘要压缩** - 用 LLM 总结历史对话
- **关键信息提取** - 只保留核心事实
- **分层存储** - 热数据在内存，冷数据在数据库

---

## 10. Agent 的安全与可靠性如何保障？

### 安全性保障

**① 工具调用权限控制**
- 白名单机制
- 敏感操作需人工确认
- 审计日志

**② 输入验证**
- 参数类型检查
- SQL 注入防护
- XSS 过滤

**③ 输出审查**
- 敏感信息脱敏
- 幻觉检测
- 事实核查

### 可靠性保障

**① 错误处理**
- 工具调用失败重试（最多 3 次）
- 降级策略（工具不可用时的备选方案）
- 优雅失败（给出部分结果而非完全失败）

**② 幻觉缓解**
- 引用来源
- 事实验证工具
- 不确定性表达

**③ 监控告警**
- Token 消耗监控
- 响应时间监控
- 错误率告警

---

## 11. RAG 和 Agent 是什么关系？

### 核心区别

**RAG：** 检索增强生成，解决知识时效性问题
**Agent：** 自主任务执行系统

### RAG 可以是 Agent 的一个工具

```
Agent 工具箱：
- search_web()
- query_database()
- rag_search()  ← RAG 作为知识检索工具
- send_email()
```

### 典型集成场景

**智能客服 Agent：**
1. 用户问"你们的退款政策是什么？"
2. Agent 调用 RAG 工具检索知识库
3. RAG 返回相关政策文档片段
4. Agent 基于检索结果生成回答

### RAG vs Agent 对比

|  | RAG | Agent |
|--|-----|-------|
| 核心能力 | 知识检索 | 任务执行 |
| 是否调用工具 | 否 | 是 |
| 是否有规划 | 否 | 是 |
| 典型应用 | 问答系统 | 任务助手 |

---

## 12. 大厂真实面试追问汇总

### 字节跳动

**Q1：** "你说你们用了 ReAct，遇到过死循环吗？怎么解决的？"
**A：** 三个机制：最大步数限制（15 步）、重复动作检测（连续 3 次相同调用退出）、超时控制

**Q2：** "Function Call 和 MCP 有什么本质区别？"
**A：** Function Call 是 LLM 调用函数的协议，MCP 是工具集成的标准化方案。Function Call 解决"怎么调"，MCP 解决"调什么"

**Q3：** "如果让你设计一个 Agent 的记忆系统，你会怎么做？"
**A：** 三层架构：短期记忆（上下文窗口）、长期记忆（向量数据库）、程序性记忆（Skills）

### 阿里巴巴

**Q1：** "Agent 和 Workflow 在什么场景下该选哪个？"
**A：** 固定流程用 Workflow（订单处理），开放式目标用 Agent（研究分析）。生产环境常用混合架构

**Q2：** "你们的 Agent Token 消耗怎么控制的？"
**A：** Plan-and-Execute 模式（相比 ReAct 省 80%）、并行 Function Call、记忆压缩、缓存机制

**Q3：** "Agent 出现幻觉怎么办？"
**A：** 引用来源、事实验证工具、Reflection 模式自我校正、不确定性表达

### 腾讯

**Q1：** "Skills 和 System Prompt 有什么区别？"
**A：** System Prompt 是全局规范，Skills 是按需激活的领域指导。Skills 模块化、可维护性更好

**Q2：** "Multi-Agent 什么时候用？"
**A：** 任务明确需要并行处理或专业分工时。不要过早引入，单 Agent 往往更稳定

**Q3：** "如何保证 Agent 的安全性？"
**A：** 工具调用权限控制、输入验证、输出审查、审计日志、敏感操作人工确认

### 美团

**Q1：** "A2A 协议解决什么问题？"
**A：** Agent 之间的通信协作。MCP 解决 Agent ↔ 工具，A2A 解决 Agent ↔ Agent

**Q2：** "如何防止 Agent 陷入死循环？"
**A：** 最大步数限制、重复动作检测、超时控制

**Q3：** "RAG 和 Agent 是什么关系？"
**A：** RAG 可以作为 Agent 的一个知识检索工具，解决知识时效性问题

---

## 面试准备建议

### 必须掌握的核心概念

1. **LLM vs Agent** - 能用一句话说清楚
2. **ReAct 工作流程** - Thought → Action → Observation
3. **Function Call 四步流程** - 定义工具 → LLM 生成指令 → 执行 → 返回结果
4. **MCP 解决的问题** - N × M → N + M
5. **三层记忆架构** - 短期、长期、程序性

### 加分项

1. **实际项目经验** - 能说出具体场景、遇到的问题、解决方案
2. **性能优化** - Token 消耗控制、并行调用、缓存策略
3. **安全性考虑** - 权限控制、输入验证、审计日志
4. **框架对比** - LangChain vs LangGraph vs CrewAI

### 常见陷阱

1. **不要混淆 LLM 和 Agent** - LLM 只会说，Agent 会做
2. **不要认为 LLM 执行 Function Call** - LLM 只输出指令，应用程序执行
3. **不要过早引入 Multi-Agent** - 单 Agent 往往更稳定
4. **不要忽视安全性** - 工具调用必须有权限控制
