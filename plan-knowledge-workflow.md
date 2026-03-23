# Knowledge Workflow — 知识工作自动化元工作流实施计划

## 一、设计总纲

### 核心理念

Knowledge-Workflow 是一个**知识工作自动化元工作流**。用户提交需求后，Claude 全自动完成从主题调研到分析构建到内容产出到质量审核的完整流程。人类介入被压缩到最少：只在需求确认阶段和最终交付时参与。

**定义**：输入是需求/问题/素材，输出是结构化文本（报告、指南、分析、规划等），中间需要调研、分析、组织、写作。边界判定——只要最终产出是"文档"而非"可运行的代码"，都属于此范畴。

### 与 Development-Workflow（dev 工作流）的关系

Knowledge-Workflow 是独立项目，不是 dev 工作流的子模块。两者共享相同的**架构模式**（A/B/C 三阶段、Subagent 串行调度、Agent Team 并行执行、检查点恢复、8 轮安全阀），但在每个阶段的具体内容、模板、检查清单、Agent 角色上完全不同。

独立而非合并的理由：
- Agent 指令不应做"双模式切换"——塞两套逻辑进一个文件只会让指令冗长脆弱
- 根 CLAUDE.md 加路由层会复杂度陡增，影响 Lead 执行质量
- 真正共享的部分（Git 策略、安全阀、检查点）是机制性模式，复制即可，不值得为此引入路由层

### 场景全景

Knowledge Workflow 覆盖的场景分类：

**调研分析类**
- 深度调研报告、竞品分析、市场调研
- 可行性评估（商业计划、投资标的）
- SWOT 分析、风险评估、政策影响分析

**策略规划类**
- 参会指南、策略分析、立场文件
- 日程规划、OKR/KPI 体系设计
- 流程 SOP 编写、项目复盘报告

**教育培训类**
- 课程大纲、教学计划、学习路径规划
- 出题、出试卷、案例分析编写
- 知识图谱梳理

**写作编辑类**
- 文献综述、白皮书、技术文档（纯文档，非代码）
- 提案、标书、申请书撰写
- 翻译 + 本地化

**评判审核类**
- 批改考试、合同条款分析
- 合规检查报告、法律意见书草稿
- 数据分析报告、问卷结果分析

### 两阶段架构

| 阶段 | 模式 | 原因 |
|------|------|------|
| **阶段 B：规划与设计** | **Subagents**（串行调度） | 全程串行（Research → QA → Research → QA...），不需要并行和 Agent 间通信 |
| **阶段 C：执行产出** | **Agent Teams**（并行协作） | 多个写作 Agent 并行产出独立章节/文档，需要共享任务列表和 mailbox 通信 |

### 全自动化原则

**人类介入分类法（知识工作版）：**

| 类别 | 处理方式 | 示例 |
|------|---------|------|
| **机器可验证 + 影响后续产出** | 立即自动检查，不通过则阻塞 | 章节完整性、来源引用数量达标、字数在范围内、交叉引用无矛盾 |
| **机器可验证 + 不影响后续产出** | 自动检查，结果记录但不阻塞 | 格式规范、Markdown 语法 |
| **需人类判断 + 影响后续产出** | 尽一切可能转化为机器可验证；实在无法转化的提前告知人类 | 核心立场方向是否正确（如：对美国外交立场的判断） |
| **需人类判断 + 不影响后续产出** | 延迟到最终交付时通知人类 | 语气风格是否满意、排版美观度 |

---

## 二、完整流程

```
用户提交需求
    |
    v
+----------------------------------------------+
|  阶段 A: 需求审阅（不启动任何 Agent）            |
|                                               |
|  Lead 直接与人类对话：                          |
|  1. 审阅需求的完整性和可执行性                    |
|  2. 识别场景类型，确定 B2 的适配方向               |
|  3. 识别需要人类判断的主观决策点                   |
|  4. 提出改进建议，与人类确认最终版本               |
+----------------------------------------------+
    | 需求定稿 + 场景类型确定
    v
+----------------------------------------------+
|  阶段 B: 规划与设计（Subagents 串行调度）        |
|                                               |
|  Lead 创建项目目录，串行调用 Subagents：         |
|                                               |
|  B1: 调用 Research Subagent                   |
|      -> 产出 topic-research.md                |
|      -> 调用 QA Subagent 审批                  |
|      -> 不通过 -> 再调 Research 修改（<=8轮）    |
|                                               |
|  B2: 调用 Research Subagent                   |
|      -> 产出 analysis-structure.md            |
|      -> 调用 QA Subagent 审批                  |
|      -> 不通过 -> 再调 Research 修改（<=8轮）    |
|                                               |
|  B3: 调用 Research Subagent                   |
|      -> 产出 deliverables-plan.md             |
|      -> 调用 QA Subagent 审批                  |
|      -> 不通过 -> 再调 Research 修改（<=8轮）    |
|                                               |
|  B4: 调用 Design Subagent                     |
|      -> 产出项目 CLAUDE.md + .claude/agents/   |
|      -> 产出 deferred-human-review.md         |
|      -> 调用 QA Subagent 审批                  |
|      -> 不通过 -> 再调 Design 修改（<=8轮）      |
|                                               |
|  每个子阶段完成后 git commit + 更新检查点         |
+----------------------------------------------+
    | 全部产出审批通过
    v
+----------------------------------------------+
|  阶段 C: 执行产出（Agent Teams 并行协作）        |
|                                               |
|  Lead cd 到 projects/<name>/                  |
|  读取项目 CLAUDE.md，创建唯一的 Agent Team       |
|  按 deliverables-plan 全自动产出                |
|                                               |
|  每个 Phase:                                   |
|  1. Lead 分配写作任务、质量标准和审核用例           |
|     同时写入 .agents/test-cases/phase-N-...     |
|  2. Writer Agents 并行撰写（可参考 test-case）   |
|  3. QA 按 test-case 审核 + 事实核查              |
|     问题写入 .agents/issues/phase-N/            |
|  4. 不通过 -> Writer 读 issue 文件自行修复        |
|  5. 通过 -> git commit -> 下一 Phase            |
|                                               |
|  最后一个 Phase 完成后：                         |
|  -> Lead 输出延迟人工审阅清单                     |
|  -> 通知人类：项目完成，以下项需你确认              |
+----------------------------------------------+
    |
    v
人类收到完成通知 + 延迟审阅清单
```

---

## 三、目录结构

