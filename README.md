<p align="center">
  <h1 align="center">🛠️ Feishu PRD Workflow</h1>
  <p align="center">
    <strong>飞书 PRD → 技术方案 → 代码交付 · 一条指令跑通全流程</strong>
  </p>
  <p align="center">
    <img src="https://img.shields.io/badge/platform-Hermes%20|%20Claude%20Code%20|%20Cursor%20|%20OpenCode%20|%20Windsurf-blue" alt="platforms">
    <img src="https://img.shields.io/badge/license-MIT-green" alt="license">
    <img src="https://img.shields.io/badge/skills-10%20modular%20sub--skills-orange" alt="skills">
    <img src="https://img.shields.io/badge/stages-8%20phase%20pipeline-purple" alt="stages">
  </p>
</p>

---

## 📖 这是什么

一套**平台无关的 AI Agent Skill 集合**，让任何 AI Agent（Hermes / Claude Code / Cursor / OpenCode / Windsurf 等）都能按照标准化流程，把飞书 PRD 文档一步步转化为可交付的代码。

- 🧩 **9 个独立子 Skill**：每个阶段可单独使用
- 🔄 **8 阶段完整链路**：PRD → API → 方案 → 用例 → 分支 → 代码 → MR → 验收
- 🌍 **不绑定平台**：Agent 自己判断用什么工具，skill 只定义"做什么"
- 📁 **产物落盘**：每个阶段产出结构化文件，可追溯、可断点续跑

---

## 🚀 30 秒快速开始

```bash
# 1. 安装（任选一种）
hermes skill install SKILL.md          # Hermes
cp -r . .claude/skills/feishu-prd/     # Claude Code
cp -r . .cursor/rules/feishu-prd/      # Cursor

# 2. 对你的 Agent 说
按 feishu-prd-workflow 处理：

PRD：https://xxx.feishu.cn/wiki/xxx
API：https://xxx.feishu.cn/wiki/xxx
项目路径：/path/to/project
项目名称：my-project
Release：release/1.0.0
PingCode：#PROJ-123
```

Agent 会自动按阶段执行，每步播报进度，产物落在 `项目路径/outputs/` 下。

---

## 🧩 子 Skill 一览

| # | 子 Skill | 做什么 | 输入 | 输出 |
|---|----------|--------|------|------|
| 1 | `read-prd` | 读取飞书 PRD，提取结构化摘要 | 飞书 URL / 导出文本 | `prd-summary.md` |
| 2 | `read-api` | 读取 API 文档，提取接口清单 | 飞书 URL / 导出文本 | `api-summary.md` |
| 3 | `tech-doc` | 生成前端技术方案 | PRD 摘要 + API 摘要 | `tech-doc-summary.md` |
| 4 | `test-cases` | 生成测试用例（冒烟/主流程/边界/联动） | PRD + API + 技术方案 | `test-cases-summary.md` |
| 5 | `git-branch` | 创建 feature / hotfix 开发分支 | 项目路径 + 任务名 | 分支已推送 |
| 6 | `write-code` | 按技术方案写代码 | 技术方案 + 测试用例 | `changes-summary.md` |
| 7 | `git-finish` | 提交代码 + 推送 + 创建 MR | PingCode ID + 功能摘要 | `mr-info.md` |
| 8 | `git-workflow` | 双模通用 Git 工作流（脏写自愈+MR一键收口） | 项目路径 + 任务名 + PingCode ID | 一键合规交付创建 MR |
| 9 | `browser-qa` | 浏览器自动化验收 | 测试用例 + 目标 URL | `qa-verification.md` |
| 10 | `final-verify` | 最终复核 + 结论 + 下一步建议 | 产物目录 | `final-verification.md` |

---

## 🎯 三种使用姿势

### 姿势 1：完整工作流（最常用）

```
按 feishu-prd-workflow 处理：

PRD：https://xxx.feishu.cn/wiki/xxx
API：https://xxx.feishu.cn/wiki/xxx
项目路径：/path/to/project
项目名称：my-project
Release：release/1.0.0
PingCode：#PROJ-123
```

Agent 自动走完 Phase 1→8，每阶段播报 + 产物落盘。

### 姿势 2：只用某个阶段

