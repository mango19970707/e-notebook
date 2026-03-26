# Deep Agents进阶：Skill系统+沙箱课程总结

> 对应原文 PDF：`D:\notebook\agent开发\deep agent框架实战\第四课：Skill 系统 + 沙箱，Agent 能力上限被拉高了.pdf`
> 说明：在不改变原文含义的前提下做精简总结；代码示例保持完整，不删减。

本课程聚焦Deep Agents的两大核心进阶能力——**Skill系统**（提升Agent任务执行专业性与稳定性）和**沙箱（Sandboxes）**（保障Agent代码/命令执行安全），基于`deepagents==0.4.3`（Python3.11+）展开实操讲解，解决Agent“会用工具但不会做事”“执行代码易搞乱环境”的核心痛点，拉高Agent能力上限与安全阈值。

## 一、Skill系统：Agent的“专业工作手册”
### 1. Skill与Tool的核心区别
| 维度         | 工具（Tool）                | Skill（技能）                  |
|--------------|-----------------------------|--------------------------------|
| 定位         | Agent的“手”，具体操作能力    | Agent的“工作手册”，专业任务SOP |
| 核心价值     | 提供基础操作（搜索、读文件等）| 提供方法论，确保任务执行标准化 |
| 表现差异     | 无Skill时，Agent随机调用工具，结果不稳定 | 有Skill时，Agent按流程执行，输出质量可控 |
| 形式         | Python函数（带type hints+docstring） | Markdown文件（含元数据+流程指令） |

简单总结：**工具给能力，Skill给方法**。例如`internet_search`是工具（能上网），而`tech-research`是Skill（教Agent如何做专业调研）。

### 2. Skill的结构与规范
一个Skill对应一个独立文件夹，核心文件为`SKILL.md`（可搭配辅助脚本），文件夹结构如下：
```
skills/
├── langgraph-docs/       # Skill文件夹（命名自定）
│   └── SKILL.md         # 核心：元数据+流程指令
└── arxiv-search/
    ├── SKILL.md         # 核心文件
    └── arxiv_search.py  # 辅助脚本（可选）
```

#### `SKILL.md`的两大核心部分
1. **前置元数据（Frontmatter）**：Agent用于匹配任务的关键依据，格式为YAML，核心字段如下：
    - `name`：Skill名称（唯一标识）；
    - `description`：Skill适用场景描述（Agent靠此判断是否激活，1024字符内，超则截断）；
    - `allowed-tools`：该Skill允许调用的工具（可选）；
    - 其他自定义元数据（如作者、版本）。
2. **正文指令**：Markdown格式的详细执行流程，需分步骤写清“用什么工具→做什么操作→注意事项”，确保Agent可按流程复现。

#### 示例：LangGraph文档查询Skill
```markdown
---
name: langgraph-docs
description: Use this skill for requests related to LangGraph to fetch relevant documentation and provide accurate guidance.
allowed-tools: fetch_url
---
# LangGraph文档查询Skill
## Overview
该技能用于查询LangGraph官方文档，为用户提供准确的实现指导和概念解释。
## Instructions
### 1. 获取文档索引
用fetch_url工具访问：https://docs.langchain.com/llms.txt，获取所有文档的结构化列表。
### 2. 筛选相关文档
根据用户问题，从索引中筛选2-4个最相关的URL，优先级：
- 实现类问题→优先How-to指南
- 理解类问题→优先核心概念页
- 完整示例→优先教程
- API细节→优先参考文档
### 3. 抓取选中文档
用fetch_url工具读取筛选后的文档内容。
### 4. 生成指导方案
整合文档信息，精准回应用户需求。
```

### 3. Skill的核心机制：渐进式披露（Progressive Disclosure）
Agent加载Skill的逻辑的是“按需加载”，而非一次性全部注入，核心目的是**节省Token**：
1. Agent启动时，仅读取所有`SKILL.md`的**元数据（Frontmatter）**，不加载正文；
2. 接收用户请求后，Agent匹配元数据中的`description`，找到适配的Skill；
3. 仅加载适配Skill的完整正文指令，执行任务。