```
Knowledge-Workflow/
├── CLAUDE.md                              # 元工作流 Lead 指令（阶段 A + B）
├── .claude/
│   ├── settings.json                      # 启用 Agent Teams
│   └── agents/
│       ├── research-agent.md              # Subagent: 主题调研+分析构建+产出规划 (opus, 50 turns)
│       ├── design-agent.md                # Subagent: 工作流设计 (opus, 50 turns)
│       └── qa-agent.md                    # Subagent: 质量审批 (sonnet, 60 turns)
├── templates/
│   ├── topic-research-template.md         # B1: 主题调研模板
│   ├── analysis-structure-template.md     # B2: 分析结构模板（含多场景适配指引）
│   ├── deliverables-plan-template.md      # B3: 产出物规划模板
│   ├── project-claude-md-template.md      # B4: 项目 CLAUDE.md 模板
│   ├── writer-agent-template.md           # B4: Writer Agent 配置模板
│   └── qa-agent-template.md              # B4: 项目 QA Agent 配置模板
├── .gitignore                             # 忽略 projects/
└── projects/
    └── <project-name>/                    # 阶段 B 产出 -> 阶段 C 运行
        ├── CLAUDE.md                      # Design Subagent 定制的 Agent Team Lead 指令
        ├── .claude/
        │   ├── settings.json
        │   └── agents/
        │       ├── <topic-1>-writer.md
        │       ├── <topic-2>-writer.md
        │       └── qa.md
        ├── docs/
        │   ├── requirement.md
        │   ├── topic-research.md
        │   ├── analysis-structure.md
        │   └── deliverables-plan.md
        ├── .agents/
        │   ├── style-guide/              # Lead 定义的写作规范和术语表
        │   ├── reviews/                  # QA 审批报告（阶段 B，按子阶段和轮次归档）
        │   │   ├── B1-topic-research/
        │   │   │   ├── round-1.md
        │   │   │   └── round-2.md
        │   │   ├── B2-analysis-structure/
        │   │   ├── B3-deliverables-plan/
        │   │   └── B4-workflow/
        │   ├── test-cases/               # Lead 前置生成的审核用例（阶段 C）
        │   │   └── phase-N-test-cases.md
        │   ├── issues/                   # QA 审核发现的问题（阶段 C，按 phase 分子目录）
        │   │   └── phase-N/
        │   │       └── 001-description.md
        │   ├── status/
        │   │   └── phase
        │   └── deferred-human-review.md
        └── output/                        # 最终交付物目录
```

**Git 策略：**
- 元工作流仓库的 `.gitignore` 忽略 `projects/` 整个目录
- `projects/<name>/` 内部独立 `git init`，不污染元仓库历史

---

## 四、阶段 A：需求审阅（不启动任何 Agent）

Lead 在主对话中直接完成，不调用任何 Subagent。

### Lead 的审阅检查清单

1. **主题覆盖面**：需求是否明确了核心问题？是否有遗漏的关键维度？
2. **可量化验收标准**：每份产出物是否有明确的通过/不通过判定？（篇幅、章节、来源数量等）
3. **约束合理性**：篇幅要求、深度要求、受众定位、格式要求是否合理？
4. **场景类型识别**：
   - 判断属于哪类场景（调研分析 / 策略规划 / 教育培训 / 写作编辑 / 评判审核）
   - 确定 B2 的适配方向（分析框架 / 知识点矩阵 / 约束条件梳理 / 评估维度 / ...）
5. **人类判断点识别**：
   - 立场/倾向性决策（如：代表哪一方的利益？态度应该多强硬？）
   - 语气风格偏好（学术风 / 职场风 / 通俗风）
   - 受众适配（专家 / 非专业人士 / 特定群体）
6. **自动化可行性评估**：对每个人类判断点评估能否转化为机器验证

### 场景-B2 适配表

Lead 在阶段 A 结束时确定 B2 的产出方向：

| 场景类型 | B2 产出名称 | B2 核心内容 |
|---------|-----------|------------|
| 调研分析 | analysis-structure.md | 分析框架：分析维度、指标、方法论 |
| 策略规划 | analysis-structure.md | 策略框架：立场矩阵、应对策略树、情景分析 |
| 教育培训 | analysis-structure.md | 知识结构：知识点矩阵、难度分级、覆盖要求 |
| 写作编辑 | analysis-structure.md | 内容架构：论述结构、章节逻辑、论据链 |
| 评判审核 | analysis-structure.md | 评判框架：评估维度、评分标准、权重分配 |
| 日程规划 | analysis-structure.md | 约束模型：时间约束、资源约束、优先级规则 |

B2 文件名统一为 `analysis-structure.md`，但内部结构根据场景完全不同。模板中提供多场景适配指引，research-agent 根据 Lead 传递的场景类型选择对应结构。

### 输出

- 改进建议列表（与人类讨论）
- 场景类型 + B2 适配方向（与人类确认）
- 人类判断点清单（提前告知）
- 确认后的最终需求文档

### 进入阶段 B 的条件

人类明确确认需求文档定稿，且场景类型已确认。

---

## 五、阶段 B：规划与设计（Subagents 串行调度）

### 5.1 Subagent 角色

| 角色 | Agent 文件 | 模型 | maxTurns | 工具 | 调用时机 |
|------|-----------|------|----------|------|---------|
| **Lead** | -- | 当前会话模型 | -- | 全部 | 始终在线 |
| **Research Subagent** | research-agent.md | opus | 50 | Read, Write, Edit, Glob, Grep, Bash, WebSearch, WebFetch | B1-B3 |
| **QA Subagent** | qa-agent.md | sonnet | 60 | Read, Write, Glob, Grep, Bash | B1-B4 每个产出后 |
| **Design Subagent** | design-agent.md | opus | 50 | Read, Write, Edit, Glob, Grep, Bash | B4 |

**Subagents 的工作方式（继承自 dev 工作流，完全相同）：**
- Lead 通过 Agent 工具调用 Subagent，Subagent 在自己的上下文窗口中执行任务
- Subagent 完成后结果返回给 Lead
- Lead 根据结果决定下一步（调用 QA 审批、或让 Research 修改）
- 每个 Subagent 调用都是独立的，不保留之前调用的上下文

### 5.2 初始化

