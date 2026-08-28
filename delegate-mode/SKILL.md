---
name: delegate-mode
description: 精简主上下文的分派工作模式:主 agent(Fable)只保留对话与结论,实质工作分派给 Workflow/子 agent 在干净短上下文里执行;读文件收敛到 workflow 第一阶段产出 context pack 供后续阶段继承(结构化、无依赖的任务尽量多 agent 并行分片,尤其阅读类);按任务性质分级用模型(规划/创新/分析→fable,明确目标写码→opus,简单搜读→sonnet);指令用英文、结构化返回,结果译成中文向用户汇报。用户说"开分派模式/delegate mode/省着点context干/开workflow干活"时启用,启用后本会话持续生效。Delegation work-mode: lean main context, workflow-first execution with a read-once context pack and per-task model tiers.
---

# 分派工作模式(Delegate Mode)

启用后本会话持续生效:主 agent 的上下文是稀缺资源,只装三样东西——**与用户的对话、任务状态、各子任务的结论**。原文级的读、搜、改、验,发生在子 agent 的干净上下文里。本 skill 的工作方式以调用 Workflow 工具为主。

## 一、核心判据:不是"能分就分",而是"会灌多少"

分派省的是**主 agent 的上下文和注意力**,不是总 token——子 agent 要自己读材料,总消耗通常更高。唯一判据:

> 这个任务做完后,我上下文里会留下多少**不需要长期保留的原文**(文件全文、搜索结果堆、日志、diff)?

- 灌得多(≳ 数千 token 的原文只为得出一个小结论)→ 分派。
- 三五个 tool call、结果本身就是我要的 → 直接干。
- 拿不准 → 分派。主上下文污染不可逆,多花子 agent 的 token 可逆。

## 二、Workflow 结构:读一次,处处继承

Workflow 里的 agent **不自动共享上下文**。继承靠脚本手工传递,所以把"读"收敛成第一阶段:

0. **派发前先倾倒已知信息**:主 agent 把自己**已经知道**的关键事实通过 `args` 或直接写进 prompt 带进 workflow——确切文件路径、验证过能用的命令与工具、端口、配置字段名、会话里已定的结论与口径、已知的坑。子 agent 每一次"重新发现"主 agent 已知的事,都是白花的 tool call;stage 0 只去读主 agent 手里没有的东西。
1. **Stage 0 — Context Pack(读且只读一次,但不是一个 agent 读)**:context pack 的采集以主 agent 倾倒的已知信息为起点(不重新验证已确认的事实),返回结构化 **context pack**(JSON schema 强制):相关代码摘录(带 `file:line`)、约束、既有约定、术语表。宁可略厚,不可缺关键段——后续 agent 拿不到它没打包的东西。已知信息足够覆盖任务时,stage 0 可以整个跳过,把主 agent 给的信息直接当 pack 用。
   **分片原则(0828 用户令)**:任务较为结构化、适合分片、无依赖时,尽量分成多个 agent 并行操作——**尤其是阅读类型的任务**。分片结果用纯 JS merge 成一个 pack。每片 schema 保持小(大结构化输出会撞 180s 打断)。
2. **后续每个 agent 的 prompt = 英文指令骨架 + 注入 context pack(或其相关切片)**。后续 agent 原则上不再自己去读同一批文件;只有当它发现 pack 缺料时才允许补读,并在返回里注明补了什么(脚本可把补读结果合并回 pack)。
3. **多级传递**:pipeline 里前一阶段的返回值就是后一阶段的输入,把 pack 一路 thread 下去;需要的话在阶段间用纯 JS 做切片/合并,不要为搬运数据开 agent。

```js
// stage 0: sharded parallel reads, merged in pure JS
const shards = await parallel(fileGroups.map((g, i) => () => agent(
  `You are one of ${fileGroups.length} parallel readers building a context pack. Read ONLY your shard: ${g.files.join(', ')}. Return excerpts with file:line, constraints, conventions. Under 1500 words.`,
  { label: `pack:${g.name}`, model: 'sonnet', effort: 'xhigh', schema: SHARD_SCHEMA })))
const ok = shards.filter(Boolean)
const pack = { excerpts: ok.flatMap(s => s.excerpts), conventions: ok.flatMap(s => s.conventions), gaps: ok.flatMap(s => s.gaps) }
const results = await pipeline(tasks,
  (t) => agent(`${t.instruction}\n\n## Context pack\n${JSON.stringify(pack)}`, { model: t.model, effort: 'xhigh', schema: t.schema }))
