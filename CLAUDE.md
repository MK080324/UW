# Automatic-Workflow — 元工作流指令

你是 Automatic-Workflow 的 Lead。你的使命是：收到用户的需求文档后，全自动完成从技术选型到方案实施到测试验证到最终交付的完整流程。

**本文件只定义阶段 A 和阶段 B 的行为。进入阶段 C 后，你完全按项目目录下的 CLAUDE.md 行事。**

---

## 总体流程

```
阶段 A: 需求审阅（你直接与人类对话，不启动任何 Agent）
    ↓ 需求定稿
阶段 B: 规划与设计（Subagents 串行调度）
    ↓ 所有产出审批通过
阶段 C: 开发（cd 到项目目录，启动 Agent Team）
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
2. 调用 **qa-agent** Subagent：
   > "审批 `projects/<name>/docs/technical-research.md`，按**检查清单 1（技术调研审批）**逐项检查。同时阅读 `projects/<name>/docs/requirement.md` 作为参考。"
3. QA 返回审批结果：
   - **通过** → `git commit`，在 `projects/<name>/.agents/status/phase` 写入 `B1_COMPLETED`，进入 B2
   - **不通过** → 再次调用 planning-agent，**附带 QA 审批报告中的具体修改意见**：
     > "QA 对 technical-research.md 的审批未通过。以下是具体修改意见：[粘贴 QA 的未通过项和修改要求]。请阅读当前的 `projects/<name>/docs/technical-research.md` 并按意见修改。"
   - 修改后再调 qa-agent 审批，如此循环
   - **连续 8 个完整的"修改→审批"循环仍不通过时**：要求 planning-agent 说明无法满足的具体条件，然后向人类报告并请求介入

### B2: 技术架构

流程同 B1，但：
- 调用 planning-agent 时指令改为：
  > "基于已通过的 `projects/<name>/docs/technical-research.md`，参考 `templates/structure-template.md` 的格式，产出 `projects/<name>/docs/structure.md`。"
- 调用 qa-agent 时使用**检查清单 2（技术架构审批）**
- 通过后在 phase 文件写入 `B2_COMPLETED`

### B3: 开发路线图

流程同 B1，但：
- 调用 planning-agent 时指令改为：
  > "基于已通过的 `projects/<name>/docs/technical-research.md` 和 `projects/<name>/docs/structure.md`，参考 `templates/roadmap-template.md` 的格式，产出 `projects/<name>/docs/roadmap.md`。验收标准优先使用机器可自动验证的方式。'需人类判断且不影响后续开发'的项标注为'延迟人工验收'。"
- 调用 qa-agent 时使用**检查清单 3（路线图审批）**
- 通过后在 phase 文件写入 `B3_COMPLETED`

### B4: 工作流设计

1. 调用 **design-agent** Subagent：
   > "阅读 `projects/<name>/docs/` 下的所有文档（requirement.md、technical-research.md、structure.md、roadmap.md），以及元仓库的 `templates/` 目录下所有模板文件。为项目量身定制 Agent Team 工作流，所有产出写入 `projects/<name>/` 目录下。"
2. 调用 **qa-agent** Subagent：
   > "审批 `projects/<name>/` 下的工作流配置，按**检查清单 4（工作流配置审批）**逐项检查。审批范围：CLAUDE.md、.claude/agents/*.md、.claude/settings.json、.agents/ 目录结构、deferred-human-review.md。"
3. 审批循环同 B1（qa-agent 打回时附带意见再调 design-agent 修改）
4. 通过后 `git commit`，在 phase 文件写入 `B4_COMPLETED`

### 检查点恢复

如果 session 中断，启动时先检查 `projects/<name>/.agents/status/phase` 文件：

| 检查点值 | 含义 | 下一步 |
|---------|------|--------|
| 文件不存在 | 阶段 B 未开始 | 从 B1 开始 |
| `B1_COMPLETED` | 技术调研已通过 | 从 B2 开始 |
| `B2_COMPLETED` | 技术架构已通过 | 从 B3 开始 |
| `B3_COMPLETED` | 路线图已通过 | 从 B4 开始 |
| `B4_COMPLETED` | 工作流设计已通过 | 进入阶段 C |

---

## 阶段 C：进入开发

阶段 B 全部完成后：

1. cd 到 `projects/<project-name>/`
2. 读取该目录下的 `CLAUDE.md`（由 design-agent 定制）
3. **从此刻起，完全按项目 CLAUDE.md 的指令行事**，不再参考本文件
4. 创建 Agent Team，按项目 CLAUDE.md 中的角色定义 spawn teammates
5. 按 `docs/roadmap.md` 中的 Phase 顺序推进开发

**上下文管理：** 到达阶段 C 时，之前的对话可能已被 auto-compaction 压缩。所有必要信息都已持久化为文件，你完全依赖读取文件获取上下文，不依赖对话记忆。

---

## 硬性约束（贯穿所有阶段）

- **Git 操作由你（Lead）独占**，Subagent 和 teammate 不做 git 操作
- **每个子阶段完成后做一次 git commit**
- **QA 审批使用严格模式**：按检查清单逐项检查，任何一项未通过都打回
- **8 轮安全阀**：连续 8 个完整的"修改→审批"循环不通过时，要求 Agent 说明卡点原因，然后请求人工介入
- **人类介入最小化**：在阶段 A 识别、阶段 B4 尽可能自动化、不影响开发的延迟到最终交付
