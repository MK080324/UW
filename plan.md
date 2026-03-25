# 全自动多 Agent 元工作流 — 实施计划 v3

## 一、设计总纲

### 核心理念

Development-Workflow 是一个**元工作流仓库**。用户提交需求文档后，Claude 全自动完成从技术选型到方案实施到测试验证到最终交付的完整流程。人类介入被压缩到最少：只在需求确认阶段和最终交付时参与。

### 两阶段架构

| 阶段 | 模式 | 原因 |
|------|------|------|
| **阶段 B：规划与设计** | **Subagents**（串行调度） | 全程串行（Planning → QA → Planning → QA...），不需要并行和 Agent 间通信 |
| **阶段 C：开发** | **Agent Teams**（并行协作） | 多个开发 Agent 并行编码，需要共享任务列表和 mailbox 通信 |

这个架构彻底消除了"Team 切换"问题——阶段 B 不创建任何 Team，阶段 C 才创建唯一的 Agent Team。Lead 在阶段 B 结束后提示人类新开 session 到项目目录，启动 Agent Team 开始开发。

### 全自动化原则

**人类介入分类法：**

| 类别 | 处理方式 | 示例 |
|------|---------|------|
| **机器可验证 + 影响后续开发** | 立即自动测试，不通过则阻塞 | 帧率是否达标、编译是否通过、API 响应格式正确 |
| **机器可验证 + 不影响后续开发** | 自动测试，结果记录但不阻塞 | 代码风格警告、文档格式 |
| **需人类判断 + 影响后续开发** | 尽一切可能转化为机器可验证；实在无法转化的提前告知人类 | 核心 UI 布局是否正确 |
| **需人类判断 + 不影响后续开发** | 延迟到最终交付时通知人类 | 渐变色是否美观、主题配色好不好看 |

**需人类操作的系统行为（如 sudo）：**
- 在需求审阅阶段就识别出来，提前告知人类
- 在工作流设计阶段尽一切可能规避（如用 npm prefix 替代全局安装、用 docker 替代本地环境依赖）
- 实在无法规避的，集中到特定 Phase 的开头一次性处理，而不是散落在各处

---

## 二、完整流程

```
用户提交需求文档
    │
    ▼
┌──────────────────────────────────────────┐
│  阶段 A: 需求审阅（不启动任何 Agent）        │
│                                           │
│  Lead 直接与人类对话：                      │
│  1. 审阅需求文档的完整性和可执行性            │
│  2. 识别所有可能需要人类介入的操作             │
│  3. 提出改进建议，与人类确认最终版本           │
│  4. 明确告知哪些步骤必须人类操作               │
└──────────────────────────────────────────┘
    │ 需求文档定稿
    ▼
┌──────────────────────────────────────────┐
│  阶段 B: 规划与设计（Subagents 串行调度）    │
│                                           │
│  Lead 创建项目目录，串行调用 Subagents：     │
│                                           │
│  B1: 调用 Planning Subagent                │
│      → 产出 technical-research.md          │
│      → 调用 QA Subagent 审批               │
│      → 不通过 → 再调 Planning 修改（≤8轮）   │
│                                           │
│  B2: 调用 Planning Subagent                │
│      → 产出 structure.md                   │
│      → 调用 QA Subagent 审批               │
│      → 不通过 → 再调 Planning 修改（≤8轮）   │
│                                           │
│  B3: 调用 Planning Subagent                │
│      → 产出 roadmap.md                     │
│      → 调用 QA Subagent 审批               │
│      → 不通过 → 再调 Planning 修改（≤8轮）   │
│                                           │
│  B4: 调用 Design Subagent                  │
│      → 产出项目 CLAUDE.md + .claude/agents/ │
│      → 产出 deferred-human-review.md       │
│      → 调用 QA Subagent 审批               │
│      → 不通过 → 再调 Design 修改（≤8轮）     │
│                                           │
│  每个子阶段完成后 git commit + 更新检查点     │
└──────────────────────────────────────────┘
    │ 全部产出审批通过
    ▼
┌──────────────────────────────────────────┐
│  阶段 C: 开发（Agent Teams 并行协作）       │
│                                           │
│  人类新开 session: cd projects/<name>/ && claude │
│  项目 CLAUDE.md 自动加载，创建 Agent Team    │
│  按 roadmap 全自动开发                      │
│                                           │
│  每个 Phase:                               │
│  1. Lead 定义任务/接口                      │
│  2. 开发 Agents 并行实现                    │
│  3. QA 自动验收（只验机器可验证的项）          │
│  4. 不通过 → 修复循环                       │
│  5. 通过 → git commit → 下一 Phase          │
│                                           │
│  最后一个 Phase 完成后：                     │
│  → Lead 输出延迟人工验收清单                  │
│  → 通知人类：项目完成，以下项需你确认          │
└──────────────────────────────────────────┘
    │
    ▼
人类收到完成通知 + 延迟验收清单
```