```

## 三、模型分级

| 任务性质 | model | 说明 |
|---|---|---|
| 规划、创新、开放分析、方案取舍、验证裁决、综合成文 | `fable` | **默认倾向**:多数实质工作开 fable 做 dynamic workflow,prompt 给足自由度(给目标和边界,不给步骤清单),让它自己决定怎么达成 |
| 写代码、修 bug、执行明确 goal 的任务(目标清晰、成败可验) | `opus` | prompt 要有明确的 done 判据(测试过、编译过) |
| 简单搜索、读文件、不需要知识或思考的压缩式总结、context pack 采集、机械核对 | `sonnet` | 凡是"把长的变短、把散的变结构"且无判断成分的都归这档 |

拿不准就升一档,不降档——省错了模型比多花 token 贵。

**Effort 一律 xhigh(0827 用户令)**:所有 workflow `agent()` 调用与 `Agent` 派发,**无论模型档位**,显式传 `effort: 'xhigh'`,不留默认继承。仅两个例外:①用 `resumeFromRunId` 恢复时,已完成调用的 opts 必须保持一字不动(effort 在缓存键里,改了会让已完成的昂贵阶段重跑);②用户对某次任务显式另有指示。fable agent 的 prompt 风格:陈述 goal、约束、返回格式,明确允许它自主选择路径("You decide the approach; report what you chose and why in one line")。

## 四、分派路由(按任务形状)

| 任务形状 | 用什么 |
|---|---|
| 有任何多阶段/多项结构(读→做→验,N 个同构项,fan-out→synthesize) | `Workflow`(默认选择,套第二节结构) |
| 真正孤立的单发任务,连"读"都只有一小撮 | `Agent` 单发后台跑,模型按第三节选 |
| 需要**完整对话上下文**才能做对 | `Agent` 的 `fork`,或 inline |
| 琐碎(改一行、答一句、看一个值) | inline 直接干 |
| 会话本身太重、结论都在 | 先考虑 `/auto-compact`,那是压缩问题不是分派问题 |

并行子任务之间无依赖时,一条消息同时发出。

## 五、子 agent 的 prompt 规范

**默认英文**。每个 prompt 自包含:

1. **Task** — one unambiguous sentence(fable 档给 goal,opus 档给 goal + done 判据,sonnet 档给精确操作)。
2. **Context** — the context pack slice plus absolute paths. Never say "as discussed".
3. **Return format** — structured English; Workflow 一律传 JSON `schema`,Agent 单发则写明形状("Return: list of {file, line, finding}. No prose.")。告诉它 final text 是数据不是给人的消息。
4. **Scope fence** — what NOT to do。

**中文例外**:任务**内容本身**是中文时(中文文案/文档、中文材料解读、ASR 转写、本项目中文口径),子 agent 直接用中文干活,只保留结构化骨架;中文内容英译再回译只有损耗。

**质量结构**(要高置信度时):找→验分离,verifier 用 fable、以反驳视角独立复核,只把 confirmed 带回主上下文。

## 六、回收与汇报

- 英文结论**翻译成中文**汇报;专有名词、代码标识符、路径保持原文。
- 提炼成"结论 + 影响 + 下一步",不逐字转述;细节留在 transcript,用户要再取。
- 子 agent 返回**先抽查再采信**:关键改动看 diff 关键段,关键结论抽验一条;对用户负责的是主 agent。
- 如实说明哪些部分分派完成、验证到什么程度;失败/跳过的子任务不许吞。空结果先读 `journal.jsonl` 再下结论。

## 七、主上下文卫生

- 要留住的中间结论(work list、已定口径、待办)落盘成文件,主上下文只留一句指针。
- 不为"了解一下"inline 读大文件——问题连路径丢给子 agent,只收答案。
- 分派出去的任务不自己再做一遍。
- 每完成一个大阶段,评估是否 `/auto-compact`。

## 八、不适用信号(该任务回普通模式)

- 用户在**讨论/决策**而非要产出物——对话本身是交付。
- 任务强依赖会话中未落盘的隐性上下文,提炼成本超过任务本身。
- 破坏性/对外操作(删除、发布、外发)——主 agent 亲自做,按确认规则走,不分派。
