---
name: git-workflow
description: 整合分支生命周期管理（gitBranch）与提交收口（gitFinish）的驼峰命名 Git 工作流技能。支持自动 stash 暂存、基线同步、提交推送，以及在创建 MR 前确认指派人/审核人后通过 GitLab REST API + curl 稳定创建 MR 的能力。
parent: feishu-prd-workflow
phase: 5-and-6
depends_on: []
optional: false
---

# Git Workflow — 全生命周期 Git 工作流技能

此技能集成了从开发分支的拉取准备，到最终代码的自动化推送与 MR（Merge Request）合规创建。全面采用驼峰命名并内置了跨 IDE/Agent 的通用偏好配置文件兼容嗅探机制；MR 创建阶段使用自建 GitLab REST API + curl，并在创建前确认 assignee/reviewer，当鉴权缺失或 API 调用异常时，支持一键降级为网页自助直达通道。

---

## 🚀 动作一：gitBranch (分支创建与准备)

### 1. 触发方式示例 (Example)

> [!NOTE]
> 本技能是通用的分支创建组件，支持 Feature（日常迭代）和 Hotfix（线上热修复）双模式，且不绑定任何具体业务。以下是 Agent 调用本技能的位置示例：

**Feature 模式 (日常迭代)：**

```
用 feishu-prd-workflow/git-workflow 执行 gitBranch：
项目路径：/path/to/project
Release：release-1.0.6
任务名：apiIntegration
```

**Hotfix 模式 (线上修复)：**