该机制避免了“加载所有Skill正文导致Token浪费”的问题，契合人类“按需调用知识”的思维模式。

### 4. Skill实操：编写与加载
#### 步骤1：编写中文技术调研Skill
1. 创建文件夹结构：
```
skills/
└── tech-research/
    └── SKILL.md
```
2. 编写`SKILL.md`：
```markdown
---
name: tech-research
description: 用于技术调研任务。当用户要求调研某个技术、框架、工具时，按标准流程执行调研并输出报告。
allowed-tools: internet_search, fetch_url
metadata:
  author: nange
  version: "1.0"
---
# 技术调研Skill
## 调研流程
### 1. 明确调研范围
收到任务后，先列出3-5个核心问题（如“核心功能”“适用场景”“优缺点”），避免盲目搜索。
### 2. 信息收集（按优先级）
- 第一轮：官方文档+GitHub仓库（记录Star数、最近更新时间）
- 第二轮：技术博客+教程（辨别时效性，优先近1年内容）
- 第三轮：社区评价（Reddit、掘金等真实用户反馈）
- 所有信息用write_file工具存入/research/目录，标注来源链接和日期。
### 3. 整理报告
报告结构：
- 一句话技术概述
- 核心特点（3-5点）
- 优缺点对比
- 适用/不适用场景
- 参考链接
### 注意事项
- 标注信息版本号和调研日期
- 区分“官方宣传”和“用户实际评价”
- 矛盾信息需同时呈现，不主观判断
- 报告控制在2000字以内
```

#### 步骤2：加载Skill并运行Agent
```python
from deepagents import create_deep_agent
from deepagents.backends.utils import create_file_data
from langgraph.checkpoint.memory import MemorySaver

# 初始化checkpointer（记忆/中断必备）
checkpointer = MemorySaver()

# 读取Skill文件，生成标准文件格式
with open("skills/tech-research/SKILL.md", "r", encoding="utf-8") as f:
    skill_content = f.read()
skills_files = {
    "/skills/tech-research/SKILL.md": create_file_data(skill_content)
}

# 定义搜索工具（示例）
def internet_search(query: str) -> str:
    """互联网搜索工具，返回相关信息"""
    # 实际搜索逻辑（如对接Tavily）
    return f"搜索结果：{query} 的相关信息..."

# 创建Agent并加载Skill
agent = create_deep_agent(
    skills=["./skills/"],  # 指定Skill目录
    tools=[internet_search],
    checkpointer=checkpointer,
    system_prompt="你是专业调研助手，严格按Skill流程执行调研任务"
)

# 运行Agent：调研CrewAI与AutoGen的区别
result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "帮我调研一下 CrewAI 和 AutoGen 的区别"}],
        "files": skills_files
    },
    config={"configurable": {"thread_id": "12345"}},
)

print(result["messages"][-1].content)
```

### 5. Skill的高级用法
#### （1）Skill带辅助脚本
当任务逻辑复杂（如复杂搜索、数据处理），可在Skill文件夹中添加Python脚本，在`SKILL.md`中说明使用方法：
```markdown
---
name: arxiv-search
description: 搜索arXiv上的学术论文，适用于需要最新研究成果的场景。
allowed-tools: execute
---
# arXiv论文搜索Skill
## 使用方法
本Skill包含辅助脚本arxiv_search.py，功能：按关键词搜索论文并返回JSON结果。
1. 用execute工具运行脚本：
   python /skills/arxiv-search/arxiv_search.py --query "multi-agent systems" --max_results 5
2. 脚本输出格式：包含论文标题、作者、摘要、链接。
## 搜索建议
- 关键词用英文，搜索精度更高
- 加年份限定（如"2025"）过滤旧论文
- 搜索结果存入文件，便于后续引用
```

#### （2）Skill优先级：后加载覆盖前加载
当从多个目录加载同名Skill时，**后加载的Skill覆盖先加载的**，适配“项目级Skill覆盖通用Skill”的场景：
```python
# /skills/project/的web-search覆盖/skills/user/的同名Skill
agent = create_deep_agent(
    skills=["/skills/user/", "/skills/project/"],
    tools=[internet_search]
)
```

