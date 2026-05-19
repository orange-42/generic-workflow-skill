---
name: feishu-prd-workflow/test-cases
description: 基于 PRD + API + 技术方案生成测试用例：冒烟、主流程、边界条件、UI/API 联动。
parent: feishu-prd-workflow
phase: 4
depends_on: [read-prd, read-api, tech-doc]
---

# Test Cases — 生成测试用例

## 触发方式

```
用 feishu-prd-workflow/test-cases 生成测试用例，
PRD 摘要：./outputs/xxx/prd/prd-summary.md
API 摘要：./outputs/xxx/api/api-summary.md
技术方案：./outputs/xxx/tech-doc/tech-doc-summary.md
```

## 输入

| 参数 | 必需 | 说明 |
|------|------|------|
| `prd_summary` | ✅ | Phase 1 产物 |
| `api_summary` | 推荐 | Phase 2 产物 |
| `tech_doc` | 推荐 | Phase 3 产物 |

## 做什么

1. 基于 PRD 真实页面文案和接口信号生成用例（不写死业务词）
2. 区分"浏览器可验证项"和"代码级验证项"
3. 覆盖：冒烟 → 主流程 → 边界条件 → UI/API 联动

## 输出产物

`{{output_dir}}/test-cases/test-cases-summary.md`

```markdown
## 冒烟用例
| 编号 | 步骤 | 预期 |

## 主流程用例
| 编号 | 步骤 | 预期 |

## 边界条件
| 编号 | 场景 | 步骤 | 预期 |

## UI/API 联动
| 编号 | UI 操作 | 预期 API | 预期 UI 变化 |
```

## 验收

- [ ] 主流程覆盖关键交互
- [ ] 边界：空状态、错误态、权限边界
- [ ] UI/API 联动点已列出
- [ ] 不写死具体业务数据
