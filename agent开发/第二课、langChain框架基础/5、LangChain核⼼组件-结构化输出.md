# LangChain核心组件-结构化输出核心笔记
LangChain的「结构化输出」组件解决了模型输出“非标准化、难解析”的痛点，通过预先定义数据格式（如字段名、类型、校验规则），让模型直接返回可直接用的结构化数据（Pydantic对象/字典），无需手动解析自然语言，大幅提升开发效率。

## 一、核心价值与基础认知
### 1. 核心解决的问题
- 传统输出：模型返回大段自然语言，需手动用正则/字符串截取关键信息，易出错、效率低；
- 结构化输出：模型按预设格式返回数据（如JSON/Pydantic对象），字段明确、类型固定，可直接通过`.字段名`调用，无需二次解析。

### 2. 基础示例（提取联系人信息）
```python
from pydantic import BaseModel, Field
from langchain.agents import create_agent

# 1. 定义结构化格式（Pydantic模型）
class ContactInfo(BaseModel):
    name: str = Field(description="姓名")
    email: str = Field(description="邮箱地址")
    phone: str = Field(description="电话号码")

# 2. 创建Agent并指定输出格式
agent = create_agent(model="gpt-4o", response_format=ContactInfo)

# 3. 调用Agent获取结构化结果
result = agent.invoke({
    "messages": [{"role": "user", "content": "帮我提取:张三,邮箱 zhangsan@example.com,手机 13800138000"}]
})

# 4. 直接使用结果（无需解析）
contact = result["structured_response"]
print(contact.name)  # 张三
print(contact.email) # zhangsan@example.com
```

## 二、两种结构化输出策略（按需选择）
LangChain支持两种底层实现策略，自动适配模型能力，也可手动指定：
| 策略 | 核心原理 | 适用场景 | 优势 | 实现方式 |
|------|----------|----------|------|----------|
| ProviderStrategy | 利用模型厂商原生结构化输出能力（如OpenAI的JSON模式） | 支持原生结构化输出的模型（GPT-4o、Claude 3等） | 可靠性最高，格式不易出错 | 直接传schema（自动识别）或显式指定`ProviderStrategy(schema)` |
| ToolStrategy | 将结构化格式伪装成工具，通过工具调用实现 | 不支持原生结构化输出，但支持工具调用的模型 | 兼容性强，适配所有支持工具调用的模型 | 显式指定`ToolStrategy(schema)` |

### 关键结论
- 优先让LangChain自动选择（直接传schema），无需手动指定策略；
- 模型能力强（如GPT-4o、Claude 3）→ 自动用ProviderStrategy；
- 模型能力一般（仅支持工具调用）→ 自动用ToolStrategy。

## 三、实战场景：商品评论分析器
以电商评论分析为例，展示结构化输出的完整应用，包含字段校验、枚举限制等高级功能：
### 1. 定义结构化格式
```python
from pydantic import BaseModel, Field
from typing import Literal

class ReviewAnalysis(BaseModel):
    """商品评论分析结果"""
    rating: int | None = Field(description="评分，1到5分", ge=1, le=5)  # 数值范围校验
    sentiment: Literal["positive", "negative"] = Field(description="情感倾向")  # 枚举限制
    key_points: list[str] = Field(description="评论要点，每条1-3个词")  # 列表类型
```

### 2. 调用Agent分析评论
```python
agent = create_agent(
    model="gpt-4o",
    tools=[],
    response_format=ReviewAnalysis  # 自动选择策略
)

# 分析用户评论
result = agent.invoke({
    "messages": [{"role": "user", "content": "物流很快，两天就到了。但是包装有点简陋，东西倒是没问题，好评。"}]
})

# 提取结果并使用
review = result["structured_response"]
print(f"评分: {review.rating}")       # 输出：4
print(f"情感: {review.sentiment}")    # 输出：positive
print(f"要点: {review.key_points}")   # 输出：['物流快', '包装简陋', '商品完好']
```

### 3. 核心亮点
- 字段校验：`rating`超出1-5分会报错，强制模型输出合法值；
- 枚举限制：`sentiment`仅允许“positive/negative”，避免模糊表述；
- 类型约束：`key_points`固定为列表，模型自动拆分要点，格式统一。