```
# 只要 PRD 摘要（不需要写代码）
用 feishu-prd-workflow/read-prd 读这个：
https://xxx.feishu.cn/wiki/xxx

# 只要测试用例（已有 PRD 和 API 摘要）
用 feishu-prd-workflow/test-cases 生成测试用例：
PRD 摘要：./outputs/xxx/prd-summary.md
API 摘要：./outputs/xxx/api-summary.md

# 创建 feature 分支
用 feishu-prd-workflow/git-branch 创建 feature 分支：
项目路径：/path/to/project
Release：release/1.24.0
任务名：refund-photo-lock

# 创建 hotfix 分支（线上紧急修复）
用 feishu-prd-workflow/git-branch 创建 hotfix 分支：
项目路径：/path/to/project
模式：hotfix
任务名：fix-payment-crash

# 只提交 MR（代码已写完）
用 feishu-prd-workflow/git-finish 收口：
项目路径：/path/to/project
PingCode：#PROJ-123
功能摘要：退款照片锁定功能

# 🌟 推荐：用大一统通用双模工作流（支持脏写自愈、中文驼峰重构与动态 MR 指派）
# 动作 1：拉取并安全搬迁分支
用 feishu-prd-workflow/git-workflow 执行 gitBranch：
项目路径：/path/to/project
Release：1.0.5                  # 智能自愈：1.0.5 -> release-1.0.5
任务名：退款图片锁定             # 智能翻译并驼峰化 -> refundPhotoLock

# 动作 2：一键规范提交并指派 MR
用 feishu-prd-workflow/git-workflow 执行 gitFinish：
项目路径：/path/to/project
PingCode：#PROJ-123
功能摘要：退款照片锁定功能

# 只跑浏览器验收
用 feishu-prd-workflow/browser-qa 跑测试：
测试用例：./test-cases-summary.md
目标页面：https://xxx.com/page
```

### 姿势 3：半路接续

```
# 从产物目录继续（中断了也不怕）
从 outputs/20260519_143000_#PROJ-123_my-project_refund-lock/ 继续

# 从指定阶段开始（前面阶段手动完成了）
从 feishu-prd-workflow/write-code 开始，
前面产物在：./outputs/xxx/
项目路径：/path/to/project
```

---

## 📁 产物在哪

每次执行会在项目路径下生成：

```
outputs/
└── 20260519_143000_#PROJ-123_my-project_refund-lock/
    │                        │         │        │
    │                        │         │        └─ 任务名
    │                        │         └─ 项目名
    │                        └─ PingCode ID
    └─ 时间戳
    │
    ├── workflow-state.json          # 🔄 工作流状态（断点续跑靠它）
    ├── prd/
    │   └── prd-summary.md           # 📋 PRD 结构化摘要
    ├── api/
    │   └── api-summary.md           # 🔌 API 接口清单
    ├── tech-doc/
    │   └── tech-doc-summary.md      # 🏗️ 技术方案
    ├── test-cases/
    │   └── test-cases-summary.md    # 🧪 测试用例
    ├── git/
    │   ├── branch-info.md           # 🌿 分支信息
    │   └── mr-info.md               # 📬 MR 信息
    ├── code/
    │   └── changes-summary.md       # 💻 代码变更记录
    ├── qa/
    │   └── qa-verification.md       # ✅ 浏览器测试结果
    └── summary/
        └── final-verification.md    # 🎯 最终验收结论
```

> 💡 **想换目录？** Agent 启动时会先问你"产物放在哪？"——直接告诉它新路径就行。

---

## 🔄 断点续跑

工作流支持断点续跑。Agent 每完成一个阶段都会更新 `workflow-state.json`：

```json
{
  "current_phase": 4,
  "status": "running",
  "phases": {
    "1": { "status": "completed" },
    "2": { "status": "completed" },
    "3": { "status": "completed" },
    "4": { "status": "running" }
  }
}
```

中断后只需告诉 Agent：

```
从 outputs/20260519_143000_xxx/ 继续上次的工作流
```

Agent 会自动从 `current_phase` 继续，不会重复执行已完成的阶段。

---

## 🌍 跨平台兼容

相同的工作流，不同 Agent 怎么跑：