1. 从需求中提取项目名称（如未明确指定，根据内容推断一个简短的英文项目名）
2. 创建项目目录：`projects/<project-name>/`
3. 创建子目录：`projects/<project-name>/docs/`、`projects/<project-name>/.agents/status/`
4. 将需求文档写入 `projects/<project-name>/docs/requirement.md`
5. 在 `projects/<project-name>/` 下执行 `git init`
6. 做一次初始 commit

### 5.3 B1: 主题调研

**目标**：对主题进行全方位的信息收集和事实梳理。

**Lead 调用 Research Subagent：**
> "阅读 `projects/<name>/docs/requirement.md`，参考 `templates/topic-research-template.md` 的格式，产出 `projects/<name>/docs/topic-research.md`。使用 WebSearch 广泛调研，确保信息来源多元、事实有据可查。不要凭空推测。"

**Lead 调用 QA Subagent：**
> "审批 `projects/<name>/docs/topic-research.md`，按**检查清单 1（主题调研审批）**逐项检查。同时阅读 `projects/<name>/docs/requirement.md` 作为参考。审批报告写入 `projects/<name>/.agents/reviews/B1-topic-research/round-{N}.md`。"

**审批报告 frontmatter 格式：**
```yaml
---
result: PASS  # 或 FAIL
round: 1
phase: B1
reviewed_file: docs/topic-research.md
date: YYYY-MM-DD
---
```

**Lead 读取审批报告文件**（检查 frontmatter 中的 `result` 字段）：
- **PASS** -> `git commit`，在 `projects/<name>/.agents/status/phase` 写入 `B1_COMPLETED`，进入 B2
- **FAIL** -> 再次调用 Research Subagent：
  > "QA 审批未通过，审批意见见 `projects/<name>/.agents/reviews/B1-topic-research/round-{N}.md`，请阅读该文件并按意见修改 `topic-research.md`。"
- 修改后再调 QA 审批（QA 自行扫描 reviews 目录确定轮次编号），如此循环
- **连续 8 个完整的"修改->审批"循环仍不通过时**：要求 Research Subagent 说明无法满足的具体条件，然后向人类报告并请求介入

### 5.4 B2: 分析结构

**目标**：基于主题调研，建立场景驱动的结构化分析框架。

B2 的核心创新：**不固定结构含义，由场景类型驱动**。

- B1 永远是"信息收集"（通用）
- **B2 根据场景适配**——可以是分析框架、知识点矩阵、评估维度、约束条件梳理等
- B3 永远是"产出物规划"（通用）

**Lead 调用 Research Subagent：**
> "基于已通过的 `projects/<name>/docs/topic-research.md`，参考 `templates/analysis-structure-template.md` 的格式，产出 `projects/<name>/docs/analysis-structure.md`。本项目的场景类型是**[场景类型]**，请按模板中对应场景的指引构建分析结构。"

**审批循环同 B1**（QA 写审批报告到 `reviews/B2-analysis-structure/round-{N}.md`，Research Agent 从文件读取修改意见），使用**检查清单 2（分析结构审批）**。通过后在 phase 文件写入 `B2_COMPLETED`。

### 5.5 B3: 产出物规划

**目标**：明确最终交付物的清单、结构、写作计划和验收标准。

**Lead 调用 Research Subagent：**
> "基于已通过的 `projects/<name>/docs/topic-research.md` 和 `projects/<name>/docs/analysis-structure.md`，参考 `templates/deliverables-plan-template.md` 的格式，产出 `projects/<name>/docs/deliverables-plan.md`。验收标准优先使用机器可自动验证的方式（章节完整性、来源引用数量、字数范围、交叉引用一致性）。纯主观判断项标注为'延迟人工审阅'。"

**审批循环同 B1**（QA 写审批报告到 `reviews/B3-deliverables-plan/round-{N}.md`，Research Agent 从文件读取修改意见），使用**检查清单 3（产出物规划审批）**。通过后在 phase 文件写入 `B3_COMPLETED`。

### 5.6 B4: 工作流设计

**目标**：为具体项目量身定制 Agent Team 工作流。

**Lead 调用 Design Subagent：**
> "阅读 `projects/<name>/docs/` 下的所有文档（requirement.md、topic-research.md、analysis-structure.md、deliverables-plan.md），以及元仓库的 `templates/` 目录下所有模板文件。为项目量身定制 Agent Team 工作流，所有产出写入 `projects/<name>/` 目录下。"

**Lead 调用 QA Subagent：**
> "审批 `projects/<name>/` 下的工作流配置，按**检查清单 4（工作流配置审批）**逐项检查。审批范围：CLAUDE.md、.claude/agents/*.md、.claude/settings.json、.agents/ 目录结构、deferred-human-review.md。"

**审批循环同 B1**（QA 写审批报告到 `reviews/B4-workflow/round-{N}.md`，Design Agent 从文件读取修改意见）。通过后 `git commit`，在 phase 文件写入 `B4_COMPLETED`。

### 5.7 检查点与恢复机制

每个子阶段（B1/B2/B3/B4）完成后：
1. Lead 在 `projects/<name>/.agents/status/phase` 写入当前进度
2. Lead 做一次 git commit

如果 session 中断，新 session 启动时：
1. Lead 检查 `projects/<name>/.agents/status/phase` 文件
2. 从上次完成的检查点之后继续
3. 已完成的子阶段不重复执行

| 检查点值 | 含义 | 下一步 |
|---------|------|--------|
| 文件不存在 | 阶段 B 未开始 | 从 B1 开始 |
| `B1_COMPLETED` | 主题调研已通过 | 从 B2 开始 |
| `B2_COMPLETED` | 分析结构已通过 | 从 B3 开始 |
| `B3_COMPLETED` | 产出物规划已通过 | 从 B4 开始 |
| `B4_COMPLETED` | 工作流设计已通过 | 进入阶段 C |

---

## 六、阶段 C：执行产出（Agent Teams 并行协作）

### 6.1 进入阶段 C

Lead 在阶段 B 全部完成后：
1. cd 到 `projects/<name>/`
2. 读取该目录下的 CLAUDE.md（由 Design Subagent 定制）
3. **从此刻起，完全按项目 CLAUDE.md 的指令行事**，不再参考元仓库 CLAUDE.md
4. 创建 Agent Team，按项目 CLAUDE.md 中的角色定义 spawn teammates
5. 按 `docs/deliverables-plan.md` 中的 Phase 顺序推进产出

**上下文管理**：到达阶段 C 时，之前的对话可能已被 auto-compaction 压缩。所有必要信息都已持久化为文件，Lead 完全依赖读取文件获取上下文，不依赖对话记忆。

