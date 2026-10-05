# 技能编写最佳实践（Skill authoring best practices）

> 怎么写让代理能发现、也能用起来的技能。

好的技能简洁、结构清楚，并且经过真实使用验证。本文给出实用的编写决策，帮代理发现并有效使用技能。

配套读物：技能机制的背景见本目录 SKILL.md 的“什么是技能”“技能发现优化”两节；TDD 视角的创作流程（RED-GREEN-REFACTOR）也见 SKILL.md。

## 核心原则

### 简洁是关键

上下文窗口是公共资源。你的技能要和代理需要知道的其他一切共享它，包括：

- 系统提示
- 对话历史
- 其他技能的元数据
- 实际任务本身

技能里的 token 并非每个都有即时成本。启动时只预加载元数据（name 与 description）；技能变得相关时代理才读 SKILL.md，需要时才继续读附加文件。但 SKILL.md 里保持简洁仍然重要：一旦被加载，每个 token 都在和对话历史及其他上下文竞争。

**默认假设：代理已经非常聪明**

只补充代理本来不知道的上下文。对每一条信息都要追问：

- “代理真的需要这个解释吗？”
- “我能假定代理已经知道这个吗？”
- “这段文字对得起它花的 token 吗？”

**好例子：简洁**（约 50 个 token）：

````markdown
## 提取 PDF 文本

用 pdfplumber 做文本提取：

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**坏例子：太啰嗦**（约 150 个 token）：

```markdown
## 提取 PDF 文本

PDF（Portable Document Format）是一种常见文件格式，包含文本、图像和其他内容。
要从 PDF 提取文本，你需要用一个库。PDF 处理库有很多，我们推荐 pdfplumber，
因为它易用且能覆盖大多数情况。首先你需要用 pip 安装。然后可以用下面的代码……
```

简洁版本假定代理知道 PDF 是什么、库是怎么工作的。

### 设定合适的自由度

让具体程度匹配任务的脆弱性与可变性。

**高自由度**（基于文字的指令）：

适用：

- 多种做法都可行
- 决策依赖上下文
- 靠启发式引导做法

例子：

```markdown
## 代码评审流程

1. 分析代码结构与组织
2. 检查潜在 bug 与边界情况
3. 就可读性与可维护性提改进建议
4. 核实是否符合项目约定
```

**中自由度**（伪代码，或带参数的脚本）：

适用：

- 存在偏好模式
- 允许一定变化
- 配置会影响行为

例子：

````markdown
## 生成报告

用这个模板，按需调整：

```python
def generate_report(data, format="markdown", include_charts=True):
    # 处理数据
    # 按指定格式产出
    # 可选地包含可视化
```
````

**低自由度**（具体脚本，参数很少或没有）：

适用：

- 操作脆弱易错
- 一致性至关重要
- 必须遵循特定顺序

例子：

````markdown
## 数据库迁移

原样执行这个脚本（由主代理用 terminal 执行）：

```bash
python scripts/migrate.py --verify --backup
```

不要改命令，也不要加额外参数。
````

**类比**：把代理想成沿路径探索的机器人：

- **两侧是悬崖的窄桥**：只有一条安全通路。给具体的护栏和精确指令（低自由度）。例子：必须严格按序执行的数据库迁移。
- **没有危险的旷野**：多条路都能成功。给大致方向，让代理自己找最佳路线（高自由度）。例子：上下文决定最佳做法的代码评审。

### 用真实任务测试，覆盖你打算用的所有配置

技能是模型的附加物，效果取决于底层模型与运行配置。用你打算用到的各种配置都测一遍：worker 由用户配置决定，所以不要假定某一种配置就能代表全部。

**测试时注意：**
- 能力较弱的配置下，技能给的信息够不够？
- 能力较强的配置下，技能是不是解释过头了？

跨越多种配置使用时，目标写成对所有配置都好用的指令。

## 技能结构

**YAML frontmatter**：SKILL.md 的 frontmatter 需要两个字段：