| 平台 | 飞书直读 | 代码实现 | 浏览器测试 | Git/MR | 安装方式 |
|------|:---:|:---:|:---:|:---:|------|
| **Hermes** | ✅ feishu MCP | ✅ 内置 | ✅ chrome-devtools | ✅ glab | `hermes skill install` |
| **Claude Code** | ⚠️ 需导出文本 | ✅ 内置 | ✅ 如配 MCP | ✅ 内置 git | 放 `.claude/skills/` |
| **Cursor** | ⚠️ 需导出文本 | ✅ 内置 | ❌ 需手动 | ✅ 终端 | 放 `.cursor/rules/` |
| **OpenCode** | ⚠️ 需导出文本 | ✅ 内置 | ❌ 需手动 | ✅ 终端 | 放 `~/.opencode/skills/` |
| **Windsurf** | ⚠️ 需导出文本 | ✅ 内置 | ❌ 需手动 | ✅ 终端 | 放 skills 目录 |

> **核心原则**：skill 只定义"做什么"，Agent 自己判断"用什么做"。飞书读不了就把文档导出 Markdown 粘贴给 Agent 即可。

---

## 🎛️ Agent 行为协议

每个 Agent 加载 skill 后必须遵守：

### 0. 环境检查——缺什么补什么（不在空白环境硬跑）

每个阶段开始前，Agent 必须先**静默检查**自己有没有对应的工具能力：

```
🔍 正在检查环境...
✅ 飞书文档读取就绪（feishu MCP）
✅ Git 就绪
⚠️ glab 未安装 → 将用 GitLab API 创建 MR
⚠️ 浏览器工具未检测到 → 问用户要不要装
```

具备 → 直接开始。不具备 → 引导安装，给出精确命令让用户确认。**永不在空白环境硬跑。**

### 1. 首轮对话——确认配置（不猜不跳步）

```
我将按 feishu-prd-workflow 开始处理。

📁 产物目录：/path/to/project/outputs/  ← 要改吗？

可选阶段开关（默认关闭）：
  ✍️  代码实现：[开/关]
  📬 Git 收口：[开/关]
  ✅ 浏览器测试：[开/关]

📋 输入确认：
  PRD：https://xxx.feishu.cn/wiki/xxx ✅
  API：https://xxx.feishu.cn/wiki/xxx ✅
  项目路径：/path/to/project ✅
  PingCode：#PROJ-123 ✅

确认无误后我立即开始。
```

### 2. 每阶段切换——播报进度

```
━━━ Phase 3/8：技术方案 ━━━
📊 PRD 摘要：N 个模块 / M 个交互流程
📊 API 摘要：N 个接口 / M 个页面映射
🏗️ 技术方案完成：
   新建 3 个文件 / 修改 2 个文件 / 风险点 4 处
📄 产物：tech-doc-summary.md + json

▶️  下一步 Phase 4：生成测试用例
```

### 3. 产物落盘——完成即写

每个阶段完成后**立即**写入产物目录，不攒到最后。

### 4. 不懂就问

遇到需要用户决策的地方（文件冲突、方案选择、权限边界），停下来问，不猜。

---

## 📦 安装

```bash
# Hermes（推荐）
hermes skill install SKILL.md

# Claude Code
mkdir -p .claude/skills/feishu-prd-workflow
cp SKILL.md skills/*.md .claude/skills/feishu-prd-workflow/

# Cursor
mkdir -p .cursor/rules/feishu-prd-workflow
cp SKILL.md skills/*.md .cursor/rules/feishu-prd-workflow/

# OpenCode
mkdir -p ~/.opencode/skills/feishu-prd-workflow
cp SKILL.md skills/*.md ~/.opencode/skills/feishu-prd-workflow/

# 其他平台
# 把整个文件夹放到对应平台的 skills/rules 目录
```

---

## 🗺️ 工作流全景图