### 6.2 Team 组成（由 Design Subagent 动态决定）

Design Subagent 根据项目特征决定 Agent 角色。决策规则：

```
产出物数量与类型:
├── 单一报告（< 3 章节）     -> 1 Writer Agent (teammate)
├── 多章节报告（3-6 章节）   -> 1-2 Writer Agents (teammates, 按主题分工)
├── 多文档产出（如：指南 + 数据包 + 附录） -> 2-3 Writer Agents (teammates, 按文档分工)
└── 大型项目（5+ 独立文档）  -> 最多 4 Writer Agents (teammates)

所有配置均额外包含 1 个 QA (subagent, 由 Lead 按需调用)

文档间依赖:
├── 强依赖（后文引用前文结论） -> 减少并行 Agent，增加串行 Phase
└── 弱依赖（独立章节/独立文档） -> 增加并行 Agent
```

**QA 的角色定位**：QA 在阶段 C 中作为 **Subagent**（不是 teammate），由 Lead 在每个 Phase 完成后按需调用。原因：QA 审核是串行的（必须等写作完成），作为常驻 teammate 会在等待期间空耗 token。

### 6.3 阶段 C 的 QA 审核循环与检查点

每个 Phase 开始时：
1. Lead 定义写作任务和接口，**同时生成 `.agents/test-cases/phase-N-test-cases.md`**
2. test-case 必须覆盖 deliverables-plan 中该 Phase 的所有验收标准，可增加额外用例但不能遗漏

每个 Phase 完成后：
1. Lead 调用 QA Subagent 审核：**按 test-case 文件逐条执行**，问题写入 `.agents/issues/phase-N/`
2. QA 审核结果：
   - 通过（不存在 Critical 或 Major 级别的 `open` issue） -> Lead 做 git commit，更新检查点（`C_PHASE_N_COMPLETED`），进入下一 Phase
   - 不通过 -> Lead 通过 SendMessage 告知对应 Writer：`"QA 审核未通过，请阅读 .agents/issues/phase-N/ 下的 issue 文件并逐一修复。"` Writer 自行读取 issue 文件修复，修复后将 issue 的 `status` 改为 `resolved`
   - QA 每轮重新跑全部 test-case，可新增 issue 也可将已 resolved 的 reopen
   - 连续 8 个完整的"修改->审核"循环仍不通过时，Lead 要求 Writer Agent 说明无法满足的具体条件，然后请求人工介入

**Issue 生命周期管理：**

issue 文件使用 status frontmatter：
```yaml
---
status: open  # open / resolved
severity: Major
phase: 1
issue_type: 事实错误  # 事实错误 / 逻辑缺陷 / 来源缺失 / 格式问题 / 一致性问题
date: YYYY-MM-DD
---
```

- Writer 修复后将 `status` 改为 `resolved`
- QA 可将已 resolved 的 issue reopen（改回 `open`）
- **最终通过判定**：不存在 Critical 或 Major 级别的 `open` issue。Minor 级别的 `open` issue 不阻塞通过，但记录在案
- `assigned_to` 为可选字段，QA 不需要填写，由 Lead 根据文件归属分配

**test-case 文件格式（知识工作版）：**

```markdown
# Phase N 审核用例

## 基本信息
- Phase: N
- 生成时间: YYYY-MM-DD
- 依据: deliverables-plan.md Phase N 验收标准

## 审核用例

### TC-N-001: [审核项名称]
- **类型**: 章节完整性 / 来源引用 / 字数范围 / 交叉引用 / 逻辑连贯 / 术语一致 / 事实核查
- **验证方式**: grep/wc/人工阅读/WebSearch 交叉验证
- **通过标准**: [预期结果]
- **失败时严重程度**: Critical / Major / Minor

### TC-N-002: ...
```

阶段 C 的检查点值定义：
```
C_PHASE_0_COMPLETED  -> 基础框架已通过
C_PHASE_1_COMPLETED  -> Phase 1 已通过
C_PHASE_N_COMPLETED  -> Phase N 已通过
C_COMPLETED          -> 所有 Phase 已通过，项目完成
```

### 6.4 QA 审核的自动化分层（知识工作版）

Design Subagent 在设计 Team 的 QA 验收标准时，必须按以下规则分层：

**自动审核层（QA Agent 直接执行，阻塞后续 Phase）：**
- 章节完整性：deliverables-plan 中规划的所有章节是否已产出
- 来源引用：关键事实是否有来源标注，引用数量是否达标
- 字数范围：是否在规划的篇幅范围内
- 交叉引用一致性：同一事实在不同文档/章节中的表述是否一致
- 逻辑连贯性：段落间是否有明确的逻辑过渡，结论是否有论据支撑
- 术语一致性：同一概念是否使用相同术语
- 格式规范：Markdown 格式、标题层级、列表风格是否统一

**延迟人工审阅层（不阻塞，记录到 deferred-human-review.md）：**
- 语气风格是否符合预期
- 立场倾向性是否恰当
- 排版美观度
- 表述的"味道"和"调性"

### 6.5 最终交付

所有 Phase 完成后，Lead 输出：

```
项目完成

延迟人工审阅清单（以下项需要你确认）：
1. [Phase 2] 核心立场的表述力度是否恰当
2. [Phase 3] 应对策略的语气是否过于强硬/柔和
3. [Phase 4] 附录中数据表格的排版是否清晰
...

所有产出物位于 projects/<name>/output/ 目录下。
```

---

## 七、Agent 配置文件设计要点

### 7.1 research-agent.md

- **模型**: opus, **maxTurns**: 50
- **工具**: Read, Write, Edit, Glob, Grep, Bash, WebSearch, WebFetch
- **职责**: 主题调研 -> 分析结构 -> 产出物规划，三个产出分阶段调用
- **关键约束**:
  - 严格按 templates/ 下的模板格式产出
  - 调研必须基于 WebSearch 结果，不能凭空推测
  - 关键事实必须标注来源
  - 产出写入项目的 docs/ 目录
  - 不做 git 操作
- **收到修改意见时**：Lead 会告知审批意见文件的路径（`.agents/reviews/` 下），Research Agent 自行读取文件获取修改意见，不从 Lead 指令中获取意见全文
- **与 dev 工作流 planning-agent 的区别**:
  - planning-agent 调研的是技术方案（框架、库、API），research-agent 调研的是主题知识（事实、数据、观点、文献）
  - planning-agent 不需要标注来源（技术文档即来源），research-agent 必须标注来源（知识工作的基本要求）
  - planning-agent 的 B2 是固定的"技术架构"，research-agent 的 B2 根据场景类型适配

