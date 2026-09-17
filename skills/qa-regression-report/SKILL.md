---
name: qa-regression-report
description: >-
  Builds a QA functional regression report from developer-document MCP
  search_worklog (today / this week / current iteration via git vs shipped
  baseline, current project, all creators). Use when the user asks for
  回归报告, 功能回归, 给测试的报告, qa regression, regression report,
  本次迭代回归, or 今天改动回归.
---

# 功能回归报告（search_worklog）

用 **user-developer-document** MCP 的 `search_worklog` 拉当前项目时间窗内的工作日志，整理成给测试执行的 Markdown 功能回归报告。  
禁止只扫本地 `~/.kne_document/worklog` 凑合；禁止用 `search_experience` 代替。  
与 `weekly-report` 不同：默认**项目全员**、产出是**可执行用例**而非周报叙事。

## 何时使用

- 用户说：回归报告 / 功能回归 / 给测试的报告 / qa regression / regression report
- 本次迭代回归 / 今天改动回归 / 按本周 worklog 出回归用例

## 与 weekly-report 的差异

| | weekly-report | qa-regression-report |
|--|---------------|----------------------|
| 目的 | 对人汇报本周工作 | 给测试可执行回归 |
| 默认人员 | `mine: true` | 项目全员（不传 `mine`） |
| 时间 | 本周/上周为主 | 今天 / 本周 / **本次迭代（git）** |
| 产出 | 周报叙事 | 冒烟 + 用例表 + 追溯 |

## 流程

### 1. 确认 MCP 工具

1. `GetDynamicTools` → namespace `user-developer-document`，确认有 `search_worklog`
2. 若工具缺失：提示用户刷新 MCP / 确认已发布含该工具的 server；不要编造报告或用例

### 2. 解析时间窗

| 意图 | 做法 |
|------|------|
| 今天 | `startAt` / `endAt` = Asia/Shanghai 当日（ISO 日期或当日 00:00～次日 00:00） |
| 本周 | `week: "this"`（周一～周日，Asia/Shanghai） |
| 近 N 天 | `days: N` |
| 指定区间 | 用户给的 `startAt` / `endAt` |
| **本次迭代** | 见下方算法 |
| 未指明 | 默认 **本次迭代**；非 git 仓则回退 `week: "this"` 并说明 |

#### 本次迭代算法（当前打开的 git 仓库）

1. 按顺序找第一个存在的基线 ref（**禁止**为此 `git fetch`）：
   `origin/master` → `origin/main` → `master` → `main`
2. 都不存在 → 停止并说明，不要猜分支
3. `git rev-list --count <base>..HEAD`；为 0 → 输出「相对基线无独有提交」，可结束或仅写空范围说明
4. 取独有提交 author 日期：

```bash
git log <base>..HEAD --format=%aI
```

5. `startAt` = 最早日期的 `YYYY-MM-DD`；`endAt` = 最晚日期的次日（闭开）或同日结束 ISO
6. 报告头必须写明：基线 ref、当前分支、相对基线提交数、时间窗

### 3. 解析项目名

1. `git rev-parse --show-toplevel` → `basename` 作为项目名
2. 可用 `git remote get-url origin` 解析仓库名做校验（不一致时以 remote 仓库名为准并注明）
3. 传给 `search_worklog` 的 `project`

用户明确指定其他项目名时，以用户为准。

### 4. 拉取与去重

```text
search_worklog({
  project: "<repo>",
  startAt / endAt 或 week / days,
  mode: "report",
  limit: 200
})
```

- **不要**默认传 `mine: true`（项目全员）
- 用户明确只要自己时再加 `mine: true`
- 按 **标题 + 项目 + 日期（或 PR URL）** 去重
- 测试用 / 无实质内容（如纯 sync test）可略
- 无数据 → 写「该筛选无工作日志」，**不编造用例**

### 5. worklog → 测试条目映射

每条有效日志产出 **1～N** 条可执行回归项（优先 1 条主路径；必要时再加 1～2 条边界/关联）：

| worklog 字段 | 映射到 |
|-------------|--------|
| `title` / `requirement.summary` | 功能点 / 用例标题 |
| `finalSolution.keyChanges` | 验证点拆分、步骤线索 |
| `finalSolution.summary` | 预期结果依据 |
| `requirement.acceptanceNotes` | 验收关注（备注） |
| `pr.url` / `versionBump` | 追溯与版本 |
| `description` | 背景 / 变更说明 |

**优先级**：明确 bugfix / 验收修复 → P0；主流程功能 → P1；文案 / 文档 / 纯工程 → P2（可抽测）。

