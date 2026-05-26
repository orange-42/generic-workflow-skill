---
name: feishu-prd-workflow
description: >-
  飞书 PRD → 代码交付完整工作流（编排器）。
  自动按序调用子 skill：read-prd → read-api → tech-doc → test-cases → [write-code] → [git-finish] → [browser-qa] → final-verify。
  平台无关，任何 AI Agent 可独立闭环。
version: 2.0.0
author: allen
license: MIT
includes:
  - feishu-prd-workflow/read-prd
  - feishu-prd-workflow/read-api
  - feishu-prd-workflow/tech-doc
  - feishu-prd-workflow/test-cases
  - feishu-prd-workflow/git-branch
  - feishu-prd-workflow/write-code
  - feishu-prd-workflow/git-finish
  - feishu-prd-workflow/git-workflow
  - feishu-prd-workflow/browser-qa
  - feishu-prd-workflow/final-verify
---

# 飞书 PRD → 代码交付 完整工作流

## 这是什么

一套模块化的飞书 PRD 工作流 Skill 集合。包含 8 个独立子 Skill + 1 个编排器。

**你可以：**
- 完整跑：一句话触发全部 8 阶段
- 挑着跑：只用某个子 Skill（例如只要 PRD 摘要、只要测试用例）
- 半路跑：从中间某个阶段开始

---

## 子 Skill 速查

| 子 Skill | 做什么 | 单独触发 |
|----------|--------|----------|
| read-prd | 读取飞书 PRD，输出结构化摘要 | `feishu-prd-workflow/read-prd` |
| read-api | 读取飞书 API 文档，输出接口清单 | `feishu-prd-workflow/read-api` |
| tech-doc | 基于 PRD+API 生成技术方案 | `feishu-prd-workflow/tech-doc` |
| test-cases | 生成测试用例 | `feishu-prd-workflow/test-cases` |
| git-branch | 创建 feature/hotfix 分支 + 推送 | `feishu-prd-workflow/git-branch` |
| write-code | 按技术方案写代码 | `feishu-prd-workflow/write-code` |
| git-finish | 提交代码 + 创建 MR | `feishu-prd-workflow/git-finish` |
| git-workflow | 双模通用 Git 工作流（脏写自愈+MR一键收口） | `feishu-prd-workflow/git-workflow` |
| browser-qa | 浏览器自动化验收 | `feishu-prd-workflow/browser-qa` |
| final-verify | 最终复核 + 结论 | `feishu-prd-workflow/final-verify` |

> 子 Skill 文件在 `skills/` 目录下，每个都是独立的、可安装的 Skill。

---

## 快速开始

### 完整工作流（推荐）

对 Agent 说：

```
按 feishu-prd-workflow 处理：

PRD：https://xxx.feishu.cn/wiki/xxx
API：https://xxx.feishu.cn/wiki/xxx
项目路径：/path/to/project
项目名称：your-project
Release：release/1.0.0
PingCode：#STORE-0000
```

### 只用某个阶段

```
用 feishu-prd-workflow/read-prd 帮我读这个 PRD：
https://xxx.feishu.cn/wiki/xxx
```

```
用 feishu-prd-workflow/test-cases 生成测试用例，
PRD 摘要已在这：./outputs/xxx/prd-summary.md
API 摘要已在这：./outputs/xxx/api-summary.md
```

```
从 feishu-prd-workflow/write-code 开始，
前面阶段已完成，产物在：./outputs/xxx/
项目路径：/path/to/project
```

---

## Agent 首轮对话协议

无论触发完整工作流还是子 Skill，Agent **必须先确认**：

1. **产物目录**：默认 `{{project_path}}/outputs/<timestamp>_<pingcode>_<project>_<task>/`，是否更改？
2. **可选阶段开关**（完整工作流时）：代码实现/Git 收口/浏览器测试 开还是关？
3. **输入完整性**：缺什么就问什么

---

## 完整工作流阶段

