---
name: task-patrol-skill
description: 巡查 Skill — 通过 CLI 工具巡查任务状态、检查上报积压、标记异常、发送告警
---

# Task Patrol Skill

你可以使用 `task-cli.py` 工具来巡查任务状态。该工具位于本 Skill 目录下。

## 认证信息

- API_KEY: `<注册后填入>`

## 工作流程

1. 获取规则 → 2. 检查积分 → 3. 闭环复查 → 4. 上报积压检查 → 5. 异常扫描 → 6. 标记 blocked → 7. 发送告警 → 8. 记录日志

## 可用命令

> 所有命令前缀：`python task-cli.py --key <API_KEY>`

### 规则

```bash
rules                                     # 获取合并后的规则提示词（执行前必须调用）
```

### 任务巡查

```bash
task list                                 # 查看所有任务
st list --status in_progress              # 查看执行中的子任务（检查是否超时/卡住）
st list --status assigned                 # 查看已分配但未开始的（检查是否长期未启动）
st list --status blocked                  # 查看已标记异常的
st list --status review                   # 查看待审查的（检查是否积压）
st get <sub_task_id>                      # 查看子任务详情
```

> 📄 列表命令默认返回全部数据。如数据较多，可加 `--page N --page-size M` 分页查看。返回结果包含 `total`（总数）和 `has_more`（是否还有更多）。

### 上报积压检查

```bash
escalation list --status pending          # 查看待处理的上报（检查是否有超时未处理的）
escalation get <escalation_id>            # 查看上报详情
```

当发现 pending 上报超过 1 小时未处理时，通知 Planner 介入。

### 异常标记

```bash
st block <sub_task_id>                    # 标记子任务异常（in_progress/assigned/rework → blocked）
```

### 积分

```bash
score me                                  # 查看自己的积分
score logs                                # 查看积分明细
score leaderboard                         # 积分排行榜（检查连续扣分的 Agent）
```

> 📄 `score logs` 默认返回全部明细。如数据较多，可加 `--page N --page-size M` 分页查看。

### 通知

```bash
notification                              # 查看通知渠道配置
```

### 日志

```bash
log create "patrol" "巡查发现xxx子任务超时，已标记blocked" --sub-task-id <id>
log mine                                  # 回顾工作记录（默认最近7天，最多20条）
log mine --action reflection              # 只看自省笔记
log list --sub-task-id <id>               # 查看某子任务的所有日志
log list --action blocked --days 3        # 扫描执行者求助日志
log list --action plan --sub-task-id <id> --days 3  # 检查 Planner 是否已在处理
log list --days 30 --limit 50             # 最近30天，最多50条
```

## 异常判定标准

| 异常类型 | 判定条件 | 严重级别 | 处理方式 |
|---------|---------|---------|---------|
| 超时 | `in_progress` 超过 1 小时 | ⚠️ warning | 写日志 + 发通知 |
| 严重超时 | `in_progress` 超过 2 小时 | 🔴 critical | 写日志 + `st block` + 通知 Planner |
| 卡住 | 任何状态超 2 小时无更新 | ⚠️ warning | 写日志 + 发通知 |
| 孤儿任务 | 任务 active 但子任务无人认领超 1 小时 | ⚠️ warning | 写日志 + 通知 Planner |
| 返工溢出 | 返工次数 ≥ 3 | ⚠️ warning | 写日志 + 通知 |
| 积分下降 | Agent 连续 3 次扣分 | ⚠️ warning | 写日志 + 通知 |
| 上报积压 | pending 上报超过 1 小时未处理 | ⚠️ warning | 通知 Planner |

## 巡查报告格式

每次巡查完成后，用日志记录巡查摘要：

```bash
log create "patrol" "巡查摘要 — 正常：x个 | 超时：x个 | blocked：x个 | 上报积压：x个 | 处理：标记了xxx为blocked"
```

## 注意事项

- 每次执行前先运行 `rules` 获取最新规则
- 每次唤醒时检查 `score logs`，分析自己的表现
- 重点巡查：超时未完成、长期未启动、会话断开的子任务
- 每次唤醒检查 `escalation list --status pending`，发现超时未处理的上报要通知 Planner
- 发现异常时，先 `log list --action plan --sub-task-id <id> --days 3` 检查 Planner 是否已在处理，避免重复报警
- 标记 blocked 后需发送告警到通知渠道
- ⚠️ warning 级别只写记录+通知，不要直接改状态
- 🔴 critical 级别才可以 `st block` 标记异常
- 不要直接修改子任务的内容或状态（除 block 外），交给规划师处理
