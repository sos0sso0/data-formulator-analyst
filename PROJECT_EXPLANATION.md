# Data Formulator 项目功能实现逻辑详解

> 写给"只懂一点点 Python"的技术合伙人

---

## 一、这个项目是做什么的？

**Data Formulator** 是微软研究院开发的一款"AI 驱动的数据可视化工具"。

用一句话来说：**你把数据扔给它，然后用普通话（或英文）告诉它你想看什么图，它就帮你画出来——不需要你自己写代码。**

举个例子：
- 你上传一份销售 Excel 表格
- 你说："帮我看看每个月的销售趋势，按地区分开"
- 它自动生成一段 Python 代码，运行后生成折线图展示给你

---

## 二、项目整体结构（像一家餐厅）

把这个项目想象成一家餐厅：

| 餐厅组成 | 对应项目部分 | 技术名称 |
|---------|------------|---------|
| 餐厅大堂（顾客看到的界面） | 前端界面 | React + TypeScript（`src/` 目录） |
| 后厨（真正干活的地方） | 后端服务 | Python + Flask（`py-src/` 目录） |
| 传菜员 | 前后端通信 | HTTP API 接口（`/api/agent/...`） |
| 大厨助理（AI）| 语言模型 | LLM（如 GPT-4o、Claude 等） |
| 菜谱 | 提示词（Prompt） | 每个 Agent 文件里的 `SYSTEM_PROMPT` |

---

## 三、用户的一次操作完整流程

下面用"用户问了一个问题"来走一遍完整流程：

```
用户在网页上输入：
"帮我看看销售额最高的前10个城市"

↓ 前端（浏览器）把这句话 + 当前数据表格 → 发送给后端

↓ 后端选择合适的"Agent（智能助手）"处理这个请求

↓ Agent 组装一个完整的"提示词"发给 LLM（比如 GPT-4o）

↓ LLM 返回：
   1. 一段 JSON：分析了用户的意图，决定了图表类型和字段
   2. 一段 Python 代码：用来处理数据

↓ 后端在沙盒中运行这段 Python 代码，得到新的数据表格

↓ 把结果（新数据 + 图表配置）返回给前端

↓ 前端用 Vega-Lite 渲染出图表显示给用户
```

---

## 四、最核心的部分：用户指令是如何变成操作的？

这是整个系统最重要的设计，分为三步：

### 第一步：组装"提示词"（Prompt）

当你说"帮我看看销售额最高的前10个城市"，后端会把这些信息组装成一个完整的"提示词"发给 AI：

```
系统指令（System Prompt）:
  你是一个数据科学家，帮用户做数据转换...
  [详细的规则和输出格式要求]

用户上下文（Context）:
  [数据集描述]
  表格: sales_data（1000行 × 5列）
  字段:
    - city: 城市名称（字符串）
    - date: 日期
    - sales: 销售额（数字）
    ...
  前5行数据:
    | city     | date       | sales |
    | 北京     | 2024-01-01 | 50000 |
    ...

用户目标（Goal）:
  "帮我看看销售额最高的前10个城市"
```

### 第二步：LLM 生成"分析 + 代码"

LLM 收到这个提示词后，返回两样东西：

**① 分析结果（JSON格式）**：告诉系统它"理解"了什么
```json
{
  "mode": "infer",
  "recap": "显示销售额最高的前10个城市",
  "display_instruction": "按**销售额**显示前10个城市",
  "input_tables": ["sales_data"],
  "output_fields": ["city", "total_sales", "rank"],
  "chart_type": "bar",
  "chart_encodings": {"x": "city", "y": "total_sales"}
}
```

**② Python 转换代码**：实际操作数据的脚本
```python
import pandas as pd
import numpy as np

def transform_data(df_sales_data):
    # 按城市汇总销售额
    city_sales = df_sales_data.groupby('city')['sales'].sum().reset_index()
    city_sales.columns = ['city', 'total_sales']
    
    # 排序并取前10
    city_sales = city_sales.sort_values('total_sales', ascending=False).head(10)
    
    # 添加排名
    city_sales['rank'] = range(1, len(city_sales) + 1)
    
    transformed_df = city_sales[['city', 'total_sales', 'rank']]
    return transformed_df
```

### 第三步：执行代码，返回结果

后端接收到这段 Python 代码后，在一个"安全沙盒"里运行它（防止恶意代码损坏服务器），得到处理好的数据，再返回给前端画图。

---

## 五、有哪些"智能助手"（Agent）？各自负责什么？

项目里有多个专门的 Agent，就像一家公司里有不同岗位的员工：

### 1. `PythonDataRecAgent`（推荐助手）
**场景**：用户问题比较模糊，比如"帮我看看这份数据有什么有趣的地方"

**它做什么**：
- 分析数据的特征
- **主动推荐**适合的图表类型和分析维度
- 生成 Python 代码来处理数据

**文件位置**：`py-src/data_formulator/agents/agent_py_data_rec.py`

---

### 2. `PythonDataTransformationAgent`（转换助手）
**场景**：用户已经指定了图表设计（比如拖拽了字段到 X/Y 轴），但数据还需要处理