```
┌─────────────────────────────────────────────────────────┐
│                    完整工作流（8 阶段）                    │
│                                                         │
│  Phase 1         Phase 2         Phase 3        Phase 4 │
│ ┌─────────┐    ┌─────────┐    ┌──────────┐   ┌────────┐│
│ │ read-prd│───▶│read-api │───▶│ tech-doc │──▶│test-   ││
│ │         │    │         │    │          │   │cases   ││
│ └─────────┘    └─────────┘    └──────────┘   └────────┘│
│      │              │               │              │    │
│      ▼              ▼               ▼              ▼    │
│  ┌──────────────────────────────────────────────────┐   │
│  │              可选阶段（用户确认后开启）            │   │
│  │                                                  │   │
│  │  ┌────────────┐   ┌───────────┐   ┌───────────┐ │   │
│  │  │ git-branch │──▶│write-code │──▶│git-finish │ │   │
│  │  │ (拉分支)   │   │ (写代码)  │   │ (提交+MR) │ │   │
│  │  └────────────┘   └───────────┘   └───────────┘ │   │
│  │                                                  │   │
│  │  ┌───────────┐                                   │   │
│  │  │browser-qa │  (浏览器验收，独立分支)            │   │
│  │  └───────────┘                                   │   │
│  └──────────────────────────────────────────────────┘   │
│                         │                               │
│                         ▼                               │
│                    ┌─────────────┐                      │
│                    │final-verify │  (最终复核 + 结论)    │
│                    └─────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

---

## 🔌 前置依赖

使用前确保你的 Agent 具备以下能力（至少一项）：

| 能力 | 用途 | 没有怎么办 |
|------|------|-----------|
| 飞书文档读取 | 读 PRD / API 文档 | 导出 Markdown 粘贴给 Agent |
| 代码读写 | 写代码（Phase 5） | 跳过 Phase 5，Agent 只出方案 |
| Git 操作 | 分支 / 提交 / MR | 跳过 Phase 5-6，手动操作 |
| 浏览器操控 | 自动化验收（Phase 7） | 跳过 Phase 7，手动测试 |

> **最小可用**：只要 Agent 能读文档 + 输出文本，就能跑 Phase 1-4（PRD 分析 + 技术方案 + 测试用例）。

---

## ❓ 常见问题

<details>
<summary><b>Q: Agent 读不了飞书怎么办？</b></summary>

把飞书文档导出为 Markdown 或 PDF，然后把内容粘贴给 Agent，或者放到本地文件让 Agent 读取。

```
按 feishu-prd-workflow 处理，
PRD 内容在文件 /path/to/prd-export.md
```
</details>

<details>
<summary><b>Q: 只想跑到测试用例，不写代码？</b></summary>

Agent 首轮确认时，把"代码实现"关掉就行。或者说：

```
按 feishu-prd-workflow 处理，只要前 4 个阶段（PRD→用例）
```
</details>

<details>
<summary><b>Q: 产物目录在哪？</b></summary>

默认 `{{项目路径}}/outputs/`。Agent 首轮会跟你确认，你可以随时改成任意路径。
</details>

<details>
<summary><b>Q: 中断了怎么续跑？</b></summary>

```
从 outputs/20260519_143000_xxx/ 继续上次的工作流
```

Agent 读 `workflow-state.json`，从上次中断的阶段继续，不重复已完成阶段。
</details>

<details>
<summary><b>Q: 不同 Agent 跑出来的结果一样吗？</b></summary>

流程和产物结构一样。具体内容质量取决于 Agent 自身的模型能力。建议用推理能力强的模型（Claude Sonnet 4 / GPT-5 / DeepSeek V4 等）。
</details>

<details>
<summary><b>Q: 这个和 workflow-orchestrator 的关系？</b></summary>

`workflow-orchestrator` 是一个 TypeScript 项目，需要 Node.js 环境运行，专为 Hermes + Claude Code 协同设计。

`feishu-prd-workflow`（本项目）是一个纯 Skill 定义文件，不需要任何运行时，装在任意 AI Agent 里就能用。更轻量、更通用。
</details>

---

## 📂 项目结构

```
feishu-prd-workflow/
├── SKILL.md                 # 主编排器
├── README.md                # 你正在看的这个文件
├── skills/                  # 9 个子 Skill（每个可独立使用）
│   ├── read-prd.md
│   ├── read-api.md
│   ├── tech-doc.md
│   ├── test-cases.md
│   ├── git-branch.md
│   ├── write-code.md
│   ├── git-finish.md
│   ├── git-workflow.md
│   ├── browser-qa.md
│   └── final-verify.md
└── examples/
    └── prompts.md           # 各平台触发示例
```

---

## 📄 License

MIT © allen

---

<p align="center">
  <sub>如果这个项目帮到了你，给个 ⭐ Star 吧～</sub>
</p>
