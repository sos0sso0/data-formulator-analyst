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

---

## 十四、`agents/` 目录下所有文件全览

`py-src/data_formulator/agents/` 目录下共有 **17 个文件**，除了之前已介绍的，下面补充介绍全部内容：

### 文件一览表

| 文件名 | 对应类名 | 一句话功能 |
|--------|---------|-----------|
| `agent_code_explanation.py` | `CodeExplanationAgent` | 解释 AI 生成的代码，让非技术用户看懂数据转换过程 |
| `agent_concept_derive.py` | `ConceptDeriveAgent` | 用 **TypeScript** 生成"单列派生"函数（逐行操作） |
| `agent_data_clean.py` | `DataCleanAgent` | 从图片/文字/网页 URL 中提取结构化数据表格 |
| `agent_data_clean_stream.py` | `DataCleanAgentStream` | 与上面功能相同，但以流式输出方式实时返回进度 |
| `agent_data_load.py` | `DataLoadAgent` | 识别上传数据的字段类型和语义类型，给表格命名 |
| `agent_exploration.py` | `ExplorationAgent` | 智能体模式下的"规划者"，决定继续还是停止探索 |
| `agent_interactive_explore.py` | `InteractiveExploreAgent` | 交互式推荐下一步探索问题（流式输出） |
| `agent_py_concept_derive.py` | `PyConceptDeriveAgent` | 用 **Python** 生成"单列派生"函数（逐行操作） |
| `agent_py_data_rec.py` | `PythonDataRecAgent` | 当用户意图模糊时，推荐分析方向并生成 Python 转换代码 |
| `agent_py_data_transform.py` | `PythonDataTransformationAgent` | 当用户明确了图表设计，生成 Python 数据转换代码 |
| `agent_report_gen.py` | `ReportGenAgent` | 根据图表和数据生成 Markdown 分析报告（流式） |
| `agent_sort_data.py` | `SortDataAgent` | 利用 AI 常识对分类字段做自然排序（如月份、年级） |
| `agent_sql_data_rec.py` | `SQLDataRecAgent` | 当用户意图模糊时，推荐分析方向并生成 SQL 查询 |
| `agent_sql_data_transform.py` | `SQLDataTransformationAgent` | 当用户明确了图表设计，生成 SQL 数据转换查询 |
| `agent_utils.py` | —（工具函数集） | 提供代码提取、数据摘要生成等通用工具函数 |
| `client_utils.py` | `Client` | 统一封装各 AI 提供商（OpenAI/Azure/Claude 等）的调用 |
| `web_utils.py` | —（工具函数集） | 提供安全的网页抓取功能（含 SSRF 防护） |

---

### 之前未详细介绍的 Agent 补充说明

#### ① `CodeExplanationAgent`（代码解释助手）

**文件**：`agent_code_explanation.py`

**触发场景**：用户点击图表旁的"解释代码"按钮时。

**它做什么**：
- 接收 AI 生成的 Python/SQL 代码和数据表摘要
- 输出两部分内容：
  1. **代码步骤说明**：用自然语言描述代码做了什么（像写操作步骤清单）
  2. **新字段解释**：对代码中新计算出来的字段（如"ROI"、"归一化评分"）给出数学定义

**特点**：输出包含 LaTeX 数学公式，方便用户理解复杂指标的计算方式。

---

#### ② `ConceptDeriveAgent`（TypeScript 单列派生助手）

**文件**：`agent_concept_derive.py`

**触发场景**：用户在前端"概念（Concept）"区域输入自定义字段时（基于 TypeScript 的轻量计算）。

**它做什么**：
- 接收输入字段列表、目标新字段名、以及用户描述
- 生成一个 **TypeScript 箭头函数**，该函数以单行数据的字段值为输入，返回新字段值
- 这个函数随后在浏览器前端执行，对每一行数据做 `map()` 映射

**示例输出**：
```typescript
// 从 Date 字段提取月份
(date: string) => {
    const month = new Date(date).getMonth() + 1;
    return month;
}
```

**与 PyConceptDeriveAgent 的区别**：
- `ConceptDeriveAgent` → 生成 **TypeScript** 代码 → 在**浏览器**中运行
- `PyConceptDeriveAgent` → 生成 **Python** 代码 → 在**服务器**中运行

---

#### ③ `PyConceptDeriveAgent`（Python 单列派生助手）

**文件**：`agent_py_concept_derive.py`

**它做什么**：功能与 `ConceptDeriveAgent` 相同，但生成的是 Python 函数，运行在后端沙盒中。适用于需要更复杂计算（如正则表达式、日期解析）的场景。