```
用 feishu-prd-workflow/git-workflow 执行 gitBranch：
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
| `task`         | 条件必需             | 任务名（**必须使用驼峰命名**，例如 `apiIntegration` 或 `fixPaymentCrash`，严禁使用中划线或下划线）。若用户未显式提供，Agent 先从当前需求/PRD/summary/PingCode 标题提炼并展示确认；无法可靠推断时必须询问。 |
| `developer`    | 否（推导自愈） | 开发者简称（Agent 需优先通过本地 git config 或项目偏好配置自动嗅探，若无则主动问询，支持中文简称智能自愈转换为拼音/英文，严禁直接默认 `allen`） |

### 3. 💡 基线分支智能推理与自愈机制

> [!TIP]
> 当用户口语化输入时，可能会提供简写的基线名称（例如：`1.0.5`、`v1.0.5`）或直接指定 hotfix。为了保障极致顺畅的无感开发体验，Agent **必须具备智能推理与前缀自愈心智**：

1. **Feature 模式 - 版本号前缀智能补全**：
   - 若在 `feature` 模式下，用户输入的 `release` 为纯版本号（如 `1.0.5`）或带有小写 `v` 前缀（如 `v1.0.5`）：
     - Agent 应当**智能推断并自动重构参数**，在内部将其自动补全为规范的标准分支名格式：`release-{version}`（例如将 `1.0.5` 转化为 `release-1.0.5`）。
2. **Hotfix 模式 - 生产主分支自适应**：
   - 若在 `hotfix` 模式下，基线默认指向生产分支。Agent 需自动探测远端是 `master` 还是 `main` 分支，自适应锁定正确的基线名。
3. **远端分支校验与探测**：
   - 在执行 `git checkout -b` 之前，先以补全或锁定后的标准分支名在远端（如 `origin/release-1.0.5` 或 `origin/master`）进行探测，匹配成功后直接以此为基准拉取，无需强行命令用户重新修改。
4. **任务名 task 缺失推断与小驼峰自愈机制**：
   - **task 是分支业务语义的核心参数**：Agent 不能在没有业务语义的情况下硬造 `task`，也不能使用 `task`、`temp`、`dev`、`fix`、工单号等无意义占位作为分支业务名。
   - **缺失时的推断优先级**：若用户未显式提供 `task`，Agent 应按以下顺序尝试提炼：
     1. 从用户当前自然语言需求中提炼 2-5 个英文业务词，并转为小驼峰。
     2. 若当前流程来自 PRD / 飞书文档 / PingCode 标题 / commit summary，优先从这些更稳定的业务描述中提炼。
     3. 若只能拿到 PingCode ID、项目名、release 版本等非业务语义信息，视为无法推断。
   - **推断成功时必须确认展示**：如果 Agent 能从上下文推断出 task，必须在拉分支前明确展示：
     > _“我从当前需求中推断本次分支业务名为 `{task}`，最终分支将是 `{branch_name}`。如果不合适请直接告诉我新的业务名。”_
     在同一轮用户已经明确要求立即执行且推断语义高度确定时，可以继续执行，但必须在执行日志中展示该推断结果。
   - **无法可靠推断时必须询问**：若上下文不足以形成明确业务名，必须暂停并询问用户：
     > _“这次分支名里的业务 task 我还缺一个语义名称。请给我一个简短中文/英文任务名，例如 `退款图片锁定` 或 `apiIntegration`，我会自动转成小驼峰。”_
   - **中文任务名智能翻译**：若用户输入的 `task` 为中文（例如：`退款图片锁定` 或 `修复支付崩溃`）：
     - Agent **绝对不能直接将中文拼入分支名中**，也**绝对不能退回报错**强制用户手动输入。
     - Agent 必须在后台自动将其翻译为**简短、专业、符合行业习惯**的英文，并严格重构为**小驼峰命名（camelCase）**。
     - **自愈实例**：
       - `退款图片锁定` 自动重构为 `refundPhotoLock`。
       - `修复支付崩溃` 自动重构为 `fixPaymentCrash`。
   - **非标准英文任务名自愈**：若用户输入 `api-integration`、`api_integration`、`API Integration`、`ApiIntegration` 等非小驼峰格式，Agent 应自动规范化为 `apiIntegration`，并在执行前展示规范化结果。
   - **透明确认**：在确认执行拉取分支前，需在交互信息中清晰呈现最终小驼峰 `task` 与完整标准分支名称，保障流程透明与可控。
5. **基于 Remote 的拉取优化方案 (默认) 与本地基线偏好**：
   - **默认行为 (优化方案)**：为了保持本地分支列表的极度纯净，默认不创建、不更新本地基线分支。在获取最新的远程分支数据后，直接基于 `origin/{release}` 检出开发分支。
   - **用户偏好配置 (原方案)**：若用户有特定习惯需要在本地保留并对齐基线分支，可通过项目根目录下的偏好配置文件（如 `.agent_preferences.json`）进行显式配置：
     - 配置项：`git.keep_local_baseline: true`（布尔值，默认为 `false`）。
     - 当检测到该配置为 `true` 时，Agent 将自动降级回原方案（即：本地检出基线分支 -> `git pull` 对齐远端 -> 基于本地基线分支检出新开发分支）。
6. **开发者简称 (developer) 三级探测与中文拼音自愈机制 (去硬编码设计)**：
   - **核心安全红线**：Agent **绝对不能**直接在代码中硬编码任何特定的默认开发者简称（如 `"allen"`）。这会导致其他团队成员拉分支时发生严重命名冲突。
   - **三级探测机制 (智能推导)**：
     - **第一级：项目偏好配置**：优先嗅探项目根目录下偏好配置文件（如 `.agent_preferences.json`）中的 `git.developer` 或 `git.username` 配置项。
     - **第二级：本地 Git 环境探测**：若偏好文件未配置，静默执行 `git config user.name`，动态拉取并使用当前系统配置的 Git 用户名。
     - **第三级：友好问询（绝对兜底）**：若以上级均未能成功解析出有效简称，Agent 必须主动触发熔断交互，以亲和的中文问询用户：
       > _“检测到您未指定开发者简称。为了规范拼装 Git 分支名称，请问您的英文简称或拼音是什么？（例如：`zhangsan`，后续会自动记录在此环境）”_
   - **中文简称智能自愈**：
     - **拦截策略**：在上述任何一级探测中，若获取到的开发者名称中包含**中文字符**（例如：`张三` 或 `李四`），Agent **绝对不能直接将其拼入分支名中**，以防分支名出现非 ASCII 字符导致流水线或部署兼容性报错。
     - **自动拼音化**：Agent 必须在后台自动将其转换为**规范 of 拼音全拼或声母缩写（纯小写英文）**。
       - **自愈实例**：`张三` 自动自愈重构为 `zhangsan` 或 `zs`；`李四` 自动自愈重构为 `lisi` 或 `ls`。

### 4. ⚙️ 前置检查与智能偏好探测

1. **多 Agent 偏好配置文件与本地 Git 简称嗅探 (三级探测与去硬编码)**：
   - 按照以下优先级，主动在 `{{project_path}}` 根目录下嗅探并读取第一个存在的配置文件：
     1. `.agent_preferences.json` （跨 IDE 与 Agent 框架的通用标准偏好文件）
     2. `.workflow_preferences.json` （工作流专属标准偏好文件）
     3. `.gemini_preferences.json` （历史兼容偏好文件）
   - **解析本地基线偏好**：
     - 提取并解析配置项 `git.keep_local_baseline`。若存在且为 `true`，则标记启用“本地保留基线”原方案；若不存在或为 `false`，则默认启用“直接基于 Remote 检出”的优化方案。
   - **执行开发者简称 (developer) 三级探测**：
     - 1) 优先嗅探上述偏好配置文件中的 `git.developer` 或 `git.username`。
     - 2) 若无，自动执行 `git config user.name` 获取本地用户名。
     - 3) 若仍无，主动向用户发出中文提示进行交互问询。
     - 4) 对获取的简称执行中文智能拼音/简写转换自愈（纯小写 ASCII），锁定最终 `developer` 参数。**绝对禁止在任何地方硬编码 "allen" 为默认值。**
     - 问询示例调整为非硬编码：
       > _“检测到您未指定开发者简称。为了规范拼装 Git 分支名称，请问您的英文简称或拼音是什么？（例如：`zhangsan`，后续会自动记录在此环境）”_
2. **前置 Token 提取与剪枝防空转规范**：
   - 为了防止在本地环境无配置、无凭证时 AI 反复在大范围内通过命令寻找 Token 导致无限空转，**必须将 Token 的寻找路径严格限定在以下 3 个直接读取操作，绝对禁止任何模糊全局遍历或 printenv 全局筛选**：
     - 动作一：检查系统环境变量中是否有 `GITLAB_PRIVATE_TOKEN`。
     - 动作二：读取项目偏好配置文件（如 `.agent_preferences.json` 等）中的 `gitlab.token` 字段。
     - 动作三：检查并读取 `~/.config/glab-cli/config.yml`（或其兼容路径 `~/.glab-cli/config.yml`）中对应 `code.hzmantu.com` 的 token 字段（仅限直接读取，若文件不存在则瞬间忽略）。
   - **2 秒快速熔断降级**：如果上述 3 个步骤读取后均未获取到 Token，**立即判定为“无 Token 凭证”状态。此时必须立刻切断 Token 嗅探流程，严禁执行 `printenv` 过滤、`ls -la ~` 遍历或对其他目录搜索。直接触发极速降级到一键网页直达自助创建 MR 通道**。
3. **工作区状态智能嗅探与“无感脏写迁移”**：
   - **智能状态嗅探**：
     - 在开始拉取分支前，执行 `git status --porcelain` 嗅探本地工作区。
     - 若发现工作区存在未提交的修改（Dirty Working Tree）：
       - **主动汇报与承诺**：Agent 需以极其专业的中文温馨告知用户：_“检测到您在当前分支有未提交的本地修改，我们将自动为您安全贮藏，并无缝搬迁至即将创建的全新开发分支上，请您放心！”_

### 5. 操作时序与逻辑

1. **工作区安全暂存**：
   - 执行 `git stash save "stash before branching for {task}"`。确保用户工作树上未提交的修改得到绝对安全的保护。
2. **远端同步**：
   - 执行 `git fetch --all --prune`。保持本地远程追踪分支为最新，确保基线引用已是最新的远端状态。
3. **根据偏好策略创建开发分支**：
   - **分支命名规范**：
     - **Feature 分支**：`feature-{release}/{developer}_{task}_{YYYYMMDD}`
     - **Hotfix 分支**：`hotfix/{developer}_{task}_{YYYYMMDD}`
   - **分支检出时序 (核心分支拉取策略)**：
     - **Feature 模式 (默认：直接基于 Remote 检出)**：
       - 若 `git.keep_local_baseline` 为 `false`（默认优化方案）：
         - 直接检出新开发分支并追踪远端基线：
           `git checkout -b feature-{release}/{developer}_{task}_{YYYYMMDD} origin/{release}`
       - 若 `git.keep_local_baseline` 为 `true`（原方案）：
         - 1) 切换并更新本地基线分支：`git checkout {release} && git pull origin {release}`
         - 2) 基于本地基线创建开发分支：`git checkout -b feature-{release}/{developer}_{task}_{YYYYMMDD}`
     - **Hotfix 模式**：
       - 先自动切换并拉取主干：`git checkout master && git pull origin master`（或自适应 `main`），再执行 `git checkout -b hotfix/{developer}_{task}_{YYYYMMDD} origin/master`
4. **一键推送并建立远程追踪关系 (Upstream 绑定)**：
   - 为了免去后续拉取和推送时反复手动指定分支的繁琐，在开发分支创建成功后，**立即将其推送到远端并建立追踪关系**。
   - **变量约定**：gitBranch 阶段统一使用 `new_branch` 表示“新创建的开发分支 / hotfix 分支”；`target_branch` 仅保留给 gitFinish 阶段表示“MR 目标分支”，避免语义混淆。
   - **执行命令**：
     ```bash
     git push -u origin {new_branch}
     ```
     *(注：`-u`/`--set-upstream` 参数会自动将本地新分支与远端新创建的同名分支建立强追踪关系，效果完全等同于 `git branch --set-upstream-to=origin/{new_branch}`)*
5. **恢复工作区与释放**：
   - 执行 `git stash pop`。安全地将第 1 步中贮藏的代码释放并应用回当前新拉出的分支，完美衔接开发流。
6. **智能下一步问询 (闭环引导)**：
   - 成功将修改迁移至新开发分支并建立上游绑定后，Agent **必须主动向用户发起交互式问询**：
     - _“🎉 您的本地修改已安全搬迁至已自动绑定追踪的新分支 `{new_branch}`！请问接下来您需要：_
     - _1. **立即对这些修改进行规范化提交与 MR 收口 (直接为您唤起 `gitFinish` 流程)？**_
     - _2. **保留在本地工作区，继续您的开发工作？**_
     - _请回复数字或说明您的意向。”_

---

## 🚀 动作二：gitFinish (提交与 MR 收口)

### 1. 触发方式示例 (Example)

> [!NOTE]
> 本技能是通用的代码提交与 MR 收口组件，不绑定任何具体业务。以下是 Agent 调用本技能的自然语言触发示例：

```
用 feishu-prd-workflow/git-workflow 执行 gitFinish：
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
| `mode`           | 否   | 分支模式：`feature`（默认）或 `hotfix` |
| `target_branch`  | 条件必需 | 目标合并分支。Feature 模式通常为 `release-1.0.6`；Hotfix 阶段流中由 `hotfix_stage` 自动推导，除非用户显式指定 |
| `release_branch` | 兼容旧参数 | 旧参数名，等价于 `target_branch`；若同时存在，以 `target_branch` 为准 |
| `hotfix_stage`   | 否 | Hotfix MR 阶段：`dev` / `pre` / `master` / `auto`。默认 `auto`：按已合并状态推进 `hotfix -> dev -> pre -> master` |

