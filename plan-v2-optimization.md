# 优化计划：QA 意见文件化 + 测试用例机制

## Context

当前工作流存在三个问题：
1. **QA 审批意见仅存在于对话上下文中**，由 Lead 手动复制粘贴转发给 planning-agent/design-agent。上下文压缩后信息丢失，且 agent 无法自主查阅历史审批意见。
2. **阶段 C 缺少系统性的测试用例设计环节**，QA 只是对着 roadmap 中的验收标准表格跑命令，没有 Lead 生成的结构化 test-case 文件。
3. **"审批意见"与"测试 issues"概念混淆**，两者产生时机、针对对象、存在阶段都不同，但目前共用 `.agents/issues/` 目录。

## 修改范围

共 6 个文件需要修改：

| 文件 | 修改类型 | 说明 |
|------|---------|------|
| `CLAUDE.md` | 编辑 | 阶段 B 流程：QA 写审批文件 + agent 读文件；阶段 C 流程：Lead 生成 test-case |
| `.claude/agents/qa-agent.md` | 编辑 | 阶段 B QA：审批报告写入文件而非返回给 Lead |
| `.claude/agents/design-agent.md` | 编辑 | 初始化目录结构变更 + 必须设计测试阶段 |
| `.claude/agents/planning-agent.md` | 编辑 | 收到修改意见时去读文件而非从 Lead 指令中获取 |
| `templates/qa-agent-template.md` | 编辑 | 阶段 C QA：按 test-case 执行 + issues 按 phase 分子目录 |
| `templates/project-claude-md-template.md` | 编辑 | 流程模板更新：Lead 生成 test-case 步骤 |

## 详细修改方案

### 1. 新的目录结构规范

```
.agents/
├── reviews/                          # 审批意见（仅阶段 B 使用）
│   ├── B1-technical-research/
│   │   ├── round-1.md                # 第 1 轮审批报告
│   │   └── round-2.md                # 第 2 轮（如有）
│   ├── B2-structure/
│   ├── B3-roadmap/
│   └── B4-workflow/
├── test-cases/                       # Lead 生成的测试用例（阶段 C）
│   ├── phase-1-test-cases.md
│   └── phase-2-test-cases.md
├── issues/                           # 运行测试发现的 bug（阶段 C）
│   ├── phase-1/
│   │   └── 001-download-progress.md
│   └── phase-2/
├── interfaces/                       # 接口定义（不变）
├── status/
│   └── phase
└── deferred-human-review.md
```

**关键区分：**
- `reviews/` = 审阅文档/配置时提出的意见（文档质量、完整性、逻辑问题）
- `test-cases/` = Lead 设计的结构化测试用例，QA 按此执行
- `issues/` = 实际运行代码/测试时发现的 bug（编译错误、测试失败、运行时问题）

### 2. CLAUDE.md 修改

#### 2.1 阶段 B 初始化 — reviews/ 按需创建

不在初始化时预创建 reviews/ 子目录。QA agent 在需要写审批报告时自行 `mkdir -p` 创建。

#### 2.2 阶段 B1-B4 流程 — QA 写文件 + Agent 读文件

**以 B1 为例（B2-B4 同理）：**

```
原流程：
  QA 返回审批结果 → 不通过 → Lead 粘贴 QA 意见给 planning-agent

新流程：
  1. 调用 qa-agent：
     > "审批 projects/<name>/docs/technical-research.md，按检查清单 1 逐项检查。
     >  审批报告写入 projects/<name>/.agents/reviews/B1-technical-research/round-{N}.md"
  2. Lead 读取审批报告文件（检查 frontmatter 中的 result 字段），判断通过/不通过
  3. 不通过 → 调用 planning-agent：
     > "QA 审批未通过，审批意见见 projects/<name>/.agents/reviews/B1-technical-research/round-{N}.md，
     >  请阅读该文件并按意见修改 technical-research.md。"
```