**示例输出**：
```python
import re
import datetime

def derive_new_column(df):
    df['month'] = df['Date'].apply(
        lambda x: datetime.datetime.strptime(x, '%m/%d/%Y').month
    )
    return df['month']
```

---

#### ④ `SortDataAgent`（自然排序助手）

**文件**：`agent_sort_data.py`

**触发场景**：数据中有月份、年级、尺码等有"自然顺序"的分类字段，但系统无法自动判断顺序时。

**它做什么**：
- 接收一个字段名和它所有的取值列表
- 利用 AI 的常识知识，把这些值排成正确的自然顺序
- 返回排序后的数组

**示例**：
```
输入: ["April", "August", "December", "February", ...]
输出: ["January", "February", "March", "April", ..., "December"]
```

---

#### ⑤ `SQLDataRecAgent`（SQL 推荐助手）

**文件**：`agent_sql_data_rec.py`

**触发场景**：与 `PythonDataRecAgent` 相同，但数据存在 DuckDB 数据库中（SQL 模式）。

**它做什么**：
- 和 `PythonDataRecAgent` 逻辑一致：用户意图模糊时主动推荐
- 区别在于生成的是 **SQL 查询语句**，由 DuckDB 执行
- 数据摘要通过查询 DuckDB 的元数据生成，而不是读取完整数据到内存

---

#### ⑥ `SQLDataTransformationAgent`（SQL 转换助手）

**文件**：`agent_sql_data_transform.py`

**触发场景**：用户明确了图表设计（拖拽了字段），数据存在 DuckDB 中（SQL 模式）。

**它做什么**：与 `PythonDataTransformationAgent` 逻辑一致，但：
- 生成 **SQL 查询**（而非 Python 代码）
- 在 DuckDB 中以"视图（View）"的形式保存查询结果，支持更大数据量

> 这个文件是下一章深度剖析的对象，详见第十五节。

---

#### ⑦ `DataCleanAgentStream`（流式数据清洗助手）

**文件**：`agent_data_clean_stream.py`

与 `DataCleanAgent` 功能完全相同（从图片/文字/URL 提取数据），区别在于：
- 返回数据时使用**流式输出**（Streaming）
- 用户可以实时看到 AI 正在逐步生成数据表的过程，而不是等待结果全部完成

---

#### ⑧ `agent_utils.py`（通用工具函数库）

**不是一个 Agent 类**，而是整个 agents 模块共用的**工具函数集**，被其他所有 Agent 引用。

主要功能：

| 函数名 | 功能 |
|-------|------|
| `generate_data_summary()` | 把数据表格转成自然语言摘要，发给 AI 当上下文 |
| `extract_json_objects()` | 从 AI 返回的文本中解析出 JSON 对象 |
| `extract_code_from_gpt_response()` | 从 AI 返回的文本中提取代码块（```python...```） |
| `get_field_summary()` | 生成单个字段的摘要（类型 + 示例值） |
| `table_hash()` | 计算数据表的哈希值（用于判断数据是否变化） |

---

#### ⑨ `client_utils.py`（LLM 客户端封装）

封装了与各 AI 提供商通信的逻辑，提供统一接口：

```
client.get_completion(messages)  → 普通对话（返回完整响应）
client.get_completion(messages, stream=True)  → 流式对话（边生成边返回）
client.get_response(messages, tools)  → 支持工具调用的对话
```

支持的提供商：OpenAI、Azure OpenAI、Anthropic Claude、Google Gemini、Ollama（本地模型）。

---

#### ⑩ `web_utils.py`（网页工具函数库）

**不是 Agent 类**，提供安全的网页抓取能力，供 `DataCleanAgent` 使用。

核心功能：
- **SSRF 防护**：防止攻击者让服务器访问内网地址（比如 `192.168.0.1`）
- 只允许访问公网 HTTP/HTTPS 地址
- 自动解析 HTML，提取可读文本

---

## 十五、深度剖析：`agent_sql_data_transform.py` 代码结构详解

这个文件是整个项目里非常有代表性的 Agent，同时处理了"与 AI 通信"和"与数据库交互"两件事。我们把它从头到尾逐块拆解。

### 文件整体骨架

```
agent_sql_data_transform.py
├── 第①块：导入依赖
├── 第②块：SYSTEM_PROMPT（AI 的任务说明书）
├── 第③块：EXAMPLE（示例 — 教 AI 如何回答）
├── 第④块：sanitize_table_name()（安全工具函数）
├── 第⑤块：SQLDataTransformationAgent（主 Agent 类）
│   ├── __init__()   ← 初始化
│   ├── process_gpt_sql_response()  ← 解析 AI 回答 + 执行 SQL
│   ├── run()        ← 第一次调用入口
│   └── followup()   ← 追问/修改调用入口
├── 第⑥块：generate_sql_data_summary()（数据摘要生成器）
└── 第⑦块：get_sql_table_statistics_str()（单表摘要生成器）
```