### 3. ⚙️ 前置检查与偏好探测

1. **PingCode ID 与提交摘要完整性校验**：
   - **PingCode ID 拦截条件**：执行前必须严格检查 `pingcode_id` 参数。
   - **PingCode ID 交互策略**：若 `pingcode_id` 缺失或为空，Agent **绝对不能直接报错或强制中断**。必须主动触发熔断提示，以极具亲和力且明确的中文问询用户：
     > _“当前提交未指定关联的 PingCode 任务 ID。请问本次提交对应的 PingCode ID 是什么？（例如：#STORE-5857，若本次提交不对应任何卡片，可直接回复 'none'）”_
   - **summary 轻量补全策略**：若 `summary` 缺失，不要做长时间推理，也无需消耗大量上下文；优先用当前用户需求、PRD 标题、PingCode 标题、分支 `task` 或已知变更意图提炼一个简短中文摘要。
   - **summary 推断成功**：若能明确提炼，直接展示：`本次提交摘要我先按“{summary}”处理。` 然后继续流程。
   - **summary 无法可靠推断**：用一句话询问用户：`本次提交摘要写什么？例如：判断次卡的有效性类型。` 用户回复后继续。
   - **summary 禁止占位**：不得使用 `更新代码`、`修复问题`、`提交代码`、`优化功能` 等空泛摘要，除非用户明确指定。