#### （3）子Agent的Skill配置
自定义子Agent**不会继承主Agent的Skill**，需单独指定，实现技能隔离：
```python
# 自定义调研子Agent，加载专属Skill
research_subagent = {
    "name": "researcher",
    "description": "专业调研助手",
    "system_prompt": "你是调研专家，严格按Skill流程执行",
    "tools": [internet_search],
    "skills": ["/skills/research/", "/skills/web-search/"]  # 子Agent专属Skill
}

# 主Agent加载通用Skill，子Agent加载专属Skill
agent = create_deep_agent(
    skills=["/skills/main/"],  # 主Agent+通用子Agent用
    subagents=[research_subagent],
    checkpointer=checkpointer
)
```

#### （4）Skill与Memory的区别（避免混淆）
| 维度         | Skill（技能）                | Memory（记忆/AGENTS.md）       |
|--------------|-----------------------------|--------------------------------|
| 定位         | 专业任务流程（方法论）       | 用户偏好、项目背景（习惯/常识） |
| 加载方式     | 按需加载（匹配任务才加载）   | 启动即注入（全程占用Token）     |
| 适用场景     | “怎么做技术调研”“怎么查文档” | “用户喜欢简洁风格”“项目用React” |
| Token消耗    | 低（不匹配不消耗）           | 高（固定占用）                 |

## 二、沙箱（Sandboxes）：Agent的“安全隔离环境”
### 1. 核心价值
解决Agent执行Shell命令、安装依赖、运行代码时的安全风险，实现“隔离执行”——Agent在沙箱内任意操作（删文件、装包、跑代码），均不会影响本地/服务器环境，避免误操作、恶意攻击导致的系统损坏。

### 2. 沙箱的本质与架构
- **本质**：Deep Agents中的特殊Backend，在普通文件Backend基础上新增`execute`工具，支持Agent执行Shell命令；
- **核心隔离**：计算环境隔离（沙箱为远程云环境，如Modal/Daytona），Agent无法访问本地文件、环境变量、密钥；
- **两种使用架构**：
    1. 模式一（Agent进沙箱）：Agent打包成Docker镜像在沙箱内运行，开发体验贴近本地，但密钥需进沙箱（有安全隐患）；
    2. 模式二（沙箱当工具）：Agent跑在本地/服务器，需执行代码时调用远程沙箱API，密钥不进沙箱（推荐生产使用）。

### 3. 主流沙箱提供商（Deep Agents原生支持）
| 提供商   | 核心特点                     | 适用场景                     |
|----------|------------------------------|------------------------------|
| Modal    | 支持GPU，适配AI/ML工作负载    | 模型推理、训练、深度学习任务 |
| Daytona  | 冷启动快（3-10秒），轻量灵活 | Web开发、普通代码执行、快速调试 |
| Runloop  | 一次性Devbox，用完即焚        | 高安全需求、临时任务         |

三者用法一致，仅初始化方式不同，后续Agent创建逻辑完全复用。

### 4. 沙箱实操：Daytona沙箱编程Agent
以Daytona为例，搭建“写代码→跑测试→输出结果”的安全编程Agent：

#### 步骤1：安装依赖
```bash
pip install deepagents langchain-daytona
```

