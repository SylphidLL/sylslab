---
name: cruiser-mode
description: "Use when a task has multiple steps that can be batched into phases, but each phase needs user review. The agent executes steps within a phase autonomously, monitors reply length via self-calibrating estimation, and stops at a natural phase boundary before hitting the model's output limit."
version: 2.0.0
author: Sylphid
license: MIT
metadata:
  hermes:
    tags: [task-mode, phased, checkpoint, batch, self-calibrating]
    related_skills: [long-task-mode, dev-mode, task-mode-selector]
---

# Cruiser Mode (巡航者模式)

## Overview

The task is organized into **phases** (logical batches of steps). The agent executes steps within a phase autonomously — no need to confirm after every step. But it monitors its own reply length via a self-calibrating estimation system and stops at a **natural phase boundary** before hitting the model's output limit.

**Key distinction**: this mode guards against **per-reply truncation** (single response too long), not context exhaustion (cumulative conversation too long). Context compression is handled separately by `context-monitor-and-compress` in any mode.

## Self-Calibrating Output Limit

Instead of hardcoded thresholds, this mode uses a **calibration script** (`cruiser-calibrate.py`) to probe the actual model limits and continuously refine its estimation.

### Calibration Script

Location: `~/.hermes/tasks/globle/agents/sylphid/scripts/cruiser-calibrate.py`

Commands:
```bash
python3 cruiser-calibrate.py calibrate     # Full calibration
python3 cruiser-calibrate.py show          # Show current calibration data
python3 cruiser-calibrate.py record-truncation <estimated> <tools> <chars>  # Record a truncation event
```

### Data Collected

| Metric | Description | Source |
|--------|-------------|--------|
| `max_output_tokens` | Model's hard output limit | Probe via API |
| `soft_stop` | Safe stop threshold (default: 85% of hard limit) | Auto-calculated |
| `chinese_ratio` | Chinese chars per token | tiktoken estimation |
| `english_ratio` | English chars per token | tiktoken estimation |
| `code_ratio` | Code chars per token | tiktoken estimation |
| `tool_call_overhead` | Tokens per tool call definition | Estimated |
| `tool_result_overhead` | Tokens per tool result | Estimated |
| `truncation_history` | Log of actual truncation events | Recorded on truncation |

### Adaptive Threshold

When a truncation is detected (the reply was cut off mid-step or `finish_reason: "length"`), the agent records the estimated tokens at that point and adjusts the soft_stop threshold downward. Over multiple sessions, the threshold converges to a reliable value.

```
Initial: hard limit = 8192, soft_stop = 6963 (85%)
After truncation at ~7500: soft_stop = 6375 (85% of 7500)
After truncation at ~6800: soft_stop = 5780 (85% of 6800)
... converges to safe value
```

## Output Limit Estimation (Heuristic)

The agent estimates its current reply length during generation using:

```
estimated_tokens = chinese_chars / chinese_ratio
                 + english_chars / english_ratio
                 + code_chars / code_ratio
                 + tool_calls * tool_call_overhead
                 + tool_results * tool_result_overhead
```

Where the ratios come from the calibration script. When `estimated_tokens >= soft_stop`, stop starting new steps and wrap up.

**This is heuristic, not precise.** The calibration data improves over time as truncation events are recorded.

## Real-Time Usage Tracking

In cruiser mode, every tool call is tracked mentally during the reply, and the aggregated data is logged at checkpoints or end-of-round.

### Per-Round Process

```
本轮回合计数组：
  tool_count = 0       ← 本轮调用了几个工具
  total_output = 0     ← 所有工具返回数据的 bytes 之和
  total_text = 0       ← 我写的文字（展示、说明、提问）的 chars 之和

每调用一个工具：
  tool_count += 1
  total_output += 该工具返回的数据大小（bytes）
  total_text += 这次调用前后写的文字（chars）

到检查点或本轮结束时：
  python3 cruiser-calibrate.py log-step <task_id> <step_name> <tool_count> <total_output> <total_text>
  
  如果本轮被截断：
  python3 cruiser-calibrate.py record-truncation <estimated_tokens> <tool_count> <total_text>
  （estimated_tokens 为空时传 0，脚本会用历史数据推算）
```

### Data Accumulation

`<task>/history/usage-log.json` stores all logged rounds. Over time, patterns emerge:

- "平均每个工具调用产生 ~X bytes 输出"
- "平均每轮 ~Y 个工具调用"
- "写 ~Z chars 文字 + ~W bytes 工具输出 ≈ 被截断"

These patterns feed back into the soft_stop threshold without needing exact token counting.

## When to Use

- Tasks with natural phases (e.g., "user module" → "payment module" → "admin panel")
- Multi-step coding tasks where you want to review each batch but don't need per-step confirmation
- Medium-complexity tasks where precise mode would be too slow but long mode is too hands-off

## Mode Entry (first turn after selection)

When cruiser mode is activated (via selector or manual switch):

1. **Auto-calibrate** — run `python3 cruiser-calibrate.py calibrate` in background (takes ~30s)
   - If last calibration was < 3 days ago and model hasn't changed: skip, use cached data
   - If model changed since last calibration: force re-calibrate
2. **Show intro** — immediately display mode intro (calibration can finish in background)
3. **Load calibration data** — `python3 cruiser-calibrate.py show` to confirm limits