2. **目标分支解析与确认（Feature / Hotfix 分流）**：
   - **Feature 模式**：目标合并分支必须是用户指定的 `target_branch`（兼容旧参数 `release_branch`），通常为 `release-x.y.z`。若缺失，必须先询问用户，不能默认猜测。
   - **Hotfix 模式采用同一 hotfix 分支的三段式投递引导**：线上修复不是一次性直进生产，而是同一个 `hotfix/{developer}_{task}_{YYYYMMDD}` 分支在不同测试/审核阶段分别创建 MR：
     1. 第一段：`hotfix/{developer}_{task}_{YYYYMMDD}` -> `dev`
     2. 第二段：`hotfix/{developer}_{task}_{YYYYMMDD}` -> `pre`（通常在 dev MR 已合并后，由用户再次触发）
     3. 第三段：`hotfix/{developer}_{task}_{YYYYMMDD}` -> `master`（通常在 pre MR 已合并后，由用户再次触发）
   - **Hotfix 是引导式阶段流**：Agent 每次只处理用户当前要推进的一段 MR，并在完成后提示下一步；不得在一个请求中自动连续创建多个 MR。
   - **Hotfix 阶段自动推导**：若 `mode=hotfix` 且 `hotfix_stage=auto` 或缺失，Agent 可通过 GitLab API 查询当前 hotfix 分支相关 MR，辅助判断下一步建议：
     - 若不存在 `hotfix -> dev` MR 或其尚未 merged：建议本次创建 / 处理 `hotfix -> dev`。
     - 若 `hotfix -> dev` 已 merged、但不存在 `hotfix -> pre` MR 或其尚未 merged：建议本次创建 / 处理 `hotfix -> pre`。
     - 若 `hotfix -> pre` 已 merged、但不存在 `hotfix -> master` MR 或其尚未 merged：建议本次创建 / 处理 `hotfix -> master`。
   - **Hotfix 阶段显式指定**：若用户指定 `hotfix_stage=dev/pre/master`，按对应阶段推导 `source_branch` 与 `target_branch`：source 始终是当前 hotfix 分支，target 分别为 `dev`、`pre`、`master`。
   - **参数兼容**：若同时存在 `target_branch` 与 `release_branch`，以 `target_branch` 为准；若只存在 `release_branch`，内部统一映射为 `target_branch`。