核心变化：
- QA 将审批报告**写入文件**（`reviews/B{N}-xxx/round-{N}.md`），带机器可读的 frontmatter
- Lead 不再复制粘贴意见，只告诉 planning-agent/design-agent **去读哪个文件**
- 每轮审批都有独立文件，可追溯历史
- 轮次编号由 QA 自行扫描目录确定（`ls .agents/reviews/B{N}-xxx/` 已有文件数 + 1）

#### 2.3 审批报告 frontmatter 格式

```yaml
---
result: PASS  # 或 FAIL
round: 1
phase: B1
reviewed_file: docs/technical-research.md
date: YYYY-MM-DD
---
```

Lead 通过 `result` 字段快速判断通过/不通过，无需解读自由文本。

#### 2.4 阶段 C 流程 — test-case 前置生成（TDD 思维）

```
原流程：
  1. Lead 定义任务和接口
  2. 并行开发
  3. 接口沟通
  4. 完成通知
  5. QA 验收
  6. 问题修复
  7. 通过 → commit

新流程：
  1. Lead 定义任务、接口和测试用例 — 同时写入 .agents/test-cases/phase-N-test-cases.md
  2. 并行开发（开发 Agent 可参考 test-case 理解验收预期）
  3. 接口沟通
  4. 完成通知
  5. QA 验收 — 按 test-case 文件逐条执行，issues 写入 .agents/issues/phase-N/
  6. 问题修复 — Agent 读取 .agents/issues/phase-N/ 下的 issue 文件自行修复，修复后标记 status: resolved
  7. QA 重新验收 — 重新跑全部 test-case，可新增 issue 也可 reopen 旧的
  8. 通过 → commit
```

test-case 前置生成的好处：先定义"什么算通过"再开发，开发 Agent 也能参考来理解验收标准。

**约束：test-case 必须覆盖 roadmap 中该 Phase 的所有验收标准，可以增加额外用例但不能遗漏 roadmap 中的任何验收项。**

#### 2.5 issue 生命周期管理

issue 文件增加 status frontmatter：

```yaml
---
status: open  # open / resolved
severity: Major
phase: 1
date: YYYY-MM-DD
---
```

> **注意：** `assigned_to` 为可选字段，QA 不需要填写。由 Lead 在转发 issue 时根据文件归属自行分配给对应 Agent。

- Agent 修复后将 status 改为 `resolved`
- QA 每轮重新跑全部 test-case，可新增 issue 也可将已 resolved 的 reopen（改回 `open`）
- **最终通过判定：** 不存在 Critical 或 Major 级别的 `open` issue。Minor 级别的 `open` issue **不阻塞通过**，但会记录在案供后续处理

#### 2.6 阶段 C 问题修复流程变更

```
原：Lead 将 QA 验收报告中的具体问题和修复意见通过 SendMessage 转发给对应 Agent 修复
改：QA 将问题写入 .agents/issues/phase-N/ 下的独立文件。Lead 通过 SendMessage 告知 Agent：
    "QA 验收未通过，请阅读 .agents/issues/phase-N/ 下的 issue 文件并逐一修复。"
    Agent 自行读取 issue 文件进行修复。
```

### 3. .claude/agents/qa-agent.md 修改

- 增加 Write 工具权限（需要写审批报告文件）
- 审批输出不再只是"返回给 Lead"，而是**写入文件**
- 新增指令：审批报告写入 `projects/<name>/.agents/reviews/B{N}-xxx/round-{N}.md`，带 frontmatter
- 文件格式保留检查清单格式，增加 frontmatter 头
- QA 自行扫描目录确定轮次编号
- 检查清单 4 新增检查项：工作流中是否包含 Lead 前置生成 test-case 的步骤
- 检查清单 4 更新 .agents/ 目录结构检查项（reviews/、test-cases/、issues/）
- **Write 权限硬性约束：** 禁止写入 `docs/`、`CLAUDE.md`、`.claude/agents/` 等被审批文件。Write 工具**仅用于** `.agents/reviews/` 目录，防止 QA 篡改被审批文档

