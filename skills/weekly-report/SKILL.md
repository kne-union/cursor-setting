---
name: weekly-report
description: >-
  Writes a weekly work report from developer-document MCP search_worklog
  (time + creator filters, markdown by project). Use when the user asks for
  周报, 本周周报, 上周周报, weekly report, or to summarize worklogs for a week.
---

# 周报（search_worklog）

用 **user-developer-document** MCP 的 `search_worklog` 拉日志，再整理成可读周报。  
禁止只扫本地 `~/.kne_document/worklog` 凑合；禁止用 `search_experience` 代替。

## 何时使用

- 用户说：写周报 / 本周周报 / 上周周报 / weekly report / 汇总本周工作
- 需要按时间段 + 创建人拉取已同步 worklog

## 流程

### 1. 确认 MCP 工具

1. `GetDynamicTools` → namespace `user-developer-document`，确认有 `search_worklog`
2. 若工具缺失：提示用户刷新 MCP / 确认已发布含该工具的 server；不要编造周报内容

### 2. 拉取素材（默认本周自己）

先调 `search_worklog`，参数按用户意图选：

| 意图 | 参数 |
|------|------|
| 本周自己（默认） | `week: "this"`, `mine: true`, `mode: "report"` |
| 上周自己 | `week: "last"`, `mine: true` |
| 近 N 天 | `days: N`, `mine: true` |
| 指定区间 | `startAt` / `endAt`（ISO 日期） |
| 指定同事 | `creator: "昵称或邮箱"`，或 `createdUserId`（精确） |
| 某项目 | 再加 `project: "仓库名"` |
| 只要紧凑列表 | `mode: "list"` |

建议：`limit: 200`（上限 200）。自然周按 **Asia/Shanghai，周一～周日**。

`creator` 模糊匹配多人时：展示候选，让用户用返回的 `createdUserId`，不要猜。

### 3. 去重与归并

MCP 返回可能有 path 重复（带/不带 `worklog/` 前缀）：

- 按 **标题 + 项目 + 日期（或 PR URL）** 去重，同一事项只保留一条
- 测试用 / 无实质内容的日志（如纯 sync test）可略写或不入正文

### 4. 写成周报（不要贴原始 MCP 全文）

在素材基础上重组，默认结构：

```markdown
# 本周工作周报（{创建人}）

**周期**：{start} ～ {end}
**数据来源**：developer-document MCP · search_worklog

## 一、本周概要
用 2～4 条主线概括（按主题，不是按仓库罗列）。

## 二、重点工作
按主题分节（如：文档平台与 MCP、弹层依赖链、业务交付…）。
每节用表格或短列表：事项 | PR 链接 | 版本（有则写）。

## 三、产出与影响
3～5 条可感知结果（能力闭环、发版、修复等）。

## 四、下周建议（可选）
仅当素材或对话里有线索时写；不要空编。
```

若用户指定公司模板（钉钉/飞书/「完成+计划」两段），改用其格式，仍只用 MCP 数据。

### 5. 诚实边界

- 只写 MCP 返回里有的事项；缺失就写「本周无记录」或「该筛选无数据」
- 不要把经验卡、未同步本地文件冒充已同步周报素材
- 用户未要求时不要 commit / 不要改业务代码

## 快速调用示例

```text
search_worklog({ week: "this", mine: true, mode: "report", limit: 200 })
search_worklog({ week: "last", creator: "张三", project: "components-core" })
search_worklog({ days: 7, mode: "list", limit: 50 })
```

## 禁止

- 跳过 MCP、凭记忆或聊天记录编造完成项
- 用 `search_experience` / 本地扫盘代替 `search_worklog`
- 把去重前的 40+ 条原始列表当最终周报交给用户
- `creator` 多人命中时擅自选一个
