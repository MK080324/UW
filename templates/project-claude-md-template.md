# [项目名称] — 项目指令

<!--
此文件由 Design Subagent 为具体项目量身定制。
它是阶段 C（Agent Teams 并行开发）中 Lead 的行为指令。
-->

## 项目概述

<!-- 从需求文档提炼的项目简述 -->

## 核心架构

<!-- 从 structure.md 提炼的架构摘要，让所有 Agent 快速理解全局 -->

## 目录边界

<!-- 明确哪个 Agent 负责哪些目录，确保不重叠 -->

- `<dir-1>/` — [Agent 1] 负责
- `<dir-2>/` — [Agent 2] 负责
- 根级配置文件 — 谁需要谁改，改前通知 Lead

## 接口/通信约定

<!-- 如果项目有跨模块接口（如前后端 API、IPC），在此定义接口模板 -->

### 接口定义模板

Lead 在每个 Phase 开始前，按以下格式定义本 Phase 的所有接口：

```
### [接口类型]: [接口名]
- 方向: [调用方] → [被调方]
- 参数: { field: type, ... }
- 返回值: { field: type, ... }
- 错误: 可能的错误类型及含义
```

开发 Agent 必须严格按此接口开发。接口变更必须通过 SendMessage 通知相关 Agent 和 Lead，等待确认后才能继续。

## Agent Teams 工作流

### 角色

| 角色 | 职责 | 工作目录 |
|------|------|---------|
| **Lead** | 任务分解、接口定义、Phase 推进、QA 调度、git 操作 | 全局 |
| **[Dev Agent 1]** | ... | `<dir>/` |
| **[Dev Agent 2]** | ... | `<dir>/` |
| **QA** (Subagent) | 代码验收、测试执行、issue 产出 | 全局（由 Lead 按需调用） |

### 流程

每个 Phase 按以下步骤执行：

1. **Lead 定义任务和接口** — 按接口定义模板写明本 Phase 的所有接口，发布到 .agents/interfaces/
2. **并行开发** — 开发 Agent 按接口并行实现各自部分
3. **接口沟通** — 如需调整接口，Agent 必须通过 SendMessage 通知相关方和 Lead，等待确认后才能继续
4. **完成通知** — 每个 Agent 完成后通过 SendMessage 向 Lead 发送完成通知
5. **QA 验收** — Lead 收到所有开发 Agent 的完成通知后，调用 QA Subagent 验收
6. **问题修复** — QA 不通过时，Lead 将 **QA 验收报告中的具体问题和修复意见** 通过 SendMessage 转发给对应 Agent 修复。无硬性轮数上限，连续 8 个完整的"修复→验收"循环仍不通过时，Lead 要求 Agent 说明无法满足的具体条件，然后请求人工介入
7. **通过 → git commit → 更新检查点 → 下一个 Phase**

### 并行策略

<!-- 哪些 Phase 可以并行，哪些必须串行 -->

### Git 操作约定

- **Lead 负责所有 git 操作**（commit、push、branch）
- 开发 Agent 和 QA 只写代码/测试，不做 git 操作
- 每个 Phase 验收通过后，Lead 做一次 commit

### 检查点机制

每个 Phase 验收通过后，Lead 更新 `.agents/status/phase` 文件：
```
C_PHASE_0_COMPLETED
C_PHASE_1_COMPLETED
...
C_COMPLETED
```

如果 session 中断，新 session 启动时 Lead 先检查此文件，从上次完成的 Phase 之后继续。

## 技术要求

<!-- 编码规范、lint 规则、测试要求等 -->

## 验收标准速查

| Phase | 自动验收（阻塞） | 延迟人工验收（不阻塞） |
|-------|----------------|---------------------|
| 0 | ... | — |
| 1 | ... | ... |

## 参考文档

- 需求文档：docs/requirement.md
- 技术调研：docs/technical-research.md
- 技术架构：docs/structure.md
- 开发路线图：docs/roadmap.md