---

## 三、目录结构

```
Development-Workflow/
├── CLAUDE.md                              # 元工作流 Lead 指令（阶段 A + B）
├── .claude/
│   ├── settings.json                      # 启用 Agent Teams
│   └── agents/
│       ├── planning-agent.md              # Subagent: 技术规划 (opus, 50 turns)
│       ├── design-agent.md                # Subagent: 工作流设计 (opus, 50 turns)
│       └── qa-agent.md                    # Subagent: 质量审批 (sonnet, 60 turns)
├── templates/
│   ├── technical-research-template.md
│   ├── structure-template.md
│   ├── roadmap-template.md
│   ├── project-claude-md-template.md
│   ├── dev-agent-template.md
│   └── qa-agent-template.md
├── .gitignore                             # 忽略 projects/
└── projects/
    └── <project-name>/                    # 阶段 B 产出 → 阶段 C 运行
        ├── CLAUDE.md                      # Design Subagent 定制的 Agent Team Lead 指令
        ├── .claude/
        │   ├── settings.json
        │   └── agents/                    # Design Subagent 定制的 Team 成员
        │       ├── <role-1>-dev.md
        │       ├── <role-2>-dev.md
        │       └── qa.md
        ├── docs/
        │   ├── requirement.md
        │   ├── technical-research.md
        │   ├── structure.md
        │   └── roadmap.md
        ├── .agents/
        │   ├── interfaces/                # Lead 在每个 Phase 定义的接口
        │   ├── issues/
        │   │   └── .gitkeep
        │   ├── status/
        │   │   └── phase                  # 检查点文件（当前进度）
        │   └── deferred-human-review.md   # 延迟人工验收清单
        └── <src>/
```

**Git 策略：**
- 元工作流仓库的 `.gitignore` 忽略 `projects/` 整个目录
- `projects/<name>/` 内部独立 `git init`，不污染元仓库历史

---

## 四、阶段 A：需求审阅（不启动任何 Agent）

Lead 在主对话中直接完成，不调用任何 Subagent。

### Lead 的审阅检查清单

1. **功能完整性**：需求是否覆盖了核心功能、边界场景
2. **可量化验收标准**：每个功能是否有明确的通过/不通过判定
3. **技术约束合理性**：指定的技术栈、平台、性能要求是否合理
4. **人类介入识别**：
   - 系统特权操作（sudo、全局安装、修改系统配置）
   - GUI/视觉验证（颜色、布局、动画效果）
   - 硬件依赖（特定设备、外部服务连接）
   - 交互体验（手势、拖拽、实时响应感受）
5. **自动化可行性评估**：对每个人类介入点评估能否转化为机器验证

### 输出

- 改进建议列表（与人类讨论）
- 人类介入操作清单（提前告知人类）
- 确认后的最终需求文档

---

## 五、阶段 B：规划与设计（Subagents 串行调度）

### 5.1 Subagent 角色

