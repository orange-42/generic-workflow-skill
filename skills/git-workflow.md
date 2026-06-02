---
name: git-workflow
description: 整合分支生命周期管理（gitBranch）与提交收口（gitFinish）的驼峰命名 Git 工作流技能。支持自动 stash 暂存、基线同步、提交推送，以及在创建 MR 前确认指派人/审核人后通过 GitLab REST API + curl 稳定创建 MR 的能力。
parent: feishu-prd-workflow
phase: 5-and-6
depends_on: []
optional: false
---

# Git Workflow — 全生命周期 Git 工作流技能

此技能集成了从开发分支的拉取准备，到最终代码的自动化推送与 MR（Merge Request）合规创建。全面采用驼峰命名并内置了跨 IDE/Agent 的通用偏好配置文件兼容嗅探机制；MR 创建阶段默认使用 GitLab REST API + curl，并在创建前确认 assignee/reviewer。

---

## 🚀 动作一：gitBranch (分支创建与准备)

### 1. 触发方式示例 (Example)

> [!NOTE]
> 本技能是通用的分支创建组件，支持 Feature（日常迭代）和 Hotfix（线上热修复）双模式，且不绑定任何具体业务。以下是 Agent 调用本技能的自然语言触发示例：

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
> 当用户着急使用或口语化输入时，可能会提供简写的基线名称（例如：`1.0.5`、`v1.0.5`）或直接指定 hotfix。为了保障极致顺畅的无感开发体验，Agent **必须具备智能推理与前缀自愈心智**：

1. **Feature 模式 - 版本号前缀智能补全**：
   - 若在 `feature` 模式下，用户输入的 `release` 为纯版本号（如 `1.0.5`）或带有小写 `v` 前缀（如 `v1.0.5`）：
     - Agent 应当**智能推断并自动重构参数**，在内部将其自动补全为规范的标准分支名格式：`release-{version}`（例如将 `1.0.5` 转化为 `release-1.0.5`）。
2. **Hotfix 模式 - 生产主分支自适应**：
   - 若在 `hotfix` 模式下，基线默认指向生产分支。Agent 需自动探测远端是 `master` 还是 `main` 分支，自适应锁定正确的基线名。
3. **远端分支校验与探测**：
   - 在执行 `git checkout -b` 之前，先以补全或锁定后的标准分支名在远端（如 `origin/release-1.0.5` 或 `origin/master`）进行探测，匹配成功后直接以此为基准拉取，无需强行命令用户重新修改，极大提升敏捷度。
4. **任务名 task 缺失推断与小驼峰自愈机制**：
   - **task 是分支业务语义的核心参数**：Agent 不能在完全没有业务语义的情况下硬造 `task`，也不能使用 `task`、`temp`、`dev`、`fix`、工单号等无意义占位作为分支业务名。
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
   - **核心安全红线**：Agent **绝对不能**直接在代码中硬编码任何特定的默认开发者简称（如 `"allen"`）。这会导致其他团队成员拉分支时发生严重命名冲突与尴尬。
   - **三级探测机制 (智能推导)**：
     - **第一级：项目偏好配置**：优先嗅探项目根目录下偏好配置文件（如 `.agent_preferences.json`）中的 `git.developer` 或 `git.username` 配置项。
     - **第二级：本地 Git 环境探测**：若偏好文件未配置，静默执行 `git config user.name`，动态拉取并使用当前系统全局或局部配置的 Git 用户名。
     - **第三级：友好问询（绝对兜底）**：若以上级均未能成功解析出有效简称，Agent 必须主动触发熔断交互，以亲和的中文问询用户：
       > _“检测到您未指定开发者简称。为了规范拼装 Git 分支名称，请问您的英文简称或拼音是什么？（例如：`allen`，后续会自动记录在此环境）”_
   - **中文简称智能自愈**：
     - **拦截策略**：在上述任何一级探测中，若获取到的开发者名称中包含**中文字符**（例如：`张三` 或 `李四`），Agent **绝对不能直接将其拼入分支名中**，以防分支名出现非 ASCII 字符导致流水线或部署兼容性报错。
     - **自动拼音化**：Agent 必须在后台自动将其转换为**规范 of 拼音全拼或声母缩写（纯小写英文）**。
       - **自愈实例**：`张三` 自动自愈重构为 `zhangsan` 或 `zs`；`李四` 自动自愈重构为 `lisi` 或 `ls`。

