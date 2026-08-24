# claude-skills

Claude Code 自定义 skill 集合。

## 安装

把需要的 skill 目录复制到 `~/.claude/skills/`(全局)或项目的 `.claude/skills/`(仅该项目):

```bash
cp -r delegate-mode ~/.claude/skills/
```

## Skills

### delegate-mode — 分派工作模式

精简主 agent 上下文的工作模式:

- 主 agent 只保留对话、任务状态和结论;实质工作分派给 Workflow / 子 agent 在干净短上下文里执行。
- 读文件收敛到 workflow 的第一个 agent,产出结构化 context pack,后续阶段通过脚本注入继承,不重复读取。
- 模型分级:规划、创新、分析用 fable(默认,给足自由度);目标明确的写码任务用 opus;简单搜索、读文件、无判断成分的压缩式总结用 sonnet。
- 子 agent 指令默认英文、结构化返回;任务内容本身是中文时直接用中文。结果译成中文向用户汇报。

触发:对 Claude 说「开分派模式」/「delegate mode」/「省着点 context 干」。
