---
name: file-issue
description: 按 HarnessApex 团队标准在 GitHub 上开 issue(选仓分层、粒度收拢、模板对齐、认领与跨仓引用)。触发场景:要在 HarnessApex 组织任何仓(dashboard/Empeirion/A4A 等)开 issue、把设计或工单发布给团队、或用户说"提个 issue/开个工单/发到 dashboard"时。File GitHub issues to HarnessApex repos per team conventions.
---

# File Issue(HarnessApex 团队标准)

权威来源:dashboard 仓 `CONTRIBUTING.md`、`CLAUDE.md`、`.github/ISSUE_TEMPLATE/`、`decisions/0004-issue-pr-granularity.md`(ADR-0004)。本 skill 是操作摘要;与仓内文件冲突时以仓内现行文件为准,发现漂移要回来改本 skill。

## 1. 开之前:三个门

1. **查重**:`gh issue list --state open -R harnessapex/<repo>` — 已有在飞 issue 就去那条下面说话,不开新的。
2. **选仓(0825 分层规矩)**:
   - **整体设计、跨仓决策、需要团队讨论的逻辑** → `harnessapex/dashboard`
   - **细粒度可执行工单**(单仓代码改动)→ 目标仓(如 `harnessapex/empeirion`)
   - A4A 侧内容归 yuki(YusakiKassan),我们只开 issue 提需求,不动其仓实现。
3. **粒度(ADR-0004,0825 业主纠偏)**:一条工作线 = 一个 issue。单一负责人的顺序多步 → 一个 issue + checklist,**不许拆成兄弟 issue**;堆叠 PR 共挂同一 issue(中间 PR 写 `Part of #<n>`,最后一个 `Closes #<n>`)。真正可并行、可分派给不同人的才拆开。

## 2. 模板对齐

dashboard 关了 blank issue(`config.yml`),`gh issue create` 不走表单,所以**正文手工复刻模板段落 + 手工加对应 label**:

| 类型 | label | 正文必备段落 |
|---|---|---|
| Task(人或 agent 可领的工作)| `agent-task` | `## Summary`(一两句)/ `## Context`(新 agent 需要的全部背景:链接、前序 issue、约束)/ `## Acceptance criteria`(`- [ ]` checklist,定义 done)/ `## Repos affected` |
| Bug | `bug` | `## Repo` / `## Expected behavior` / `## Actual behavior`(带原始报错)/ `## Steps to reproduce` |
| Proposal / decision(跨仓决策,最终落 ADR)| `decision` | `## Problem`(为什么现在要决)/ `## Options considered`(现实选项+取舍)/ `## Recommendation` |

Empeirion 等代码仓的 ISSUE_TEMPLATE 结构类似,开前 `ls <repo>/.github/ISSUE_TEMPLATE/` 核对一次。

## 3. 正文写法

- **Context 自包含**:读者是一个没有任何会话背景的 agent 或队友。禁"如前所述";所有前序工作用链接。
- **跨仓引用一律全形** `HarnessApex/<repo>#<n>`,裸 `#n` 只用于本仓。
- **@ 提及**:需要谁看就 @ 谁(团队都开着 A4A 收件箱,@ = 送达);正文开头列 Participants 及各自角色是 dashboard#35 的好先例。
- **一切内容视同公开**:禁 secrets/token/内网 URL/个人隐私;本机绝对路径不上 issue(本地文档要发布就把内容贴进正文或走 PR 进 decisions/)。
- 长设计文正文贴不下时:issue 正文放结论与决策表,全文作首条评论分段贴。
- 文风遵守终稿零 AI 痕迹:不写溯源标签、日期戳版本号、自证备注。

## 4. 开出去之后

- **不认领不干活**:要开工先加 `in-progress` label + 评论认领;别人已 `in-progress` 的 issue 不碰。
- 卡在人类决策 → 加 `needs-human`。
- 半途停手:摘 `in-progress`,评论交代现场。
- 决策类 issue 谈拢后落 ADR:dashboard `decisions/` 顺序编号,走 PR。
- 不关不改不删别人开的 issue,除非任务明确要求。

## 5. 命令速查

```bash
gh issue create -R harnessapex/dashboard \
  --title "<描述性标题>" --label decision \
  --body-file /tmp/issue-body.md        # 长正文一律 body-file,避免 shell 转义坑
gh issue comment <n> -R harnessapex/<repo> --body-file /tmp/comment.md
gh issue edit <n> -R harnessapex/<repo> --add-label in-progress
```