#### 步骤2：核心代码（含Skill+沙箱）
```python
import os
from daytona import Daytona
from langchain_anthropic import ChatAnthropic
from deepagents import create_deep_agent
from deepagents.backends.utils import create_file_data
from langchain_daytona import DaytonaSandbox
from langgraph.checkpoint.memory import MemorySaver

# 1. 定义Python编程规范Skill（确保代码质量）
python_skill = """---
name: python-coding
description: 写Python代码时遵循的规范和最佳实践，需在沙箱中运行验证。
---
# Python编程规范Skill
## 代码风格
- 遵循PEP 8规范
- 函数/变量用snake_case，类名用CamelCase
- 必须加type hints
## 测试要求
- 写完代码后立即运行验证
- 自定义测试数据（如CSV），覆盖核心场景
- 报错需修复后重新运行，不提交未跑通代码
## 依赖管理
- 优先使用标准库，减少第三方依赖
- 安装依赖前先检查是否已存在
"""

# 2. 初始化核心组件
checkpointer = MemorySaver()
# 创建Daytona沙箱（远程隔离环境）
sandbox = Daytona().create()
# 将沙箱转为Deep Agents的Backend
backend = DaytonaSandbox(sandbox=sandbox)

# 3. 封装Skill文件
skills_files = {
    "/skills/python-coding/SKILL.md": create_file_data(python_skill)
}

# 4. 创建带沙箱+Skill的编程Agent
agent = create_deep_agent(
    model=ChatAnthropic(model="claude-sonnet-4-20250514"),
    system_prompt="你是Python编程助手，所有代码在沙箱中编写和运行，确保跑通后再输出结果",
    backend=backend,  # 绑定沙箱Backend
    skills=["./skills/"],  # 加载编程规范Skill
    checkpointer=checkpointer,
)

# 5. 流式运行Agent（查看执行过程）
try:
    for event in agent.stream(
        {
            "messages": [{"role": "user", "content": "写一个脚本，读取CSV文件（自己造测试数据），做均值、中位数、标准差统计，输出结果"}],
            "files": skills_files
        },
        config={"configurable": {"thread_id": "coding-001"}},
        stream_mode="updates"
    ):
        for node_name, node_output in event.items():
            if "messages" in node_output:
                for msg in node_output["messages"]:
                    if hasattr(msg, "content") and msg.content:
                        print(f"\n[{node_name}] {msg.content[:200]}")
                    if hasattr(msg, "tool_calls") and msg.tool_calls:
                        for tc in msg.tool_calls:
                            print(f"\n[工具] {tc['name']}")
finally:
    # 关键：用完关闭沙箱，避免持续计费
    sandbox.stop()
```

#### 步骤3：核心执行流程（Agent自动完成）
1. 制定计划（`write_todos`）：创建测试CSV→编写统计脚本→安装依赖→运行验证；
2. 操作沙箱：用`write_file`创建CSV和Python脚本，用`execute`工具安装`pandas`、运行脚本；
3. 结果反馈：脚本运行成功后，输出统计结果；失败则自动修复代码重新运行。

### 5. 沙箱核心操作
#### （1）文件进出沙箱
沙箱与本地的文件交互分两套逻辑，不可混淆：
- **Agent视角（沙箱内部）**：用`read_file`/`write_file`等工具读写沙箱内文件（通过`execute`工具翻译为Shell命令）；
- **开发者视角（本地→沙箱）**：用`upload_files`/`download_files`传输文件（通过沙箱提供商原生API）：
```python
# 1. 本地→沙箱：上传预置代码/配置
backend.upload_files([
    ("/src/main.py", b"print('hello sandbox')\n"),
    ("/config.json", b'{"debug": true}\n')
])

# 2. 沙箱→本地：下载生成的报告/结果
results = backend.download_files(["/output/report.md"])
for r in results:
    if r.content is not None:
        print(r.content.decode("utf-8"))  # 打印文件内容
    else:
        print(f"下载失败：{r.error}")
```

#### （2）直接执行Shell命令
通过`backend.execute()`直接调用沙箱的Shell命令，适用于自定义操作：
```python
# 执行命令：查看Python版本
result = backend.execute("python --version")
print(result.output)  # 输出：Python 3.11.x

# 执行命令：安装pandas并查看版本
result = backend.execute("pip install pandas && python -c 'import pandas; print(pandas.__version__)'")
print(result.output)  # 输出：2.2.x
```

#### （3）沙箱生命周期管理
沙箱按使用时长计费，需严格管理生命周期：
1. **用完即关**：用`try/finally`确保异常时也关闭沙箱；
2. **设置TTL自动销毁**：用户场景下无法预判关闭时机时，设置自动删除间隔：
```python
from daytona import CreateSandboxFromSnapshotParams

# 配置1小时无操作自动销毁
params = CreateSandboxFromSnapshotParams(
    labels={"thread_id": "user-123"},
    auto_delete_interval=3600  # 单位：秒
)
sandbox = Daytona().create(params)

# 复用沙箱：用户返回时查找已有沙箱
try:
    sandbox = Daytona().find_one(labels={"thread_id": "user-123"})
except Exception:
    sandbox = Daytona().create(params)
```