```
Phase 1          Phase 2          Phase 3          Phase 4
[read-prd]  →   [read-api]   →   [tech-doc]   →  [test-cases]
    │                │                │               │
    ▼                ▼                ▼               ▼
              ┌──── 可选阶段 ────┐
              ▼                 ▼
         git-branch         browser-qa
         (拉分支)           (浏览器验收)
              │                 │
              ▼                 │
         write-code             │
         (写代码)               │
              │                 │
              ▼                 │
         git-finish             │
         (提交+MR)              │
              │                 │
              └────────┬────────┘
                       ▼
                 final-verify
                 (最终验收)
```

- Phase 1-4 + git-branch：核心链路
- write-code / git-finish：可选（用户确认后开启）
- browser-qa：可选（用户明确要求时才跑）
- final-verify：必跑（至少复核已完成的阶段）

> [!TIP]
> **🌟 新版推荐：`git-workflow` 终极单兵武器**
> 本工作流现已引入全新的 `git-workflow` 通用大一统技能。它完美合并并超越了原本离散的 `git-branch` 与 `git-finish` 两个步骤，天生支持**脏写自动安全迁移**、**中文任务名称智能地道翻译与小驼峰化**、**指派人 GitLab ID 动态检索与降级兜底**等超一流心智。在开发日常中，强烈建议优先使用 `git-workflow` 闭环承接整个分支到提交生命周期！

---

## 产物目录

```
outputs/<timestamp>_<pingcode>_<project>_<task>/
├── workflow-state.json          # 工作流状态（断点续跑）
├── prd/
│   └── prd-summary.md/json      # Phase 1
├── api/
│   └── api-summary.md/json      # Phase 2
├── tech-doc/
│   └── tech-doc-summary.md/json # Phase 3
├── test-cases/
│   └── test-cases-summary.md    # Phase 4
├── code/
│   └── changes-summary.md       # Phase 5
├── git/
│   ├── branch-info.md            # Phase 5-prep：分支信息
│   └── mr-info.md                # Phase 6：MR 信息
├── qa/
│   └── qa-verification.md       # Phase 7
└── summary/
    └── final-verification.md    # Phase 8
```

---

## 断点续跑

```
从 outputs/20260519_143000_xxx/ 继续上次的工作流
```

Agent 读 `workflow-state.json`，从 `current_phase` 继续。

---

## ⚙️ Agent 环境检查协议（重要）

Agent 加载此 skill 后，**每个阶段开始前必须先做环境检查**。具备条件直接开始，不具备则引导用户补齐。

### 检查矩阵

| 阶段 | 需要的能力 | 检查方式 | 不具备时 |
|------|-----------|---------|----------|
| read-prd | 飞书文档读取 | feishu MCP / lark-cli / 用户提供文本 | 引导安装 lark-cli 或导出文档 |
| read-api | 飞书文档读取 | 同上 | 同上 |
| tech-doc | 无特殊依赖 | — | — |
| test-cases | 无特殊依赖 | — | — |
| git-branch | Git + 远端连通 | `git --version` + `git fetch --dry-run` | 引导安装 Git / 检查 SSH |
| write-code | 无特殊依赖 | — | — |
| git-finish | Git + MR 创建 | `glab --version` → 降级 GitLab API → 手动指引 | 逐级降级，不阻塞 |
| git-workflow | Git + MR 创建 | `git --version` + `glab --version` (或 GitLab API) | 动态检索 ID 并自适应降级 |
| browser-qa | 浏览器工具 | chrome-devtools MCP → Playwright → Agent 自带 | 引导安装或跳过 |
| final-verify | 无特殊依赖 | — | — |

### 检查原则

1. **先静默检查，再出声**：先自己检测，具备条件直接开始，只在不具备时才告诉用户
2. **给出安装命令，不自动安装**：安全第一，让用户确认后再执行安装
3. **逐级降级**：glab 不可用 → API；API 不可用 → 手动指引。永不完全阻塞
4. **可以跳过**：浏览器测试、Git 收口都是可选阶段，用户说"跳过"就跳过

---

## 安装

| 平台 | 命令 |
|------|------|
| Hermes | `hermes skill install SKILL.md` + 把 `skills/` 下所有文件也装上 |
| Claude Code | 整个文件夹放到 `.claude/skills/` |
| Cursor | 放到 `.cursor/rules/` |
| 其他 | 放到对应平台的 skills/rules 目录 |

---

## 子 Skill 文件

见 `skills/` 目录，每个子 Skill 可独立安装使用。