---

### 第①块：导入依赖

```python
import json
import random
import string

from data_formulator.agents.agent_utils import extract_json_objects, extract_code_from_gpt_response
import pandas as pd

import logging
import re

logger = logging.getLogger(__name__)
```

**作用**：
- `json`：把 Python 对象转成 JSON 字符串（或反过来），用于和前端交换数据
- `random` + `string`：生成随机字符串，用于给 DuckDB 视图命名（避免冲突）
- `extract_json_objects`、`extract_code_from_gpt_response`：从 AI 返回的纯文本里"挖出"JSON 和 SQL 代码块
- `pandas`：把 JSON 数据转成表格（DataFrame），便于注册到 DuckDB
- `logging`：记录运行日志，方便排查问题
- `re`：正则表达式，用于清理表名中的非法字符

---

### 第②块：`SYSTEM_PROMPT`（AI 的任务说明书）

```python
SYSTEM_PROMPT = '''You are a data scientist to help user to transform data ...
```

这是整个文件最重要的部分之一。它是发给 AI 的**固定角色说明**，告诉 AI：

**① 你的身份**：你是一个数据科学家

**② 你的任务**（两步走）：

- **第一步**：理解用户目标，输出一个 JSON 对象，包含：
  - `input_tables`：需要用到哪些数据表
  - `detailed_instruction`：详细理解后的用户意图
  - `display_instruction`：简短的展示说明（显示在 UI 上）
  - `output_fields`：输出数据应该包含哪些字段
  - `chart_encodings`：图表的 x/y/color 等视觉通道映射
  - `reason`：为什么这样分析

- **第二步**：写一段 SQL 查询，查询结果就是输出数据

**③ 特殊约束**：
- 所有 SQL 必须符合 **DuckDB** 语法（与标准 SQL 略有差异）
- 日期处理需要显式类型转换（避免 DuckDB 函数重载歧义）
- 正则表达式不能用 Unicode 转义序列（DuckDB 限制）

**设计意图**：通过在提示词里给出精确的输出格式要求，保证 AI 的输出"可机器解析"——程序可以直接从 AI 的回复里提取 JSON 和 SQL，无需人工处理。

---

### 第③块：`EXAMPLE`（示例教学）

```python
EXAMPLE = '''
[CONTEXT]
...（示例数据）

[GOAL]
...（示例用户指令）

[OUTPUT]
...（示例 JSON + SQL 输出）
'''
```

这是一个**少样本示例（Few-shot example）**。把它插入提示词，相当于告诉 AI："你应该输出的格式长这个样子"。AI 通过模仿示例，能更准确地生成符合格式的输出。

> 注意：`EXAMPLE` 在这个文件里定义了，但查看代码可以发现，这个变量目前**并未**被添加到实际发送的 `messages` 中。这是一个常见的提示词工程技术决策——示例会占用大量 token（消耗费用），当 `SYSTEM_PROMPT` 本身已经足够清晰时，往往会省略示例以降低成本。`EXAMPLE` 变量在代码中保留，一方面作为开发文档说明输出格式，另一方面便于未来需要时重新启用。

---

### 第④块：`sanitize_table_name()`（安全工具函数）

```python
def sanitize_table_name(table_name: str) -> str:
    """Sanitize table name to be used in SQL queries"""
    sanitized_name = table_name.replace(" ", "_")
    sanitized_name = sanitized_name.replace("-", "_")
    sanitized_name = re.sub(r'[^a-zA-Z0-9_\.$]', '', sanitized_name)
    return sanitized_name
```

**为什么需要这个函数？**

用户上传的数据文件名可能包含空格、特殊符号（如 `sales data (Q1).csv`），这些字符在 SQL 语句中会导致语法错误。这个函数把表名转换为合法的 SQL 标识符：
- 空格和连字符 → 下划线
- 其余特殊字符（除字母、数字、`_`、`.`、`$`）→ 删除

**例子**：`"sales data (Q1)"` → `"sales_data_Q1"`

---

### 第⑤块：`SQLDataTransformationAgent` 类（主 Agent）

#### `__init__()`：初始化