### 4. ⚙️ 前置检查与智能偏好探测

1. **多 Agent 偏好配置文件与本地 Git 简称嗅探 (三级探测)**：
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
     - 4) 对获取的简称执行中文智能拼音/简写转换自愈（纯小写 ASCII），锁定最终 `developer` 参数。
2. **工作区状态智能嗅探与“无感脏写迁移”**：
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
     > _“🎉 您的本地修改已安全搬迁至已自动绑定追踪的新分支 `{new_branch}`！请问接下来您需要：_
     > \*1. **立即对这些修改进行规范化提交与 MR 收口 (直接为您唤起 `gitFinish` 流程)？\***
     > \*2. **保留在本地工作区，继续您的开发工作？\***
     > _请回复数字或说明您的意向。”_

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
   - **summary 轻量补全策略**：若 `summary` 缺失，不要做长时间推理，也不要为了生成摘要大量消耗上下文；优先用当前用户需求、PRD 标题、PingCode 标题、分支 `task` 或已知变更意图提炼一个简短中文摘要。
   - **summary 推断成功**：若能明确提炼，直接展示：`本次提交摘要我先按“{summary}”处理。` 然后继续流程。
   - **summary 无法可靠推断**：用一句话询问用户：`本次提交摘要写什么？例如：判断次卡的有效性类型。` 用户回复后继续；不要因为 summary 缺失做复杂分析或长时间阻塞。
   - **summary 禁止占位**：不得使用 `更新代码`、`修复问题`、`提交代码`、`优化功能` 等空泛摘要，除非用户明确指定。
2. **目标分支解析与确认（Feature / Hotfix 分流）**：
   - **Feature 模式**：目标合并分支必须是用户指定的 `target_branch`（兼容旧参数 `release_branch`），通常为 `release-x.y.z`。若缺失，必须先询问用户，不能默认猜测。
   - **Hotfix 模式采用同一 hotfix 分支的三段式投递引导**：线上修复不是一次性直进生产，也不是 `dev -> pre -> master` 的环境分支整体晋级；而是同一个 `hotfix/{developer}_{task}_{YYYYMMDD}` 分支在不同测试/审核阶段分别创建 MR：
     1. 第一段：`hotfix/{developer}_{task}_{YYYYMMDD}` -> `dev`
     2. 第二段：`hotfix/{developer}_{task}_{YYYYMMDD}` -> `pre`（通常在 dev MR 已合并且 dev 环境测试通过后，由用户再次触发）
     3. 第三段：`hotfix/{developer}_{task}_{YYYYMMDD}` -> `master`（通常在 pre MR 已合并且 pre 环境验证通过后，由用户再次触发）
   - **Hotfix 是引导式阶段流，不是强制一口气自动推进**：三段之间通常存在人工审核、环境部署、测试验证的时间差。Agent 每次只处理用户当前要推进的一段 MR，并在完成后提示下一步可能是等待测试 / 等待合并 / 继续创建下一阶段 MR；不得在一个请求中自动连续创建 dev、pre、master 三个 MR。
   - **Hotfix 阶段自动推导**：若 `mode=hotfix` 且 `hotfix_stage=auto` 或缺失，Agent 可通过 GitLab API 查询当前 hotfix 分支相关 MR，辅助判断下一步建议：
     - 若不存在 `hotfix -> dev` MR 或其尚未 merged：建议本次创建 / 处理 `hotfix -> dev`。
     - 若 `hotfix -> dev` 已 merged、但不存在 `hotfix -> pre` MR 或其尚未 merged：建议本次创建 / 处理 `hotfix -> pre`。
     - 若 `hotfix -> pre` 已 merged、但不存在 `hotfix -> master` MR 或其尚未 merged：建议本次创建 / 处理 `hotfix -> master`。
     - 若三段均已 merged：告知用户 Hotfix 投递链路已完成。
   - **Hotfix 阶段显式指定**：若用户指定 `hotfix_stage=dev/pre/master`，按对应阶段推导 `source_branch` 与 `target_branch`：source 始终是当前 hotfix 分支，target 分别为 `dev`、`pre`、`master`。
   - **前置状态只做风险提示与引导**：若用户要求创建 `hotfix -> pre` 但未检测到 `hotfix -> dev` 已 merged，或要求创建 `hotfix -> master` 但未检测到 `hotfix -> pre` 已 merged，Agent 应提示当前链路状态与风险，并请用户确认是否仍要继续；不要误创建 `dev -> pre` 或 `pre -> master`。
   - **参数兼容**：若同时存在 `target_branch` 与 `release_branch`，以 `target_branch` 为准；若只存在 `release_branch`，内部统一映射为 `target_branch`。Hotfix 模式下显式 `target_branch` 只作为高级覆盖，但 source 仍应保持为当前 hotfix 分支。
   - **安全红线**：不得把 Hotfix MR 误写成 `dev -> pre` 或 `pre -> master`；不得在用户只要求推进一段时连续创建多段 MR；不得把 Feature MR 默认合并到 master/main，除非用户明确指定。
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
     - 在执行 MR 创建 API 前，Agent 必须先用中文询问：`本次 MR 要指派给谁、由谁审核？可以回复“机器人”、 “加里克”、具体 GitLab 用户名/ID，或分别指定 assignee/reviewer。`
     - 若用户回复“机器人 / Team Robot / team_robot”，则 assignee 与 reviewer 均使用 `170`。
     - 若用户回复“加里克”，则 assignee 与 reviewer 均使用 `267`。
     - 若用户只指定一个人且未区分角色，默认 assignee 与 reviewer 都给同一人；若用户分别指定，则分别解析。
     - 只有当用户在当前请求中已明确给出 assignee/reviewer 时，才可以跳过再次询问。