3. **多 Agent 偏好配置文件嗅探**：
   - 按照以下优先级，主动在 `{{project_path}}` 根目录下嗅探并读取第一个存在的配置文件：
     1. `.agent_preferences.json` （跨 IDE 与 Agent 框架的通用标准偏好文件）
     2. `.workflow_preferences.json` （工作流专属标准偏好文件）
     3. `.gemini_preferences.json` （历史兼容偏好文件）
   - **解析候选指派人与审核人参数**：
     - 优先提取并解析 `gitlab.default_assignee_id`、`gitlab.default_reviewer_ids`（数字 ID，推荐）或 `gitlab.default_assignee_username`、`gitlab.default_reviewer_usernames`（用户名，兼容旧配置），作为“建议默认值”。
     - 若上述文件均不存在，或文件中未配置指派人与审核人，则建议默认值为 Team Robot（历史 numeric ID：`170`）。
     - 对 code.hzmantu.com 内部协作场景，若用户明确指定加里克作为 reviewer/assignee，则使用加里克 numeric ID：`267`，避免显示名解析失败。
   - **创建 MR 前必须询问用户确认指派人与审核人**：
     - 在执行 MR 创建 API 前，Agent 必须先用中文询问：`本次 MR 要指派给谁、由谁审核？可以回复“机器人”、“加里克”、具体 GitLab 用户名/ID，或分别指定 assignee/reviewer。`
     - 若用户回复“机器人 / Team Robot / team_robot”，则 assignee 与 reviewer 均使用 `170`。
     - 若用户回复“加里克”，则 assignee 与 reviewer 均使用 `267`。
     - 若用户只指定一个人且未区分角色，默认 assignee 与 reviewer 都给同一人；若用户分别指定，则分别解析。
     - 只有当用户在当前请求中已明确给出 assignee/reviewer 时，才可以跳过再次询问。
4. **GitLab API 鉴权与 ID 自愈（直连 API 与防 403 健壮性设计）**：
   - **去 glab CLI 依赖**：技能全过程不再依赖、不再检测 `glab` CLI，完全使用原生 `curl` 与 GitLab REST API 进行网络通信与 MR 创建。
   - **Token 提取唯一限制**：严格遵循前置 Token 提取的 3 条直读路径，未找到时瞬间熔断降级，杜绝任何对宿主系统的空转探测。
   - **当前用户 ID 获取（绕过 403 越权）**：
     - 获取当前登录开发者的 ID 时，**严禁**使用全局用户列表接口。必须直接调用个人的账户接口：`GET https://code.hzmantu.com/api/v4/user`。此接口只需带上 Private Token，不会因为权限等级不足而触发 403 Forbidden。
   - **其他用户名转 ID**：
     - 欲将指派人（assignee）或审核人（reviewer）用户名转换为 ID，调用：`GET https://code.hzmantu.com/api/v4/users?username={username}`，**必须在 Header 中携带 Private Token**。
     - **防 403 / 失败弹性降级**：若该接口仍然返回 403 Forbidden、解析错误或无返回，**绝对不能报错中断流程**。此时应直接采取防御性重构：
       - 如果无法识别 `assignee_id` 或 `reviewer_ids`，在创建 MR 的 POST 请求中**去掉这两个参数**，让 MR 成功创建。并在终端最后以醒目的方式提示用户：
         > _“由于 GitLab 用户查询接口返回 403 权限受限，已为您成功创建 MR，但未自动指派负责人与审核人，请点击下方链接进入页面手动指派。”_