### 4. .claude/agents/planning-agent.md 修改

- "收到 QA 修改意见后"章节改为：
  > Lead 会告诉你审批意见文件的路径。你需要：
  > 1. 读取审批意见文件（`.agents/reviews/` 下对应文件）
  > 2. 逐条按意见修改文档
  > 3. 返回结果给 Lead

### 5. .claude/agents/design-agent.md 修改

#### 5.1 目录初始化变更

`.agents/` 目录模板更新：
```
.agents/
├── reviews/               # 阶段 B 已有内容（阶段 C 不使用此目录）
├── test-cases/            # Lead 生成的测试用例
│   └── .gitkeep
├── issues/                # 运行测试发现的 bug（按 phase 分子目录）
│   └── .gitkeep
├── interfaces/
├── status/
│   └── phase
└── deferred-human-review.md
```

#### 5.2 新增硬性要求：必须设计测试阶段

在 design-agent 的产出要求中增加：

> **工作流中必须包含 Lead 前置生成测试用例的步骤。** 在项目 CLAUDE.md 的流程定义中，每个 Phase 开始时 Lead 在定义任务和接口的同时生成 `test-cases/phase-N-test-cases.md`，QA 按此文件执行验收。

#### 5.3 收到修改意见 — 读文件

同 planning-agent，改为读取 `reviews/B4-workflow/round-{N}.md` 文件。

### 6. templates/qa-agent-template.md 修改

- QA 的 issue 输出路径从 `.agents/issues/phase{N}-{seq}-{brief}.md` 改为 `.agents/issues/phase-{N}/{seq}-{brief}.md`（按 phase 分子目录）
- 新增输入来源：`读取 .agents/test-cases/phase-N-test-cases.md 作为测试执行依据`
- issue 增加 status frontmatter（open/resolved），定义生命周期
- QA 每轮重新跑全部 test-case
- 新增写入权限：`.agents/issues/phase-N/` 目录

### 7. templates/project-claude-md-template.md 修改

流程步骤更新（与 2.4 一致），增加：
- test-case 前置生成步骤
- 文件化 issue 修复流程
- issue 生命周期管理说明

### 8. test-case 文件格式

```markdown
# Phase N 测试用例

## 基本信息
- Phase: N
- 生成时间: YYYY-MM-DD
- 依据: roadmap.md Phase N 验收标准 + interfaces/phase-N-interfaces.md

## 测试用例

### TC-N-001: [测试名称]
- **类型**: 编译检查 / 单元测试 / 集成测试 / 接口一致性 / 性能
- **验证命令**: `command`
- **通过标准**: [预期结果]
- **失败时严重程度**: Critical / Major / Minor

### TC-N-002: ...
```

## 不修改的内容

- 检查清单内容不变（清单 1-4，仅清单 4 新增两个检查项）
- 8 轮安全阀机制不变
- Git 操作由 Lead 独占不变
- 检查点机制不变
- QA 作为 Subagent 模式不变

## 验证方式

修改完成后逐一检查：
1. 阅读 CLAUDE.md，确认阶段 B 流程中 QA 写文件（带 frontmatter）、agent 读文件的逻辑通顺
2. 阅读 CLAUDE.md，确认阶段 C 流程中有 test-case 前置生成步骤
3. 阅读 qa-agent.md，确认审批报告输出到文件 + frontmatter 格式 + 轮次自动编号
4. 阅读 planning-agent.md，确认从文件读取修改意见
5. 阅读 design-agent.md，确认目录结构更新 + 测试阶段硬性要求
6. 阅读 templates/qa-agent-template.md，确认按 test-case 执行 + issues 按 phase 分目录 + issue 生命周期
7. 阅读 templates/project-claude-md-template.md，确认流程更新
8. 检查各文件之间的交叉引用一致性（路径、文件名、流程步骤编号）