```python
def __init__(self, client, conn, system_prompt=None, agent_coding_rules=""):
    self.client = client      # LLM 客户端（负责调用 AI）
    self.conn = conn          # DuckDB 数据库连接

    if system_prompt is not None:
        self.system_prompt = system_prompt
    else:
        base_prompt = SYSTEM_PROMPT
        if agent_coding_rules and agent_coding_rules.strip():
            self.system_prompt = base_prompt + "\n\n[AGENT CODING RULES]\n..." + agent_coding_rules
        else:
            self.system_prompt = base_prompt
```

**存储两个关键对象**：
1. `client`：LLM 调用接口（通过 `client_utils.py` 的 `Client` 类）
2. `conn`：DuckDB 数据库连接，用于后续执行 SQL

**系统提示词组装**：允许用户或管理员附加"编码规则"（`agent_coding_rules`）追加到提示词末尾，覆盖 AI 的默认行为（如"禁止使用子查询"）。

---

#### `process_gpt_sql_response()`：解析 AI 回答并执行 SQL

这是最复杂的方法，完整流程如下：

```
AI 返回的原始文本
    ↓
① 提取 JSON 块（extract_json_objects）
    → 得到 refined_goal（分析意图）
    ↓
② 提取 SQL 块（extract_code_from_gpt_response）
    → 得到 query_str（SQL 查询语句）
    ↓
③ 生成随机视图名（如 "view_xkqz"）
    ↓
④ 在 DuckDB 中创建视图：
   CREATE VIEW IF NOT EXISTS view_xkqz AS <query_str>
    ↓
⑤ 查询行数，若超过 5000 行则截断
    ↓
⑥ 把结果转成 JSON 格式返回
    ↓
⑦ 附带 dialog（对话历史）、agent 名称、refined_goal
```

关键代码段详解：

```python
# ③ 生成随机视图名，避免多次调用时命名冲突
random_suffix = ''.join(random.choices(string.ascii_lowercase, k=4))
table_name = f"view_{random_suffix}"

# ④ 把 AI 生成的 SQL 创建为 DuckDB 视图
# 使用视图而非普通表：视图是"虚拟表"，不实际存储数据，节省内存
create_query = f"CREATE VIEW IF NOT EXISTS {table_name} AS {query_str}"
self.conn.execute(create_query)

# ⑤ 大数据保护：超过 5000 行时自动截断，避免把海量数据传回前端
row_count = self.conn.execute(f"SELECT COUNT(*) FROM {table_name}").fetchone()[0]
if row_count > 5000:
    query_output = self.conn.execute(f"SELECT * FROM {table_name} LIMIT 5000").fetch_df()
else:
    query_output = self.conn.execute(f"SELECT * FROM {table_name}").fetch_df()

# ⑥ 结果转 JSON，附带虚拟表信息（表名 + 总行数）
result = {
    "status": "ok",
    "code": query_str,
    "content": {
        'rows': json.loads(query_output.to_json(orient='records')),
        'virtual': {
            'table_name': table_name,   # DuckDB 中的视图名
            'row_count': row_count      # 总行数（即使超过 5000 也保留真实值）
        }
    },
}
```

**为什么用"视图"而不是普通表？**
- 视图（View）是保存在数据库里的一条查询语句，不实际存储数据
- 用户后续想"修改"这个分析时，只需重新覆盖视图定义，不用复制数据
- 大数据场景下，视图比复制数据表效率高得多

---

#### `run()`：首次调用入口

```python
def run(self, input_tables, description, chart_type, chart_encodings, prev_messages=[], n=1):
```

**工作步骤**：

1. **把数据表注册到 DuckDB**（如果还没有的话）：
   ```python
   for table in input_tables:
       table_name = sanitize_table_name(table['name'])
       try:
           self.conn.execute(f"DESCRIBE {table_name}")  # 检查是否已存在
       except Exception:
           df = pd.DataFrame(table['rows'])
           self.conn.register('df_temp', df)  # 临时注册 DataFrame
           self.conn.execute(f"CREATE TABLE {table_name} AS SELECT * FROM df_temp")
           # 把前端传来的 JSON 数据持久化为 DuckDB 表
   ```

2. **生成数据摘要**（发给 AI 当上下文）：
   ```python
   data_summary = generate_sql_data_summary(self.conn, input_tables)
   ```

3. **组装用户目标**（Goal）：
   ```python
   goal = {
       "instruction": description,      # 用户的自然语言描述
       "chart_type": chart_type,        # 图表类型（bar/line/scatter...）
       "chart_encodings": chart_encodings,  # X/Y/Color 轴映射
   }
   user_query = f"[CONTEXT]\n\n{data_summary}\n\n[GOAL]\n\n{json.dumps(goal)}"
   ```

