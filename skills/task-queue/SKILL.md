---
name: task-queue
description: Accept a list of tasks (JSON or natural language), execute shell commands sequentially with fail-fast logic, and report a structured summary.
license: BSD 3-Clause
compatibility: Requires git CLI configured with remote access and PR creation tools (e.g., gh).
metadata:
  agent: coding
---

# Task Queue

You are an autonomous executor. Your job is to accept a task list, execute shell commands sequentially with fail-fast logic, and report a structured summary.

## 1. Ingest & Parse

Accept the task list. It may be provided as:
- **JSON Array:** `[{ "id": "1", "description": "Run lint", "command": "npm run lint" }]`
- **Natural Language:** A numbered or bulleted list.

**Action:** Parse the input into a standardized structure. Ensure every task has:
- `id` — unique string/number
- `description` — human-readable
- `command` — shell command to execute

**Handle edge cases:**
- **Empty task list:** If the input is empty or contains zero tasks, report "No tasks to execute." and stop.
- **Malformed JSON:** If the input is supposed to be JSON but is malformed, report the parse error and stop.
- **Missing fields:** If a task is missing `id`, `description`, or `command`, skip that task and log a warning. Do not abort.
- **Duplicate IDs:** If two tasks have the same `id`, keep the first occurrence and skip duplicates with a warning.

## 2. Execute (Fail-Fast)

Iterate through tasks in order. **Do not pause for user confirmation.**

**Important:** This skill manages its own task tracking. Do not rely on the system prompt's todo management — create and manage a structured todo list for visibility:

```bash
# Before starting, create todo items for each task
# (Create a todo list with each task from the parsed input)
```

For each task:
1. **Run Command:** Execute the `command` using `terminal`. Set a timeout to prevent hanging:
   ```bash
   timeout 300 bash -c "$COMMAND" 2>&1
   ```
   This limits each command to 5 minutes. Adjust as needed.

2. **Capture output:** Always capture both stdout and stderr:
   ```bash
   OUTPUT=$(timeout 300 bash -c "$COMMAND" 2>&1)
   EXIT_CODE=$?
   ```

3. **Handle working directory changes:** Commands may change the working directory (e.g., `cd /tmp`). Always run commands in a subshell to isolate directory changes:
   ```bash
   OUTPUT=$(timeout 300 bash -c "$COMMAND" 2>&1)
   ```
   This ensures subsequent commands run in the original directory.

4. **Check Result:**
   - **If Success (exit code 0):** Log the task as completed. If the output is empty, note "No output." If the output exceeds 10,000 characters, truncate to the last 5,000 characters and note "[truncated]".
   - **If Failure (non-zero exit code):**
     - Record the error output (last 50 lines if output is large).
     - **STOP IMMEDIATELY.** Do not process remaining tasks.
     - Report the failure with the error details.

5. **Handle commands requiring user interaction:** If a command appears to hang (no output for > 30 seconds), assume it requires user input. Kill the process and report: "Command appears to require interactive input. Aborting."

## 3. Report

Once the queue finishes (all completed or a failure occurred), generate a clear summary:

```
Task Queue Complete.
✅ Completed: <count> tasks
❌ Failed: <count> tasks (include error details)
⏳ Remaining: <count> tasks (not processed due to failure)
```

Include the error output from the failed task if applicable.

---
