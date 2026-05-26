---
name: feishu-prd-workflow/gitWorkflow
description: 整合分支生命周期管理（gitBranch）与提交收口（gitFinish）的驼峰命名 Git 工作流技能。支持自动 stash 暂存、基线同步、以及基于项目通用偏好配置文件（如 .agent_preferences.json）自动指派默认指派人（如 Team Robot）创建 MR 的能力。
parent: feishu-prd-workflow
phase: 5-and-6
depends_on: []
optional: false
---

# Git Workflow — 全生命周期 Git 工作流技能

此技能集成了从开发分支的拉取准备，到最终代码的自动化推送与 MR（Merge Request）合规指派。全面采用驼峰命名并内置了跨 IDE/Agent 的通用偏好配置文件兼容嗅探机制。

---

## 🚀 动作一：gitBranch (分支创建与准备)

### 1. 触发方式示例 (Example)

> [!NOTE]
> 本技能是通用的分支创建组件，支持 Feature（日常迭代）和 Hotfix（线上热修复）双模式，且不绑定任何具体业务。以下是 Agent 调用本技能的自然语言触发示例：

**Feature 模式 (日常迭代)：**

```
用 feishu-prd-workflow/gitWorkflow 执行 gitBranch：
项目路径：/path/to/project
Release：release-1.0.6
任务名：apiIntegration
```

**Hotfix 模式 (线上修复)：**

```
用 feishu-prd-workflow/gitWorkflow 执行 gitBranch：
项目路径：/path/to/project
模式：hotfix
任务名：fixPaymentCrash
```

### 2. 输入参数

| 参数           | 必需           | 说明                                                                                               |
| :------------- | :------------- | :------------------------------------------------------------------------------------------------- |
| `project_path` | ✅             | 项目本地路径                                                                                       |
| `mode`         | 否             | 分支模式：`feature`（默认）或 `hotfix`                                                             |
| `release`      | `feature` 必需 | 基线 release 分支，如 `release-1.0.6`                                                              |
| `task`         | ✅             | 任务名（**必须使用驼峰命名**，例如 `apiIntegration` 或 `fixPaymentCrash`，严禁使用中划线或下划线） |
| `developer`    | 否             | 开发者简称，默认 `allen`                                                                           |

### 3. 💡 基线分支智能推理与自愈机制

> [!TIP]
> 当用户着急使用或口语化输入时，可能会提供简写的基线名称（例如：`1.0.5`、`v1.0.5`）或直接指定 hotfix。为了保障极致顺畅的无感开发体验，Agent **必须具备智能推理与前缀自愈心智**：

1. **Feature 模式 - 版本号前缀智能补全**：
   - 若在 `feature` 模式下，用户输入的 `release` 为纯版本号（如 `1.0.5`）或带有小写 `v` 前缀（如 `v1.0.5`）：
     - Agent 应当**智能推断并自动重构参数**，在内部将其自动补全为规范的标准分支名格式：`release-{version}`（例如将 `1.0.5` 转化为 `release-1.0.5`）。
2. **Hotfix 模式 - 生产主分支自适应**：
   - 若在 `hotfix` 模式下，基线默认指向生产分支。Agent 需自动探测远端是 `master` 还是 `main` 分支，自适应锁定正确的基线名。
3. **远端分支校验与探测**：
   - 在执行 `git checkout -b` 之前，先以补全或锁定后的标准分支名在远端（如 `origin/release-1.0.5` 或 `origin/master`）进行探测，匹配成功后直接以此为基准拉取，无需强行命令用户重新修改，极大提升敏捷度。
4. **中文任务名智能翻译与小驼峰化 (自愈机制)**：
   - **拦截场景**：若用户输入的 `task` 为中文（例如：`退款图片锁定` 或 `修复支付崩溃`）：
     - Agent **绝对不能直接将中文拼入分支名中**，也**绝对不能退回报错**强制用户手动输入。
     - Agent 必须在后台自动将其翻译为**简短、专业、符合行业习惯**的英文，并严格重构为**小驼峰命名（camelCase）**。
     - **自愈实例**：
       - `退款图片锁定` 自动重构为 `refundPhotoLock`。
       - `修复支付崩溃` 自动重构为 `fixPaymentCrash`。
     - 在确认执行拉取分支前，需在交互信息中清晰地呈现翻译并驼峰化后的标准分支名称，保障流程的透明与掌控。

