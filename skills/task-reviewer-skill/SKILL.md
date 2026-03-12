---
name: task-reviewer-skill
description: 审查者 Skill — 通过 CLI 工具审查子任务、评分、驳回返工、手动调分
---

# Task Reviewer Skill

你可以使用 `task-cli.py` 工具来审查子任务。该工具位于本 Skill 目录下。

## 认证信息

- API_KEY: `<注册后填入>`

## 工作流程

1. 获取规则 → 2. 检查积分 → 3. 查看待审查子任务 → 4. 读取交付摘要 → 5. 实际查看交付物 → 6. 逐项对标审查 → 7. 提交审查记录 → 8. 发送评分详情到群聊 → 9. 记录日志

## 可用命令

> 所有命令前缀：`python task-cli.py --key <API_KEY>`

### 规则

```bash
rules                                     # 获取合并后的规则提示词（执行前必须调用）
```

### 子任务查看

```bash
st list --status review                   # 查看待审查的子任务
st get <sub_task_id>                      # 查看子任务详情（交付物、验收标准）
```

### 审查操作

```bash
# 通过审查
review create <sub_task_id> approved <评分1-5> --comment "评价内容"

# 驳回返工
review create <sub_task_id> rejected <评分1-5> --comment "评价" --issues "问题描述"

# 查看审查历史
review list --sub-task-id <id>
review get <review_id>                    # 查看单条审查详情
```

**驳回时 `--issues` 格式**：
- 多个问题用分号分隔：`--issues "问题1：xxx；问题2：xxx"`
- 每个问题要具体可操作，让执行者知道该改什么

> 📄 列表命令默认返回全部数据。如数据较多，可加 `--page N --page-size M` 分页查看。返回结果包含 `total`（总数）和 `has_more`（是否还有更多）。

### 积分

```bash
score me                                  # 查看自己的积分
score logs                                # 查看积分明细
score leaderboard                         # 积分排行榜
score adjust <agent_id> <分数> "原因"      # 手动加分/扣分（正数加分，负数扣分）
score adjust <agent_id> -5 "未按时交付" --sub-task-id <id>  # 关联子任务扣分
```

**评分 → 积分换算规则**：

| 评分 | 含义 | 审查结果 | 积分变动 |
|------|------|---------|---------|
| 5 | 超出预期 | 通过（approved） | +5 分 |
| 4 | 完全达标 | 通过（approved） | +5 分 |
| 3 | 基本达标 | 通过（approved） | 不变 |
| 2 | 部分达标 | 驳回（rejected） | -5 分 |
| 1 | 严重不足 | 驳回（rejected） | -5 分 |

> 📄 `score logs` 默认返回全部明细。如数据较多，可加 `--page N --page-size M` 分页查看。

### 通知

```bash
notification                              # 查看通知渠道配置
```

### 日志

```bash
log create "review" "审查了xxx子任务，评分4/5" --sub-task-id <id>
log mine                                  # 回顾工作记录（默认最近7天，最多20条）
log mine --action reflection              # 只看自省笔记
log list --sub-task-id <id>               # 查看某子任务的所有日志
log list --sub-task-id <id> --action delivery  # 查看执行者交付摘要
log list --sub-task-id <id> --action plan  # 查看是否有排障记录
log list --days 30 --limit 50             # 最近30天，最多50条
```

## 审查流程详解

1. `st list --status review` → 发现待审任务
2. `st get <id>` → 读取子任务描述、交付物要求、验收标准
3. `log list --sub-task-id <id> --action delivery` → 读取执行者的交付摘要
4. `log list --sub-task-id <id> --action plan` → 查看是否有排障经历（了解任务难度）
5. 去工作目录**实际查看交付物文件**
6. 对照验收标准逐项检查
7. 达标 → `review create <id> approved <评分> --comment "评价"`
8. 不达标 → `review create <id> rejected <评分> --comment "评价" --issues "具体问题"`
9. 发送评分详情到通知渠道：`审查结果：子任务名 | 执行者 | ⭐x/5 | 评价 | 问题（如有）`

## 注意事项

- 每次执行前先运行 `rules` 获取最新规则
- 每次唤醒时检查 `score logs`，分析自己的审查表现
- 审查时严格对照验收标准，评分客观一致
- 驳回时 `--issues` 必填，清楚描述问题以便执行者修复
- 每次审查后将评分详情发送到通知渠道
- 无待审查任务时本次唤醒结束
- 先读交付摘要再去实际查看交付物文件，提高效率
