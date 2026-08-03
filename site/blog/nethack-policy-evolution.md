---
locale: en
page: nethack-policy-evolution
title: "Into the dungeon: autonomous Policy evolution in NetHack"
description: "How coding agents used bounded raw NLE trajectories to diagnose behavior, rewrite executable Policies, and face held-out NetHack assessment."
lead: "Coding agents turned complete Policy-visible NetHack trajectories into executable strategies for navigation, obstacle handling, and dungeon progress."
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

We gave coding agents a NetHack Environment, an executable baseline, and
complete Policy-visible raw trajectories. Their task was not to describe a
strategy, but to write one that could survive held-out evaluation on its own.

<!-- truncate -->

## NetHack through NLE

NetHack is a classic terminal roguelike. The player explores a procedurally
generated dungeon, fights or avoids monsters, survives hunger and traps,
manages equipment and consumables, and searches for routes to deeper levels.
The long-term objective is to recover the Amulet of Yendor and ascend, but even
early progress requires interpreting terse messages and acting under partial
observability across a long sequence of decisions.

The NetHack Learning Environment (NLE) exposes that game as a Python interface
for agent research. This independently installable distribution integrates NLE
1.3.0 `NetHackScore-v0`, backed by NetHack 3.6.7. Every Policy sees a 21 × 79
terminal map, named status values, the current message, inventory entries, and
input mode, then chooses among 23 public NLE Actions. Learning is retained as
executable source code—map memory, movement rules, obstacle handling, state
estimation, and exploration strategy—not as changing model weights.

Many failures are not clean crashes. A Policy may walk into one boulder for
thousands of steps, repeatedly try to cross iron bars, oscillate between two
tiles, or stand on a staircase without using it. The experiment tests whether
a coding agent can turn that execution evidence into a better strategy system.

## How the Environment scores feedback

The Environment uses NLE's shaped score reward. Each Action contributes the
change in NetHack game score and, when the NetHack turn counter does not
advance, a constant frozen-step penalty of -0.01. The Episode return is the sum
of those step rewards:

```text
Episode return = Σ (game-score delta + frozen-step penalty)
frozen-step penalty = -0.01 when the NetHack turn does not advance, else 0
Benchmark score = mean Episode return
```

A Policy execution failure receives a return of -5,000, the negative Episode
horizon. Feedback also reports game score, dungeon depth, Episode length,
deaths, truncations, Policy failures, and the frozen-step fraction. These
diagnostics distinguish genuine dungeon progress from a Policy that merely
keeps issuing Actions without advancing the game.

## Experiment protocol

Three GPT-5.6 model variants ran through Codex under the same Benchmark
configuration. The optional NetHack optimization Skill was disabled.

| Setting | Value |
| --- | --- |
| Benchmark | `nle/NetHackScore-v0/mean-return-v1` |
| Runtime | NLE 1.3.0 · NetHack 3.6.7 · Python 3.12 |
| Character | neutral human male Monk |
| Episode horizon | 5,000 Policy steps |
| Training allowance | up to 128 Episodes |
| Submission allowance | up to 32 submissions · 64 Episodes each |
| Validation | up to 3 candidates · 64 private Episodes each |
| Assessment | 256 held-out Episodes |
| Experiment seed | `20260801` |
| Reproducibility | uv 0.11.16 and a committed lockfile |
| Environment visualization | disabled |

Training is the only optimization phase. After the Agent calls `finish`, the
Host evaluates handed-off candidates on identical private Validation Episodes,
selects one, and measures it on a disjoint held-out Assessment split.
Validation and Assessment retain aggregate results and never reopen the Agent
feedback loop.

> The training allowance is a ceiling, not forced consumption. Sol used all
> 128 training Episodes; Terra used 68 and Luna used 40 before voluntarily
> finishing. These results are not a strictly resource-matched model leaderboard.

## Held-out results

The primary score is the mean shaped NLE return defined above, measured over
256 Assessment Episodes.

| Agent lane | Training used | Submissions | Assessment | Mean game score | Mean / max depth | Frozen steps |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| **GPT-5.6 Sol + Codex** | 128 / 128 | 8 | **204.026** | 208.230 | 2.867 / **11** | **27.53%** |
| **GPT-5.6 Terra + Codex** | 68 / 128 | 4 | **80.237** | 87.094 | 1.082 / 4 | 33.77% |
| **GPT-5.6 Luna + Codex** | 40 / 128 | 4 | **63.773** | 70.777 | 1.094 / 4 | 35.75% |

All three selected Policies completed Assessment with zero Policy failures;
none ascended. Every Episode ended in death or at the 5,000-step horizon, so
the measurements describe early-game survival, exploration, and progress—not
complete NetHack mastery. Sol was strongest in these Runs, but the result is
not yet a general claim about the three models.

## One complete Policy Run

![Complete semantic replay of a NetHack training Episode from Sol's selected
Policy. The replay covers all 1,269 Policy steps, reaches dungeon depth 11 and
a maximum game score of 860, and ends in death.](/images/blog/nle-sol-policy-training-replay.gif)

*Submission 000008, training Episode 16. This complete replay shows an
Agent-written Policy acting autonomously from the first observation to the end
of the Episode. It is a representative training trajectory, not part of the
held-out Assessment results above.*

## From stuck loops to dungeon progress

**Luna — obstacle memory.** Luna found a baseline Episode dominated by walking
into a boulder and another with 1,470 attempts to cross iron bars. It added
local memory for failed directions and used messages and position changes to
avoid retrying static obstacles.

**Terra — private selection preserved an earlier revision.** Terra introduced
anti-loop memory and stair descent. Its strongest private Validation candidate
was an early 12-Episode revision, showing why the last edited Program should
not automatically become the final result.

**Sol — spatial memory and a progression objective.** Sol added remembered-map
routing and explicit stair use, then repaired blocked edges, peaceful-monster
interactions, stale stair targets, mimic confusion, and repeated thump loops.
Its held-out depth and frozen-step statistics reflect that structural change.

Each idea had to leave the Agent transcript and survive in code that
independently receives observations and returns Actions. The Program is the
durable result of the Run.

## Findings and boundaries

- Return, game score, depth, frozen fraction, deaths, and truncations explain
  different aspects of behavior; the primary score ranks candidates while the
  diagnostics explain them.
- Private Validation matters because bounded public training batches are noisy.
- With a short training allowance of at most 128 Episodes, coding-agent search
  may produce score variation because of randomness. A separate Terra Run
  under the same Environment configuration scored 49.464, compared with
  80.237 in this Run.

The distribution pins the simulator version, derives split-scoped hidden seeds
deterministically, gives every Episode a fresh Environment, disables bones
files, ttyrec capture, saved games, and anti-TAS reseeding, and keeps RNG seeds
and privileged NLE state outside the Policy interface.

This is an initial experiment with one Benchmark configuration and one primary
Run per model lane. Repeated Runs, multiple Host seeds, and enforced equal
budget consumption are needed before making strong model comparisons.

## Code and notes

- [NLE NetHack Benchmark](https://github.com/Linzwcs/EvoPolicyGym/tree/main/environments/nle/nethack)
- [Evaluation and Runs](/docs/evaluation/)
- [Policy boundary](/docs/policy/)
