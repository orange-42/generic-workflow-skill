---
name: feishu-prd-workflow/tech-doc
description: 基于 PRD + API 摘要生成前端技术方案：文件范围、状态管理、实现路径、回归风险。
parent: feishu-prd-workflow
phase: 3
depends_on: [read-prd, read-api]
---

# Tech Doc — 生成技术方案

## 触发方式

```
用 feishu-prd-workflow/tech-doc 生成技术方案，
PRD 摘要：./outputs/xxx/prd/prd-summary.md
API 摘要：./outputs/xxx/api/api-summary.md
```

## 输入

| 参数 | 必需 | 说明 |
|------|------|------|
| `prd_summary` | ✅ | Phase 1 产物路径或内容 |
| `api_summary` | 推荐 | Phase 2 产物路径或内容 |
| `project_path` | 推荐 | 项目路径（用于分析现有代码） |

## 做什么

1. 划分页面模块
2. 规划状态管理（全局 vs 局部）
3. 明确文件范围（新建哪些文件、修改哪些文件——给具体文件名）
4. 核心实现路径（关键逻辑怎么走）
5. 标注回归风险范围

## 输出产物

`{{output_dir}}/tech-doc/tech-doc-summary.md` + `json`

| 内容 | 说明 |
|------|------|
| 模块划分 | 页面 → 组件树 |
| 状态管理 | 全局/局部状态设计 |
| 文件范围 | 新建：N 个文件 / 修改：M 个文件 |
| 实现路径 | 关键步骤 |
| 回归风险 | 改这里可能影响哪里 |

## 验收

- [ ] 文件范围有具体文件名（不是空话）
- [ ] 实现路径可执行
- [ ] 回归风险已标注
