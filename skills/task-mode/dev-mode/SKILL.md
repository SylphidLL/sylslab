---
name: dev-mode
description: "Use when each step needs user confirmation before proceeding. Agent focuses on one step per turn, reports progress with full context overview, then stops and asks '继续？' before the next step."
version: 2.0.0
description: "Use when each step needs user confirmation and full file diff review. Agent focuses on one step per turn, reports progress and shows modified file contents, then stops and asks '继续？' before the next step."

metadata:
  hermes:
    tags: [task-mode, step-by-step, manual, interactive, dev]
    related_skills: [task-mode-selector, cruiser-mode, long-task-mode]
---

# Dev Mode (开发者模式)

## Overview

One step per turn. Before executing, show the full task context so the user knows where we are. Execute the step. Report results and show the modified file contents. Then stop and ask "继续？" before proceeding to the next step.

## When to Use

- User wants to review each step's output before proceeding
- Exploratory/debugging tasks where the next step depends on current output
- High-risk operations where user oversight is mandatory
- **This is the default mode** if no mode is selected at session start

## Mode Entry (first turn after selection)

When entering dev mode (via selector or mode switch), show a brief intro:

```
💻 开发者模式
接下来每做一步我会报告进度、展示修改后的文件内容，等你确认后再继续。
当前任务：<任务名>
```

## How It Works

```
Turn 1: 
  📋 任务总览 (M 步)
    已完成: ✅ 步A  ✅ 步B
    当前:   🔄 第 X/M 步：步C
    待完成: ○ 步D  ○ 步E
  
  执行步C → 报告结果 → "继续？"
  
  ↑                                    ↑
  用户看到全景                    我停下来等你

Turn 2:
  📋 任务总览 (M 步)
    已完成: ✅ 步A  ✅ 步B  ✅ 步C
    当前:   🔄 第 X+1/M 步：步D
    待完成: ○ 步E
  
  执行步D → 报告结果 → "继续？"

Turn N:
  全部完成 → "🎉 全部完成！"
```

## Output Format

```
📋 <任务名>（共 M 步）
  已完成: ✅ <已完成步骤列表>
  当前:   🔄 第 X/M 步：<当前步骤名>
  待完成: ○ <待完成步骤列表>

<当前步骤执行结果>
  - 改了哪些文件 / 运行了什么 / 结果如何
  - 关键输出（1-2行）

📄 修改后的文件内容（完整或关键部分）：
  ┌─ path/to/file.py ─────────────────────
  │ def hello():
  │     print("Hello, World!")
  │     return True
  └───────────────────────────────────────

进度: X/M
继续？
```

## Output Limit Awareness

Dev mode reuses **cruiser mode's calibration data** (`cruiser-calibrate.py`) for output estimation, but handles it differently:

### Shared with Cruiser
- Auto-calibrate on mode entry (3-day cache / model change detection)
- Token density data (Chinese/English/code ratios)
- Real-time mental tracking: tool_count, total_output, total_text
- Usage logging to `usage-log.json` (per-step stats accumulation)

### Different Behavior

| Situation | Cruiser | Dev |
|-----------|---------|-----|
| Step output < soft_stop | Auto-advance | Show result + ask "继续？" |
| Step output >= soft_stop | Stop + checkpoint | Show result + **warn** + ask "继续？" |
| Step truncated (8K) | Record to calibration | Record to calibration |

### Soft Stop Warning Format

If a single step produces output approaching soft_stop (~6963 tokens):

```
⚠️ 本轮输出较大（~6500 / 6963 tokens），接近上限
结果已完整展示，可正常确认。

后续步骤如果也这么大，建议切换到巡航者或长任务模式。
```

### After Confirmation

- "继续" → proceed to next step normally
- "切换到巡航者" → switch modes if the task is better suited for phased execution

## Key Rules

1. **Always show task overview** — every turn shows completed/current/pending so the user never loses the big picture
2. **One step per turn** — focus on the current step only. Do NOT pre-execute future steps
3. **Multiple independent tool calls per step allowed** — if a step needs reading a file, modifying it, and running a test, those are fine in one turn
4. **Report briefly but informatively** — 3-5 lines: what changed, key outputs, any issues
5. **Always end with "继续？"** — never assume the answer
6. **When all steps done** — "🎉 全部完成！" + brief summary of what was accomplished

## Anti-Patterns

- ❌ 做完步骤1，没问你继续，直接做步骤2
- ❌ 做完步骤1只说了"完成"，没告诉你改了啥
- ❌ 展示任务总览时用太多行（步骤列表超长时折叠）
- ❌ 预执行未来步骤（"做了步骤1，顺便也把步骤2做了"）
- ❌ 长篇大论的分析（把分析放在工具调用里，报告只给结论）

## Verification

- [ ] Task overview shown at start of each turn
- [ ] One conceptual step per turn
- [ ] Results reported (what changed, key outputs)
- [ ] Explicit "继续？" at end
- [ ] User triggered next turn
