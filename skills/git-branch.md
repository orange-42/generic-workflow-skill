---
name: feishu-prd-workflow/git-branch
description: 创建开发分支：feature 分支（日常迭代）或 hotfix 分支（线上修复）。自动 stash、切基线、拉分支、推远端。前置检查 git 和远端连通性。
parent: feishu-prd-workflow
phase: 5-prep
depends_on: []
optional: false
---

# Git Branch — 创建开发分支

## 触发方式

**Feature 分支：**
```
用 feishu-prd-workflow/git-branch 创建 feature 分支：
项目路径：/path/to/project
Release：release/1.24.0
任务名：refund-photo-lock
```

**Hotfix 分支：**
```
用 feishu-prd-workflow/git-branch 创建 hotfix 分支：
项目路径：/path/to/project
模式：hotfix
任务名：fix-payment-crash
```

---

## ⚙️ 前置检查（Agent 必须先做）

### 1. 检查 Git

```bash
git --version
```

不具备时：
```
⚠️ Git 未安装。

macOS：xcode-select --install
Linux：sudo apt-get install git
安装后重启终端即可。要我等你装好吗？
```

### 2. 检查项目目录

```bash
cd {{project_path}} && git remote -v
```

不具备时：
```
⚠️ {{project_path}} 不是一个 Git 仓库。

请确认项目路径是否正确，或先 clone 项目：
git clone <repo_url> {{project_path}}
```

### 3. 检查远端连通性

```bash
cd {{project_path}} && git fetch --dry-run
```

不具备时（网络问题/权限问题）：
```
⚠️ 无法连接远端仓库。

可能原因：
1. SSH Key 未配置：ssh -T git@your-gitlab.com
2. VPN 未连接
3. Git 凭证过期

请先解决远端连通问题再继续。
```

### 全部通过

```
✅ Git 就绪
✅ 项目仓库正常
✅ 远端连通正常
正在创建分支...
```

---

## 分支命名规范

**Feature 分支：**
```
feature-{release}/{developer}_{task}_{YYYYMMDD}
例：feature-release-1.24.0/allen_refund-photo-lock_20260519
```

**Hotfix 分支：**
```
hotfix/{developer}_{task}_{YYYYMMDD}
例：hotfix/allen_fix-payment-crash_20260519
```

---

## 输入

| 参数 | 必需 | 说明 |
|------|------|------|
| `project_path` | ✅ | 项目本地路径 |
| `mode` | 否 | `feature`（默认）或 `hotfix` |
| `release` | feature 必需 | 基线 release 分支，如 `release/1.24.0` |
| `task` | ✅ | 任务名（英文连字符） |
| `developer` | 否 | 默认 `allen` |

---

## 做什么

1. `git stash`（保存未提交改动）
2. `git fetch --all --prune`
3. **Feature**：`git checkout {release}` → `git pull`
4. **Hotfix**：`git checkout master` → `git pull`
5. `git checkout -b {分支名}`
6. `git push -u origin {分支名}`
7. 如有 stash，提示可选 `git stash pop`

---

## 输出

```
✅ 分支已创建并推送

分支名：feature-release-1.24.0/allen_refund-photo-lock_20260519
基线：release/1.24.0
项目：/path/to/project
远端：已推送，upstream 已设置

💡 有 stash 需要恢复？执行：git stash pop
```

---

## 验收

- [ ] 分支名符合命名规范
- [ ] 基于正确基线创建（feature→release，hotfix→master）
- [ ] 已推送 + upstream 已设置
