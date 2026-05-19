---
name: feishu-prd-workflow/read-api
description: 读取飞书 API 文档，提取接口定义和页面-接口映射。自动检测飞书读取能力，缺失时引导配置。
parent: feishu-prd-workflow
phase: 2
depends_on: [read-prd]
---

# Read API — 读取 API 文档

## 触发方式

```
用 feishu-prd-workflow/read-api 帮我读这个 API 文档：
https://xxx.feishu.cn/wiki/xxx
```

## ⚙️ 前置检查

与 [read-prd](./read-prd.md) 完全相同的检查逻辑。复用 read-prd 的环境检测结果——如果 read-prd 刚通过，直接开始，不重复检查。

如果**单独使用**此 skill（不经过主编排器），则必须先做环境检查：

- 🥇 feishu MCP → 🥈 lark-cli → 🥉 用户提供文本
- 都不具备时引导用户安装或导出

---

## 输入

| 参数 | 必需 | 说明 |
|------|------|------|
| `api_url` | ✅ | 飞书 API 文档 URL，或已导出文本 |
| `prd_summary` | 推荐 | Phase 1 产物（帮助区分页面接口 vs 旁路接口） |

---

## 做什么

1. 读取 API 文档全文（正文 + 表格 + 截图）
2. 提取接口清单：方法、路径、关键请求/响应字段
3. 区分"页面依赖接口"和"旁路接口"
4. 标注错误码和边界条件

---

## 输出产物

`{{output_dir}}/api/api-summary.md` + `api-summary.json`

| 内容 | 说明 |
|------|------|
| 接口清单 | 方法、路径、用途（表格） |
| 关键字段 | 每个接口核心请求/响应字段 |
| 页面-接口映射 | 哪个页面依赖哪些接口 |
| 错误码 | 边界条件 |

---

## 验收

- [ ] 接口清单完整
- [ ] 关键字段已标注（不追求全字段）
- [ ] 页面-接口映射清晰