- `name` —— 技能的可读名称（最长 64 字符）
- `description` —— 一句话说明技能做什么、何时使用（最长 1024 字符）

### 命名约定

用一致的命名模式，便于在引用和讨论时辨认。推荐用**动名词形式**（动词 + -ing）给技能命名，因为它清楚描述技能提供的活动或能力。

**好的命名（动名词）：**

- “Processing PDFs”
- “Analyzing spreadsheets”
- “Managing databases”
- “Testing code”
- “Writing documentation”

**也可接受：**

- 名词短语：“PDF Processing”、“Spreadsheet Analysis”
- 动作导向：“Process PDFs”、“Analyze Spreadsheets”

**避免：**

- 含糊的名字：“Helper”、“Utils”、“Tools”
- 过于宽泛：“Documents”、“Data”、“Files”
- 技能集合内部命名模式不一致

一致的命名让这些事更容易：在文档和对话中引用技能、一眼看出技能做什么、组织和检索多个技能、维护一个专业而统一的技能库。

### 写出有效的 description

`description` 字段决定技能能否被发现，应当同时包含技能做什么与何时使用。

**永远用第三人称。** 它会被注入系统提示，人称不一致会造成发现问题。

- **好：** “Processes Excel files and generates reports”
- **避免：** “I can help you process Excel files”
- **避免：** “You can use this to process Excel files”

**具体，并包含关键术语。** 既写技能做什么，也写具体的触发条件/上下文。

每个技能只有一个 description 字段。它极其关键：代理要从可能上百个技能里挑出对的那个。description 必须给出足够信息，让代理知道何时该选这个技能，而 SKILL.md 的其余部分提供实现细节。

有效示例：

**PDF 处理技能：**

```yaml
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel 分析技能：**

```yaml
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git 提交信息技能：**

```yaml
description: Generate descriptive commit messages by analyzing git diffs. Use when the user asks for help writing commit messages or reviewing staged changes.
```

避免这类含糊的 description：

```yaml
description: Helps with documents
```

```yaml
description: Processes data
```

```yaml
description: Does stuff with files
```

**注意：** 上头这些“what + when”的官方例子与本仓库 `writing-skills` 技能的 SDO 一节有张力。那个技能实测发现：description 里概括工作流会让代理照 description 做、而不去读技能正文。实践取法：本仓库技能的 description **以触发条件为主**，可以对行为做最小说明，但绝不要概括流程步骤。

### 渐进披露（Progressive disclosure）

SKILL.md 是一份总览，按需把代理引向细节材料，像入职指南里的目录。

**实用建议：**

- SKILL.md 正文保持在 500 行以内
- 接近这个上限就拆成独立文件
- 用下面的模式组织指令、代码和资源

#### 从简单到复杂

最基础的技能只有一个含元数据与指令的 SKILL.md：

```
pdf/
├── SKILL.md              # 主指令（被触发时加载）
```

技能长大后，可以附带只在需要时加载的内容：

```
pdf/
├── SKILL.md              # 主指令（被触发时加载）
├── FORMS.md              # 表单填写指南（按需加载）
├── reference.md          # API 参考（按需加载）
├── examples.md           # 用法示例（按需加载）
└── reference/            # 分域参考
```

#### 模式 1：高层指南 + 参考文件

````markdown
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing

## 快速开始

用 pdfplumber 提取文本：
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## 进阶功能

**表单填写**：完整指南见 [FORMS.md](FORMS.md)
**API 参考**：全部方法见 [REFERENCE.md](REFERENCE.md)
**示例**：常见模式见 [EXAMPLES.md](EXAMPLES.md)
````

代理只在需要时才读 FORMS.md、REFERENCE.md 或 EXAMPLES.md。

#### 模式 2：按领域组织

有多个领域的技能，按领域组织内容，避免加载无关上下文。这样 token 用量低、上下文聚焦。

```
bigquery-skill/
├── SKILL.md（总览与导航）
└── reference/
    ├── finance.md（收入、计费指标）
    ├── sales.md（商机、管道）
    ├── product.md（API 用法、功能）
    └── marketing.md（活动、归因）
```

