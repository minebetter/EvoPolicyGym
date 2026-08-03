---
locale: zh
page: nethack-policy-evolution
title: "深入地下城：在 NetHack 中自主演化 Policy"
description: "Coding Agent 如何利用有界的 NLE 原始轨迹诊断行为、重写可执行 Policy，并接受 held-out NetHack Assessment。"
lead: "Coding Agent 将完整的 Policy 可见 NetHack 轨迹转化为能够导航、处理障碍并向地下城深处推进的可执行策略。"
publishedAt: "2026-08-03"
date: "2026-08-03"
authors: [evopolicygym]
tags:
  - Benchmark
  - NetHack
  - Experiment
  - Policy Evolution
status: published
---

我们向 Coding Agent 提供 NetHack Environment、一个可执行 baseline，以及完整的
Policy 可见原始轨迹。它们的任务不是描述一套策略，而是写出一套能够独立接受
held-out 评测的策略。

<!-- truncate -->

## 通过 NLE 进入 NetHack

NetHack 是一款经典的终端 Roguelike 游戏。玩家需要探索程序生成的地下城，战斗或
避开怪物，应对饥饿与陷阱，管理装备和消耗品，并寻找通往更深层的道路。游戏的
长期目标是取得 Yendor 护符并成功 ascend；但即使只想在前期推进，也需要在部分
可观测条件下理解简短的文字消息，并连续做出大量相互关联的决策。

NetHack Learning Environment（NLE）将这款游戏封装成面向 Agent 研究的 Python
接口。本实验使用可独立安装的 distribution，接入 NLE 1.3.0 的
`NetHackScore-v0`，底层为 NetHack 3.6.7。每个 Policy 会看到 21 × 79 的终端地图、
具名状态值、当前消息、背包条目与输入模式，再从 23 个公开 NLE Actions 中选择
动作。学习结果保存在可执行源代码中，包括地图记忆、移动规则、障碍处理、状态
估计和探索策略，而不是变化的模型权重。

很多失败并不会表现为程序崩溃。Policy 可能连续数千步撞击同一块巨石，反复尝试
穿过铁栏，在两个格子之间振荡，或者站在楼梯上却不知道如何使用它。本实验检验
Coding Agent 能否将这些执行证据转化为更好的策略系统。

## Environment 如何计算反馈分数

Environment 使用 NLE 的 shaped score reward。每个 Action 的 reward 由 NetHack
游戏分数的变化量构成；如果 NetHack 的 turn 计数没有前进，则额外加入固定的
-0.01 冻结步惩罚。一个 Episode 的 return 是全部 step rewards 之和：

```text
Episode return = Σ（游戏分数变化量 + 冻结步惩罚）
冻结步惩罚 = NetHack turn 未前进时为 -0.01，否则为 0
Benchmark score = Episode return 的平均值
```

如果 Policy 执行失败，该 Episode 的 return 记为 -5,000，即 Episode horizon 的
负值。Feedback 还会报告游戏分数、地下城深度、Episode 长度、死亡、截断、Policy
failure 和冻结步比例。这些诊断信息可以区分真正的地下城推进，以及只是不断发出
Actions、却没有推动游戏进程的 Policy。

## 实验协议

三个 GPT-5.6 模型变体通过 Codex 在同一套 Benchmark 配置下运行。实验关闭了可选的
NetHack 优化 Skill。

| 设置 | 值 |
| --- | --- |
| Benchmark | `nle/NetHackScore-v0/mean-return-v1` |
| Runtime | NLE 1.3.0 · NetHack 3.6.7 · Python 3.12 |
| 角色 | 中立、人类、男性 Monk |
| Episode horizon | 5,000 个 Policy steps |
| 训练额度 | 最多 128 Episodes |
| Submission 额度 | 最多 32 次 submissions · 每次最多 64 Episodes |
| Validation | 最多 3 个 candidates · 每个 64 个私有 Episodes |
| Assessment | 256 个 held-out Episodes |
| 实验 seed | `20260801` |
| 可复现性 | uv 0.11.16 与已提交的 lockfile |
| Environment 可视化 | 关闭 |

训练是唯一的优化阶段。Agent 调用 `finish` 后，Host 在完全相同的私有 Validation
Episodes 上评估交接的 candidates，选择其中一个，再在不相交的 held-out
Assessment split 上进行测量。Validation 与 Assessment 只保留聚合结果，不会重新
打开 Agent Feedback 循环。