| 角色 | Agent 文件 | 模型 | maxTurns | 工具 | 调用时机 |
|------|-----------|------|----------|------|---------|
| **Lead** | — | 当前会话模型 | — | 全部 | 始终在线 |
| **Planning Subagent** | planning-agent.md | opus | 50 | Read, Write, Edit, Glob, Grep, Bash, WebSearch, WebFetch | B1-B3 |
| **QA Subagent** | qa-agent.md | sonnet | 60 | Read, Glob, Grep, Bash | B1-B4 每个产出后 |
| **Design Subagent** | design-agent.md | opus | 50 | Read, Write, Edit, Glob, Grep, Bash | B4 |

**Subagents 的工作方式：**
- Lead 通过 Agent 工具调用 Subagent，Subagent 在自己的上下文窗口中执行任务
- Subagent 完成后结果返回给 Lead
- Lead 根据结果决定下一步（调用 QA 审批、或让 Planning 修改）
- 每个 Subagent 调用都是独立的，不保留之前调用的上下文

### 5.2 流程详细

**B1: 技术调研**
1. Lead 调用 Planning Subagent："阅读 docs/requirement.md，参考 templates/technical-research-template.md，产出 docs/technical-research.md"
2. Planning Subagent 返回结果
3. Lead 调用 QA Subagent："审批 docs/technical-research.md，按技术调研检查清单"
4. QA 返回审批报告：
   - 通过 → Lead 做 git commit，更新检查点，进入 B2
   - 不通过 → Lead 调用 Planning Subagent 修改，**附带 QA 审批报告中的具体修改意见作为 prompt 的一部分**，修改完成后再调 QA 审批
   - 无硬性轮数上限。连续 8 个完整的"修改→审批"循环仍不通过时，Lead 要求 Agent 说明无法满足的具体条件，然后请求人工介入

**B2: 技术架构**（审批循环同 B1，产出 docs/structure.md）

**B3: 开发路线图**（审批循环同 B1，产出 docs/roadmap.md）