````markdown
# BigQuery 数据分析

## 可用数据集

**财务**：收入、ARR、计费 → 见 [reference/finance.md](reference/finance.md)
**销售**：商机、管道、客户 → 见 [reference/sales.md](reference/sales.md)
**产品**：API 用法、功能、采用率 → 见 [reference/product.md](reference/product.md)
**市场**：活动、归因、邮件 → 见 [reference/marketing.md](reference/marketing.md)

## 快速检索

按关键词定位具体指标：用 `read_file` 打开对应参考文件后按关键词检索（需要按文件批量检索时，
由主代理用 terminal 执行再交回结果）。
````

#### 模式 3：条件式细节

先给基础内容，再链接进阶内容：

```markdown
# DOCX 处理

## 创建文档

新文档用 docx-js。见 [DOCX-JS.md](DOCX-JS.md)。

## 编辑文档

简单编辑直接改 XML。

**修订痕迹**：见 [REDLINING.md](REDLINING.md)
**OOXML 细节**：见 [OOXML.md](OOXML.md)
```

只有用户需要这些功能时，代理才读 REDLINING.md 或 OOXML.md。

### 避免过深的引用层级

被引用的文件再引用别的文件时，代理可能只部分读取。遇到嵌套引用，它可能只预览一部分而不读全文，结果信息不完整。

**引用保持离 SKILL.md 只有一层。** 所有参考文件都应当直接从 SKILL.md 链接，确保代理需要时读到完整文件。

**坏例子：太深**：

```markdown
# SKILL.md
见 [advanced.md](advanced.md)...

# advanced.md
见 [details.md](details.md)...

# details.md
这里才是真正的信息...
```

**好例子：一层**：

```markdown
# SKILL.md

**基本用法**：[指令写在 SKILL.md 里]
**进阶功能**：见 [advanced.md](advanced.md)
**API 参考**：见 [reference.md](reference.md)
**示例**：见 [examples.md](examples.md)
```

### 长参考文件加目录

超过 100 行的参考文件，在顶部加目录。这样即使只读到一部分，代理也能看到全部可用信息的范围。

**例子**：

```markdown
# API 参考

## 目录
- 认证与初始化
- 核心方法（增删改查）
- 进阶功能（批量操作、webhook）
- 错误处理模式
- 代码示例

## 认证与初始化
...

## 核心方法
...
```

这样代理可以整份读完，也可以跳到具体章节。

## 工作流与反馈回环

### 复杂任务用工作流

把复杂操作拆成清楚的顺序步骤。特别复杂的流程，给一份代理可以复制到回复里逐项打勾的清单（Eta 没有 todo 工具，清单就写在回复里，用编号列表跟踪）。

**例子 1：研究综合工作流**（不含代码的技能）：

````markdown
## 研究综合工作流

把这份清单复制到回复里并跟踪进度：

```
研究进度：
- [ ] 步骤 1：读完所有源文档
- [ ] 步骤 2：归纳关键主题
- [ ] 步骤 3：交叉核对主张
- [ ] 步骤 4：产出结构化摘要
- [ ] 步骤 5：核实引用
```

**步骤 1：读完所有源文档**

逐份阅读 `sources/` 目录下的文档。记下主要论点与支撑证据。

**步骤 2：归纳关键主题**

跨源找模式：哪些主题反复出现？来源在哪里一致、哪里冲突？

**步骤 3：交叉核对主张**

每条主要主张都要核实它确实出现在源材料里。记下每一点由哪份资料支撑。

**步骤 4：产出结构化摘要**

按主题组织结论，包含：
- 主要主张
- 来自来源的支撑证据
- 冲突观点（如有）

**步骤 5：核实引用**

检查每条主张引用的是不是正确的源文档。引用不全就回到步骤 3。
````

这个例子说明工作流也适用于不需要代码的分析任务。清单模式对任何复杂的多步流程都有效。

**例子 2：PDF 表单填写工作流**（含代码的技能）：