**它做什么**：
- 理解用户指定的图表配置（x轴用什么，y轴用什么）
- 生成 Python 代码对数据进行转换

**文件位置**：`py-src/data_formulator/agents/agent_py_data_transform.py`

---

### 3. `InteractiveExploreAgent`（问题推荐助手）
**场景**：用户完成了一次分析，想知道"接下来可以探索什么"

**它做什么**：
- 分析当前已探索的内容
- 推荐4个有意思的后续探索问题
- 支持流式输出（实时显示问题，而不是等全部生成完再显示）

**文件位置**：`py-src/data_formulator/agents/agent_interactive_explore.py`

---

### 4. `ExplorationAgent`（规划助手）
**场景**：用户开启了"智能体模式"（Agent Mode），让 AI 自动连续探索数据

**它做什么**：
- 看完当前步骤的结果
- 决定"继续探索"还是"停下来汇报结论"
- 如果继续，调整接下来的探索计划

**文件位置**：`py-src/data_formulator/agents/agent_exploration.py`

---

### 5. `DataLoadAgent`（数据识别助手）
**场景**：用户上传了一份数据文件

**它做什么**：
- 分析每一列的数据类型（字符串、数字、日期...）
- 识别语义类型（是"年份"还是"城市名"还是"百分比"）
- 给数据表建议一个合适的名字

**文件位置**：`py-src/data_formulator/agents/agent_data_load.py`

---

### 6. `DataCleanAgent`（数据清洗助手）
**场景**：用户粘贴了一段混乱的文字，或者上传了一张截图

**它做什么**：
- 从图片/文字中提取结构化数据
- 处理网页 URL，抓取内容转换为数据

**文件位置**：`py-src/data_formulator/agents/agent_data_clean.py`

---

### 7. `ReportGenAgent`（报告生成助手）
**场景**：用户完成探索，想生成一份报告

**它做什么**：
- 接收用户选择的多张图表
- 生成一篇 Markdown 格式的分析报告
- 报告可以是博客文章、社交媒体帖子、管理层摘要等风格

**文件位置**：`py-src/data_formulator/agents/agent_report_gen.py`

---

## 六、"智能体模式"（Agent Mode）是怎么工作的？

这是最强大也最复杂的功能。用户只需要说"帮我全面分析这份销售数据"，AI 就会自动制定计划、连续执行多步分析。

```
用户输入一个高层目标：
"全面分析我们公司的销售数据"

↓

【规划阶段】
AI 制定探索计划：
  步骤1: 看看各地区销售总额分布
  步骤2: 分析月度销售趋势
  步骤3: 找出最畅销的前5个产品
  ...

↓ 循环执行每一步 ↓

【执行步骤1】
  - RecAgent 生成代码 → 运行 → 得到"地区销售条形图"

【规划决策】
  ExplorationAgent 看到结果 → 决定继续 → 调整接下来的计划

【执行步骤2】
  - RecAgent 生成代码 → 运行 → 得到"月度趋势折线图"

...（重复直到达到最大步数或 AI 认为探索足够了）

【汇报结论】
  ExplorationAgent 决定停止 → 输出简短总结
```

这个循环的代码在：`py-src/data_formulator/workflows/exploration_flow.py`

---

## 七、如何与不同的 AI 模型通信？（`client_utils.py`）

项目支持多种 AI 提供商（OpenAI、微软 Azure、Anthropic Claude、谷歌 Gemini、本地 Ollama 等），靠的是一个叫 **LiteLLM** 的库。

LiteLLM 就像一个"万能插座"——不管你用哪家的 AI，都用同一套接口调用：

```python
# 无论是 GPT-4o、Claude 还是本地模型，调用方式一样
client = Client(endpoint="openai", model="gpt-4o", api_key="你的密钥")
response = client.get_completion(messages=[...])
```

核心文件：`py-src/data_formulator/agents/client_utils.py`

---

## 八、代码在哪里运行？（安全沙盒）

AI 生成的 Python 代码需要实际运行才能处理数据。但直接运行 AI 生成的代码是危险的——万一 AI 生成了删文件的代码怎么办？

项目用了一个"沙盒"（Sandbox）机制：

```
AI 生成的代码
    ↓
在独立的子进程中运行（和主程序隔离）
    ↓
安装了"审计钩子"：
  - 禁止写文件
  - 禁止调用危险的系统函数
    ↓
结果通过管道安全传回主进程
```

核心文件：`py-src/data_formulator/py_sandbox.py`

---

## 九、SQL 模式（大数据时怎么办？）

当数据量很大时（比如几百万行），把所有数据都发给 AI 是不现实的。项目有一个 SQL 模式：

- 数据存在本地 **DuckDB**（一个嵌入式数据库）里
- AI 生成 **SQL 查询语句**（而不是 Python 代码）
- DuckDB 执行 SQL，只返回结果
- 这样就不用把所有数据都传给 AI，只传数据的"摘要"（字段名、类型、部分示例值）