4. **GitLab API 鉴权与 ID 自愈（curl-first）**：
   - **默认原则**：MR 创建默认直接调用 GitLab REST API + `curl`，不要先试 `glab mr create`。`glab` 只能作为辅助查询工具，不作为创建 MR 的主路径。
   - **Token 提取**：优先从 `~/.config/glab-cli/config.yml` 中读取 `code.hzmantu.com` 对应 token，并强制使用 HTTPS API：`https://code.hzmantu.com/api/v4/...`。
   - **用户名转 ID**：若偏好配置或用户输入只提供用户名，Agent 需通过 GitLab API 静默检索：
     - **API 路径**：`GET https://{domain}/api/v4/users?username={username}`
     - **提取逻辑**：从返回用户数组中提取 `id`，作为后续 `assignee_id` 或 `reviewer_ids[]`。
   - **健壮性降级兜底**：若用户输入无法解析，必须回问确认；不得静默改派。只有当用户明确说“机器人”或使用偏好建议默认值并确认后，才使用 `170`（Team Robot）。若用户明确指定加里克，则使用 `267`。
5. **Feature 提交前分支复用安全检查（防止已合并 feature 分支重复 MR）**：
   - **适用范围（严格限制）**：该检查仅适用于 `feature-*` 开发分支合并到目标 `release-*` 分支的场景，用于防止“某条 feature 分支近期已经合过目标 release，用户又忘记并继续在旧 feature 分支上提交、重复创建 MR”。
   - **不适用范围**：Hotfix 阶段流（`hotfix -> dev/pre/master`）、普通 `dev/pre/master` 环境分支、非 release 目标分支、以及用户明确指定的特殊合并场景，不应用这道自动迁移拦截。对这些场景最多做信息提示，不得自动阻断或迁移。
   - **触发条件**：仅当 `mode=feature` 且 `current_branch` 匹配 `feature-*`，并且本次 `target_branch` 匹配 `release-*` 时，才在执行 `git add` / `git commit` / 创建 MR 之前检查当前 `source_branch` 是否已经有合并到本次 `target_branch` 的 merged MR。
   - **检查方式**：优先通过 GitLab API 查询当前项目中 `source_branch={current_branch}`、`target_branch={target_branch}`、`state=merged` 的 MR；必要时可辅以 `git fetch origin {target_branch}` + `git merge-base --is-ancestor origin/{current_branch} origin/{target_branch}` 判断当前远端 source 是否已被目标分支包含。
   - **未发现已合并记录**：继续正常 `git add`、`commit`、`push`、创建 MR。
   - **发现当前 feature 分支已合并到目标 release 分支，且工作区没有新改动**：停止提交流程，提醒用户“当前 feature 分支已合并到 `{target_branch}`，且没有待提交改动，无需重复创建 MR”。
   - **发现当前 feature 分支已合并到目标 release 分支，但工作区存在新改动**：不要在旧 feature 分支上继续提交，也不要重复创建旧分支到同一 release 的 MR；应执行“无感迁移到新开发分支”：
     1. 先展示原因：`当前 feature 分支 {current_branch} 已合并到 {target_branch}，继续在该分支提交容易造成重复 MR 或历史混淆。我会先暂存当前工作区改动，再从最新 {target_branch} 拉一条新的开发分支继续提交。`
     2. 执行 `git status --porcelain` 确认有改动后，`git stash push -m "stash before new branch because {current_branch} already merged"`。
     3. `git fetch origin {target_branch}`，基于 `origin/{target_branch}` 创建新的 `new_branch`。新分支名沿用原业务 `task`，若当天同名分支已存在，则追加短后缀（如 `Retry`、`V2` 或时间片）避免冲突。
     4. `git checkout -b {new_branch} origin/{target_branch}`，再 `git stash pop` 把用户改动恢复到新分支。
     5. 继续执行后续 `git add`、`commit`、`push`、MR 创建流程；MR 的 `source_branch` 必须改为新的 `new_branch`。
   - **用户体验原则**：该检查只兜底 feature -> release 的重复合并风险，避免拦截过宽。只要能安全迁移，就帮用户把工作区代码先保护起来、拉新分支、恢复改动，再继续收口。