**类型**：`冒烟` / `功能回归` / `边界` / `兼容`；素材不足时默认 `功能回归`。

日志写不出具体步骤时：步骤写「待补充步骤」，保留变更摘要；**禁止臆造 UI 路径或点击流程**。

### 6. 输出报告（不要贴原始 MCP 全文）

默认聊天气泡交付 Markdown；用户要求落盘时再写文件。使用下方模板。

## 报告模板

```markdown
# 功能回归报告 · {projectName}

| 项 | 内容 |
|----|------|
| 项目 | {projectName} |
| 范围周期 | {start} ～ {end}（{今天/本周/本次迭代}） |
| 迭代基线 | {baseRef}（仅迭代模式） |
| 分支 | {currentBranch} · 相对基线提交数 {N} |
| 数据来源 | developer-document MCP · search_worklog |
| 日志条数 | {去重后条数} |
| 生成时间 | {ISO+08:00} |

## 1. 变更与风险摘要

- 用 3～8 条概括本窗口**对用户可见/可测**的变更（按模块，不按作者罗列）
- **建议重点回归**（P0/P1）：高风险模块与原因（bugfix、主流程、验收备注）
- **可降级/抽测**（P2）：文档、纯脚手架、无行为变化的工程项

## 2. 冒烟清单（建议先跑）

| ID | 模块 | 检查项 | 通过标准 | 关联 |
|----|------|--------|----------|------|
| S-01 | … | 一句话可执行检查 | 可见结果/无报错 | PR# / 日志标题 |

（从 P0/P1 提炼 5～15 条；无则写「本窗口无冒烟项」）

## 3. 功能回归用例

默认**表格紧凑版**；步骤过长的项在表下用小节展开。

| 用例ID | 模块 | 标题 | 优先级 | 类型 | 前置条件 | 步骤（简） | 预期结果 | 追溯（PR/版本/日志） | 结果栏 |
|--------|------|------|--------|------|----------|------------|----------|---------------------|--------|
| R-001 | … | … | P0/P1/P2 | 功能回归 | … | 1.… 2.… | … | PR / version / title | ☐ |

字段约定：

- **用例ID**：`R-xxx` 回归；`S-xxx` 冒烟；同报告内唯一
- **模块**：从标题/keyChanges 归类
- **优先级**：P0 必测 / P1 应测 / P2 抽测
- **前置条件**：环境、账号、数据、依赖版本；无则「常规测试环境」
- **步骤**：可执行、可判定；一步一动作
- **预期结果**：可观察、可判定（禁止「正常」「没问题」）
- **追溯**：PR 链接、versionBump、worklog 标题
- **结果栏**：留给测试（通过/失败/阻塞 + 备注）

复杂项展开示例：

### R-00x {标题}
- **优先级 / 类型**：…
- **前置条件**：…
- **步骤**：1. … 2. …
- **预期结果**：…
- **备注 / 验收关注**：…
- **追溯**：…

## 4. 兼容与回归关注（有则写）

- 升级依赖 / 发版顺序（日志提到下游仓时）
- 国际化、移动端、权限、空数据等仅在素材明确时列入

## 5. 不在范围 / 无法从日志确认

- 日志缺失细节、需产品/开发补步骤的项
- 明确「未覆盖」，避免测试误以为已全覆盖

## 6. 执行建议

- 顺序：冒烟 → P0 → P1 → P2
- 阻塞分级：环境 / 数据 / 缺陷（测试填写）
- 失败回传：环境、复现步骤、截图/录屏、关联 PR
```

## 快速调用示例

```text
# 本次迭代（先算 git 时间窗，再查）
search_worklog({ project: "components-core", startAt: "2026-09-10", endAt: "2026-09-18", mode: "report", limit: 200 })

# 本周 · 当前项目全员
search_worklog({ project: "components-core", week: "this", mode: "report", limit: 200 })

# 今天
search_worklog({ project: "components-core", startAt: "2026-09-17", endAt: "2026-09-18", mode: "report", limit: 200 })
```

## 禁止

- 跳过 MCP、凭记忆或聊天记录编造变更与用例
- 用 `search_experience` / 本地扫盘代替 `search_worklog`
- 为「本次迭代」执行 `git fetch`
- 默认只拉 `mine`（除非用户明确只要自己）
- 臆造 UI 路径、点击步骤或预期结果
- 把去重前的原始 MCP 列表当最终报告交给用户
- 用户未要求时不要 commit / 不要改业务代码
