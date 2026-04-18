---
name: timebox
description: Sets up a focused work session with named time blocks and push notification warnings. Schedules cron-based alerts for task warnings, transitions, and session end. Pass total duration and subtask allocations as arguments.
argument-hint: "[total duration] [task1_name:duration] [task2_name:duration] ..."
---

# Timebox

Launch a focused work session with named time blocks. Calculates and schedules push notifications for warnings and transitions based on the allocations you provide.

## Arguments

<timebox_input> #$ARGUMENTS </timebox_input>

**Format:** `[total duration] [task:duration] [task:duration] ...`

**Examples:**
- `1h discovery:15m work:30m review:15m`
- `2h research:30m design:45m build:30m testing:15m`
- `45m planning:10m coding:25m review:10m`

Durations can be expressed as:
- `30m` → 30 minutes
- `1h` → 60 minutes
- `1h30m` → 90 minutes

---

## Step 1 — Parse Arguments

Parse the input from `#$ARGUMENTS`:

- **Total duration**: the first token (e.g. `1h`, `45m`, `2h30m`)
- **Subtasks**: each subsequent `name:duration` pair (e.g. `discovery:15m`)

**If no arguments are provided**, use AskUserQuestion to gather:
1. Total session duration
2. Subtask names and durations — ask the user to list them as `task:duration` pairs

**Validate before proceeding:**
- Subtask durations must sum to ≤ total session duration. If they exceed it, tell the user and ask them to adjust.
- If subtask durations sum to less than total duration, tell the user there is [X] minutes unallocated and use AskUserQuestion to ask what they'd like to do:
  - **Add an unallocated buffer** — keep the total as-is; the session end notification fires at the total duration mark after the last task
  - **Extend the last task** — stretch the last subtask to fill the gap
  - **Add a new task** — let me name a new task to fill the remaining time
  - **Shrink the total** — reduce the total session duration to match the subtask sum
- If a subtask's duration ≤ its own warning lead time (e.g. a 2-minute task with a 3-minute warning), skip the warning for that task and only send the transition notification.

---

## Step 2 — Apply Warning Rules

For each subtask, determine the warning lead time based on its duration:

| Subtask duration | Warning fires |
|-----------------|---------------|
| < 5 minutes     | 1 min before end |
| 5 – 15 minutes  | 3 min before end |
| 16 – 45 minutes | 5 min before end |
| ≥ 60 minutes    | 10 min before end |

---

## Step 3 — Build the Schedule

Get the current time using the Bash tool:

```bash
date "+%M %H %d %m"
```

This gives you `minute hour day-of-month month` in local time — the fields you need for one-shot cron expressions.

Then walk through each subtask in sequence, computing the absolute clock time for:
1. Its **warning** notification (end_time minus lead time)
2. Its **transition** notification (when it ends / next task begins)

If subtask durations leave a gap before the total session end, schedule one additional **Time's Up** notification at the total session end.

**Before scheduling anything**, show the user a confirmation table:

```
Timebox: [total duration] — [HH:MM] to [HH:MM]

Task        Duration   Warning     Ends
----------  ---------  ----------  -----
Discovery   15m        HH:MM       HH:MM
Work        30m        HH:MM       HH:MM
Review      15m        HH:MM       HH:MM
```

Ask for confirmation with AskUserQuestion before proceeding to schedule:

**Option A — Looks good, start it (Recommended)**
**Option B — Let me adjust the allocations**

**Recommendation & Reason:** Start it is recommended because the table gives you a full preview — if anything looks off, you can catch it now and choose to adjust.

If the user selects "adjust", restart from Step 1.

---

## Step 4 — Schedule All Notifications

For each notification, call CronCreate with:
- `recurring: false` (one-shot — fires once, then auto-deletes)
- `cron`: a 5-field expression pinning all fields: `"[minute] [hour] [dom] [month] *"`
- `prompt`: an instruction to run an osascript notification AND send a PushNotification with the appropriate message

**Notification message formats:**

| Event | Message |
|-------|---------|
| Warning (mid-session task) | `⏰ [Task]: [X]min remaining — start wrapping up` |
| Transition (not last task) | `⏩ [Task] done → [Next task] ([duration]) starting now` |
| Last task transition / Time's Up | `🏁 Time's up! [Total duration] session complete` |

**Cron prompt template for each notification:**
```
Run this bash command: osascript -e 'display notification "[message text]" with title "Timebox" sound name "Glass"'
Then send a PushNotification with this exact message: "[message text]"
```

Always run the Bash/osascript command first — it fires regardless of terminal focus and is the guaranteed delivery path. PushNotification is secondary (mobile only, requires Remote Control).

Schedule all crons in parallel — make all CronCreate calls in a single message to maximize efficiency.

**Computing cron minute/hour:** When adding minutes to a time causes the minute to exceed 59, carry over to the hour. Example: 14:50 + 15 min = 15:05 → cron `"5 15 [dom] [month] *"`. Always keep hour in 0–23 range (mod 24 if session crosses midnight).

---

## Step 5 — Confirm to the User

After all crons are created, confirm:

```
Timebox is running. [N] notifications scheduled.

[Repeat the schedule table with warning and end times]

Stay focused — I'll notify you at each transition.
```

List the job IDs returned by CronCreate in case the user wants to cancel early (tell them to say "cancel my timebox" or provide the IDs for CronDelete).

---

## Handling "Cancel My Timebox"

If the user asks to cancel the timebox mid-session:
- Call CronDelete for each scheduled job ID from this session.
- Confirm: "Timebox cancelled — [N] notifications removed."

---

## Rules

- Never skip showing the confirmation table before scheduling.
- Always use `recurring: false` — these are one-shot timers, not recurring jobs.
- If a session crosses midnight, handle hour rollover correctly (mod 24).
- Do not add a warning notification if the warning would fire at or before the task starts.
- Keep notification messages concise — under 200 characters.
