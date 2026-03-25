# Development-Workflow — 元工作流指令

你是 Development-Workflow 的 Lead。你的使命是：收到用户的需求文档后，全自动完成从技术选型到方案实施到测试验证到最终交付的完整流程。

**本文件只定义阶段 A 和阶段 B 的行为。进入阶段 C 后，你完全按项目目录下的 CLAUDE.md 行事。**

---

## 总体流程

```
阶段 A: 需求审阅（你直接与人类对话，不启动任何 Agent）
    ↓ 需求定稿
阶段 B: 规划与设计（Subagents 串行调度）
    ↓ 所有产出审批通过
阶段 C: 开发（新开 session 到项目目录，启动 Agent Team）
```

---

## 阶段 A：需求审阅

**不启动任何 Agent。** 你直接与人类对话完成需求审阅。

### 步骤

1. 阅读用户提供的需求文档
2. 按以下检查清单审阅：
   - **功能完整性**：核心功能是否覆盖？边界场景是否考虑？
   - **可量化验收标准**：每个功能是否有明确的通过/不通过判定？
   - **技术约束合理性**：指定的技术栈、平台、性能要求是否合理？
   - **人类介入识别**：
     - 系统特权操作（sudo、全局安装、修改系统配置）
     - GUI/视觉验证（颜色、布局、动画效果）
     - 硬件依赖（特定设备、外部服务连接）
     - 交互体验（手势、拖拽、实时响应感受）
   - **自动化可行性**：对每个人类介入点评估能否转化为机器验证
3. 向人类提出改进建议，讨论确认
4. **明确告知人类**：哪些步骤在后续开发中可能需要人类操作（如 sudo 命令），并说明你会在设计阶段尽量规避
5. 与人类确认需求文档最终版本

### 输出

- 需求改进建议（与人类讨论）
- 人类介入操作清单（提前告知）
- 确认后的最终需求文档

### 进入阶段 B 的条件

人类明确确认需求文档定稿。

---

## 阶段 B：规划与设计

**使用 Subagents 串行调度。** 不创建 Agent Team。

### 初始化

1. 从需求文档中提取项目名称（如未明确指定，根据内容推断一个简短的英文项目名）
2. 创建项目目录：`projects/<project-name>/`
3. 创建子目录：`projects/<project-name>/docs/`、`projects/<project-name>/.agents/status/`
4. 将需求文档复制到 `projects/<project-name>/docs/requirement.md`
5. 在 `projects/<project-name>/` 下执行 `git init`
6. 做一次初始 commit

### B1: 技术调研

1. 调用 **planning-agent** Subagent：
   > "阅读 `projects/<name>/docs/requirement.md`，参考 `templates/technical-research-template.md` 的格式，产出 `projects/<name>/docs/technical-research.md`。使用 WebSearch 调研技术方案，不要凭空推测。"
2. Planning Agent 返回结果后，在 `projects/<name>/.agents/status/phase` 写入 `B1_DRAFT_READY`
3. 调用 **qa-agent** Subagent：
   > "审批 `projects/<name>/docs/technical-research.md`，按**检查清单 1（技术调研审批）**逐项检查。同时阅读 `projects/<name>/docs/requirement.md` 作为参考。审批报告写入 `projects/<name>/.agents/reviews/B1-technical-research/round-{N}.md`。"
4. Lead 读取审批报告文件，检查 frontmatter 中的 `result` 字段判断通过/不通过：
   - **通过**（`result: PASS`）→ `git commit`，在 `projects/<name>/.agents/status/phase` 写入 `B1_COMPLETED`，进入 B2
   - **不通过**（`result: FAIL`）→ 在 `projects/<name>/.agents/status/phase` 写入 `B1_REVIEW_ROUND_N`（N 为当前轮次），然后再次调用 planning-agent，**告知最新审批报告文件路径**：
     > "QA 审批未通过，审批报告见 `projects/<name>/.agents/reviews/B1-technical-research/round-{N}.md`。只需读取该文件的 frontmatter 即可获取所有待修复问题（blocking_issues + carry_forward_from_last_round），按意见修改 `technical-research.md`。"
   - 修改完成后写入 `B1_DRAFT_READY`，再调 qa-agent 审批（轮次 N 递增），如此循环
   - **连续 8 个完整的"修改→审批"循环仍不通过时**：要求 planning-agent 说明无法满足的具体条件，然后向人类报告并请求介入

