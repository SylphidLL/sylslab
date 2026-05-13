---
name: task-mode-selector
description: "Use at the START of every new session. Presents the three task modes to the user, lets them pick one, and can switch modes mid-session on request. The agent may also suggest a mode switch if the task nature changes."
version: 2.0.0
author: Sylphid
license: MIT
metadata:
  hermes:
    tags: [task-mode, selector, switching, workflow]
    related_skills: [long-task-mode, cruiser-mode, dev-mode]
---

# Task Mode Selector (任务模式选择器)

## Overview

Every new session begins with a mode selection. The mode controls how the agent interacts with you — whether it waits after every step, pauses at checkpoints, or runs autonomously. You can switch modes mid-session any time by saying "切换模式" or "切换到 XX 模式". The agent may also proactively suggest a mode switch if the task nature warrants it.

Mode is **session-level**, not task-level — it governs how we communicate, not what work gets done.

## Three Modes

| Mode | User Involvement | Risk | Use Case |
|------|-----------------|------|----------|
| 🏃 **长任务** (long) | 无需在场，自动桥接 | 低 | 超长任务、多会话自动执行 |
| 👁️ **巡航者** (cruiser) | 每阶段审核 | 中 | 分阶段任务，需中间审查 |
| 💻 **开发者** (dev) | 每步确认+文件展示 | 高 | 开发调试、需逐行审查代码 |

## Trigger Points

### 1. New Session Start (automatic prompt)

The first time I speak to you in a new session (new `hermes` invocation, `/new`, or your first message after startup):

```
请选择任务模式:
  [1] 🏃 长任务模式 — 超长任务自动桥接，无需你在场
  [2] 👁️ 巡航者模式 — 分阶段执行，每阶段做完暂停审核
  [3] 💻 开发者模式 — 每步做完展示文件内容，等你确认

(10秒内未选择则默认使用精确任务模式)
```

**If no mode is selected within ~10 seconds:**
- If your task/request clearly fits one mode → auto-select that mode and proceed
- If mode choice is ambiguous → default to **开发者模式** (safest: shows everything, waits for confirmation)

### 2. Suggested Switch (agent-initiated)

If the agent detects that a different mode would suit the current task better:

```
这个任务比较复杂，需要多个步骤。
要不要切换到巡航者模式？这样阶段内我自动走，到阶段边界再停。
还是保持当前模式？
```

Only suggest — never auto-switch. If user says no, stay on current mode.

### 3. Manual Switch (user-initiated)

You say "切换模式", "切换到巡航者", "换长任务模式", etc.

Response format:
```
好的，切换到巡航者模式。接下来分阶段执行，每阶段做完我暂停让你审核。
继续做当前步骤。
```

### 3. Agent Suggestion (proactive)

If the agent detects that the current session's task nature has changed significantly and a different mode would suit it better:

```
这个任务周期比较长，后面步骤我可以自动跑完。
要不要切换到长任务模式？还是保持当前模式？
```

Only suggest — never auto-switch. If user says no, stay on current mode.

## What Switching Does NOT Do

Switching mode does NOT:
- Save or compress progress (that's `context-monitor-and-compress`'s job)
- Write to `history/` (only compression writes there)
- Write to `memory/`
- Update `task.json` (mode is session-level, not task-level)

The only thing that happens is: the agent changes its interaction pattern for subsequent turns.

## Key Rules

1. **Prompt at new session start** — always. Three options, default to precise after 10s timeout.
2. **Switch immediately on explicit request** — no confirmation needed.
3. **Only suggest, never auto-switch** — user has final say.
4. **No persistence** — mode is a session-level preference. Next session asks again.
5. **Switch is instant** — no file writes, no compression, no delay.

## Mode Behavior Matrix

|| Situation | 长任务 | 巡航者 | 开发者 |
||-----------|--------|--------|---------|
|| 完成一步 | 自动下一步 | 自动下一步 | 展示文件+等确认 |
|| 接近回复上限 | 自动压缩+桥接下一会话 | 收尾当前阶段+等审核 | 不可能发生（每步很短） |
|| 完成任务 | 自动通知+摘要 | 展示结果+等确认 | 展示结果 |
|| 用户不在 | 正常运行 | 暂停（等阶段结束） | 暂停（等确认） |

## Common Pitfalls

1. **Confusing mode with task state.** Mode is how we talk, not what we're doing. Don't write mode to `task.json.steps` or `memory/`.
2. **Over-suggesting.** Don't suggest a mode switch more than once per major phase change. Nobody wants "要不要切换模式?" every 3 turns.
3. **Auto-switching.** Never auto-switch modes. Always ask.
4. **Saving progress on switch.** Mode switch is instant — no progress save needed.