### 4. ⚙️ 前置检查与智能工作区搬迁 (脏开发自适应机制)

> [!TIP]
> 本技能天然支持**“无感脏写迁移”**高阶开发流：用户在错乱分支上已写完代码但未建分支时，直接运行本技能即可实现一键安全搬迁，Agent 将遵循以下自适应机制：

1. **智能状态嗅探**：
   - 在开始拉取分支前，执行 `git status --porcelain` 嗅探本地工作区。
   - 若发现工作区存在未提交的修改（Dirty Working Tree）：
     - **主动汇报与承诺**：Agent 需以极其专业的中文温馨告知用户：_“检测到您在当前分支有未提交的本地修改，我们将自动为您安全贮藏，并无缝搬迁至即将创建的全新开发分支上，请您放心！”_

### 5. 操作时序与逻辑

1. **工作区安全暂存**：
   - 执行 `git stash save "stash before branching for {task}"`。确保用户工作树上未提交的修改得到绝对安全的保护。
2. **远端同步**：
   - 执行 `git fetch --all --prune`。保持本地的远程追踪分支为最新。
3. **拉取基线创建本地新分支**：
   - **分支命名规范**：
     - **Feature 分支**：`feature-{release}/{developer}_{task}_{YYYYMMDD}`
     - **Hotfix 分支**：`hotfix/{developer}_{task}_{YYYYMMDD}`
   - **执行命令**：
     - **Feature 模式**：`git checkout -b feature-{release}/{developer}_{task}_{YYYYMMDD} origin/{release}`
     - **Hotfix 模式**：先自动切换并拉取主干：`git checkout master && git pull origin master`（或自适应 `main`），再执行 `git checkout -b hotfix/{developer}_{task}_{YYYYMMDD} origin/master`
4. **恢复工作区与释放**：
   - 执行 `git stash pop`。安全地将第 1 步中贮藏的代码释放并应用回当前新拉出的分支，完美衔接开发流。
5. **智能下一步问询 (闭环引导)**：
   - 成功将修改迁移至新开发分支后，Agent **必须主动向用户发起交互式问询**：
     > _“🎉 您的本地修改已安全搬迁至新分支 `{target_branch}`！请问接下来您需要：_
     > \*1. **立即对这些修改进行规范化提交与 MR 收口 (直接为您唤起 `gitFinish` 流程)？\***
     > \*2. **保留在本地工作区，继续您的开发工作？\***
     > _请回复数字或说明您的意向。”_

---

## 🚀 动作二：gitFinish (提交与 MR 收口)

### 1. 触发方式示例 (Example)

> [!NOTE]
> 本技能是通用的代码提交与 MR 收口组件，不绑定任何具体业务。以下是 Agent 调用本技能的自然语言触发示例：

```
用 feishu-prd-workflow/gitWorkflow 执行 gitFinish：
项目路径：/path/to/project
PingCode：#STORE-5857
功能摘要：接口联调及商品推荐批量导入UI优化
Target分支：release-1.0.6
```

### 2. 输入参数

| 参数             | 必需 | 说明                               |
| ---------------- | ---- | ---------------------------------- |
| `project_path`   | ✅   | 项目本地路径                       |
| `pingcode_id`    | ✅   | PingCode 卡片 ID，如 `#STORE-5857` |
| `summary`        | ✅   | 功能摘要                           |
| `release_branch` | ✅   | 目标合并分支，如 `release-1.0.6`   |

### 3. ⚙️ 前置检查与偏好探测

1. **PingCode ID 完整性校验与主动问询**：
   - **拦截条件**：执行前必须严格检查 `pingcode_id` 参数。
   - **交互策略**：若 `pingcode_id` 缺失或为空，Agent **绝对不能直接报错或强制中断**。必须主动触发熔断提示，以极具亲和力且明确的中文问询用户：
     > _“当前提交未指定关联的 PingCode 任务 ID。请问本次提交对应的 PingCode ID 是什么？（例如：#STORE-5857，若本次提交不对应任何卡片，可直接回复 'none'）”_
   - **动态更新**：接收到用户输入后，将获取的值赋给 `pingcode_id` 参数，并继续后面的流程。
