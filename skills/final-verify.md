---
name: feishu-prd-workflow/final-verify
description: 最终复核：独立验证关键事实、检查产物完整性、给出结论和下一步建议。
parent: feishu-prd-workflow
phase: 8
---

# Final Verify — 最终验收

## 触发方式

```
用 feishu-prd-workflow/final-verify 做最终验收，
产物目录：./outputs/20260519_143000_xxx/
```

## 输入

| 参数 | 必需 | 说明 |
|------|------|------|
| `output_dir` | ✅ | 产物目录路径 |

## 做什么

1. 独立复核 2-3 个关键事实（不依赖前面阶段的结论）
2. 检查产物完整性（缺了什么？）
3. 区分"已完成"和"已验证"
4. 给出明确的下一步建议

## 输出产物

`{{output_dir}}/summary/final-verification.md`

| 内容 | 说明 |
|------|------|
| 已验证点 | 已独立复核的事实 |
| 未验证点 | 无法验证 + 原因 |
| 残余风险 | 可能出问题的地方 |
| 下一步建议 | 明确、可执行的建议 |

## 验收

- [ ] 关键事实已独立复核
- [ ] 不确定性边界已说明
- [ ] 下一步建议明确可执行