5. **Feature 提交前分支复用安全检查（精细化拓扑校验，防止重复 MR 与误拦截）**：
   - **适用范围与触发条件**：严格限制于 `mode=feature`、当前分支匹配 `feature-*` 且本次目标合并分支匹配 `release-*` 的场景。非此场景一律直接跳过该安全检查。
   - **双重精确检测逻辑（核心）**：
     为了防止“误判追加提交导致正常开发流中断”，**必须同时**经过以下两步校验，方可断定该分支“已完全合并”：
     - **第一步：Git 拓扑分析校验（最可靠指标）**：
       - 本地执行 `git fetch origin {target_branch}` 将目标分支的远程数据拉到最新。
       - 在本地运行命令：`git merge-base --is-ancestor HEAD origin/{target_branch}`（如果当前分支在远端也有提交，可以对 `origin/{current_branch}` 进行远端校验；若远端无此开发分支，则直接用 `HEAD` 与 `origin/{target_branch}` 校验）。
       - **若该命令返回非 0（假）**：说明本地或远端存在新的 Commit，这些改动在 `target_branch` 中**尚未**被包含（处于追加提交与迭代状态）。此时**判定为“未完全合并”**，直接跳过拦截，允许正常提交与 MR 追加。
       - **若该命令返回 0（真）**：说明本地已提交的所有历史确实已经被目标 release 分支完全覆盖了，进入第二步 API 深度核对。
     - **第二步：GitLab API 深度核对**：
       - 通过 GitLab API 查询该项目下 `source_branch={current_branch}`、`target_branch={target_branch}` 且 `state=merged` 的 MR 记录。
       - 若存在 merged MR，必须核实其最后一次合并的 `merge_commit_sha`。如果当前分支的最新 Commit SHA 与合并的 SHA 一致，才被断定为“已完全合并”。
   - **处理分支已完全合并状态**：
     - **若工作区无任何未提交改动（Dirty Write 为空）**：立即停止提交流程，友好提示用户：“当前分支所有提交已合并入 `{target_branch}` 且无新改动，无需重复创建 MR”。
     - **若工作区存在未提交改动（Dirty Write 不为空）**：为了防止在生命周期已结束的分支上继续写提交导致历史冲突，自动执行以下“无感脏写迁移”：
       1. 提示用户：“检测到当前开发分支已合并入目标基线，我们将暂存您的工作区，并自动为您基于最新 `{target_branch}` 拉取新开发分支，将修改安全移至新分支继续提交。”
       2. 执行 `git stash save "stash before auto branching because feature merged"` 保护工作区。
       3. 基于最新的 `origin/{target_branch}` 创建新开发分支 `new_branch`（沿用原 `task`，但追加短后缀或时间戳如 `V2` 或 `_V2` 以防重名冲突）。
       4. `git checkout -b {new_branch} origin/{target_branch}`。
       5. 执行 `git stash pop` 恢复工作区代码，接下来在新分支上安全地进行后面的 `gitFinish` 提交与推送（MR `source_branch` 改为新的 `new_branch`）。
6. **工具检测**：
   - 检查 `curl --version`。
   - **完全去除 glab 依赖**：严禁执行任何 `glab --version`、`glab auth status` 或 `glab config` 检测，避免因查找/运行不存在的命令消耗时间并导致多余的报错分析。

---

### 4. 操作时序与逻辑

0. **执行 Feature 提交前分支复用安全检查**：
   - 按“Feature 提交前分支复用安全检查”策略执行。
1. **暂存全部文件**：
   - 执行 `git add .`。
2. **轻量提交前自检与大模型 Review（限时，不做漫长阻断）**：
   - 在真正 `git commit` 前，基于暂存区 diff 做一轮**轻量质量自检**，用于兜住明显阻断问题，并沉淀 MR 描述素材。
   - **默认时间预算**：
     - 小改动：10-30 秒。
     - 普通改动：30-90 秒。
     - 大改动：只做摘要级扫描与风险识别，除非用户明确要求“完整 review”，否则不得升级为长时间逐行审查。
   - **建议采集**：
     - `git diff --cached --stat`：变更规模与文件分布。
     - `git diff --cached --name-status`：新增/修改/删除文件清单。
     - `git diff --cached`：大语言模型快速阅读关键 diff。
   - **只拦截 Critical / 明显阻断问题**：例如明显编译失败、需求明显未完成、明显空指针/数据损坏风险、疑似密钥/敏感信息、明显删错核心文件。发现这类问题时暂停提交并向用户说明。
   - **不得因非阻断问题拖慢流程**：命名偏好、轻微格式、可选重构、未来优化、个人风格建议，只能放入“Reviewer 重点关注”或“后续建议”，不得阻断提交/MR。
   - **产出 review_summary**：自检后生成简短 `review_summary`，后续 MR 描述直接复用该素材。
   - **review_summary 至少包含**：
     1. 本次改动目的：为什么改、解决什么问题。
     2. 核心改动点：按模块/文件归纳。
     3. 轻量自检结论：正确性、边界条件、安全/权限、性能影响是否有明显风险。
     4. 验证情况：已执行的测试/构建/人工验证；没有执行时标注“未执行”。
     5. Reviewer 重点关注：提醒 reviewer 重点看哪些接口、风险点。