**B4: 工作流设计**
1. Lead 调用 Design Subagent：
   - 读取 docs/ 下所有已审批文档 + templates/ 下的模板
   - 分析项目特征（技术栈、模块数、并行可行性）
   - 产出到 `projects/<name>/` 下：
     - CLAUDE.md（阶段 C 的 Agent Team Lead 指令）
     - .claude/agents/*.md（Team 成员配置）
     - .claude/settings.json
     - .agents/ 目录初始化
     - .agents/deferred-human-review.md（延迟人工验收清单）
2. Lead 调用 QA Subagent 审批（包括审查延迟验收清单分类的合理性）
3. 审批循环同 B1
4. 通过后 → Lead 做 git commit，进入阶段 C

### 5.3 检查点与恢复机制

每个子阶段（B1/B2/B3/B4）完成后：
1. Lead 在 `projects/<name>/.agents/status/phase` 写入当前进度（如 `B2_COMPLETED`）
2. Lead 做一次 git commit

如果 session 中断，新 session 启动时：
1. Lead 检查 `projects/<name>/.agents/status/phase` 文件
2. 从上次完成的检查点之后继续
3. 已完成的子阶段不重复执行

检查点值定义：
```
B1_COMPLETED  → 技术调研已通过
B2_COMPLETED  → 技术架构已通过
B3_COMPLETED  → 路线图已通过
B4_COMPLETED  → 工作流设计已通过，可进入阶段 C
```

---

## 六、阶段 C：开发（Agent Teams 并行协作）

### 6.1 进入阶段 C

**阶段 B 全部完成后，必须新开一个 Claude Code session 才能进入阶段 C。**

**原因：** 项目目录下的 `.claude/agents/*.md`、`.claude/settings.json` 和 `CLAUDE.md` 只有当 Claude Code 以该目录作为工作目录启动时才会被正确加载。在元仓库中 `cd` 到项目目录是无效的——Claude Code 加载的仍然是元仓库的 `.claude/` 配置，项目定制的 Agent 定义不会被识别。

**Lead 的操作：** 阶段 B 全部完成后，Lead 向人类输出提示：

```
阶段 B 已全部完成。请在项目目录下新开 Claude Code session 进入阶段 C：

cd projects/<name>/ && claude
```

**新 session 启动后：**
1. 项目 `CLAUDE.md`（由 Design Subagent 定制）会被自动加载为项目指令
2. 项目 `.claude/agents/*.md` 会被正确识别为可用 agent
3. 项目 `.claude/settings.json` 会自动生效
4. Lead 按项目 CLAUDE.md 行事，创建 Agent Team，按 CLAUDE.md 中的角色定义 spawn teammates
5. 按 roadmap.md 中的 Phase 顺序推进开发

**重要：** 元仓库的 CLAUDE.md 只定义阶段 A 和 B 的行为。进入阶段 C 后，Lead 完全按项目 CLAUDE.md 行事。元仓库 CLAUDE.md 中不包含任何阶段 C 的具体开发指令。

**上下文隔离：** 新开 session 天然实现了上下文窗口隔离——阶段 A/B 的对话历史不会占用阶段 C 的上下文空间。所有必要信息都已在阶段 B 持久化为文件，Lead 完全依赖读取文件获取上下文，不依赖对话记忆。

### 6.2 Team 组成（由 Design Subagent 动态决定）

Design Subagent 根据项目特征决定 Agent 角色。决策规则：

```
技术栈类型:
├── 纯前端 → 1 Frontend Agent (teammate)
├── 纯后端/CLI → 1 Backend Agent (teammate)
├── 全栈 Web → 1 Frontend Agent + 1 Backend Agent (teammates)
├── 桌面应用 → 1 UI Agent + 1 Core Agent (teammates, 参考 Editor 项目)
└── 复杂系统(4+ 独立模块) → 最多 4 Dev Agent (teammates)

所有配置均额外包含 1 个 QA (subagent, 由 Lead 按需调用)

模块间依赖:
├── 强依赖 → 减少并行 Agent，增加串行 Phase
└── 弱依赖 → 增加并行 Agent，契约驱动
```

"复杂系统"定义：项目有 4 个或以上可独立开发的核心模块，且模块间通过明确的接口通信。

**QA 的角色定位：** QA 在阶段 C 中作为 **Subagent**（不是 teammate），由 Lead 在每个 Phase 开发完成后按需调用。原因：QA 验收是串行的（必须等开发完成），作为常驻 teammate 会在等待期间空耗 token。按需调用更高效。

### 6.3 阶段 C 的 QA 审批循环与检查点

每个 Phase 开发完成后：
1. Lead 调用 QA Subagent 验收，附带当前 Phase 的验收标准
2. QA 返回验收报告：
   - 通过 → Lead 做 git commit，更新检查点（`C_PHASE_N_COMPLETED`），进入下一 Phase
   - 不通过 → Lead 将 **QA 验收报告中的具体问题和修复意见** 通过 SendMessage 转发给对应的开发 teammate 修复，修复后再调 QA 验收
   - 无硬性轮数上限。连续 8 个完整的"修复→验收"循环仍不通过时，Lead 要求开发 Agent 说明无法满足的具体条件，然后请求人工介入

阶段 C 的检查点值定义（追加到 `.agents/status/phase`）：
```
C_PHASE_0_COMPLETED  → 脚手架已通过
C_PHASE_1_COMPLETED  → Phase 1 已通过
C_PHASE_N_COMPLETED  → Phase N 已通过
C_COMPLETED          → 所有 Phase 已通过，项目开发完成
```

### 6.4 QA 验收的自动化分层

Design Subagent 在设计 Team 的 QA 验收标准时，必须按以下规则分层：

**自动验收层（QA Agent 直接执行，阻塞后续 Phase）：**
- 编译/构建检查
- Lint 零警告
- 单元测试全通过
- 性能指标达标（通过探针数据 / CLI 输出验证）
- 接口契约一致性

**延迟人工验收层（不阻塞，记录到 deferred-human-review.md）：**
- 视觉美观度（颜色搭配、渐变效果）
- 交互体验感受（动画流畅度主观感受）
- 主题配色审美判断

**示例 — Editor 项目的 Phase 3：**

| 验收项 | 原来的处理 | 优化后的处理 |
|--------|-----------|------------|
| 8 套主题可切换 | 自动 | 自动（检查切换命令是否响应） |
| 切换 < 16ms | 自动 | 自动（探针数据验证） |
| 渐变色 DOM 元素存在 | 自动 | 自动（gradientElementCount > 0） |
| 语法高亮上色正确 | 人工 | **转为自动**（visibleTokenCount > 0 且 token 类型覆盖率 > 80%） |
| 渐变色是否美观 | 人工 | **延迟**（写入 deferred-human-review.md） |
| 主题颜色好不好看 | 人工 | **延迟**（写入 deferred-human-review.md） |

### 6.5 最终交付

所有 Phase 完成后，Lead 输出：

```
✅ 项目开发完成

📋 延迟人工验收清单（以下项需要你确认）：
1. [Phase 3] 8 套主题的视觉效果是否满意
2. [Phase 3] 渐变色字体的美观度
3. [Phase 5] 命令面板的交互体验
...

⚠️ 需要你手动执行的操作（如有）：
1. [一次性] 运行 `sudo xxx` 安装 xxx（原因：...）
...
```

---

## 七、Agent 配置文件设计要点

### 7.1 planning-agent.md

- **模型**: opus, **maxTurns**: 50
- **工具**: Read, Write, Edit, Glob, Grep, Bash, WebSearch, WebFetch
- **职责**: 技术调研 → 架构设计 → 路线图，三个产出分阶段调用
- **关键约束**:
  - 严格按 templates/ 下的模板格式产出
  - 技术选型必须基于 WebSearch 调研结果，不能凭空推测
  - 产出写入项目的 docs/ 目录
  - 不做 git 操作

### 7.2 design-agent.md

- **模型**: opus, **maxTurns**: 50
- **工具**: Read, Write, Edit, Glob, Grep, Bash
- **职责**: 根据所有已审批文档，设计 Agent Team 的完整工作流配置
- **关键特性 — 人类介入最小化引擎**:
  - 扫描 roadmap 中所有验收标准
  - 对每个"需人工验收"项评估：能否转为机器验证？
  - 转化策略：探针数据、CLI 输出检查、文件存在性检查、正则匹配
  - 不能转化且不影响后续的 → 写入延迟验收清单
  - 不能转化且影响后续的 → 在 CLAUDE.md 中标注，尽量缩小阻塞范围
  - 需要系统特权的操作 → 寻找替代方案（docker、prefix 安装、沙箱环境）
- **产出**: 项目 CLAUDE.md + .claude/agents/* + .claude/settings.json + .agents/ 初始化 + deferred-human-review.md

### 7.3 qa-agent.md

- **模型**: sonnet, **maxTurns**: 60
- **工具**: Read, Glob, Grep, Bash（只读，不能 Write/Edit）
- **审批模式**: 严格模式，按检查清单逐项检查，任何一项未通过都打回
- **无硬性轮数上限**：QA 可自主判断通过/不通过。连续 8 轮不通过时，Lead 要求 Agent 说明无法满足的具体条件，然后请求人工介入
- **4 套审批检查清单**:
  1. **技术调研审批**：方案对比充分性（≥2 候选）、依赖版本明确、技术栈兼容性、风险评估
  2. **技术架构审批**：模块划分合理性（高内聚低耦合）、接口定义清晰、数据流完整、目录结构与架构一致
  3. **路线图审批**：Phase 划分合理（每 Phase 目标单一）、验收标准可量化、并行策略无文件冲突、Phase 0 为脚手架
  4. **工作流配置审批**：Agent 目录边界无重叠、配置格式正确（YAML frontmatter）、QA 验收分层合理（延迟项确实不影响后续开发）、Git 操作由 Lead 独占
- **审批结果直接返回给 Lead**（Subagent 模式，结果自动返回，不需要 SendMessage）

---

## 八、关键设计决策总结

| 决策项 | 选择 | 理由 |
|--------|------|------|
| 需求审阅 | Lead 直接对话，不启动 Agent | 减少 token，人机交互更高效 |
| 阶段 B 模式 | **Subagents 串行调度** | 全程串行，不需要并行和 Agent 间通信 |
| 阶段 C 模式 | **Agent Teams 并行协作** | 多开发 Agent 并行编码，需要共享任务列表和 mailbox |
| QA 重试策略 | 无硬性上限，连续 8 个完整"修改→审批"循环不通过时触发人工介入 | 信任 Agent 自主判断，8 轮安全阀防止死循环。阶段 B 和阶段 C 统一使用此机制 |
| Planning Agent | opus, 50 turns, +WebSearch/WebFetch | 深度推理 + 需要在线调研技术方案 |
| Design Agent | opus, 50 turns | 理解项目特征生成复杂配置 |
| QA Agent | sonnet, 60 turns | 审批任务 sonnet 胜任，60 turns 覆盖 4 套 × 8 轮 |
| 人类介入策略 | 需求阶段识别 → 设计阶段最大化自动化 → 不影响开发的延迟到最后 | 全自动化核心原则 |
| Git | Lead 独占，每个子阶段/Phase 通过后 commit | 集中管理 |
| 项目 git | projects/<name>/ 独立 git init | 不污染元仓库 |
| 检查点恢复 | .agents/status/phase 文件 | 覆盖阶段 B（B1-B4）和阶段 C（每个 Phase），session 中断后可恢复 |
| 阶段 C QA 角色 | Subagent（由 Lead 按需调用，不是 teammate） | QA 验收是串行的，常驻 teammate 在等待期间空耗 token |
| CLAUDE.md 职责分离 | 元仓库 CLAUDE.md 只管阶段 A+B；项目 CLAUDE.md 管阶段 C | 避免上下文冲突 |

---

## 九、需要创建的文件清单（共 12 个）

### 核心配置（3 个）

| # | 文件路径 | 说明 |
|---|---------|------|
| 1 | `CLAUDE.md` | 元工作流核心：阶段 A（需求审阅）+ 阶段 B（Subagents 串行调度流程）+ 阶段 C 入口（cd + 创建 Team）|
| 2 | `.claude/settings.json` | 启用 Agent Teams 实验功能 |
| 3 | `.gitignore` | 忽略 projects/ |

### Agent 配置（3 个）

| # | 文件路径 | 说明 |
|---|---------|------|
| 4 | `.claude/agents/planning-agent.md` | Subagent: 技术调研+架构+路线图 (opus, 50 turns) |
| 5 | `.claude/agents/design-agent.md` | Subagent: 工作流设计+人类介入最小化 (opus, 50 turns) |
| 6 | `.claude/agents/qa-agent.md` | Subagent: 4 套审批检查清单，严格模式 (sonnet, 60 turns) |

### 模板文件（6 个）

| # | 文件路径 | 说明 |
|---|---------|------|
| 7 | `templates/technical-research-template.md` | 技术调研文档格式 |
| 8 | `templates/structure-template.md` | 技术架构文档格式 |
| 9 | `templates/roadmap-template.md` | 开发路线图格式 |
| 10 | `templates/project-claude-md-template.md` | Design Subagent 生成项目 CLAUDE.md 的参考（含 QA 用 Subagent、检查点机制、8 轮升级） |
| 11 | `templates/dev-agent-template.md` | Design Subagent 生成开发 Agent 配置的参考 |
| 12 | `templates/qa-agent-template.md` | Design Subagent 生成项目 QA Agent 配置的参考（须注明：阶段 C 的 QA 聚焦代码验收——编译、测试、性能，不做文档审批） |

### 实施顺序

```
Step 1: .claude/settings.json + .gitignore           (并行，无依赖)
Step 2: templates/*.md                                (并行，无依赖)
Step 3: .claude/agents/*.md                           (并行，无依赖)
Step 4: CLAUDE.md                                     (依赖 Step 2-3，引用 Agent 名称和模板路径)
Step 5: projects/.gitkeep                             (创建空目录)
```
