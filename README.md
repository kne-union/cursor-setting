# cursor-setting

集中维护 Cursor **用户级** rules（及后续 skills）的仓库。先在本仓库改规则、评审合并，再同步到本机 `~/.cursor/`，避免直接改本机配置导致丢失或与团队约定不一致。

仓库：[kne-union/cursor-setting](https://github.com/kne-union/cursor-setting)

## 目录结构

```text
cursor-setting/
├── rules/                 # 用户级 Cursor Rules（.mdc）
│   ├── development-workflow.mdc
│   ├── business-development-workflow.mdc
│   ├── worklog-write.mdc
│   ├── experience-distill-write.mdc
│   ├── experience-similar-search.mdc
│   ├── kne-document-search.mdc
│   ├── project-prompts-compliance.mdc
│   ├── no-restart-user-services.mdc
│   └── responsive-utils-mobile.mdc
└── README.md
```

> 本机还可能有仅存在于 `~/.cursor/rules/` 的规则（如 `cursor-setting-first`），未进本仓库的不要当作本仓源文件随意覆盖。

## Rules 一览

| 文件 | 作用 | 生效范围 |
|------|------|----------|
| `development-workflow.mdc` | 改代码：有 WIP 则 stash → 建分支 → 开发 → 验收 → 升版本/提交 → Push/PR → 还原 stash；结束后输出 `DEVELOPMENT_COMPLETE` | **仅** remote 为 `kne-union/*` 的仓库 |
| `business-development-workflow.mdc` | 改代码：原仓有 WIP 则 stash → 拷到 `~/.cursor_workplace` → 立刻还原；验收后副本提交 → 原仓再 stash → 同名分支拷回并 merge 个人基线 → 立刻还原；输出 `DEVELOPMENT_COMPLETE` | **非** kne-union 的业务仓库 |
| `worklog-write.mdc` | 收到 `DEVELOPMENT_COMPLETE` 后写入工作日志，并输出 `WORKLOG_WRITTEN` | 全局（靠信号门禁） |
| `experience-distill-write.mdc` | 收到 `WORKLOG_WRITTEN` 后提炼经验卡（价值门槛过滤；business/library/process；先搜后补） | 全局（靠信号门禁） |
| `experience-similar-search.mdc` | 动手前分层检索 `experience/business`、`library` 与 `process` | 全局 |
| `kne-document-search.mdc` | 查 `@kne/*` / remote 组件文档：经 `@kne/npm-tools` 建索引再分层检索 | 涉及 KNE 文档时 |
| `project-prompts-compliance.mdc` | 项目有 `prompts/` 时先读并按规范编写文档 / 示例；禁止擅自跑 `build:docs`、写 `docs/` / `dist` 示例产物等 | 全局（有 prompts 时强制） |
| `responsive-utils-mobile.mdc` | 移动端适配统一用 `@kne/responsive-utils` | 全局（KNE 前端） |
| `no-restart-user-services.mdc` | 禁止擅自重启用户本地开发服务 | 全局 |

信号链（kne-union / 业务项目开发收尾）：

```text
development-workflow（PR 成功）
  或 business-development-workflow（原仓个人基线 merge 成功）
  → DEVELOPMENT_COMPLETE
  → worklog-write → worklog/{project}/{timestamp}/{title}.json
  → WORKLOG_WRITTEN
  → experience-distill-write → experience/business|library|process/...（价值门槛 → 先搜后补；低价值 skipped）
```

路径均相对 `~/.kne_document/`（禁止在 JSON/信号中存绝对路径）。业务项目的 workplace 副本在 `~/.cursor_workplace/`，不进本仓库。

## 使用方法

### 1. 克隆

```bash
git clone https://github.com/kne-union/cursor-setting.git
cd cursor-setting
```

### 2. 同步到本机 Cursor（确认后再做）

将仓库内规则复制到用户级 rules 目录（按需覆盖）：

```bash
# 仅同步本仓库维护的规则文件
cp rules/*.mdc ~/.cursor/rules/
```

同步后可删除本机已废弃的 `experience-summary-write.mdc`（若仍存在）。

或指定单个文件：

```bash
cp rules/development-workflow.mdc ~/.cursor/rules/
```

同步后新开 Cursor Agent 会话，或确认 User Rules / Project Rules 已加载对应 `.mdc`。

**约定：**

1. 先在本仓库改 `rules/*.mdc` 并走 PR
2. 合并且你明确确认后，再 `cp` 到 `~/.cursor/rules/`
3. 不要把本机当作唯一源；不要未确认就批量覆盖本机其它无关配置

### 3. 新增 / 修改规则

1. 有未提交改动则 `git stash push -u` 后再从 `master` 拉分支：`cursor/<简短英文描述>`
2. 在 `rules/` 下新增或编辑 `.mdc`（YAML frontmatter + Markdown 正文）
3. 本地看一眼描述与 `alwaysApply` / 适用范围是否正确
4. 提交、开 PR；合并后按上一节同步本机

`.mdc` 最小结构：

```markdown
---
description: 一句话说明，便于规则选择器展示
alwaysApply: true
---

# 标题

规则正文…
```

仅对特定文件生效时可用 `globs`，并设 `alwaysApply: false`。

### 4. 相关本机路径（规则会读写，但文件不在本仓库）

| 路径 | 用途 |
|------|------|
| `~/.cursor/rules/` | Cursor 用户级 rules 生效目录 |
| `~/.cursor_workplace/{项目}/{时间戳}/` | 业务项目开发用副本（见 `business-development-workflow`） |
| `~/.kne_document/worklog/` | 会话工作日志 |
| `~/.kne_document/experience/business/` | 项目业务经验卡 |
| `~/.kne_document/experience/library/` | 组件/库用法与场景经验卡 |
| `~/.kne_document/experience/process/` | 发版 / PR / CI 等交付流程经验卡 |
| `~/.kne_document_indexed/` | `@kne` / remote 文档切分索引（见 `kne-document-search`） |

## 与项目级规则的区别

| 类型 | 位置 | 适用 |
|------|------|------|
| 用户级（本仓库） | `rules/` → 同步到 `~/.cursor/rules/` | 所有打开的项目（再由各规则自行限定范围） |
| 项目级 | 各业务仓 `.cursor/rules/` | 仅该仓库 |

单仓库专用约定请写在业务仓项目级 rules，不必放进本仓。

## 许可证

与 kne-union 组织约定一致；未单独声明时仅供组织内使用。
