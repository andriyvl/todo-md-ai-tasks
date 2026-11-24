# The message to the agent

Send this when a session starts. Fill the placeholders; keep the rules as they are unless the user changes them.

```markdown
You have {N} task(s) from my to-do list for this project ({repo path}).
Decide how to work through them: together, one by one, or with subagents if you can use them.

## Tasks

### 1. {title}   (key: {project}:{key}) {"(important)" if the task has 🚩}
{the task's notes, as written}
Attached images: {paths under .vault-tasks/attachments/, if any}

### 2. ...

## Rules

- First check whether a task is already done. If it is, say so and report it for review.
- Work only in this repository{, on branch ai/<key> in this worktree}.
- Don't push, deploy, publish or touch production unless the task itself says to.
- Don't edit my task file and never tick a checkbox. I close tasks myself after review.
- If something is unclear or you need a decision, don't guess and don't wait: report the task as `input` with your question as the note, and move on to the next task. My answer will come back to you in this session.

{the "How to report" block below}
```

Every message carries the "How to report" block, with the keys of the tasks in that turn:

```markdown
## How to report

When you finish a task, give up on it, or need my answer, append one line per task to
.vault-tasks/reports.jsonl in this repository (create the file if it's missing):

{"task": "{project}:{key}", "state": "review", "note": "..."}

- task: the task's key exactly as listed above. Keys in this turn: {project}:{key1}, {project}:{key2}
- state: review (done, ready for me to check), failed (couldn't do it), input (you need my answer)
- note: optional, one or two sentences about this task only: what to check, why it failed, or your question.
  Leave it out when there's nothing worth saying.
```

When new tasks join an open session, send: "More tasks from my list. Same rules as before.", then the "Tasks" section with only the new tasks, then the "How to report" block with their keys.

When the user answers a ⚠️ task, send: "My answer about {project}:{key} ({title}):", the lines they added under the task since the question, then the "How to report" block with that key.

For a harness that can't resume and starts a fresh session per turn, send the whole first message each time, with the task's current notes (they hold the conversation so far).