````markdown
## PDF 表单填写工作流

把清单复制到回复里，做完一项勾一项：

```
任务进度：
- [ ] 步骤 1：分析表单（主代理用 terminal 运行 analyze_form.py）
- [ ] 步骤 2：建立字段映射（编辑 fields.json）
- [ ] 步骤 3：校验映射（运行 validate_fields.py）
- [ ] 步骤 4：填写表单（运行 fill_form.py）
- [ ] 步骤 5：验证输出（运行 verify_output.py）
```

**步骤 1：分析表单**

运行：`python scripts/analyze_form.py input.pdf`（由主代理用 terminal 执行；子代理不能执行 shell）

这会提取表单字段及其位置，存成 `fields.json`。

**步骤 2：建立字段映射**

编辑 `fields.json`，为每个字段加值。

**步骤 3：校验映射**

运行：`python scripts/validate_fields.py fields.json`

校验错误必须先修掉再继续。

**步骤 4：填写表单**

运行：`python scripts/fill_form.py input.pdf fields.json output.pdf`

**步骤 5：验证输出**

运行：`python scripts/verify_output.py output.pdf`

验证失败就回到步骤 2。
````

清楚的步骤能防止代理跳过关键校验。清单让你和代理都能跟踪多步流程的进度。

### 实现反馈回环

**常见模式**：跑校验 → 修错误 → 再来一遍

这个模式能大幅提升产出质量。

**例子 1：风格指南合规**（不含代码的技能）：

```markdown
## 内容评审流程

1. 按 STYLE_GUIDE.md 的准则起草内容
2. 对照清单评审：
   - 检查术语一致性
   - 核实示例符合标准格式
   - 确认所有必需章节都在
3. 发现问题：
   - 逐条记录问题并指明具体章节
   - 修改内容
   - 再过一遍清单
4. 所有要求满足后才继续
5. 定稿保存
```

这个例子展示用参考文档（而不是脚本）做校验回环：“校验器”是 STYLE_GUIDE.md，代理通过阅读比对来完成检查。

**例子 2：文档编辑流程**（含代码的技能）：

```markdown
## 文档编辑流程

1. 编辑 `word/document.xml`
2. **立即校验**：`python ooxml/scripts/validate.py unpacked_dir/`（主代理用 terminal 执行）
3. 校验失败：
   - 仔细看错误信息
   - 修 XML 里的问题
   - 再跑一次校验
4. **校验通过才继续**
5. 重新打包：`python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. 测试产出文档
```

校验回环能尽早抓住错误。

## 内容准则

### 不要写会过期的时间敏感信息

**坏例子：时间敏感**（以后会变成错的）：

```markdown
如果你在 2025 年 8 月之前做这件事，用旧 API。
2025 年 8 月之后用新 API。
```

**好例子**（用“旧模式”一节收纳）：

```markdown
## 当前做法

使用 v2 API 端点：`api.example.com/v2/messages`

## 旧模式

<details>
<summary>旧版 v1 API（2025-08 起废弃）</summary>

v1 API 用法：`api.example.com/v1/messages`

该端点已不再支持。
</details>
```

“旧模式”一节保留历史上下文，又不弄乱正文。

### 术语保持一致

选一个说法并贯穿整个技能：

**好 —— 一致：**

- 始终用“API 端点”
- 始终用“字段”
- 始终用“提取”

**坏 —— 不一致：**

- “API 端点”、“URL”、“API 路由”、“路径”混用
- “字段”、“输入框”、“元素”、“控件”混用
- “提取”、“拉取”、“获取”、“取出”混用

一致性帮代理理解并遵循指令。

## 常见模式

### 模板模式

为输出格式提供模板。严格程度按需要匹配。

**严格要求时**（比如 API 响应或数据格式）：

````markdown
## 报告结构

始终使用这个确切的模板结构：

```markdown
# [分析标题]

## 执行摘要
[一段话概述关键发现]

## 关键发现
- 发现 1，附支撑数据
- 发现 2，附支撑数据
- 发现 3，附支撑数据