2. **多 Agent 偏好配置文件嗅探**：
   - 按照以下优先级，主动在 `{{project_path}}` 根目录下嗅探并读取第一个存在的配置文件：
     1. `.agent_preferences.json` （跨 IDE 与 Agent 框架的通用标准偏好文件）
     2. `.workflow_preferences.json` （工作流专属标准偏好文件）
     3. `.gemini_preferences.json` （历史兼容偏好文件）
   - **解析指派人参数**：
     - 优先提取并解析 `gitlab.default_assignee_username`（推荐，例如 `"team_robot"`）或 `gitlab.default_assignee_id`（数字 ID，例如 `170`）。
     - 若上述文件均不存在，或文件中未配置指派人，则默认将默认指派人设定为用户名 `"team_robot"`。
3. **默认指派人 ID 动态检索与自愈（防 ID 变更）**：
   - **拦截逻辑**：若解析出或默认的指派人为用户名（如 `"team_robot"`）而非数字 ID，Agent 绝对不能强行填充不确定的 ID。
   - **动态解析**：Agent 需向 GitLab API 发起一次静默检索请求：
     - **API 路径**：`GET https://{domain}/api/v4/users?username={assignee_username}`
     - **提取逻辑**：从返回的用户数组（通常返回唯一的匹配项）中，动态提取 `id` 属性作为后续 MR 创建的 `assignee_id`。
   - **健壮性降级兜底**：若由于网络波动、API 限制或凭证受限导致检索失败，Agent 应当采用历史遗留的 numeric ID `170` 作为向后兼容的最终降级兜底，确保工作流百分之百不中断。
4. **鉴权工具检测**：
   - 检查 `glab --version && glab auth status`。
   - 检查 `curl --version`。

### 4. 操作时序与逻辑

1. **暂存全部文件**：
   - 执行 `git add .`。
2. **规范化提交**：
   - **提交信息模板**：`feat: {pingcode_id} {summary}`
   - **命令**：`git commit -m "feat: {pingcode_id} {summary}" --no-verify` (使用 `--no-verify` 绕过宿主环境残缺的本地 hooks 以确保提交流畅通过)。
3. **推送代码并确立追踪关系**：
   - 执行 `git push -u origin {current_branch}` 将提交推送到远端并确立追踪关系。
4. **一键 MR 创建 (逐级降级机制)**：
   - **Glab CLI (优先使用)**：
     - 执行 `glab mr create -b {release_branch} -t "feat: {pingcode_id} {summary}" -d "PingCode ID: {pingcode_id}" -a team_robot -y`
   - **GitLab API 降级 (当 CLI 报反序列化/兼容错时)**：
     - 发起 HTTPS POST 请求到 `https://{domain}/api/v4/projects/{Encoded_Project_Path}/merge_requests`
     - 注入 Header `PRIVATE-TOKEN` 并发送以下 Body。**必须指派 assignee**：
       ```json
       {
         "source_branch": "{current_branch}",
         "target_branch": "{release_branch}",
         "title": "feat: {pingcode_id} {summary}",
         "description": "PingCode ID: {pingcode_id}",
         "assignee_id": "{resolved_assignee_id}"
       }
       ```
   - **网页直达降级 (安全兜底)**：
     - 若上述均不可达，输出一键直达的 GitLab 手动创建链接：
       `https://{domain}/{project_path}/-/merge_requests/new?merge_request[source_branch]={current_branch}`

---

## 📋 验收标准

- [ ] 分支命名符合命名规范（Feature 对应 `feature-{release}/{developer}_{task}_{YYYYMMDD}`，Hotfix 对应 `hotfix/{developer}_{task}_{YYYYMMDD}`）。
- [ ] 分支基于正确基线创建（Feature 基线为 `release` 分支，Hotfix 基线为 `master`/`main` 分支）。
- [ ] 提交 Commit 信息格式完美呈现 `feat: #ID 摘要`。
- [ ] `gitFinish` 流程支持多偏好配置文件兼容嗅探，并具备默认指派人用户名到数字 ID 的动态解析与容错降级能力。
