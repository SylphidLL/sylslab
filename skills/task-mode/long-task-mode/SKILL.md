---
name: long-task-mode
description: "Use for extremely long tasks that exceed a single session's context. The agent executes phases autonomously, auto-compresses and bridges to a new session when nearing output limits, and continues until all phases are complete. No user presence required."
version: 3.0.0
author: Sylphid
license: MIT
metadata:
  hermes:
    tags: [task-mode, autonomous, auto-bridge, long-running, self-calibrating]
    related_skills: [cruiser-mode, dev-mode, task-mode-selector, context-monitor-and-compress]
---

# Long Task Mode (长任务模式)

## Overview

The task runs **fully autonomously** across multiple sessions. When nearing output limits or completing a phase, the agent automatically compresses the conversation to `history/`, creates a one-shot cron job, bridges to a new session, and continues — all without you.

**Key distinction**: this mode solves the **multi-session problem**. Unlike **cruiser mode** (which pauses at phase boundaries for your review) and **precise mode** (which waits after every step), long mode bridges itself. You only hear from it when the entire task is done.

## Relationship with Cruiser Mode

Long mode **reuses** cruiser mode's:
- Output estimation (heuristic formula with calibration data)
- Real-time usage tracking (tool_count, total_output, total_text)
- Self-calibrating thresholds (soft_stop, truncation recording)
- Token density calibration (Chinese/English/code ratios)

**Difference**: cruiser waits for your input at checkpoints; long mode automatically bridges to a new session.

## Mode Entry (first turn after selection)

When long task mode is activated (via selector or manual switch):

1. **Auto-calibrate** — run `python3 cruiser-calibrate.py calibrate` in background
   - If last calibration was < 3 days ago and model hasn't changed: skip, use cached data
   - If model changed since last calibration: force re-calibrate
2. **Load calibration data** — `python3 cruiser-calibrate.py show`
3. **Show intro**

```
🏃 长任务模式
自动校准中...（检查输出上限）
任务会在后台自动执行，自动桥接上下文直到全部完成。
完成后通知你。
```

Calibration runs async — execution starts immediately with cached data if available.

## Auto-Bridge Lifecycle

```
你设定目标 → 我拆成 N 个阶段 → 出蓝图 → 你确认 ✅
                                          ↓
                              ┌──────────────────────────────┐
                              │     每轮执行                    │
                              │  如果是第一轮 → 直接执行步骤   │
                              │  如果是桥接恢复 → 读 latest    │
                              │  _history 恢复上下文           │
                              │  阶段内自动执行步骤             │
                              │  脑内统计                      │
                              │  检查桥接条件                   │
                              │                                │
                              │  桥接条件（任一触发）：          │
                              │  ① 当前阶段完成                 │
                              │  ② 输出估算达 soft_stop        │
                              └──────┬─────────────────────────┘
                                     ↓ 触发桥接
                              ┌──────────────────────────────┐
                              │     桥接流程                    │
                              │  a. 压缩到 history/           │
                              │  b. 更新 task.json            │
                              │  c. 创建一次性 cron job (15s) │
                              │  d. 当前轮结束 🔚              │
                              └──────────────────────────────┘
                                     ↓ 15秒后 cron 触发
                              ┌──────────────────────────────┐
                              │     新会话自动启动              │
                              │  e. 加载 bridge skill          │
                              │  f. 读取 latest_history        │
                              │  g. 继续下一步骤                │
                              │  h. 回到每轮执行 🔄             │
                              └──────────────────────────────┘
                                     ↓ 所有阶段完成
                              🏃 长任务完成！通知你
```

## Task Blueprint (required before execution)

Before any long task starts, the agent produces a **task blueprint** listing phases and steps.
Bridge count estimation is based on historical data from `usage-log.json` — no preset formula.
If no similar task exists in history, bridge count is marked as unknown and will auto-calibrate after first execution.

### Bridge Estimation Logic

