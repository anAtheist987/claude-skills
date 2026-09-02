# claude-skills

Claude Code 自定义 skill 集合。

## 安装

把需要的 skill 目录复制到 `~/.claude/skills/`(全局)或项目的 `.claude/skills/`(仅该项目):

```bash
cp -r delegate-mode ~/.claude/skills/
```

## Skills

### delegate-mode — 分派工作模式 v2

人定目标与出口判据,编排者(fable)定计划,异构 agent 干活并互查,出口靠实证验收:

- 主 agent 只保留对话、任务状态和结论;实质工作分派给 Workflow / 子 agent 在干净上下文里执行,读文件收敛为一次性 context pack。
- 固定生命周期:定出口 → 烟测前置 → 找问题 → 甄别真伪 → 规格经 codex 校对 → TDD 修复 → 异构验证 → 集成(一包一 PR)→ 实证验收(装机/升级/互通)→ 发布 + 给人的变更摘要。
- codex 是独立评审者,替代缺席的人类评审,放在规格与计划阶段而不只是 diff 阶段;必须真调用本机 codex CLI。
- 模型分级:规划/裁决 fable,写码 opus,搜读 sonnet,审计/校对 codex;effort 一律 xhigh。
- 工程守则:事实块标来源与时间、成果落 git 不落返回值、flake 隔离不列名单、e2e 环境隔离、负载纪律、版本号诚实。

触发:对 Claude 说「开分派模式」/「delegate mode」/「省着点 context 干」。
