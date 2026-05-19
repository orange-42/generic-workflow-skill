---
name: feishu-prd-workflow/browser-qa
description: 浏览器自动化验收。自动检测 Chrome DevTools MCP / Playwright / Page Agent 等浏览器工具。具备直接开始，不具备引导安装。
parent: feishu-prd-workflow
phase: 7
depends_on: [test-cases]
optional: true
---

# Browser QA — 浏览器自动化验收

## 触发方式

```
用 feishu-prd-workflow/browser-qa 跑浏览器测试：
测试用例：./outputs/xxx/test-cases/test-cases-summary.md
目标页面：https://xxx.com/page
```

---

## ⚙️ 前置检查（Agent 必须先做）

Agent 按以下优先级检测浏览器工具：

| 方案 | 检测方式 | 安装难度 | 推荐场景 |
|------|---------|:---:|------|
| 🥇 chrome-devtools MCP | MCP 工具列表含 `chrome_devtools_*` | 低（Hermes 内置） | Hermes 用户 |
| 🥈 Playwright | `npx playwright --version` | 中 | 独立 Agent |
| 🥉 Page Agent | `which page-agent` 或 npm scripts | 中 | 本项目用户 |
| 🏅 Agent 自带浏览器 | Agent 平台原生支持的浏览器工具 | 无需 | 视平台而定 |

### 都不具备时的引导流程

```
⚠️ 当前环境未检测到浏览器自动化工具。

选择一种方案安装（推荐方案 1）：

【方案 1】chrome-devtools MCP（推荐，30 秒）
在 ~/.hermes/config.yaml 中添加：
  mcp_servers:
    chrome-devtools:
      command: npx
      args: ["-y", "chrome-devtools-mcp@latest"]
然后重启 Hermes。

【方案 2】Playwright（1 分钟）
  npm install playwright
  npx playwright install chromium

【方案 3】手动测试
你手动按测试用例验证，把结果告诉我，我帮你记录到产物中。

要装哪个方案？或者跳过浏览器测试，直接进入最终验收？
```

### 具备条件时——直接开始

```
✅ chrome-devtools MCP 就绪
正在打开目标页面...
```

---

## 输入

| 参数 | 必需 | 说明 |
|------|------|------|
| `test_cases` | ✅ | Phase 4 产物 |
| `target_url` | ✅ | 测试起始页面 URL |

---

## 做什么

1. 打开目标页面
2. 按测试用例执行关键动作
3. 每步确认页面状态（截图留证）
4. 观察网络请求验证 API 调用
5. 刷新页面验证状态持久化

### 遇到 CAPTCHA / 反爬时的处理

```
⚠️ 页面触发了验证码/Human Verification。

请你在打开的浏览器窗口中手动完成验证，完成后告诉我"继续"。
这通常是因为 CDP 端口暴露了自动化特征，内部项目一般不会遇到。
```

---

## 必须验证

- 关键元素是否正确渲染
- 按钮点击后状态变化
- 接口请求参数和响应
- 刷新后状态是否保持

---

## 输出产物

`{{output_dir}}/qa/qa-verification.md`

```
## 浏览器测试结果

| 编号 | 用例 | 结果 | 证据 |
|------|------|:---:|------|
| TC-01 | 打开页面 | ✅ | 截图 path |
| TC-02 | 点击按钮 | ✅ | 截图 path |
| TC-03 | 边界条件 | ❌ | 见下方说明 |

执行摘要：通过 8 / 失败 1 / 阻塞 0

❌ TC-03 失败
   预期：显示"已锁定"
   实际：显示"加载中"
   可能原因：接口返回延迟
   建议：检查 API 超时设置
```

---

## 验收

- [ ] 关键页面已截图
- [ ] 主流程用例逐一验证
- [ ] 失败项已记录现场 + 可能原因
