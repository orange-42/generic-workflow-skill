---
name: feishu-prd-workflow/read-prd
description: 读取飞书 PRD 文档，输出结构化摘要。含正文、嵌入表格、截图识别。自动检测飞书读取能力，缺失时引导配置。
parent: feishu-prd-workflow
phase: 1
---

# Read PRD — 读取 PRD 文档

## 触发方式

```
用 feishu-prd-workflow/read-prd 帮我读这个 PRD：
https://xxx.feishu.cn/wiki/xxx
```

## ⚙️ 前置检查（Agent 必须先做）

Agent 在开始读文档前，静默检查自己有没有飞书文档读取能力：

| 检查项 | 检测方式 | 优先级 |
|--------|---------|--------|
| feishu MCP 插件 | 检查 MCP 工具列表中是否有 `feishu_*` 开头的工具 | 🥇 最优先 |
| lark-cli 命令行 | `which lark-cli` 或 `lark-cli --help` | 🥈 备选 |
| 用户直接提供文本 | 用户说"PRD 内容如下：..." 或给了本地文件路径 | 🥉 兜底 |

### 都不具备时的引导流程

```
⚠️ 检测到当前环境无法读取飞书文档。

请选择一种方式提供 PRD 内容：

1. 【推荐】安装 lark-cli（30 秒）：
   npm install -g lark-cli
   然后设置飞书凭证：
   export FEISHU_APP_ID="cli_xxx"
   export FEISHU_APP_SECRET="xxx"
   需要我帮你一步步配置吗？

2. 【最快】手动导出 PRD：
   在飞书文档页面 → 导出为 Markdown → 粘贴给我

3. 【临时方案】直接粘贴内容：
   把 PRD 的关键部分复制粘贴给我
```

### 具备条件时——直接开始

```
✅ 飞书文档读取就绪（feishu MCP / lark-cli）
正在读取 PRD...
```

---

## 输入

| 参数 | 必需 | 说明 |
|------|------|------|
| `prd_url` | ✅ | 飞书文档 URL 或 Wiki Token，或已导出的文本内容 |

---

## 做什么

1. 读取文档全文：正文 + 嵌入表格 + 图片/截图
2. 图片下载到本地后进行视觉理解——不要输出 fileToken，要输出"图中是什么"
3. 输出**干净的结构化摘要**，剥离原始技术噪音

---

## 输出产物

`{{output_dir}}/prd/prd-summary.md` + `prd-summary.json`

| 内容 | 说明 |
|------|------|
| 需求概述 | 一句话 |
| 功能模块 | 模块名 + 一句话 |
| 关键交互 | 用户操作 → 系统响应 |
| 状态流转 | 条件 → 状态变化 |
| 涉及页面 | URL 路径 + 页面名 |
| 风险点 | 待确认事项 |
| 图片摘要 | 图中内容 + 业务含义 |

---

## 验收

- [ ] 正文完整提取
- [ ] 嵌入表格已解析
- [ ] 截图/图片已语义摘要
- [ ] 无 fileToken 等原始噪音
- [ ] 风险点已列出