## 建议
1. 具体的可执行建议
2. 具体的可执行建议
```
````

**灵活指导时**（适配更有用时）：

````markdown
## 报告结构

下面是一个合理的默认格式，按分析类型自行判断：

```markdown
# [分析标题]

## 执行摘要
[概述]

## 关键发现
[按你的发现调整章节]

## 建议
[贴合具体上下文剪裁]
```

按具体分析类型调整章节。
````

### 示例模式

当产出质量取决于“见过例子”时，给输入/输出对，就像平时做提示那样：

````markdown
## 提交信息格式

按下例生成提交信息：

**示例 1：**
输入：为 JWT 令牌加了用户认证
输出：
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```

**示例 2：**
输入：修复报告里日期显示错误的问题
输出：
```
fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation
```

**示例 3：**
输入：升级依赖并重构错误处理
输出：
```
chore: update dependencies and refactor error handling

- Upgrade lodash to 4.17.21
- Standardize error response format across endpoints
```

遵循这个风格：type(scope): 简述，然后详细说明。
````

例子比单纯描述更能让代理明白想要的风格与详细程度。

### 条件工作流模式

在决策点上引导代理：

```markdown
## 文档修改工作流

1. 判断修改类型：

   **创建新内容？** → 走下面的“创建工作流”
   **修改已有内容？** → 走下面的“编辑工作流”

2. 创建工作流：
   - 使用 docx-js 库
   - 从零构建文档
   - 导出为 .docx

3. 编辑工作流：
   - 解包已有文档
   - 直接改 XML
   - 每次改动后校验
   - 完成后重新打包
```

如果工作流步骤太多太复杂，考虑把它们拆到独立文件里，并告诉代理按任务类型读对应文件。

## 评估与迭代

### 先建评估

**在写大量文档之前先建评估。** 这保证技能解决的是真实问题，而不是你想象出来的问题。

**评估驱动开发：**

1. **找出缺口**：让代理在**没有技能**的情况下做代表性任务。记录具体失败或缺失的上下文
2. **建立评估**：为这些缺口写三个场景
3. **建立基线**：测量没有技能时代理的表现
4. **写最小指令**：只写够覆盖缺口、通过评估的内容
5. **迭代**：跑评估、和基线对比、继续打磨

这样你解决的是实际问题，而不是永远不会出现的需求。

**评估结构**：

```json
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file and save it to output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "Successfully reads the PDF file using an appropriate PDF processing library or command-line tool",
    "Extracts text content from all pages in the document without missing any pages",
    "Saves the extracted text to a file named output.txt in a clear, readable format"
  ]
}
```

**说明：** 这个例子演示的是数据驱动的评估与简单的测试标准。平台不提供运行这些评估的内置方式；评估由你自己组织（例如由主代理派子代理跑场景，再用 `get_task_result` 读结论）。评估是你衡量技能有效性的唯一真相来源。

### 与代理一起迭代开发技能

最有效的技能开发流程会把代理本身拉进来。用主代理（或一个专门用来打磨技能的子代理）负责设计和改进指令，用真实干活的子代理去测试它们。之所以有效，是因为模型既懂怎么写有效的代理指令，也懂代理需要什么信息。

**创建一个新技能：**

1. **先不带技能完成一次任务**：用常规方式把问题做一遍。过程中你会自然提供上下文、解释偏好、分享流程性知识。注意你反复提供的是哪些信息。

2. **提炼可复用的模式**：任务完成后，找出哪些上下文对未来的类似任务有用。

   **例子**：如果你刚做了一次 BigQuery 分析，你可能提供了表名、字段定义、过滤规则（比如“永远排除测试账号”）和常见查询模式。

3. **让代理起草技能**：把要沉淀的内容说清楚 —— “把我们刚才这次 BigQuery 分析的套路沉淀成技能，包含表结构、命名约定，以及过滤测试账号的规则。”

   **提示：** 现在的代理本身就理解技能格式与结构。你不需要特殊系统提示，也不需要某个“写技能”的技能来帮你创建技能；直接让它写，它就会产出结构正确、frontmatter 齐备的 SKILL.md。