6. **工具检测**：
   - 检查 `curl --version`。
   - 可选检查 `glab --version`，仅用于辅助查询；不得因 `glab auth status` 异常阻断 curl API 主流程。

### 4. 操作时序与逻辑

0. **执行 Feature 提交前分支复用安全检查**：
   - 仅当 `mode=feature`、当前分支是 `feature-*`、且本次 MR 目标分支是 `release-*` 时，才按“Feature 提交前分支复用安全检查”判断当前 feature 分支是否已经合并到目标 release 分支；若需要迁移，完成 stash -> 基于目标 release 分支创建新分支 -> stash pop 后，再继续下面步骤。Hotfix 与其他非 release 目标分支不得被此检查自动阻断。
1. **暂存全部文件**：
   - 执行 `git add .`。
2. **轻量提交前自检（限时，不做完整 Code Review）**：
   - 在真正 `git commit` 前，必须基于暂存区 diff 做一轮**轻量质量闸门**，用于兜住明显阻断问题，并顺手沉淀 MR 描述素材；它不是完整 Code Review，不能因追求完美而显著拖慢提交/MR。
   - **默认时间预算**：
     - 小改动：10-30 秒。
     - 普通改动：30-90 秒。
     - 大改动：只做摘要级扫描与风险识别；除非用户明确要求“完整 review”，否则不得升级为长时间逐行审查。
   - **建议采集**：
     - `git diff --cached --stat`：变更规模与文件分布。
     - `git diff --cached --name-status`：新增/修改/删除文件清单。
     - `git diff --cached`：快速阅读关键 diff；大 diff 可抽样关键文件，不必逐行穷尽。
   - **只拦截 Critical / 明显阻断问题**：例如明显编译失败、需求明显未完成、明显空指针/权限/数据损坏风险、疑似密钥/敏感信息、明显删错核心文件。发现这类问题时暂停提交并向用户说明。
   - **不得因非阻断问题拖慢流程**：命名偏好、轻微格式、可选重构、未来优化、个人风格建议，只能放入“Reviewer 重点关注”或“后续建议”，不得阻断提交/MR。
   - **产出 review_summary**：自检后生成简短 `review_summary`，后续 MR 描述直接复用该素材，避免 MR 创建阶段重复 review。
   - **review_summary 至少包含**：
     1. 本次改动目的：为什么改、解决什么问题。
     2. 核心改动点：按模块/文件归纳，而不是逐行罗列。
     3. 轻量自检结论：正确性、边界条件、安全/权限、性能影响是否有明显风险。
     4. 验证情况：写清楚已执行的测试/构建/人工验证；没有执行时必须诚实标注“未执行”，不得编造。
     5. Reviewer 重点关注：提醒 reviewer 重点看哪些接口、字段、兼容性、数据迁移或风险点。
   - **快速通道**：若用户明确要求“快速提交/快速 MR”，可压缩为最小自检（stat + name-status + 敏感信息/明显错误扫描）和 3-5 行 MR 描述，但仍不能跳过 Critical 风险检查。