3. **规范化提交**：
   - **提交信息模板**：`feat: {pingcode_id} {summary}`
   - **命令**：`git commit -m "feat: {pingcode_id} {summary}" --no-verify` (使用 `--no-verify` 绕过宿主环境残缺的本地 hooks 以确保提交流程通过)。
4. **生成 MR 描述 mr_description**：
   - 基于第 2 步的 `review_summary` 快速整理生成 `mr_description`，作为 GitLab MR 页面“描述”输入。
   - 推荐模板：
     ```markdown
     ## 关联需求
     - PingCode：{pingcode_id}
     - 标题：{summary}

     ## 本次改动
     - ...

     ## 变更复盘
     - 改动范围：...
     - 关键决策：...
     - 兼容性/影响面：...

     ## 自检 / Code Review
     - 正确性：...
     - 边界条件：...
     - 安全/权限：...
     - 性能影响：...

     ## 验证情况
     - [ ] 已执行：...
     - [ ] 未执行：...（原因）

     ## Reviewer 重点关注
     - ...
     ```
5. **推送代码并确立追踪关系**：
   - 执行 `git push -u origin {current_branch}` 将提交推送到远端并确立追踪关系。
6. **已有 Open MR 检查（避免重复创建）**：
   - 在真正创建 MR 前，必须用 GitLab API 简单查询是否已经存在相同 `resolved_source_branch -> resolved_target_branch` 的 open MR。
   - 若已存在 open MR：不要重复创建，直接提示用户“当前分支到目标分支已有对应 MR 申请”，并返回已有 MR 的 `web_url`。
7. **一键 MR 创建（GitLab REST API + curl 与网页自助直达双通道）**：
   - **核心原则**：默认在后台使用 `curl` 调 GitLab REST API 自动创建 MR。但若无 Token 或 API 报错（如 401, 403, 500 等），**绝对不能抛错阻断**，应瞬间降级为输出预填表单的网页直达链接，引导用户通过浏览器在网页端自助完成创建。这种设计不依赖本地凭证，且百分之百会成功。
   - **项目路径解析**：优先从 `git remote get-url origin` 提取 GitLab project path（如 `himo-projects/ai-space/generic-workflow-skill`），再 URL encode：`/` 替换为 `%2F`。若已知项目 ID，也可直接使用数字 project_id。
   - **Token 提取防空转（极简版）**：
     - 严格只尝试从系统环境变量、项目偏好配置和 `~/.config/glab-cli/config.yml` 配置文件三个路径中提取。若均不存在，**瞬间进入网页自助通道，严禁通过命令模糊搜寻**。
     - 稳健读取 config.yml 的 python 段落：
       ```bash
       TOKEN=$(python3 - <<'PY'
       import os, re
       path = os.path.expanduser('~/.config/glab-cli/config.yml')
       if os.path.exists(path):
           text = open(path).read()
           in_host = False
           for line in text.splitlines():
               if line.strip().startswith('code.hzmantu.com:'):
                   in_host = True
                   continue
               if in_host:
                   m = re.match(r'\s*token:\s*(.+)\s*$', line)
                   if m:
                       print(m.group(1).strip().strip('"'))
                       break
                   if re.match(r'\S', line):
                       in_host = False
       PY
       )
       ```
   - **创建 MR（自动 API 通道）**：
     - **启用前提**：当 `$TOKEN` 变量不为空且格式有效时，在后台启动 `curl`。
     - Feature 模式：`resolved_source_branch={current_branch}`，`resolved_target_branch={target_branch}`。
     - Hotfix 模式：按阶段解析 `resolved_source_branch/resolved_target_branch`。
     ```bash
     curl -sS --request POST \
       "https://code.hzmantu.com/api/v4/projects/{project_id_or_urlencoded_path}/merge_requests" \
       --header "PRIVATE-TOKEN: $TOKEN" \
       --data-urlencode "source_branch={resolved_source_branch}" \
       --data-urlencode "target_branch={resolved_target_branch}" \
       --data-urlencode "title=feat: {pingcode_id} {summary}" \
       --data-urlencode "description={mr_description}" \
       --data-urlencode "assignee_id={resolved_assignee_id}" \
       --data-urlencode "reviewer_ids[]={resolved_reviewer_id}"
     ```
     > 注：若之前在用户名转 ID 时遇到 403 Forbidden 导致未解析出有效的数字 ID，请在 POST 参数中**去掉 `assignee_id` 与 `reviewer_ids[]`**，确保 MR 本身能越过权限校验顺利提单。
   - **创建后 API 回读校验（闭环）**：
     - 若 curl 创建响应状态码为 201 Created，可执行 `curl` 回读接口确认 `source_branch`、`target_branch`、`assignees`、`reviewers`、`web_url` 与预期一致，再将 `web_url` 交付给用户。
   - **网页自助直达通道（安全兜底与无凭证首选）**：
     - **触发条件**：当本地未获取到有效的 Token，或者上述 API 接口调用返回任何非 200/201 的错误代码（如 401 Unauthorized、403 Forbidden 或网络隔离无法访问时）。
     - **处理逻辑**：立刻判定降级，拼接好预填了源分支、目标分支、MR 标题和本地 MR 描述表单的 Web 链接（进行 URL 编码）：
       `https://code.hzmantu.com/{project_path}/-/merge_requests/new?merge_request[source_branch]={current_branch}&merge_request[target_branch]={target_branch}&merge_request[title]={url_encoded_title}&merge_request[description]={url_encoded_description}`
     - **向用户反馈**：
       - _“已自动降级为网页安全自助通道，以确保 MR 绝对创建成功（不受本地配置或 API 权限限制）。请您点击下方链接，在打开的 GitLab 页面中直接点击确认按钮即可创建 MR：_
       - _🔗 [一键点击自助创建 Merge Request]({web_url})”_