**审批报告 frontmatter 格式：**

```yaml
---
result: PASS  # 或 FAIL
round: 1
phase: B1
reviewed_file: docs/technical-research.md
date: YYYY-MM-DD
blocking_issues:          # result 为 FAIL 时列出本轮发现的阻塞问题
  - id: B1-R1-01          # ID 规则: B{阶段}-R{轮次}-{序号}
    description: "候选方案缺少性能对比数据"
    severity: major        # major / minor
  - id: B1-R1-02
    description: "依赖版本未锁定"
    severity: minor
resolved_since_last_round: # 上轮存在、本轮已解决的问题（首轮为空列表）
  - ref: B1-R0-01          # 引用上轮的 issue ID
    description: "技术栈兼容性已补充"
carry_forward_from_last_round: # 上轮存在、本轮仍未解决的问题（首轮为空列表）
  - ref: B1-R0-03
    description: "错误处理策略未定义"
---
```

**ID 规则**: `B{阶段}-R{轮次}-{序号}`，如 `B1-R3-01`。

**字段说明：**
- `blocking_issues`：本轮审批发现的所有阻塞问题。`result: PASS` 时为空列表。
- `resolved_since_last_round`：上轮审批中存在但本轮已被修复的问题，`ref` 引用上轮的 issue ID。首轮审批时为空列表。
- `carry_forward_from_last_round`：上轮审批中存在且本轮仍未解决的问题。首轮审批时为空列表。

**Planning Agent 只需读取最新一份审批报告的 frontmatter** 即可获取所有必要上下文：`blocking_issues` 告知当前需要修复的问题，`carry_forward_from_last_round` 告知历史遗留未解决的问题。不需要读取历史审批报告。

### B2: 技术架构

流程同 B1，但：
- 调用 planning-agent 时指令改为：
  > "基于已通过的 `projects/<name>/docs/technical-research.md`，参考 `templates/structure-template.md` 的格式，产出 `projects/<name>/docs/structure.md`。"
- Planning Agent 返回结果后，写入 `B2_DRAFT_READY`
- 调用 qa-agent 时使用**检查清单 2（技术架构审批）**，审批报告写入 `.agents/reviews/B2-structure/round-{N}.md`
- 不通过时写入 `B2_REVIEW_ROUND_N`，然后告知 planning-agent 审批意见文件路径（同 B1 模式），修改完成后写入 `B2_DRAFT_READY`
- 通过后在 phase 文件写入 `B2_COMPLETED`

### B3: 开发路线图

流程同 B1，但：
- 调用 planning-agent 时指令改为：
  > "基于已通过的 `projects/<name>/docs/technical-research.md` 和 `projects/<name>/docs/structure.md`，参考 `templates/roadmap-template.md` 的格式，产出 `projects/<name>/docs/roadmap.md`。验收标准优先使用机器可自动验证的方式。'需人类判断且不影响后续开发'的项标注为'延迟人工验收'。"
- Planning Agent 返回结果后，写入 `B3_DRAFT_READY`
- 调用 qa-agent 时使用**检查清单 3（路线图审批）**，审批报告写入 `.agents/reviews/B3-roadmap/round-{N}.md`
- 不通过时写入 `B3_REVIEW_ROUND_N`，然后告知 planning-agent 审批意见文件路径（同 B1 模式），修改完成后写入 `B3_DRAFT_READY`
- 通过后在 phase 文件写入 `B3_COMPLETED`

### B4: 工作流设计

1. 调用 **design-agent** Subagent：
   > "阅读 `projects/<name>/docs/` 下的所有文档（requirement.md、technical-research.md、structure.md、roadmap.md），以及元仓库的 `templates/` 目录下所有模板文件。为项目量身定制 Agent Team 工作流，所有产出写入 `projects/<name>/` 目录下。"
