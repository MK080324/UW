# Automatic-Workflow

一个元工作流：你提交一份需求文档，Claude Code 全自动完成技术选型、架构设计、路线图规划、工作流定制和完整开发。

## 工作原理

```
你写一份需求文档
        ↓
   阶段 A: Claude 与你一起审阅需求（不启动 Agent）
        ↓
   阶段 B: Subagents 串行规划项目
        ├── B1: 技术调研
        ├── B2: 架构设计
        ├── B3: 开发路线图
        └── B4: 定制工作流与 Agent Team
        ↓
   阶段 C: Agent Team 并行开发
        ↓
   完成。你收到代码 + 一份需要你确认的清单。
```

**阶段 A** — Claude 阅读你的需求文档，找出遗漏和歧义，标记出后续可能需要人类操作的步骤（如 `sudo` 命令、视觉验证），与你讨论并确认最终版本。

**阶段 B** — 三个 Subagent 串行工作。**Planning Agent**（Opus）进行技术调研、设计架构、编写路线图。**QA Agent**（Sonnet）按严格检查清单逐项审批每个产出物。**Design Agent**（Opus）根据项目特征量身定制 Agent Team 工作流——决定需要几个开发 Agent、每个负责什么、QA 如何验收。

**阶段 C** — Claude 切换到项目目录，启动刚设计好的 Agent Team。开发 Agent 并行编码，QA 在每个 Phase 完成后验收，全程自动运行直到项目完成。所有需要人类主观判断的事项（如"这个渐变色好不好看"）都被延迟到最后统一通知你。

## 前置要求

- [Claude Code](https://claude.ai/code) >= v2.1.47
- Node.js >= 18
- Git

检查 Claude Code 版本：

```bash
claude --version
```

如果版本过低：

```bash
sudo npm install -g @anthropic-ai/claude-code@latest
```

## 快速开始

### 1. 克隆仓库

```bash
git clone git@github.com:MK080324/UW.git
cd UW
```

### 2. 编写需求文档

创建一个 Markdown 文件，内容建议包含：

- **项目名称和简述** — 你在做什么
- **功能列表** — 按优先级排列，尽量附带验收标准
- **技术偏好**（可选）— 语言、框架、平台约束
- **性能要求**（可选）— 延迟、吞吐量、文件大小限制
- **目标平台** — macOS、Linux、Web 等

示例（`my-requirement.md`）：

```markdown
# 我的 CLI 工具

一个将 CSV 文件转换为 JSON 的命令行工具，支持流式处理。

## 功能
1. 从 stdin 或文件路径读取 CSV
2. 输出 JSON 到 stdout 或文件
3. 通过流式处理支持大文件（> 1GB）
4. 支持自定义分隔符（逗号、制表符、管道符）
5. 自动检测表头行

## 技术约束
- 语言：Rust
- 无外部运行时依赖
- 单二进制分发

## 性能要求
- 1GB CSV → JSON 在 30 秒内完成
- 无论输入大小，内存占用 < 100MB
```

### 3. 在本目录启动 Claude Code

```bash
claude
```

### 4. 把需求文档交给 Claude

```
这是我的需求文档：/path/to/my-requirement.md
```

Claude 会：
1. 与你一起审阅需求（阶段 A）
2. 请你确认最终版本
3. 自动完成规划与设计（阶段 B）
4. 自动切换到开发（阶段 C）
5. 完成后通知你，附带需要人工确认的事项清单

### 5. 等待

从阶段 B 到阶段 C 全程自动。你只需要在以下情况介入：

- Claude 请你澄清需求（阶段 A）
- QA 循环连续 8 轮未通过（罕见 — Claude 会解释卡在哪里）
- 最终交付包含需要你目视/手动验证的事项

## 产出物结构

阶段 B 完成后，你的项目出现在 `projects/<project-name>/` 下：

```
projects/<project-name>/
├── CLAUDE.md                  # 为阶段 C 定制的 Lead 指令
├── .claude/
│   ├── settings.json
│   └── agents/                # 定制的开发 Agent + QA Agent
├── docs/
│   ├── requirement.md         # 定稿后的需求文档
│   ├── technical-research.md  # 技术调研与选型
│   ├── structure.md           # 架构与模块设计
│   └── roadmap.md             # 分阶段开发计划
├── .agents/
│   ├── interfaces/            # 跨模块接口契约
│   ├── issues/                # QA 提出的问题
│   ├── status/phase           # 进度检查点
│   └── deferred-human-review.md  # 延迟人工验收清单
└── <src>/                     # 项目源码
```

## 关键设计决策

| 决策项 | 选择 | 原因 |
|--------|------|------|
| 阶段 B 调度方式 | Subagents（串行） | 规划是顺序的，不需要并行 |
| 阶段 C 调度方式 | Agent Teams（并行） | 多个开发 Agent 同时编码不同模块 |
| 阶段 C 的 QA 角色 | Subagent（按需调用） | QA 验收是串行的（等开发完才验收），常驻 teammate 空耗 token |
| QA 重试策略 | 无硬性上限，连续 8 轮后升级 | 信任 Agent 自主判断，安全阀防死循环 |
| 人工验收项 | 延迟到最终交付 | 最大化自动化，不因审美判断阻塞开发 |
| Git 操作 | 仅 Lead 执行 | 防止并发 commit 冲突 |

## 自定义

### 模板

`templates/` 目录包含所有产出物的格式模板。你可以修改它们以匹配你的文档规范：

- `technical-research-template.md` — 技术调研格式
- `structure-template.md` — 架构文档格式
- `roadmap-template.md` — 开发路线图格式
- `project-claude-md-template.md` — 项目 Lead 指令格式
- `dev-agent-template.md` — 开发 Agent 配置格式
- `qa-agent-template.md` — QA Agent 配置格式

### Agent 配置

`.claude/agents/` 目录定义了阶段 B 的三个 Subagent：

- `planning-agent.md` — Opus，50 turns，配备 WebSearch/WebFetch
- `design-agent.md` — Opus，50 turns，设计阶段 C 的工作流
- `qa-agent.md` — Sonnet，60 turns，严格检查清单审批

### 断点恢复

如果阶段 B 期间 Claude Code 会话中断，在本目录重新启动即可。Claude 会读取检查点文件（`.agents/status/phase`），从上次中断处继续。

阶段 C 有自己的检查点机制，定义在项目的 `CLAUDE.md` 中。

## License

[MIT](LICENSE)