3. **规范化提交**：
   - **提交信息模板**：`feat: {pingcode_id} {summary}`
   - **命令**：`git commit -m "feat: {pingcode_id} {summary}" --no-verify` (使用 `--no-verify` 绕过宿主环境残缺的本地 hooks 以确保提交流畅通过)。
4. **生成 MR 描述 mr_description**：
   - 基于第 2 步的 `review_summary` 快速整理生成 `mr_description`，作为 GitLab MR 页面“描述”输入，不要只写 `PingCode ID`。
   - **注意**：MR 创建阶段只做格式化整理，不再重复执行完整 review，避免提交/MR 流程被拖慢。
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
   - 描述应简洁但可独立阅读，目标是让 reviewer 不打开 diff 也能先理解“为什么改、改了什么、怎么验证、哪里要重点看”。
5. **推送代码并确立追踪关系**：
   - 执行 `git push -u origin {current_branch}` 将提交推送到远端并确立追踪关系。
6. **已有 Open MR 检查（避免重复创建）**：
   - 在真正创建 MR 前，必须用 GitLab API 简单查询是否已经存在相同 `resolved_source_branch -> resolved_target_branch` 的 open MR。
   - 若已存在 open MR：不要重复创建，也不必自动更新 description / reviewer；直接提示用户“当前分支到目标分支已有对应 MR 申请”，并返回已有 MR 的 `web_url`。
   - 若不存在 open MR：继续创建 MR。
   - 该检查只做轻量查询，不做复杂合并判断，避免拖慢流程。
7. **一键 MR 创建（GitLab REST API + curl 主路径）**：
   - **核心原则**：对自建 GitLab（尤其 `code.hzmantu.com`），默认直接用 `curl` 调 GitLab REST API 创建 MR。不要先试 `glab mr create`，也不要依赖 push options 创建 MR；包装层越少越稳定。
   - **原因**：`glab mr create` / `glab api` 会受 host、protocol、repo path、用户名解析、数组参数包装影响；本机曾出现 `api_protocol: http` 与稳定 API 路径 `https://code.hzmantu.com/api/v4/...` 不一致、POST 行为形状异常、JSON unmarshal 等问题。
   - **项目路径解析**：优先从 `git remote get-url origin` 提取 GitLab project path，再 URL encode：`/` 替换为 `%2F`。若已知项目 ID，也可直接使用数字 project_id。
   - **Token 提取（稳健版）**：
     ```bash
     TOKEN=$(python3 - <<'PY'
     import os, re
     text=open(os.path.expanduser('~/.config/glab-cli/config.yml')).read()
     in_host=False
     for line in text.splitlines():
         if line.strip().startswith('code.hzmantu.com:'):
             in_host=True
             continue
         if in_host:
             m=re.match(r'\s*token:\s*(.+)\s*$', line)
             if m:
                 print(m.group(1).strip().strip('"'))
                 break
             if re.match(r'\S', line):
                 in_host=False
     PY
     )
     ```
   - **创建 MR（form-urlencoded，推荐）**：
     - Feature 模式：`resolved_source_branch={current_branch}`，`resolved_target_branch={target_branch}`。
     - Hotfix 模式：按阶段解析 `resolved_source_branch/resolved_target_branch`：source 始终是当前 hotfix 分支；target 按阶段分别为 `dev`、`pre`、`master`。
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
     > 使用 `--data-urlencode` 可以透明处理分支名、中文标题、描述文本和 `reviewer_ids[]` 数组参数，比 `glab` 包装层更可控。
   - **创建后必须回读校验（闭环）**：
     ```bash
     curl -sS \
       "https://code.hzmantu.com/api/v4/projects/{project_id_or_urlencoded_path}/merge_requests/{iid}" \
       --header "PRIVATE-TOKEN: $TOKEN"
     ```
     必须确认返回 JSON 中的 `source_branch`、`target_branch`、`state`、`assignees`、`reviewers`、`web_url` 与预期一致，再把 `web_url` 交付给用户。
   - **glab 辅助查询（可选，不作为主路径）**：
     - 可用 `glab mr list` / `glab ci list` 辅助查看状态。
     - 若确需使用 `glab api` 查询，必须显式加 `--hostname code.hzmantu.com`，但关键 MR 创建与校验仍优先 curl API。
   - **网页直达降级（安全兜底）**：
     - 若 curl API 因 token/权限/网络不可达而失败，输出一键直达的 GitLab 手动创建链接：
       `https://{domain}/{project_path}/-/merge_requests/new?merge_request[source_branch]={current_branch}`