4. **检查简洁性**：确认没有加多余解释。问它：“把‘胜率是什么意思’那段解释删掉 —— 代理已经知道了。”

5. **改进信息架构**：让它重新组织内容。例如：“把表结构放到单独的参考文件里，以后可能还要加表。”

6. **在类似任务上测试**：让一个全新的、加载了这个技能的子代理去做相关的活，观察它能不能找到正确信息、正确应用规则、顺利完成。

7. **根据观察迭代**：如果它卡住或漏了东西，带着具体观察回去改：“上次用这个技能时，它忘了按日期过滤 Q4。要不要加一节讲日期过滤模式？”

**改进已有技能：**

同样的分层模式：主代理（或专用子代理）负责打磨技能，干活子代理负责用技能做真活，你负责观察并回流观察结果。

1. **在真实工作流里使用技能**：给子代理真实的活，而不是测试场景
2. **观察行为**：记下它在哪里卡住、哪里做得好、哪里做了意外选择

   **例观察**：“我让它出一份区域销售报表，它写了查询，但忘了过滤测试账号，尽管技能里写了这条规则。”

3. **回去改进**：把当前 SKILL.md 和观察结果一起给它，问：“我注意到它做区域报表时忘了过滤测试账号。技能里确实提了过滤，也许不够显眼？”

4. **评估建议**：它可能建议重新组织让规则更突出、把“always filter”换成更强的“MUST filter”，或者重构工作流章节

5. **应用并测试**：按建议更新技能，再在类似请求上测一次
6. **重复**：遇到新场景就继续这个“观察—改进—测试”的循环。每一轮都基于真实行为改进技能，而不是靠猜

**收集用户反馈：**

1. 把技能分享给用户与协作者，观察使用情况
2. 问：技能在预期的时候会被触发吗？指令清楚吗？缺什么？
3. 把反馈纳入进来，补上你自己使用中的盲点

**为什么这套做法有效**：代理理解代理的需要，你提供领域专长，干活子代理用真实使用暴露缺口，迭代基于观察到的行为而不是假设。

### 观察代理怎么用技能

迭代时注意代理实际怎么用：

- **意外的探索路径**：它读文件的顺序和你预想的不同？可能你的结构没你想的那么直观
- **错过的连接**：它没跟着引用去读重要文件？链接可能要更明确、更显眼
- **对某些章节过度依赖**：如果它反复读同一个文件，考虑那部分内容是不是该放回主 SKILL.md
- **被忽略的内容**：如果它从不读某个附带文件，那这个文件可能没必要，或者在主指令里信号太弱

按这些观察迭代，不要按假设迭代。元数据里的 `name` 和 `description` 尤其关键：代理正是靠它们决定要不要响应当前任务触发这个技能。确保它们清楚说明技能做什么、何时该用。

## 要避免的反模式

### 避免 Windows 风格路径

路径一律用正斜杠：

- ✓ **好**：`scripts/helper.py`、`reference/guide.md`
- ✗ **避免**：`scripts\helper.py`、`reference\guide.md`

正斜杠在所有平台上都能用，反斜杠在类 Unix 系统上会报错。

### 避免一次给太多选项

不是必须就不要摆出多种做法：

````markdown
**坏例子：选项太多**（让人困惑）：
“你可以用 pypdf，或者 pdfplumber，或者 PyMuPDF，或者 pdf2image，或者……”

**好例子：给一个默认值**（附逃生口）：
“文本提取用 pdfplumber：
```python
import pdfplumber
```

需要 OCR 的扫描版 PDF，改用 pdf2image 加 pytesseract。”
````

## 进阶：带可执行代码的技能

下面几节针对含可执行脚本的技能。只用 Markdown 指令的技能可以跳到“有效技能清单”。

**Eta 前提：** 子代理不能执行 shell。技能里一切命令类步骤，都要写成“由主代理用 terminal 执行，结果交给子代理”的形式。技能作者不要写“让子代理运行这个脚本”。