> 训练额度是上限，不要求必须用完。Sol 使用了全部 128 个训练 Episodes；Terra
> 使用 68 个，Luna 使用 40 个，随后主动结束。因此这些结果并不是严格资源匹配的
> 模型排行榜。

## Held-out 实验结果

主分数是在 256 个 Assessment Episodes 上测得的上述 NLE shaped return 均值。

| Agent 路线 | 使用的训练额度 | Submissions | Assessment | 平均游戏分数 | 平均 / 最大深度 | 冻结步 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| **GPT-5.6 Sol + Codex** | 128 / 128 | 8 | **204.026** | 208.230 | 2.867 / **11** | **27.53%** |
| **GPT-5.6 Terra + Codex** | 68 / 128 | 4 | **80.237** | 87.094 | 1.082 / 4 | 33.77% |
| **GPT-5.6 Luna + Codex** | 40 / 128 | 4 | **63.773** | 70.777 | 1.094 / 4 | 35.75% |

三个最终选中的 Policy 都以零 Policy failure 完成了 Assessment，但没有一个成功
ascend。每个 Episode 都以死亡或达到 5,000-step horizon 结束，因此这些测量描述的
是前期生存、探索和推进，而不是对 NetHack 的完整掌握。Sol 在这些 Runs 中最强，
但这一结果还不足以形成对三个模型的普遍结论。

## 一次完整的 Policy 执行

![Sol 最终选择的 Policy 在一个 NetHack 训练 Episode 中的完整语义回放。回放覆盖
全部 1,269 个 Policy steps，到达地下城第 11 层，最高游戏分数为 860，最终以死亡
结束。](/images/blog/nle-sol-policy-training-replay.gif)

*第 000008 次 Submission 的第 16 个训练 Episode。这段完整回放展示了 Agent 编写的
Policy 如何从第一次 observation 开始自主行动，直至 Episode 结束。它是一个具有
代表性的训练轨迹，不属于上方的 held-out Assessment 结果。*

## 从行为循环到地下城推进

**Luna——障碍记忆。** Luna 发现一个 baseline Episode 几乎一直在撞击巨石，另一个
Episode 尝试穿越铁栏 1,470 次。它加入了失败方向的局部记忆，并结合消息与位置
变化，避免反复尝试静态障碍。

**Terra——私有选择保留了较早版本。** Terra 引入 anti-loop 记忆与下楼逻辑。它在
私有 Validation 上最强的 candidate 是一个只使用了 12 个 Episodes 的早期版本，
说明最后编辑的 Program 不应该自动成为最终结果。

**Sol——空间记忆与推进目标。** Sol 加入记忆地图寻路和显式楼梯使用，随后修复
blocked edges、peaceful monster 交互、过期楼梯目标、mimic 混淆和重复 thump 循环。
其 held-out 深度和冻结步统计体现了这种结构性变化。

每个想法都必须离开 Agent transcript，最终存在于能够独立接收 observations 并返回
Actions 的代码中。Program 才是 Run 的持久结果。

## 发现与边界

- Return、游戏分数、深度、冻结比例、死亡与截断解释行为的不同方面；主分数负责
  排序 candidates，诊断信息负责解释结果。
- 有界的公开训练批次存在噪声，因此私有 Validation 很重要。
- 在最多 128 个 training Episodes 的短训练额度下，Coding Agent 搜索可能因随机性
  产生分数波动。在相同 Environment 配置下，另一次 Terra Run 得分为 49.464，
  而本次为 80.237。

该 distribution 固定 simulator 版本，按 split 确定性派生隐藏 seeds，为每个
Episode 创建全新 Environment，关闭 bones files、ttyrec capture、saved games 和
anti-TAS reseeding，并让 RNG seeds 与特权 NLE 状态保持在 Policy 接口之外。

这只是一个 Benchmark 配置下、每条模型路线各运行一次的初始实验。在形成强模型
比较结论之前，还需要重复 Runs、多个 Host seeds，以及强制相同的 budget 消耗。

## 代码与说明

- [NLE NetHack Benchmark](https://github.com/Linzwcs/EvoPolicyGym/tree/main/environments/nle/nethack)
- [Evaluation 与 Runs](/docs/evaluation/)
- [Policy 边界](/docs/policy/)