对应文件：`agent_sql_data_transform.py`、`agent_sql_data_rec.py`

---

## 十、前端是如何工作的？

前端用 **React + TypeScript** 写的（你可以理解为"一个响应很快的网页"）。

主要组成部分：

| 组件 | 功能 |
|-----|------|
| `DataFormulator.tsx` | 主界面，把所有组件拼在一起 |
| `EncodingShelfThread.tsx` | 图表编码区（拖拽字段到 X/Y 轴） |
| `ChatDialog.tsx` | 自然语言输入框 |
| `DataThread.tsx` | 数据探索历史线索面板 |
| `VisualizationView.tsx` | 图表显示区域（基于 Vega-Lite） |
| `ReportView.tsx` | 报告展示区 |

前端状态管理用 **Redux**（一种统一管理应用数据的工具），存储在 `src/app/dfSlice.tsx`。

---

## 十一、整体架构一览图

```
┌─────────────────────────────────────────────────────────┐
│                   浏览器（前端）                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐  │
│  │ 数据加载  │ │ 编码配置  │ │ 自然语言  │ │  图表展示  │  │
│  │（上传文件）│ │（拖拽字段）│ │（输入问题）│ │(Vega-Lite)│  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────┘  │
└───────────────────────────┬─────────────────────────────┘
                            │ HTTP API 请求
                            ▼
┌─────────────────────────────────────────────────────────┐
│                Python 后端（Flask 服务器）                │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Agent 路由层（agent_routes.py）      │   │
│  │  /api/agent/derive-data      → 数据转换          │   │
│  │  /api/agent/explore-data-streaming → 智能探索    │   │
│  │  /api/agent/get-recommendation-questions → 问题推荐│  │
│  │  /api/agent/generate-report-stream → 生成报告    │   │
│  └────────────────────────┬────────────────────────┘   │
│                           │                             │
│  ┌────────────────────────▼────────────────────────┐   │
│  │                各类 Agent                         │   │
│  │  DataRecAgent → 推荐分析方向 + 生成代码           │   │
│  │  TransformAgent → 转换数据 + 生成代码             │   │
│  │  ExplorationAgent → 规划和决策                    │   │
│  │  InteractiveExploreAgent → 推荐探索问题           │   │
│  │  ReportGenAgent → 生成分析报告                    │   │
│  └────────────────────────┬────────────────────────┘   │
│                           │                             │
│  ┌────────────────────────▼────────────────────────┐   │
│  │          LLM 客户端（client_utils.py）            │   │
│  │     LiteLLM（统一接口支持多种 AI 提供商）          │   │
│  └────────────────────────┬────────────────────────┘   │
│                           │                             │
│  ┌──────────┐  ┌──────────▼──────────┐  ┌──────────┐  │
│  │ py_sandbox│  │   AI 大语言模型      │  │  DuckDB  │  │
│  │（代码执行）│  │(GPT-4o/Claude/...)  │  │（大数据）  │  │
│  └──────────┘  └─────────────────────┘  └──────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## 十二、关键技术术语白话解释

| 技术术语 | 白话解释 |
|---------|---------|
| **LLM（大语言模型）** | 就是 ChatGPT 这种 AI，能理解文字并生成文字/代码 |
| **Prompt（提示词）** | 你发给 AI 的完整"任务说明书"，包含规则、上下文和问题 |
| **Agent（智能体）** | 一个专门负责某类任务的 AI 助手，有自己的提示词和处理逻辑 |
| **Flask** | Python 的轻量级 Web 框架，相当于让 Python 程序变成"网站后台" |
| **React** | 用来写网页界面的 JavaScript 框架，让界面操作反应更快 |
| **Vega-Lite** | 一个用 JSON 配置就能画图的可视化库，类似"用配置表描述图表" |
| **DuckDB** | 一个超快的本地数据库，擅长分析大量数据 |
| **LiteLLM** | 一个"万能适配器"，让代码能以同一种方式调用各家 AI |
| **沙盒（Sandbox）** | 隔离的安全运行环境，AI 生成的代码在里面跑，不会影响外部系统 |
| **Redux** | 前端的"共享记事本"，所有页面组件都从这里读取和更新数据 |
| **流式输出（Streaming）** | AI 边生成边发送，用户不用等全部生成完才能看到内容 |
| **API（接口）** | 前后端之间约定好的"暗号"，前端按格式发请求，后端按格式回复 |

---

## 十三、一句话总结整个系统

> **用户说自然语言 → 后端把数据摘要 + 用户意图 → 打包成提示词发给 AI → AI 生成 JSON（分析计划）+ Python 代码 → 在沙盒里运行代码 → 把处理好的数据 + 图表配置 → 返回给前端渲染图表。**

整个过程的"魔法"核心在于：**每个 Agent 里精心设计的 `SYSTEM_PROMPT`**。这些提示词告诉 AI "你是谁、你该做什么、输入是什么格式、输出必须是什么格式"。有了这些精确的指令，AI 才能稳定地输出可执行的 JSON + Python 代码，而不是随便聊天。