### 让脚本解决问题，而不是把问题踢回去

写技能脚本时，处理错误条件，而不是把问题抛给代理。

**好例子：显式处理错误**：

```python
def process_file(path):
    """处理文件；不存在就创建。"""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # 用默认内容创建文件，而不是直接失败
        print(f"File {path} not found, creating default")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # 给替代方案，而不是直接失败
        print(f"Cannot access {path}, using default")
        return ''
```

**坏例子：把问题踢回给代理**：

```python
def process_file(path):
    # 直接失败，让代理自己想办法
    return open(path).read()
```

配置参数也要有理由、有说明，避免“玄学常数”。如果你都不知道正确的值，代理怎么判断？

**好例子：自解释**：

```python
# HTTP 请求通常 30 秒内完成
# 更长超时是为了照顾慢连接
REQUEST_TIMEOUT = 30

# 三次重试在可靠性与速度之间取平衡
# 大多数偶发失败在第二次重试就恢复
MAX_RETRIES = 3
```

**坏例子：魔法数字**：

```python
TIMEOUT = 47  # 为什么是 47？
RETRIES = 5   # 为什么是 5？
```

### 提供工具脚本

即使代理能自己写脚本，预先写好的脚本也有优势：

**工具脚本的好处：**

- 比自己生成的代码可靠
- 省 token（不需要把代码放进上下文）
- 省时间（不需要现场生成代码）
- 保证多次使用结果一致

**重要的区分**：在指令里明确代理该**怎么**对待脚本（在 Eta 里，实际执行者是主代理）：

- **执行**（最常见）：“运行 `analyze_form.py` 提取字段”（由主代理用 terminal）
- **当参考读**（复杂逻辑）：`analyze_form.py` 里有字段提取算法，可以读它

工具脚本大多以执行为主，因为更可靠、更高效。

**例子**：

````markdown
## 工具脚本

**analyze_form.py**：提取 PDF 里所有表单字段（主代理用 terminal 执行）

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

输出格式：
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**：检查边界框是否重叠

```bash
python scripts/validate_boxes.py fields.json
# 返回 "OK" 或列出冲突
```

**fill_form.py**：把字段值写进 PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### 用视觉分析

输入可以渲染成图片时，让代理看图分析：

````markdown
## 表单版面分析

1. 把 PDF 转成图片（主代理用 terminal 执行）：
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. 逐页分析图片，识别表单字段
3. 代理可以直观看到字段位置与类型
````

**说明：** 这个例子里你需要自己写 `pdf_to_images.py`。

代理的视觉能力有助于理解版面与结构。

### 产出可校验的中间结果

代理做复杂、开放式任务时会犯错。“计划—校验—执行”模式让代理先用结构化格式写出计划，用脚本校验后再执行，从而尽早抓错。

**例子**：让代理按表格更新 PDF 里 50 个字段。没有校验，它可能引用不存在的字段、写出冲突的值、漏掉必填字段，或者应用错了。

**做法**：用上面的工作流模式，但插入一个中间文件 `changes.json`，在应用改动之前先校验。流程变成：分析 → **生成计划文件** → **校验计划** → 执行 → 验证。

**为什么有效：**

- **尽早抓错**：改动落地前校验就发现问题
- **机器可验证**：脚本给出客观验证
- **计划可回退**：代理可以不碰原件地反复改计划
- **便于调试**：错误信息指向具体问题

**什么时候用**：批量操作、破坏性改动、复杂校验规则、高风险操作。

**实现提示**：让校验脚本输出详细、具体的错误信息，比如“Field 'signature_date' not found. Available fields: customer_name, order_total, signature_date_signed”，方便代理修正。

### 声明依赖

技能在运行时环境里执行，存在平台相关限制：

- 本机（Linux/Debian）已装：git、python3、pip3、uv、node、npm、java、jadx、apktool、curl、ssh
- **没有 gh**；也没有 macOS/Windows 专有工具