---

## 📋 验收标准

- [ ] 分支命名符合命名规范（Feature 对应 `feature-{release}/{developer}_{task}_{YYYYMMDD}`，Hotfix 对应 `hotfix/{developer}_{task}_{YYYYMMDD}`）。
- [ ] 默认支持直接基于 Remote 检出分支以减少本地冗余基线分支（除非偏好配置中显式开启 `git.keep_local_baseline: true` 降级为原方案）。
- [ ] 创建分支后自动执行 `git push -u origin {new_branch}` 确立本地与远端新分支的强追踪关联（upstream 绑定）；gitBranch 阶段不得使用 `target_branch` 表示新分支。
- [ ] 开发者简称 `developer` 严格通过偏好配置 -> `git config` -> 交互问询的“三级探测机制”动态获取，支持中文简称智能转换为拼音小写简写（自愈），严禁硬编码默认 `allen`。
- [ ] 分支基于正确基线创建（Feature 基线为 `release` 分支，Hotfix 基线为 `master`/`main` 分支），且 MR 目标分支按模式正确解析：Feature 默认合并到用户指定的 `release`；Hotfix 使用同一个 hotfix 分支按阶段分别创建 `hotfix -> dev`、`hotfix -> pre`、`hotfix -> master`，阶段之间以用户测试/审核完成后的再次触发为准，不得误创建 `dev -> pre` 或 `pre -> master`。
- [ ] 提交 Commit 信息格式完美呈现 `feat: #ID 摘要`。
- [ ] Commit 前已基于 `git diff --cached` 做限时轻量自检（默认 30-90 秒，不做完整 Code Review）；只因 Critical / 明显阻断问题暂停提交，非阻断建议不得拖慢提交/MR。
- [ ] MR 描述不再只写 PingCode ID，已包含关联需求、本次改动、变更复盘、自检/Code Review、验证情况、Reviewer 重点关注。
- [ ] `gitFinish` 流程支持多偏好配置文件兼容嗅探，并具备默认指派人与审核人用户名到数字 ID 的动态解析与容错降级能力；创建 MR 前必须询问并确认本次 assignee/reviewer，用户回复“机器人”时使用 Team Robot（`170`）。
- [ ] 在 `git add` / `git commit` / 创建 MR 前，若且仅若当前是 `mode=feature`、`feature-*` source 分支、`release-*` 目标分支，才检查该 feature 分支是否已经 merged 到本次 release 目标分支；若已合并且工作区有新改动，必须先 stash 保护改动、基于目标 release 拉新 `new_branch`、恢复改动后再继续提交和 MR。Hotfix 与其他非 release 目标不得被此规则自动阻断。
- [ ] MR 创建前已轻量查询是否存在相同 source -> target 的 open MR；若已存在，只提示并返回已有 MR 链接，不重复创建。
- [ ] MR 创建默认使用 GitLab REST API + `curl`，强制命中 `https://code.hzmantu.com/api/v4/...`，不得先走 `glab mr create`。
- [ ] MR 创建后已通过 API 回读校验 `source_branch`、`target_branch`、`assignees`、`reviewers`、`web_url`。
