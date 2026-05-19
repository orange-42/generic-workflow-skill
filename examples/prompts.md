# 各平台触发示例

以下是在不同 AI Agent 平台中使用 feishu-prd-workflow 的实际对话示例。

---

## Hermes

```
按 feishu-prd-workflow 处理这个需求：

PRD：https://himo-group.feishu.cn/wiki/GoADwrwqEiWdipkBts1cFxVdn5d
API：https://himo-group.feishu.cn/wiki/Vs30wSx9ridQ1ckJ6hBcLk78ntf
项目路径：/Users/allen/himo-projects/code-manage/himo-new-store
项目名称：himo-new-store
Release：release/1.24.0
PingCode：#STORE-5416
```

> Hermes 有 feishu MCP，可以直接读飞书文档，无需导出。

---

## Claude Code

```
按 feishu-prd-workflow 处理这个需求。

飞书文档我无法直接读取，以下是导出的内容：

=== PRD 文档 ===
[paste PRD content here]

=== API 文档 ===
[paste API content here]

项目路径：/path/to/project
项目名称：my-project
Release：release/1.0.0
PingCode：#PROJ-123
```

> Claude Code 通常无法直接读飞书。把文档导出为文本后粘贴给它。

---

## Cursor

```
按 feishu-prd-workflow 处理这个需求。

PRD 文档在 @prd-export.md
API 文档在 @api-export.md

项目名称：my-project
Release：release/1.0.0
PingCode：#PROJ-123
```

> Cursor 用 `@` 引用文件。先导出飞书文档到本地文件。

---

## 通用模板（任何平台）

```
按 feishu-prd-workflow 处理：

PRD：[URL 或已导出文本或文件路径]
API：[URL 或已导出文本或文件路径]
项目路径：/path/to/project
项目名称：my-project
Release：release/1.0.0
PingCode：#PROJ-123
```

> 任何 Agent 平台都能理解这个格式。Agent 会自动问你产物目录和可选阶段开关。

---

## 单独使用子 Skill 示例

```
# 只要 PRD 摘要
用 feishu-prd-workflow/read-prd 读这个：
https://xxx.feishu.cn/wiki/xxx

# 创建 feature 分支
用 feishu-prd-workflow/git-branch 创建 feature 分支：
项目路径：/path/to/project
Release：release/1.24.0
任务名：refund-photo-lock

# 创建 hotfix 分支
用 feishu-prd-workflow/git-branch 创建 hotfix 分支：
项目路径：/path/to/project
模式：hotfix
任务名：fix-payment-crash

# 只要测试用例
用 feishu-prd-workflow/test-cases 生成测试用例：
PRD 摘要在这：./prd-summary.md
API 摘要在这：./api-summary.md

# 只做浏览器验收
用 feishu-prd-workflow/browser-qa 跑测试：
测试用例：./test-cases-summary.md
目标页面：https://xxx.com/page
```

---

## 断点续跑

```
# 从产物目录继续
从 outputs/20260519_143000_#STORE-5416_himo-new-store_refund-lock/ 继续上次的工作流

# 从指定阶段开始
从 feishu-prd-workflow/write-code 开始，
前面阶段产物在：./outputs/xxx/
项目路径：/path/to/project
```