把需要的包写进 SKILL.md，并核实它们确实可用（可用性核查由主代理用 terminal 完成）。

### 运行时环境

技能运行在有文件系统访问与命令执行能力的环境里。这对编写方式的影响：

**代理如何访问技能：**

1. **元数据预加载**：启动时，所有技能 frontmatter 里的 name 与 description 被加载进系统提示
2. **文件按需读取**：代理用 `read_file` 在自己的 worktree 里读 SKILL.md 及其他文件
3. **命令由主代理执行**：脚本不能由子代理运行，由主代理用 `terminal(environment=android/linux)` 执行后把输出交给子代理
4. **大文件没有上下文成本**：参考文件、数据、文档在被真正读取前不消耗上下文 token

**编写要点：**

- **路径很重要**：用正斜杠（`reference/guide.md`），不要用反斜杠
- **文件名要有描述性**：用能表明内容的命名，如 `form_validation_rules.md`，而不是 `doc2.md`
- **按发现需求组织**：按领域或功能组织目录
  - 好：`reference/finance.md`、`reference/sales.md`
  - 坏：`docs/file1.md`、`docs/file2.md`
- **把完整资源附上**：完整 API 文档、大量示例、大数据集都可以带，未被访问时不占上下文
- **确定性操作用脚本**：写 `validate_form.py`，而不是让代理现场生成校验代码
- **把执行意图写清楚**：
  - “运行 `analyze_form.py` 提取字段”（执行 → 主代理）
  - “`analyze_form.py` 里有提取算法，可读作参考”（代理读取）
- **测试文件访问模式**：用真实请求验证代理能按你的目录结构找到东西

**例子：**

```
bigquery-skill/
├── SKILL.md（总览，指向参考文件）
└── reference/
    ├── finance.md（收入指标）
    ├── sales.md（管道数据）
    └── product.md（使用分析）
```

用户问收入相关问题时，代理读 SKILL.md，看到 `reference/finance.md` 的引用，只读那个文件。sales.md 与 product.md 留在文件系统上，直到需要前占用零上下文 token。这种基于文件系统的模型正是渐进披露的实现基础。

### 不要假定工具已安装

````markdown
**坏例子：假定已安装**：
“用 pdf 库处理这个文件。”

**好例子：明确依赖**：
“先装依赖：`pip install pypdf`（由主代理用 terminal 执行）

然后使用：
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```”
````

## 技术说明

### YAML frontmatter 要求

SKILL.md 的 frontmatter 需要 `name`（最长 64 字符）与 `description`（最长 1024 字符）两个字段。

### Token 预算

SKILL.md 正文保持在 500 行以内以获得最佳表现。超出就按前面讲的渐进披露模式拆成独立文件。

## 有效技能清单

分享技能之前，核实：

### 核心质量

- [ ] description 具体，包含关键术语
- [ ] description 既写技能做什么、也写何时使用
- [ ] SKILL.md 正文在 500 行以内
- [ ] 附加细节放在独立文件里（如需）
- [ ] 没有时间敏感信息（或已放进“旧模式”一节）
- [ ] 术语全文一致
- [ ] 例子具体，不抽象
- [ ] 文件引用只有一层
- [ ] 恰当使用渐进披露
- [ ] 工作流步骤清楚

### 代码与脚本

- [ ] 脚本解决问题，而不是把问题踢回给代理
- [ ] 错误处理明确、有帮助
- [ ] 没有“玄学常数”（所有值都有理由）
- [ ] 需要的包已在指令中列出，并确认可用
- [ ] 脚本有清楚的文档
- [ ] 路径全用正斜杠
- [ ] 关键操作有校验/验证步骤
- [ ] 质量关键的环节有反馈回环
- [ ] 一切命令类步骤都写明由主代理用 terminal 执行（子代理不能执行 shell）

### 测试

- [ ] 至少建了三个评估
- [ ] 在多种运行配置（不同 worker 配置）下测过
- [ ] 用真实使用场景测过
- [ ] 用户反馈已纳入（如有）