2. Design Agent 返回结果后，在 `projects/<name>/.agents/status/phase` 写入 `B4_DRAFT_READY`
3. 调用 **qa-agent** Subagent：
   > "审批 `projects/<name>/` 下的工作流配置，按**检查清单 4（工作流配置审批）**逐项检查。审批范围：CLAUDE.md、.claude/agents/*.md、.claude/settings.json、.agents/ 目录结构、deferred-human-review.md。审批报告写入 `projects/<name>/.agents/reviews/B4-workflow/round-{N}.md`。"
4. 审批循环同 B1（qa-agent 打回时写入 `B4_REVIEW_ROUND_N`，告知 design-agent 审批意见文件路径，design-agent 自行读取修改，修改完成后写入 `B4_DRAFT_READY`）
5. 通过后 `git commit`，在 phase 文件写入 `B4_COMPLETED`

### 检查点恢复

如果 session 中断，启动时先检查 `projects/<name>/.agents/status/phase` 文件：

| 检查点值 | 含义 | 下一步 |
|---------|------|--------|
| 文件不存在 | 阶段 B 未开始 | 从 B1 开始 |
| `B1_DRAFT_READY` | 技术调研文档已产出，待审批 | 直接调用 QA 审批 |
| `B1_REVIEW_ROUND_N` | B1 第 N 轮审批完成（FAIL） | 扫描 reviews/ 读取最新审批报告，调用 Planning Agent 修改 |
| `B1_COMPLETED` | 技术调研已通过 | 从 B2 开始 |
| `B2_DRAFT_READY` | 技术架构文档已产出，待审批 | 直接调用 QA 审批 |
| `B2_REVIEW_ROUND_N` | B2 第 N 轮审批完成（FAIL） | 扫描 reviews/ 读取最新审批报告，调用 Planning Agent 修改 |
| `B2_COMPLETED` | 技术架构已通过 | 从 B3 开始 |
| `B3_DRAFT_READY` | 路线图已产出，待审批 | 直接调用 QA 审批 |
| `B3_REVIEW_ROUND_N` | B3 第 N 轮审批完成（FAIL） | 扫描 reviews/ 读取最新审批报告，调用 Planning Agent 修改 |
| `B3_COMPLETED` | 路线图已通过 | 从 B4 开始 |
| `B4_DRAFT_READY` | 工作流设计已产出，待审批 | 直接调用 QA 审批 |
| `B4_REVIEW_ROUND_N` | B4 第 N 轮审批完成（FAIL） | 扫描 reviews/ 读取最新审批报告，调用 Design Agent 修改 |
| `B4_COMPLETED` | 工作流设计已通过 | 进入阶段 C |

**恢复时的扫描逻辑：**

当检查点值为 `BN_REVIEW_ROUND_M` 时，Lead 需扫描 `projects/<name>/.agents/reviews/BN-xxx/` 目录，读取最新一轮审批报告文件，根据 frontmatter 中的 `result` 字段决定下一步：
- `FAIL` → 调用对应 Agent 修改（附带审批报告路径）
- `PASS` → 不应出现（PASS 时应已写入 BN_COMPLETED），但如果出现则直接标记为 BN_COMPLETED

### 回溯协议（轻量版）

当 QA 审批报告的 frontmatter 中包含 `upstream_concern` 且 `confidence: high` 时，Lead 暂停当前子阶段，向人类输出以下信息：

```
回溯预警

QA 在 B{当前阶段}（{阶段名}）审批中发现 B{目标阶段}（{阶段名}）的假设可能有误：
  "{suspect_assumption 内容}"

当前选项：
  [1] 回溯到 B{目标阶段} — 重新调研/设计该假设，后续产出物归档后重做
  [2] 继续但标记风险 — 在当前文档中标注此风险，阶段 C 实施时验证
  [3] 忽略 — QA 判断有误，当前假设成立

请选择 [1/2/3]：
```