```
出蓝图时：
  1. 检查 usage-log.json 有没有同类任务的历史数据
  2. 如果有 → 参考同类步骤的 output_size 均值估算总输出
  3. 如果总输出 < soft_stop → 标注"无需桥接，一轮完成"
  4. 如果总输出 > soft_stop → 标注"预计 N 次桥接"
  5. 如果没有历史数据 → 标注"桥接次数未知，首次执行后自动校准"

执行后：
  记录实际输出到 usage-log.json，下次同类任务可参考
```

### Blueprint Format

```
🏃 长任务计划：<任务名>

阶段1：<阶段名>
  1. <步骤描述>
  2. <步骤描述>
  ...

阶段2：<阶段名>
  3. <步骤描述>
  ...

...

桥接预估: ~N 次（基于 <N> 条历史记录）| 未知（无历史数据）
总输出预估: ~X tokens（如有历史数据）

确认执行？"确认"或"修改"后重新展示。
```

### Example Blueprint (with history)

```
🏃 长任务计划：代码质量扫描

阶段1：扫描
  1. 扫描 scripts/ 目录下所有 .py 文件
  2. 统计代码行数

阶段2：分析
  3. 检查常见问题

阶段3：报告
  4. 生成质量报告

桥接预估: 无需桥接，一轮完成
  (参考历史: 代码扫描类任务 ≈ 2300 tokens，低于 soft_stop 6963)
```

### Example Blueprint (no history)

```
🏃 长任务计划：全站数据迁移

阶段1：数据导出
  1. 连接源数据库
  2. 导出用户表（50K 行）

阶段2：数据导入
  3. 连接目标数据库
  4. 分批导入

桥接预估: 未知（无历史数据）
  首次执行后将自动校准。
```

### After Confirmation

- User says "确认" → start auto-bridge lifecycle
- User says "修改" → adjust blueprint in-place based on feedback, re-display same turn
- User says "取消" → cancel long task, switch mode or do something else

## Bridge Format (auto-compressed history file)

Location: `tasks/<task_id>/history/context-<长任务名>-YYYY-MM-DD-N.json`

```json
{
  "version": "长任务名-2026-05-13-1",
  "task_id": "my-task",
  "task_name": "数据迁移",
  "mode": "long",
  "bridge_id": 1,
  "total_bridges": 5,
  "progress": {
    "completed": ["阶段1：数据导出", "阶段2：格式转换"],
    "current": "阶段3：数据导入",
    "pending": ["阶段4：数据校验", "阶段5：报告生成"]
  },
  "stats": {
    "tool_count": 12,
    "total_output_bytes": 45000,
    "total_text_chars": 3200,
    "estimated_tokens": 6500
  },
  "decisions": [],
  "files_changed": ["..."],
  "action_required": "继续阶段3：数据导入"
}
```

The `bridge_id` / `total_bridges` fields track overall progress across all bridges.

## Cron Job for Bridging

At each bridge point, create a one-shot cron job:

```python
cronjob(
  action='create',
  name='长任务桥接-<任务名>',
  schedule='15s',
  repeat=1,                    # run once
  prompt=(
    '我是长任务桥接器。读取 tasks/<task_id>/history/'
    'context-<长任务名>-最新版本.json，恢复上下文，'
    '继续执行长任务。完成后自动桥接或通知用户。'
  ),
  skills=['sylphid-skill-bridge', 'long-task-mode'],
  enabled_toolsets=['terminal', 'file', 'skills', 'session_search', 'memory']
)
```

Only one cron job exists at a time — the next one is created after the previous completes.

## Real-Time Usage Tracking

Same as cruiser mode:

```
本轮脑内统计：
  tool_count = 0
  total_output = 0
  total_text = 0

每调用一个工具：
  tool_count += 1
  total_output += 工具返回数据大小（bytes）
  total_text += 写的文字（chars）

到桥接点时：
  python3 cruiser-calibrate.py log-step <task_id> <step> <tool_count> <total_output> <total_text>
  
  如果被截断：
  python3 cruiser-calibrate.py record-truncation <estimated> <tool_count> <total_text>
```

## Output Limit Estimation (same heuristic as cruiser)

```
estimated_tokens = chinese_chars / chinese_ratio
                 + english_chars / english_ratio
                 + code_chars / code_ratio
                 + tool_calls * tool_call_overhead
                 + tool_results * tool_result_overhead
```