## 四、结构化格式的4种定义方式
除了Pydantic，LangChain还支持3种格式定义，按需选择：
| 定义方式 | 核心特点 | 返回类型 | 推荐场景 |
|----------|----------|----------|----------|
| Pydantic Model（推荐） | 支持字段校验、IDE提示强、功能最完整 | Pydantic对象（可通过`.字段名`调用） | 生产环境、需严格校验的场景 |
| dataclass | 语法简洁、无需继承 | 字典（需通过`["字段名"]`调用） | 简单场景、无需校验 |
| TypedDict | 仅定义类型、无校验 | 字典（需通过`["字段名"]`调用） | 类型提示需求，无需校验 |
| JSON Schema | 通用格式、支持复杂结构 | 字典（需通过`["字段名"]`调用） | 跨平台兼容、复杂嵌套场景 |

### 关键建议
优先使用**Pydantic Model**，兼顾校验、提示、易用性，是最适合生产环境的方式。

## 五、高级用法：Union类型（动态选择输出格式）
当需要处理多种类型的输入（如同时处理“产品评论”和“客户投诉”），可通过`Union`类型让模型自动选择匹配的输出格式，无需手动判断：
### 1. 定义多格式Schema
```python
from typing import Union

# 产品评论格式
class ProductReview(BaseModel):
    rating: int | None = Field(description="评分1-5")
    sentiment: Literal["positive", "negative"]
    key_points: list[str]

# 客户投诉格式
class CustomerComplaint(BaseModel):
    issue_type: Literal["product", "service", "shipping", "billing"] = Field(description="问题类型")
    severity: Literal["low", "medium", "high"] = Field(description="严重程度")
    description: str = Field(description="问题描述")
```

### 2. 调用Agent动态匹配
```python
agent = create_agent(
    model="gpt-4o",
    tools=[],
    response_format=Union[ProductReview, CustomerComplaint]
)

# 输入1：产品评论 → 返回ProductReview格式
agent.invoke({"messages": [{"role": "user", "content": "商品很好用，给5分！"}]})

# 输入2：客户投诉 → 返回CustomerComplaint格式
agent.invoke({"messages": [{"role": "user", "content": "快递丢了，客服不处理，非常生气！"}]})
```

## 六、错误处理：自动重试与自定义
模型可能输出不符合格式的数据（如评分超出1-5分），LangChain提供完善的错误处理机制：
### 1. 默认行为（自动重试）
ToolStrategy默认开启自动重试，当模型输出不符合schema时，LangChain会自动反馈错误原因（如“评分必须1-5分”），让模型修正后重新输出，无需手动干预。

### 2. 自定义错误处理
可通过`handle_errors`参数自定义错误提示、指定处理的错误类型，或关闭自动重试：
```python
from langchain.agents.structured_output import ToolStrategy

# 自定义错误提示
strategy = ToolStrategy(
    schema=ReviewAnalysis,
    handle_errors="评分只能1-5分，情感只能选positive或negative，请重新输出。"
)

# 仅处理特定错误（如数值错误）
strategy = ToolStrategy(
    schema=ReviewAnalysis,
    handle_errors=ValueError  # 仅当出现ValueError时重试
)

# 关闭自动重试（直接报错）
strategy = ToolStrategy(
    schema=ReviewAnalysis,
    handle_errors=False
)
```

## 七、适用场景与核心总结
### 1. 必用结构化输出的场景
- 数据提取：从文本中提取联系人、地址、产品规格等结构化信息；
- 内容分类：情感分析、垃圾邮件检测、工单分类等需要固定标签的场景；
- 表单填充：自动提取用户输入中的表单字段（如姓名、电话、地址）；
- API对接：模型输出直接作为下游系统的输入（如存入数据库、调用其他API）；
- 批量处理：批量分析评论、工单等数据，需要统一格式汇总。

### 2. 核心总结
- 结构化输出的核心是“提前定义格式”，让模型按规则返回数据，避免手动解析；
- 优先使用Pydantic Model定义格式，兼顾校验、提示、易用性；
- 策略选择无需手动干预，LangChain自动适配模型能力；
- 支持动态格式选择（Union类型）和完善的错误处理，适配复杂业务场景；
- 下游需消费模型输出（如存库、对接API）时，结构化输出是刚需，能大幅提升开发效率和系统稳定性。