---

## 📋 验收标准

- [ ] 分支命名符合命名规范（Feature 对应 `feature-{release}/{developer}_{task}_{YYYYMMDD}`，Hotfix 对应 `hotfix/{developer}_{task}_{YYYYMMDD}`）。
- [ ] 默认支持直接基于 Remote 检出分支以减少本地冗余基线分支（除非偏好配置中显式开启 `git.keep_local_baseline: true` 降级为原方案）。
- [ ] 创建分支后自动执行 `git push -u origin {new_branch}` 确立本地与远端新分支的强追踪关联（upstream 绑定）；gitBranch 阶段不得使用 `target_branch` 表示新分支。
- [ ] 开发者简称 `developer` 严格通过偏好配置 -> `git config` -> 交互问询的“三级探测机制”动态获取，支持中文简称智能转换为拼音小写简写（自愈），严禁硬编码默认 `allen`。
- [ ] 分支基于正确基线创建（Feature 基线为 `release` 分支，Hotfix 基线为 `master`/`main` 分支），且 MR 目标分支按模式正确解析：Feature 默认合并到用户指定的 `release`；Hotfix 使用同一个 hotfix 分支按阶段分别创建 `hotfix -> dev`、`hotfix -> pre`、`hotfix -> master`，阶段之间以用户测试/审核完成后的再次触发为准，不得误创建 `dev -> pre` 或 `pre -> master`。
- [ ] 提交 Commit 信息格式完美呈现 `feat: #ID 摘要`。
- [ ] Commit 前已基于 `git diff --cached` 做限时轻量自检（默认 30-90 秒，由大模型进行 Code Review）；只因 Critical / 明显阻断问题暂停提交，非阻断建议不得拖慢提交/MR。
- [ ] MR 描述不再只写 PingCode ID，已包含关联需求、本次改动、变更复盘、自检/Code Review、验证情况、Reviewer 重点关注。
- [ ] `gitFinish` 流程支持多偏好配置文件兼容嗅探，并具备默认指派人与审核人用户名到数字 ID 的动态解析与容错降级能力；创建 MR 前必须询问并确认本次 assignee/reviewer，用户回复“机器人”时使用 Team Robot（`170`）。
- [ ] 在 `git add` / `git commit` / 创建 MR 前，若且仅若当前是 `mode=feature`、`feature-*` source 分支、`release-*` 目标分支，才检查该 feature 分支是否已经 merged 到本次 release 目标分支；必须通过 `git merge-base --is-ancestor` 与 API 合并 SHA 深度双重确认最新提交确实已被包含，才拦截并提示或启动无感迁移。对于追加开发提交的 feature 分支不得误拦截。
- [ ] MR 创建前已轻量查询是否存在相同 source -> target 的 open MR；若已存在，只提示并返回已有 MR 链接，不重复创建。
- [ ] MR 创建默认使用 GitLab REST API + `curl` 方式，完全剥离对 `glab` CLI 的依赖，无 Token 或 API 返回报错时瞬间降级到网页自助直达通道，确保 100% 成功。
- [ ] MR 创建后已通过 API 回读校验 `source_branch`、`target_branch`、`assignees`、`reviewers`、`web_url`。