At `estimated_tokens >= soft_stop`: finish current step, trigger bridge.

## Cancellation (中途终止)

If you want to stop a running long task mid-execution:

```
你说"终止xxx任务"或"取消长任务" →
  1. 设置 task.json.cancel_requested = true
  2. 删除该任务的所有桥接 cron job
  3. 通知你"长任务已终止"
```

On the next bridge session, the cron job checks `task.json.cancel_requested`:
- If `true` → clean up, delete self, notify you task was cancelled
- If `false` → continue as normal

Future enhancement: a visual task dashboard showing running bridges, progress, and cancel buttons.

## Completion Notification

When all phases are done, the last bridge session:

1. **Write to task.json.daily_log** — persistent record you can check later
2. **Send system notification** via terminal:
   ```bash
   # Windows (balloon notification in system tray)
   powershell -Command "Add-Type -AssemblyName System.Windows.Forms; \$n = New-Object System.Windows.Forms.NotifyIcon; \$n.Icon = [System.Drawing.SystemIcons]::Information; \$n.BalloonTipTitle = '🏃 长任务完成'; \$n.BalloonTipText = '<task_name>: N次桥接, 全部完成'; \$n.Visible = \$true; \$n.ShowBalloonTip(5000)"
   
   # Linux/macOS (fallback — runs in cron session, no GUI expected)
   echo "🏃 长任务完成: <task_name>"
   ```
3. **No in-chat delivery** — avoids conflicting with ongoing conversations

Task.json entry:
```json
{
  "date": "2026-05-13",
  "time": "14:35",
  "source": "长任务完成",
  "task_name": "数据迁移",
  "total_bridges": 5,
  "total_steps": "12/12",
  "summary": "数据迁移完成，所有表导入成功"
}
```

You can check completed long tasks anytime:
- "看看有没有长任务完成了" → scan task.json.daily_log for "长任务完成" entries

```
🏃 长任务完成！
  任务: 数据迁移
  总桥接: 5 次
  总耗时: ~45 分钟
  总步骤: 12/12
  改了: [文件列表]
  关键决定: [已记录]
```

## Key Rules

1. **Auto-calibrate on entry** — check model/density before first execution
2. **Borrow cruiser's estimation** — same heuristic, same calibration data
3. **First session: no latest_history** — start from step 1 directly
4. **Bridge automatically** — no user confirmation needed at checkpoints
5. **One cron job at a time** — only create the next bridge after the previous completes
6. **Track bridge count** — `bridge_id` / `total_bridges` in history files
7. **Record usage stats** — log to `usage-log.json` for threshold refinement
8. **Notify only on completion** — intermediate sessions are invisible to user

## Comparison: Long vs Cruiser vs Precise

| Aspect | Precise | Cruiser | Long |
|--------|---------|---------|------|
| Per-step confirm | Yes | No (auto within phase) | No |
| Phase checkpoints | N/A (every step) | Review + ask "继续？" | Auto-bridge (no review) |
| Multi-session | No | Manual (compress + new session) | Automatic (cron bridge) |
| User presence | Required per step | Required per phase | None (notification on completion) |
| Reply limit guard | None (one step) | Self-calibrating estimation | Same heuristic as cruiser |
| Best for | High-risk, debug | Batched, phased work | Massive, multi-session |

## Pitfalls

1. **Cron requires Hermes to be running.** If the CLI is closed, the bridge cron job won't fire. Long tasks are paused until Hermes restarts.
2. **One cron at a time.** Don't create multiple bridge jobs. Wait for the previous one to complete.
3. **Missing calibration.** If cruiser-calibrate.py hasn't been run, soft_stop defaults to 7000. For new models, run calibrate first.
4. **No rollback mid-task.** Unlike cruiser/precise, long mode doesn't support "回滚" — changes are committed per session.
5. **Protect against broken bridges.** If a bridge session writes corrupted files, subsequent sessions compound the damage. Always commit/push before starting a long task. If the task involves code changes, run from a clean git state so rollback is possible via `git reset --hard`.
6. **Long task never notified.** If the final session crashes without notification, the task may appear "incomplete" in task.json. Check `daily_log` for progress.
