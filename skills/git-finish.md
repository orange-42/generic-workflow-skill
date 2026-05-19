---
name: feishu-prd-workflow/git-finish
description: 提交代码、推送远端、创建 MR。自动检查 glab 是否可用，不可用时降级为 GitLab API 或手动指引。
parent: feishu-prd-workflow
phase: 6
depends_on: [write-code]
optional: true
---

# Git Finish — 提交 & MR 收口

## 触发方式

```
用 feishu-prd-workflow/git-finish 收口提交：
项目路径：/path/to/project
PingCode：#STORE-0000
功能摘要：退款照片锁定功能
Release 分支：release/1.0.0
```

---

## ⚙️ 前置检查（Agent 必须先做）

### 1. 复用 git-branch 的三个检查

```
git --version          # Git 是否安装
git remote -v          # 项目是否为 Git 仓库
git fetch --dry-run    # 远端是否连通
```

### 2. 检查 MR 创建工具（按优先级）

```
① glab --version && glab auth status    # GitLab CLI
② curl --version                        # 降级方案：GitLab API
```

具备 ① 时：
```
✅ glab 就绪，将自动创建 MR
```

只有 ② 时：
```
⚠️ glab 未安装，将用 GitLab API 创建 MR。

要不要装 glab？一行命令：
brew install glab && glab auth login
（或跳过安装，我用 API 方式创建）
```

都不具备时：
```
⚠️ 无法自动创建 MR。

请手动创建：
1. 打开 GitLab 项目页面
2. 创建 MR：{{source_branch}} → {{target_branch}}
3. 标题：feat: #PingCode 功能摘要
```

---

## 输入

| 参数 | 必需 | 说明 |
|------|------|------|
| `project_path` | ✅ | 项目本地路径 |
| `pingcode_id` | ✅ | 必须带 `#` 前缀 |
| `summary` | ✅ | 功能摘要 |
| `release_branch` | ✅ | MR target 分支 |

---

## 做什么

1. `git add .`
2. `git commit -m "feat: #PINGCODE 摘要"`
3. `git push`
4. 创建 MR（glab → API → 手动指引，逐级降级）

---

## 输出产物

`{{output_dir}}/git/mr-info.md`

```
✅ 代码已提交并推送

commit：abc1234
分支：feature-release-1.24.0/allen_xxx_20260519
MR：https://gitlab.com/project/-/merge_requests/123
```

---

## 验收

- [ ] 代码已提交并推送
- [ ] MR 已创建或给出明确创建指引
