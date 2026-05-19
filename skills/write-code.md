---
name: feishu-prd-workflow/write-code
description: 按技术方案和测试用例实现前端代码。先读后写、最小改动、不自扩范围。
parent: feishu-prd-workflow
phase: 5
depends_on: [tech-doc, test-cases]
optional: true
---

# Write Code — 代码实现

## 触发方式

```
用 feishu-prd-workflow/write-code 写代码，
技术方案：./outputs/xxx/tech-doc/tech-doc-summary.md
测试用例：./outputs/xxx/test-cases/test-cases-summary.md
项目路径：/path/to/project
```

## 输入

| 参数 | 必需 | 说明 |
|------|------|------|
| `tech_doc` | ✅ | Phase 3 产物 |
| `test_cases` | ✅ | Phase 4 产物（作为自检验收标准） |
| `project_path` | ✅ | 项目本地路径 |

## 原则

1. **先读后写**：先仔细阅读技术方案中指定的文件
2. **最小改动**：只改与需求相关的代码，不碰无关文件
3. **不自扩范围**：技术方案没写的不要擅自加
4. **对照自检**：写完对照测试用例逐条自检

## 做什么

1. 阅读技术方案指定的文件
2. 按方案实现代码
3. 运行 lint / type check / 单元测试
4. 对照测试用例逐条自检
5. 如修改影响现有功能，必须说明

## 输出产物

`{{output_dir}}/code/changes-summary.md`

| 内容 | 说明 |
|------|------|
| 修改文件列表 | 每个文件改了什么 |
| 验证结果 | lint ✅/❌ / typecheck ✅/❌ / 自检结论 |
| 影响说明 | 是否影响现有功能 |

## 验收

- [ ] 代码已实现目标功能
- [ ] lint / type check 通过
- [ ] 对照测试用例逐条自检通过