**人类选择 [1] 后的处理：**
1. 归档当前及后续已完成的子阶段产出物：`reviews/BN-xxx/` → `reviews/_archived/BN-xxx-rollback-from-B{当前}/`
2. 相关 `docs/` 文件标记为 `needs_revision`（不删除）
3. 检查点回退到 `B{目标阶段}_DRAFT_READY`
4. 从目标阶段重新开始

`confidence: medium` 的 upstream_concern 仅记录在审批报告中，不触发升级。

---

## 阶段 C：进入开发

**阶段 B 全部完成后，必须新开一个 Claude Code session 才能进入阶段 C。** 原因：项目目录下的 `.claude/agents/*.md`、`.claude/settings.json` 和 `CLAUDE.md` 只有当 Claude Code 以该目录作为工作目录启动时才会被正确加载。在元仓库中 `cd` 到项目目录是无效的——Claude Code 加载的仍然是元仓库的 `.claude/` 配置。

### Lead 的操作

阶段 B 全部完成后，Lead 向人类输出以下提示：

```
阶段 B 已全部完成。请在项目目录下新开 Claude Code session 进入阶段 C：

cd projects/<project-name>/ && claude
```

### 新 session 启动后的行为

新 session 以 `projects/<project-name>/` 为工作目录启动后：

1. 项目 `CLAUDE.md`（由 design-agent 定制）会被自动加载为项目指令
2. 项目 `.claude/agents/*.md` 会被正确识别为可用 agent
3. 项目 `.claude/settings.json` 会自动生效
4. **从此刻起，完全按项目 CLAUDE.md 的指令行事**，不再参考元仓库 CLAUDE.md
5. 创建 Agent Team，按项目 CLAUDE.md 中的角色定义 spawn teammates
6. 按 `docs/roadmap.md` 中的 Phase 顺序推进开发

**上下文隔离：** 新开 session 天然实现了上下文窗口隔离——阶段 A/B 的对话历史不会占用阶段 C 的上下文空间。所有必要信息都已在阶段 B 持久化为文件，Lead 完全依赖读取文件获取上下文，不依赖对话记忆。

---

## 硬性约束（贯穿所有阶段）

- **Git 操作由你（Lead）独占**，Subagent 和 teammate 不做 git 操作
- **每个子阶段完成后做一次 git commit**
- **QA 审批使用严格模式**：按检查清单逐项检查，任何一项未通过都打回
- **审批意见文件化**：阶段 B 中 QA 审批报告写入 `.agents/reviews/` 目录，Agent 从文件读取修改意见，Lead 不复制粘贴意见内容
- **8 轮安全阀**：连续 8 个完整的"修改→审批"循环不通过时，要求 Agent 说明卡点原因，然后请求人工介入
- **人类介入最小化**：在阶段 A 识别、阶段 B4 尽可能自动化、不影响开发的延迟到最终交付

## 阶段 C 补充：test-case 前置生成与 issue 生命周期

阶段 C 的具体流程由项目 CLAUDE.md 定义，但以下规则是硬性的：

### test-case 前置生成

每个 Phase 开始时，Lead 在定义任务和接口的**同时**生成 `.agents/test-cases/phase-N-test-cases.md`。test-case 必须覆盖 roadmap 中该 Phase 的所有验收标准，可以增加额外用例但不能遗漏。

**test-case 文件格式：**

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

### issue 生命周期管理

- QA 将问题写入 `.agents/issues/phase-N/` 下的独立文件（按 phase 分子目录）
- issue 文件带 status frontmatter：

```yaml
---
status: open  # open / resolved
severity: Major
phase: 1
date: YYYY-MM-DD
---
```

- Agent 修复后将 status 改为 `resolved`
- QA 每轮重新跑全部 test-case，可新增 issue 也可将已 resolved 的 reopen（改回 `open`）
- **最终通过判定：** 不存在 Critical 或 Major 级别的 `open` issue。Minor 级别的 `open` issue 不阻塞通过，但记录在案
- Lead 通过 SendMessage 告知 Agent："QA 验收未通过，请阅读 `.agents/issues/phase-N/` 下的 issue 文件并逐一修复。" Agent 自行读取 issue 文件修复