### 7.2 design-agent.md

- **模型**: opus, **maxTurns**: 50
- **工具**: Read, Write, Edit, Glob, Grep, Bash
- **职责**: 根据所有已审批文档，设计 Agent Team 的完整写作工作流配置
- **关键特性 — 人类判断最小化引擎**:
  - 扫描 deliverables-plan 中所有验收标准
  - 对每个"需人工审阅"项评估：能否转为机器验证？
  - 转化策略：
    - 章节存在性检查（grep 标题）
    - 字数统计（wc）
    - 引用数量检查（grep 引用标记）
    - 交叉引用一致性（在多个文件中 grep 同一事实）
    - 术语一致性检查（grep 术语表中的词）
  - 不能转化且不影响后续的 -> 写入延迟审阅清单
  - 不能转化且影响后续的 -> 在 CLAUDE.md 中标注，尽量缩小阻塞范围
- **产出**: 项目 CLAUDE.md + .claude/agents/* + .claude/settings.json + .agents/ 初始化 + deferred-human-review.md
- **新增职责 — 写作规范定义**:
  - 在 `.agents/style-guide/` 下产出**完整版**写作规范（语气、术语表、引用格式、行文风格），Phase 0 仅做复核和微调
  - 确保所有 Writer Agent 共享同一套规范
- **硬性要求 — 必须设计审核用例阶段**:
  - 项目 CLAUDE.md 的流程定义中，每个 Phase 开始时 Lead 必须生成 `test-cases/phase-N-test-cases.md`
  - QA 按此文件执行审核
- **硬性要求 — 目录结构**:
  - `.agents/` 目录必须包含：reviews/（阶段 B 已有内容）、test-cases/、issues/（按 phase 分子目录）、style-guide/、interfaces/、status/、deferred-human-review.md
- **评判审核类场景适配**：对于批改考试、合同审查等场景，Writer Agent 的角色定义应调整为"评判员"而非"写手"，工作模式为"逐项评判 + 产出评判报告"
- **收到修改意见时**：Lead 会告知审批意见文件路径（`.agents/reviews/B4-workflow/round-{N}.md`），Design Agent 自行读取文件获取修改意见

### 7.3 qa-agent.md

- **模型**: sonnet, **maxTurns**: 60
- **工具**: Read, Write, Glob, Grep, Bash
- **Write 权限约束**: Write 工具**仅用于** `.agents/reviews/` 目录写入审批报告。禁止写入 `docs/`、`CLAUDE.md`、`.claude/agents/` 等被审批文件，防止 QA 篡改被审批文档
- **审批模式**: 严格模式，按检查清单逐项检查，任何一项未通过都打回
- **审批报告输出**: 写入 `.agents/reviews/B{N}-xxx/round-{N}.md`，带机器可读的 frontmatter（result: PASS/FAIL）。轮次编号由 QA 自行扫描目录确定
- **8 轮安全阀**：连续 8 轮不通过时，Lead 要求 Agent 说明卡点原因，请求人工介入
- **4 套审批检查清单**:

#### 检查清单 1: 主题调研审批（docs/topic-research.md）

- [ ] 信息源覆盖面：是否涵盖多类信息源（官方文件、学术文献、权威媒体、当事方声明等）
- [ ] 事实准确性：关键事实是否有来源标注？是否存在明显事实错误？
- [ ] 时间线完整性：关键事件的时间线是否完整、有序？
- [ ] 多视角呈现：是否呈现了不同立场/阵营的观点？
- [ ] 与需求的关联度：调研内容是否紧扣需求文档中的核心问题？
- [ ] 深度与广度平衡：关键问题是否有足够深度，而非泛泛而谈？
- [ ] 信息时效性：数据和事实是否足够新，是否标注了信息获取时间？

#### 检查清单 2: 分析结构审批（docs/analysis-structure.md）

- [ ] 与场景类型匹配：结构是否适合需求中确定的场景类型？
- [ ] 分析维度完备性：是否覆盖了需求中的所有关键问题？
- [ ] 逻辑自洽性：各维度之间是否逻辑一致、不自相矛盾？
- [ ] 论据支撑：每个分析结论是否有主题调研中的事实支撑？
- [ ] 可操作性：结构是否足够具体，能指导后续的写作产出？
- [ ] 边界条件识别：是否考虑了特殊情况、例外场景、对立论点？

#### 检查清单 3: 产出物规划审批（docs/deliverables-plan.md）

- [ ] 产出清单完整性：是否覆盖了需求文档中要求的所有交付物？
- [ ] 每份文档的章节结构是否清晰、有逻辑？
- [ ] 优先级与依赖关系：产出物之间的先后顺序是否合理？
- [ ] 验收标准可评判：每份产出是否有明确的通过/不通过标准？
- [ ] 自动化审核比例：是否最大化了机器可验证的验收项？
- [ ] 延迟审阅项分类合理：标注为延迟的项是否确实不影响后续产出？
- [ ] Phase 划分合理：每个 Phase 目标单一且可完成？
- [ ] 并行策略可行：并行写作的文档/章节之间无内容冲突？

#### 检查清单 4: 工作流配置审批

- [ ] Writer Agent 角色划分与产出物类型匹配
- [ ] 每个 Agent 的写作范围清晰且不重叠
- [ ] CLAUDE.md 包含所有必要信息（角色表、流程、并行策略、Git 约定、检查点、8 轮升级机制）
- [ ] Agent 配置文件格式正确（YAML frontmatter）
- [ ] model 和 maxTurns 配置合理
- [ ] QA 配置为 Subagent 模式，聚焦内容审核（事实核查、逻辑审查、来源验证）
- [ ] QA 验收标准分层合理：自动审核（阻塞）vs 延迟人工审阅（不阻塞）
- [ ] 延迟人工审阅清单中的项确实不影响后续 Phase
- [ ] Git 操作由 Lead 独占
- [ ] .agents/ 目录结构完整（reviews/, test-cases/, issues/, style-guide/, interfaces/, status/phase, deferred-human-review.md）
- [ ] 工作流中包含 Lead 前置生成 test-case 的步骤
- [ ] issue 文件使用 status frontmatter（open/resolved）并按 phase 分子目录
- [ ] 写作规范（style-guide/）内容充分：语气、术语表、引用格式、行文风格

---

## 八、模板设计要点

### 8.1 topic-research-template.md

```markdown
# 主题调研：{主题}

## 1. 主题概述
<!-- 背景和核心问题概述 -->

## 2. 关键事件时间线
| 时间 | 事件 | 来源 |
|------|------|------|
| ... | ... | [来源标注] |

## 3. 核心利益相关方
### 3.1 [相关方名称]
- 基本立场
- 核心利益
- 关键行动
- 来源标注

## 4. 关键事实与数据
<!-- 每条事实附来源标注 -->
| 事实/数据 | 来源 | 获取时间 |
|-----------|------|---------|

## 5. 多视角分析
### 视角 1: [立场/阵营]
### 视角 2: [立场/阵营]

## 6. 信息源清单
### 一手来源（官方文件、当事方声明）
### 二手来源（学术文献、权威媒体）
### 数据来源

## 7. 争议点与不确定性
<!-- 各方说法矛盾的地方、信息不完整的地方 -->
```

### 8.2 analysis-structure-template.md

```markdown
# 分析结构：{主题}

## 场景适配指引

<!-- Research Agent 根据 Lead 传递的场景类型选择对应结构 -->

### 如果场景是"调研分析"：
构建分析框架 —
- 分析维度（每个维度含：分析方法、关键指标、预期结论方向）
- 各维度之间的逻辑关系
- 分析局限性说明

### 如果场景是"策略规划"：
构建策略框架 —
- 核心立场声明
- 立场矩阵（我方 / 对方 / 中间方）
- 应对策略树（按议题分支，每个分支含：触发条件、推荐应对、备选方案）
- 情景分析（最佳 / 最可能 / 最坏）
- 红线与底线

### 如果场景是"教育培训"：
构建知识结构 —
- 知识点矩阵（知识点 x 难度 x 重要度）
- 知识点间的前置依赖关系
- 覆盖要求（哪些是必考/核心，哪些是拓展）

### 如果场景是"写作编辑"：
构建内容架构 —
- 论述主线（中心论点 -> 分论点 -> 论据链）
- 章节间的逻辑递进关系
- 读者旅程设计（从什么认知起点到什么认知终点）

### 如果场景是"评判审核"：
构建评判框架 —
- 评估维度（每个维度含：定义、评分标准、权重）
- 评分量表（如适用）
- 判定规则（通过/不通过 或 分数段含义）

### 如果场景是"日程规划"：
构建约束模型 —
- 时间约束（固定时间点、持续时长、间隔要求）
- 资源约束（人员、场地、设备）
- 优先级规则（哪些不可移动、哪些可调整）
- 冲突解决策略

---

## 通用部分（所有场景都需要）

## N-1. 与需求的映射关系
| 需求中的问题/要求 | 本结构中的对应部分 |
|------------------|-----------------|

## N. 边界条件与风险
- 分析/框架的局限性
- 可能被挑战的假设
- 信息缺口（调研中未能完全覆盖的领域）
```

### 8.3 deliverables-plan-template.md

```markdown
# 产出物规划：{项目名}

## 1. 产出物清单

| # | 文档名 | 类型 | 预计篇幅 | 依赖 | 优先级 |
|---|--------|------|---------|------|--------|

## 2. 各文档详细结构

### 文档 1：{名称}
- 目标读者
- 章节大纲（含每章预计篇幅）
- 与其他文档的关系

### 文档 2...

## 3. 写作计划（Phase 划分）

### Phase 概览

| Phase | 目标 | 产出 | 前置依赖 | 可并行 |
|-------|------|------|---------|--------|
| 0 | 基础框架搭建 | 目录结构 + 写作规范 + 术语表 | 无 | 否 |
| 1 | ... | ... | Phase 0 | ... |

### Phase 0: 基础框架
- Lead 独立完成
- 创建 output/ 目录结构
- 复核 .agents/style-guide/（由 B4 的 Design Agent 已产出完整版），如有必要微调
- 生成 .agents/test-cases/phase-0-test-cases.md
- 为每份文档创建骨架文件（标题 + 章节框架）

### Phase N: [名称]
#### 任务列表
| 任务 | 说明 | 负责 Agent |
|------|------|-----------|

#### 并行策略
| Agent | 负责文档/章节 |
|-------|-------------|

#### 验收标准

##### 自动审核（阻塞后续 Phase）
| 验收项 | 验证方式 | 通过标准 |
|--------|---------|---------|
| 章节完整性 | grep 检查标题 | 所有规划章节已产出 |
| 来源引用 | grep 引用标记 | 关键事实均有来源 |
| 字数范围 | wc | 在规划范围内 |
| 交叉引用一致性 | 多文件 grep 对比 | 同一事实表述一致 |

##### 延迟人工审阅（不阻塞）
| 审阅项 | 原因 |
|--------|------|

## 4. MVP 范围

### 包含
- ...

### 不包含（后续版本 / 人工补充）
- ...
```

### 8.4 project-claude-md-template.md

```markdown
# [项目名称] — 项目指令

## 项目概述
<!-- 从需求文档提炼的项目简述 -->

## 分析结构摘要
<!-- 从 analysis-structure.md 提炼，让所有 Agent 快速理解整体框架 -->

## 写作范围划分
<!-- 明确哪个 Agent 负责哪些文档/章节，确保不重叠 -->
- `output/doc-1.md` -- [Writer 1] 负责
- `output/doc-2.md` -- [Writer 2] 负责

## 写作规范
<!-- 引用 .agents/style-guide/ 中的规范 -->
- 语气风格：见 .agents/style-guide/tone.md
- 术语表：见 .agents/style-guide/glossary.md
- 引用格式：见 .agents/style-guide/citation.md

## Agent Teams 工作流

### 角色
| 角色 | 职责 | 写作范围 |
|------|------|---------|
| **Lead** | 任务分配、Phase 推进、QA 调度、git 操作 | 全局 |
| **[Writer 1]** | ... | `output/...` |
| **[Writer 2]** | ... | `output/...` |
| **QA** (Subagent) | 内容审核：事实核查、逻辑审查、来源验证 | 全局（由 Lead 按需调用） |

### 流程
每个 Phase 按以下步骤执行：
1. **Lead 定义任务、质量标准和审核用例** -- 同时写入 .agents/test-cases/phase-N-test-cases.md
2. **并行写作** -- Writer Agents 并行撰写各自部分（可参考 test-case 理解验收预期）
3. **协调** -- 如需引用其他 Agent 产出的内容，通过 SendMessage 协调
4. **完成通知** -- Writer 通过 SendMessage 向 Lead 发送完成通知
5. **QA 审核** -- Lead 调用 QA Subagent，按 test-case 文件逐条执行，问题写入 .agents/issues/phase-N/
6. **问题修复** -- 不通过时，Lead 通过 SendMessage 告知 Writer 阅读 .agents/issues/phase-N/ 下的 issue 文件自行修复，修复后标记 status: resolved
7. **重新审核** -- QA 重新跑全部 test-case，可新增 issue 或 reopen 旧的
8. 连续 8 个完整"修改->审核"循环不通过时，请求人工介入
9. 通过（无 Critical/Major 级别 open issue） -> git commit -> 更新检查点 -> 下一 Phase

### Git 操作约定
- Lead 负责所有 git 操作
- Writer Agent 和 QA 不做 git 操作
- 每个 Phase 审核通过后，Lead 做一次 commit

### 检查点机制
每个 Phase 审核通过后，Lead 更新 `.agents/status/phase` 文件。
session 中断后从上次完成的 Phase 之后继续。

## 验收标准速查
| Phase | 自动审核（阻塞） | 延迟人工审阅（不阻塞） |
|-------|----------------|---------------------|

## 参考文档
- 需求文档：docs/requirement.md
- 主题调研：docs/topic-research.md
- 分析结构：docs/analysis-structure.md
- 产出物规划：docs/deliverables-plan.md
```

### 8.5 writer-agent-template.md

```markdown
---
name: <topic>-writer
description: 负责 [项目名] 的 [写作范围描述]
tools: Read, Write, Edit, Glob, Grep, Bash, WebSearch, WebFetch
model: opus
maxTurns: 100  # Design Subagent 可根据项目复杂度调整（范围 80-150）
effort: max
---

你是 [项目名] 的 [角色] 写手。

## 你的写作范围

你负责以下文档/章节：
- `output/<文件 1>` -- [描述]
- `output/<文件 2>` -- [描述]

## 写作规范

严格遵守 `.agents/style-guide/` 中的规范：
- 语气风格：.agents/style-guide/tone.md
- 术语表：.agents/style-guide/glossary.md
- 引用格式：.agents/style-guide/citation.md

## 调研补充（严格优先级规则）

写作中使用事实和数据时，必须按以下优先级：
1. **首选**：docs/topic-research.md 中已有的事实（这是经过 QA 审批的权威来源）
2. **次选**：docs/analysis-structure.md 中的分析结论
3. **补充调研**：仅当以上文档未覆盖时，使用 WebSearch/WebFetch 补充
   - 补充的事实必须标注为**"补充来源"**（区别于已审批来源）
   - QA 审核时会重点关注这些新增来源的准确性
   - 格式：`[补充来源] 事实内容 —— 来源: URL/文献名`

## 参考文档

- 主题调研：docs/topic-research.md（核心事实和数据）
- 分析结构：docs/analysis-structure.md（分析框架和逻辑结构）
- 产出物规划：docs/deliverables-plan.md（你负责的章节的详细大纲和验收标准）

## 重要

- **不要修改其他 Agent 负责的文档**
- 如需引用其他 Agent 产出的内容，通过 SendMessage 协调
- 关键事实必须标注来源
- **不做 git 操作**（git 由 Lead 负责）
- **评判审核类场景说明**：对于批改考试、合同审查等评判类场景，Writer Agent 的职责是"评判 + 产出评判报告"而非"撰写原创内容"。模板中的"写作范围"应理解为"评判范围"，工作模式为"逐题/逐条评判 + 生成结构化评判报告"
```

### 8.6 qa-agent-template.md（项目 QA）

```markdown
---
name: qa
description: 负责 [项目名] 的内容质量审核：事实核查、逻辑审查、来源验证、格式检查
tools: Read, Write, Glob, Grep, Bash, WebSearch, WebFetch
model: opus
maxTurns: 60
---

你是 [项目名] 的 QA 审核员。你由 Lead 在每个 Phase 完成后调用，负责内容审核。

## 你的职责

1. **章节完整性检查** -- 规划的所有章节是否已产出
2. **事实核查** -- 关键事实是否准确，是否有来源标注
3. **逻辑审查** -- 论述是否连贯，结论是否有论据支撑，是否自相矛盾
4. **来源验证** -- 来源标注是否充分，关键论断是否可追溯
5. **交叉引用一致性** -- 同一事实在不同文档/章节中的表述是否一致
6. **术语一致性** -- 是否遵守 .agents/style-guide/glossary.md 中的术语表
7. **格式规范** -- Markdown 格式、标题层级、引用格式是否统一
8. **Issue 产出** -- 发现的问题写入 .agents/issues/phase-N/ 下的独立文件
9. **按 test-case 执行** -- 读取 .agents/test-cases/phase-N-test-cases.md，逐条执行审核

## 输入来源

- .agents/test-cases/phase-N-test-cases.md（审核用例，逐条执行）
- .agents/style-guide/（写作规范，检查术语和格式一致性）

## Issue 格式

Issue 文件路径：`.agents/issues/phase-N/{seq}-{brief}.md`

Issue 文件内容（注意 status frontmatter）：

    ---
    status: open
    severity: Major
    phase: 1
    issue_type: 事实错误
    date: YYYY-MM-DD
    ---

    # Phase N Issue #XX: [简述]

    ## 问题描述
    [详细描述，引用具体段落]

    ## 修改建议
    [具体的修改方向]

- Writer 修复后将 status 改为 resolved
- QA 每轮重新跑全部 test-case，可新增 issue 也可将已 resolved 的 reopen

## 通过标准

当以下条件全部满足时，给出"通过"：
- test-case 中所有审核项逐一通过
- 不存在 Critical 或 Major 级别的 open issue（Minor 级别不阻塞）
- 关键事实均有来源标注
- 无逻辑矛盾

如果不通过，明确列出所有未通过项和具体修改意见。

## 重要

- **你不直接修改内容文档**（`output/` 等目录）
- 你可以写入 `.agents/issues/phase-N/` 目录来记录问题（这是你唯一允许写入的目录）
- 你可以使用 WebSearch/WebFetch 进行事实核查
- **不做 git 操作**（git 由 Lead 负责）
```

---

## 九、关键设计决策总结

| 决策项 | 选择 | 理由 |
|--------|------|------|
| 与 dev 工作流的关系 | 独立项目 | 避免双模式 Agent 的脆弱性、路由层的复杂度、调试困难 |
| 需求审阅 | Lead 直接对话，不启动 Agent | 减少 token，人机交互更高效 |
| 阶段 B 模式 | Subagents 串行调度 | 全程串行，不需要并行和 Agent 间通信 |
| 阶段 C 模式 | Agent Teams 并行协作 | 多写手并行产出独立章节/文档 |
| B2 适配机制 | 场景驱动，文件名统一，内部结构按场景变化 | 覆盖调研分析、策略规划、教育培训、写作编辑、评判审核、日程规划等全场景 |
| Research Agent | opus, 50 turns, +WebSearch/WebFetch | 深度推理 + 需要在线调研主题知识 |
| Design Agent | opus, 50 turns | 理解项目特征生成复杂配置 |
| QA Agent（元仓库） | sonnet, 60 turns | 阶段 B 审批是结构化检查清单任务，sonnet 胜任；60 turns 覆盖 4 套检查清单 x 8 轮 |
| QA Agent（项目） | opus, 60 turns, +WebSearch/WebFetch | 知识工作的事实核查和逻辑审查是深度推理任务，sonnet 不够；60 turns 覆盖大型项目多文档审核 |
| Writer Agent | opus, 80-150 turns, +WebSearch/WebFetch | 写作需要深度推理 + 可能需要补充调研 |
| 人类介入策略 | 需求阶段识别 -> 设计阶段最大化自动化 -> 不影响产出的延迟到最后 | 全自动化核心原则 |
| Git | Lead 独占 | 集中管理 |
| 项目 git | projects/<name>/ 独立 git init | 不污染元仓库 |
| 检查点恢复 | .agents/status/phase 文件 | 覆盖 B1-B4 和阶段 C 每个 Phase |
| 阶段 C QA 角色 | Subagent | QA 审核串行，常驻 teammate 空耗 token |
| CLAUDE.md 职责分离 | 元仓库只管 A+B；项目管 C | 避免上下文冲突 |
| 写作规范 | .agents/style-guide/ 独立目录 | 确保多 Writer Agent 产出风格统一 |
| 产出物目录 | output/ | 与 docs/（规划文档）分离，清晰区分"过程"和"交付" |

---

## 十、需要创建的文件清单（共 12 个）

### 核心配置（3 个）

| # | 文件路径 | 说明 |
|---|---------|------|
| 1 | `CLAUDE.md` | 元工作流核心：阶段 A（需求审阅 + 场景识别）+ 阶段 B（Subagents 串行调度）+ 阶段 C 入口 |
| 2 | `.claude/settings.json` | 启用 Agent Teams |
| 3 | `.gitignore` | 忽略 projects/ |

### Agent 配置（3 个）

| # | 文件路径 | 说明 |
|---|---------|------|
| 4 | `.claude/agents/research-agent.md` | Subagent: 主题调研+分析结构+产出规划 (opus, 50 turns) |
| 5 | `.claude/agents/design-agent.md` | Subagent: 工作流设计+人类判断最小化 (opus, 50 turns) |
| 6 | `.claude/agents/qa-agent.md` | Subagent: 4 套审批检查清单，严格模式 (sonnet, 60 turns) |

### 模板文件（6 个）

| # | 文件路径 | 说明 |
|---|---------|------|
| 7 | `templates/topic-research-template.md` | B1: 主题调研文档格式 |
| 8 | `templates/analysis-structure-template.md` | B2: 分析结构模板（含多场景适配指引） |
| 9 | `templates/deliverables-plan-template.md` | B3: 产出物规划格式 |
| 10 | `templates/project-claude-md-template.md` | B4: 项目 CLAUDE.md 参考模板 |
| 11 | `templates/writer-agent-template.md` | B4: Writer Agent 配置参考 |
| 12 | `templates/qa-agent-template.md` | B4: 项目 QA Agent 配置参考（聚焦内容审核而非代码验收） |

### 实施顺序

```
Step 1: .claude/settings.json + .gitignore           (并行，无依赖)
Step 2: templates/*.md                                (并行，无依赖)
Step 3: .claude/agents/*.md                           (并行，无依赖)
Step 4: CLAUDE.md                                     (依赖 Step 2-3，引用 Agent 名称和模板路径)
Step 5: projects/.gitkeep                             (创建空目录)
```

---

## 十一、从 dev 工作流继承的模式清单

以下模式已被验证有效，直接继承：

| 模式 | 来源 | 在知识工作流中的应用 |
|------|------|-------------------|
| A/B/C 三阶段架构 | dev plan.md 一.设计总纲 | 完全相同的骨架，内容适配 |
| Subagent 串行调度（阶段 B） | dev plan.md 五.Subagent 角色 | Research 替代 Planning，其余相同 |
| Agent Team 并行协作（阶段 C） | dev plan.md 六.Agent Teams | Writer 替代 Dev，QA 仍为 Subagent |
| QA 严格模式 + 检查清单 | dev qa-agent.md | 4 套检查清单全部重写为知识工作版 |
| 8 轮安全阀 | dev CLAUDE.md 硬性约束 | 完全相同 |
| 检查点恢复 | dev plan.md 五.3 | 完全相同的文件机制 |
| Git Lead 独占 | dev CLAUDE.md 硬性约束 | 完全相同 |
| CLAUDE.md 职责分离 | dev plan.md 八.决策总结 | 完全相同 |
| Design Subagent 人类介入最小化引擎 | dev design-agent.md | 转化策略从代码验证改为文本验证 |
| QA 作为 Subagent 而非 teammate | dev plan.md 六.2 | 完全相同的理由 |
| 延迟人工验收/审阅 | dev plan.md 六.4 | 从视觉审美类改为语气风格类 |
| 项目独立 git init | dev plan.md 三.Git 策略 | 完全相同 |
| QA 审批意见文件化 | dev plan-v2-optimization.md | 完全相同：reviews/ 目录 + frontmatter + agent 读文件 |
| test-case 前置生成 | dev plan-v2-optimization.md | 从代码测试用例适配为内容审核用例（章节完整性、来源检查等） |
| issue 生命周期管理 | dev plan-v2-optimization.md | 完全相同：status frontmatter (open/resolved) + 按 phase 分子目录 |