### 6. 沙箱安全核心原则
沙箱仅隔离执行环境，**防不住Prompt注入攻击**，需遵守以下安全规则：
1. **绝对不把密钥放进沙箱**：避免Agent被注入恶意指令后，窃取环境变量、密钥文件；
2. **密钥安全方案**：
    - 方案一（推荐）：认证逻辑放在沙箱外，Agent调用服务端工具发起请求，密钥在服务端注入：
      ```python
      # 服务端工具（跑在本地，不在沙箱）
      def call_internal_api(endpoint: str, params: dict) -> str:
          """调用内部API，密钥在服务端存储"""
          headers = {"Authorization": f"Bearer {os.environ['INTERNAL_API_KEY']}"}
          response = requests.get(endpoint, params=params, headers=headers)
          return response.text
      ```
    - 方案二：使用支持凭证注入的代理，Agent请求经代理时自动添加认证头；
3. **高危场景增强防护**：
    - 开启人工审批，所有`execute`工具调用需人工确认；
    - 关闭沙箱网络访问（部分提供商支持）；
    - 给沙箱分配最小权限、短期有效密钥；
    - 监控沙箱网络流量与命令执行日志。

## 三、核心踩坑记录
### 1. Skill相关坑
- **description写太模糊**：导致Agent误匹配（如“代码相关任务”匹配所有代码请求），浪费Token，需精准描述场景（如“沙箱中编写调试Python代码”）；
- **SKILL.md文件过大**：超过10MB会被Agent直接跳过，需拆分复杂Skill或精简内容；
- **子Agent未单独配置Skill**：自定义子Agent不会继承主Agent Skill，需手动指定，否则无Skill可用。

### 2. 沙箱相关坑
- **冷启动延迟**：Daytona/Modal冷启动需3-10秒，对延迟敏感场景需提前创建沙箱预热；
- **依赖重复安装**：沙箱为干净环境，每次需重新安装依赖，可通过`upload_files`预置`requirements.txt`优化；
- **忘记关闭沙箱**：导致持续计费，需强制用`try/finally`或设置TTL；
- **大输出导致上下文爆炸**：Shell命令输出过长（如`ls -la /usr`）会塞满上下文，需在system prompt中限制输出长度（如“加| head -50”）。

## 四、适用场景与总结
### 1. Skill系统适用/不适用场景
| 适用场景                     | 不适用场景                     |
|------------------------------|--------------------------------|
| 复杂多步骤任务（调研、文档查询） | 简单单步操作（查天气、读文件） |
| 需标准化执行的任务（代码编写、报告生成） | 无固定流程的创意类任务         |
| 多人协作的Agent产品（需统一流程） | 一次性临时任务（无需复用流程） |

### 2. 沙箱适用/不适用场景
| 适用场景                     | 不适用场景                     |
|------------------------------|--------------------------------|
| Agent需执行代码/Shell命令     | Agent仅做搜索、分析、写报告    |
| 多用户共享Agent服务（需环境隔离） | 个人开发调试（信任Agent行为）  |
| 执行用户提供的未知代码（需安全隔离） | 对延迟极其敏感（无法接受网络往返） |

### 3. 核心总结
- Skill系统让Agent从“会用工具”升级为“会按标准流程做事”，通过Markdown编写SOP，实现任务执行的标准化与稳定性，Token消耗低且可复用；
- 沙箱解决Agent执行代码/命令的安全风险，通过远程隔离环境避免影响本地系统，生产环境优先使用“沙箱当工具”模式，严格遵守密钥安全原则；
- 两者组合可搭建生产级Agent：Skill保障“做得好”，沙箱保障“做得安全”，适配编程助手、调研工具、自动化运维等复杂场景；
- 开发时需注意Skill的精准匹配与沙箱的生命周期管理，避免Token浪费与额外计费。

官方代码仓库：https://github.com/langchain-ai/deepagents