```
🚗 巡航者模式
自动校准中...（检查输出上限）
任务会分阶段执行，每阶段做完我会暂停让你审核。
当前任务：<任务名>
```

Calibration runs async — you don't need to wait for it. If calibration hasn't completed before the first tool call, the mode uses the last-known data (or defaults to conservative 7000 soft_stop if no data exists).

## How It Works — Full Flow

### Phase Execution (between checkpoints)

```
📋 <任务名>（共 M 步）
  已完成: ✅ 步骤A  ✅ 步骤B
  当前:   🔄 步骤C
  当前阶段: 用户模块（4步中已做2步）
  输出估算: ~3500 / <soft_stop> ✅

→ 自动继续下一步骤
```

The agent keeps executing steps automatically until one of:

1. **Phase complete** — all steps in the current phase are done → checkpoint
2. **Output near limit** — estimated tokens reaches soft_stop → finish current step → checkpoint
3. **All steps done** — task complete

**If a phase has only 1 step**, behavior is identical to precise mode: the step completes and the agent stops, waiting for your input. The difference only appears in multi-step phases.

### Phase Checkpoint

When a phase is complete (and there are more phases):

```
📋 第一阶段完成 ✅
  阶段: 用户模块
  已完成: 注册接口 / 登录接口 / JWT中间件 / 权限校验（4步）
  改了: api/auth.py, middleware/jwt.py, tests/test_auth.py
  关键决定: RS256 签名，token 24h 有效期
  输出估算: ~4200 tokens

→ 继续到第二阶段（支付模块）？
  → "继续"
  → "换个方向"
  → "回滚到阶段1开始前"
```

### All Phases Complete

When all phases are done (task finished):

```
🎉 任务完成！
  总览: 3个阶段 / 12步全部完成
  改了: api/auth.py, middleware/jwt.py, api/payment.py, tests/
  关键决定: [已记录]
  输出估算: ~5800 tokens

任务已全部完成，无需继续。
```

No "继续？" needed — the task is done.

### Output Limit Checkpoint

When estimated output approaches soft_stop before a phase is complete:

```
⚠️ 输出估算已达 ~6800 / 6963 tokens

已完成当前步骤（用户注册），不再启动新步骤。

📋 阶段内检查点
  当前阶段: 用户模块（4步中已做2步）
  已完成: 注册接口 ✅  登录接口 ✅
  待完成: JWT中间件 ○  权限校验 ○
  改了: api/auth.py

→ 开始新轮继续当前阶段？
  → "继续"
  → "回滚到阶段1开始前"
```

### Truncation Recording

If a reply gets truncated despite the guard:

```
⚠️ 本轮回复被截断。记录截断信息以校准阈值。
已记录: 估算 ~X tokens 时发生截断，软停止阈值已调整为 Y。
```

## Key Rules

1. **Phase-first, not step-first** — organize work into phases. Each phase is a logical batch of steps.
2. **Single-step phase = precise mode** — if a phase has only 1 step, it behaves identically to precise mode (step done, wait for input).
3. **Auto-advance within a phase** — do NOT stop after every step. Keep going until phase done or output near limit.
4. **Estimate output length** — use the calibration formula to track approximate output. At soft_stop, wrap up.
5. **Phase boundaries are natural** — don't split in the middle of a step. Finish the current step, then checkpoint.
6. **Record truncation events** — if truncation happens, log it to the calibration data so the threshold adjusts.
7. **Support "回滚" and "换个方向"** at checkpoint — same as other modes.
8. **Do NOT conflate with context compression** — this mode manages per-reply length. Context compression (>88%) is a separate mechanism.
9. **At output checkpoint: just ask to continue** — don't suggest mode switches at checkpoint. If the user wants to switch, they'll say so.

## Comparison: Cruiser vs Precise vs Long

| Aspect | Precise | Cruiser | Long |
|--------|---------|---------|------|
| Per-step confirm | Yes | No (auto within phase) | No |
| Phase checkpoints | N/A (every step) | Yes (phase end or soft_stop) | Auto-bridge |
| Reply limit guard | None (one step) | Self-calibrating estimation | Same heuristic |
| Output estimation | N/A | Calibration script + truncation history | Same |
| User presence | Required per step | Required per phase | Not required |
| Best for | High-risk, debug | Batched, phased work | Massive, multi-session |

## Pitfalls

1. **Not defining phases.** Without phase boundaries, every checkpoint is just an "output limit" checkpoint — defeats the purpose. Always organize the task into phases first.
2. **Forgetting the soft_stop.** If you don't estimate output length, you'll hit the hard limit mid-step and get truncated. This is the entire point of cruiser mode.
3. **Crossing phase boundaries in one turn.** If phase 1 has 6 steps and phase 2 has 3 steps, don't let them spill into each other. Stop at the phase boundary.
4. **Over-checkpointing.** Don't checkpoint every 3 steps within a phase. Wait for a natural boundary.
5. **Not updating calibration after truncation.** If truncation happens but the threshold isn't updated, it will keep happening. Always record it.
6. **Treating estimation as exact.** The formula is heuristic. A ~20% margin of error is normal. The 15% buffer (soft_stop = 85% of hard limit) absorbs this.