4. **组装完整消息并调用 AI**：
   ```python
   messages = [
       {"role": "system", "content": self.system_prompt},  # 角色说明
       *filtered_prev_messages,                              # 历史对话（如有）
       {"role": "user", "content": user_query}              # 当前请求
   ]
   response = self.client.get_completion(messages=messages)
   ```

5. **处理 AI 回答**：
   ```python
   return self.process_gpt_sql_response(response, messages)
   ```

---

#### `followup()`：追问/修改调用入口

```python
def followup(self, input_tables, dialog, latest_data_sample, 
             chart_type, chart_encodings, new_instruction, n=1):
```

**使用场景**：用户在看到图表后说"改一下，只看2023年的数据"，系统需要在之前的 SQL 基础上修改。

**与 `run()` 的核心区别**：

```python
# run() 发送全新的对话
messages = [system_prompt, user_query]

# followup() 在已有对话历史上追加新指令
updated_dialog = [system_prompt, *dialog[1:]]  # 保留历史对话（替换开头的 system）
messages = [
    *updated_dialog,
    {"role": "user",
     "content": f"This is the result from the latest sql query:\n\n{sample_data_str}\n\n"
                f"Update the sql query above based on the following instruction:\n\n{goal}"}
]
```

**为什么要带上"上一次的结果数据"？**
让 AI 看到上次 SQL 运行后的实际数据样本，有助于 AI 更准确地理解当前数据状态，从而生成更有针对性的修改。

---

### 第⑥块：`generate_sql_data_summary()`（数据摘要生成器）

```python
def generate_sql_data_summary(conn, input_tables, ...):
```

**作用**：把 DuckDB 中存储的表格信息汇总成自然语言描述，作为"上下文"发给 AI。

输出格式示例：

````
## Table 1: sales_data (10,000 rows × 5 columns)

### Schema (5 fields)
  - city -- type: VARCHAR, values: 北京, 上海, 广州, ...
  - date -- type: DATE, values: 2024-01-01, 2024-01-02, ...
  - sales -- type: INTEGER, range: [1000, 500000]

### Sample Data (first 5 rows)
```
...（表格前5行）
```
````

**关键设计**：只发摘要不发完整数据。即使数据库里有 100 万行数据，发给 AI 的也只是字段名、类型、几个示例值和5行样本——这样既节省 token，又让 AI 有足够的上下文。

---

### 第⑦块：`get_sql_table_statistics_str()`（单表摘要生成器）

```python
def get_sql_table_statistics_str(conn, table_name, ...):
```

**作用**：生成单张表的详细统计摘要。

**对不同字段类型的处理差异**：

```python
if col_type in ['INTEGER', 'BIGINT', 'DOUBLE', ...]:
    # 数字类型：查最小值和最大值，展示为区间
    # 例：range: [1000, 500000]
    range_result = conn.execute(f"SELECT MIN({col}), MAX({col}) FROM {table}").fetchone()
else:
    # 文字类型：查最多 14 个不同值作为示例
    # 例：values: 北京, 上海, 广州, ..., 成都, 武汉
    sample_values = conn.execute(f"SELECT DISTINCT {col} ... LIMIT 14").fetchall()
```

**设计意图**：数字字段用"范围"描述，文字字段用"枚举示例"描述，两种方式都能让 AI 快速理解该字段的数据分布，而不需要发送完整数据。

---

### 整个 `agent_sql_data_transform.py` 的数据流汇总

```
前端发来请求
  input_tables = [{"name": "sales", "rows": [...]}]
  description = "显示各城市销售额"
  chart_type = "bar"
  chart_encodings = {"x": "city", "y": "total_sales"}
         ↓
① 把 JSON 数据注册进 DuckDB（CREATE TABLE sales AS ...）
         ↓
② 查询 DuckDB 元数据，生成数据摘要文本
         ↓
③ 组装提示词（SYSTEM_PROMPT + 数据摘要 + 用户目标）
         ↓
④ 调用 LLM（GPT-4o / Claude 等）
         ↓
⑤ LLM 返回文本，包含：
   - JSON：{"input_tables": [...], "output_fields": [...], "chart_encodings": {...}}
   - SQL：SELECT city, SUM(sales) AS total_sales FROM sales GROUP BY city
         ↓
⑥ 在 DuckDB 中创建视图：CREATE VIEW view_xkqz AS <SQL>
         ↓
⑦ 查询视图，取最多 5000 行结果
         ↓
⑧ 把结果转成 JSON 返回给前端
         ↓
前端用 Vega-Lite 渲染出柱状图
```

这就是一个 SQL Agent 完整的"从用户输入到图表显示"的全过程。
