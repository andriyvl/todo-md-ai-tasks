---
name: todo-md-ai-tasks
description: Set up an automation that hands markdown to-do tasks to an AI coding agent. The user writes a task in a project's markdown file (an Obsidian vault or any folder of .md notes), marks it in progress with `- [/]`, and a small watcher starts or continues an agent session in that project's repo; the agent's progress comes back to the task as a flag (🤖 working, ⚠️ needs input, 👀 ready for review, 🛑 failed) and a short note. Works with Claude Code, Codex, Gemini CLI, opencode and other agents that run from a command line, and with IDE-only agents through an inbox file. Use when the user wants one to-do list for themselves and their AI agent, wants to delegate tasks from their notes or phone, or asks to set up, change or debug "TODO.md AI tasks".
---

# TODO.md AI Tasks: one to-do list for you and your AI agent

You are setting up a small watcher on the user's computer. When they're done, this is how they work:

1. They write a task in a project's markdown file, from any device that syncs the notes.
2. They mark it in progress: `- [ ]` becomes `- [/]`.
3. About two minutes later an agent session starts in that project's repo with the task (and any other new in-progress tasks of that project).
4. The task line gets a flag at the start: 🤖 while the agent works, ⚠️ when it needs an answer, 👀 when it's ready for review, 🛑 when it failed. A short note under the task says what happened, when there is something worth saying.
5. They review the work and close the task themselves (`[x]` or `[-]`). The agent never ticks a checkbox.

Build it for the user's own setup. Don't copy anyone else's paths, names or accounts.

## 1. Ask before building

Ask these in one short message, offering the default in brackets. Infer what you can from the machine first (installed agent CLIs, where the notes folder is, which repos exist) and only ask what's left.

- **Notes folder:** where the markdown files live (an Obsidian vault, a synced folder).
- **Projects:** which task file belongs to which repo. [one file per project, `<notes>/Projects/<repo-name>.md`]
- **Agent:** which harness runs the work. See `references/harnesses.md`. [the CLI you're running in now, if it has a headless mode]
- **Permissions:** how much the agent may do without asking. Headless runs can't stop to ask for permission, so this decides what work is possible. [edit files and run the project's own commands inside the repo; no push, no deploy]
- **Isolation:** work in the repo checkout, or in a git worktree and branch per session. [worktree when the user also works in that repo by hand]
- **Batching:** one session per project that takes every in-progress task, or one session per task. [one per project; the agent decides the order and may use subagents]
- **Check interval and settle time.** [every 60 s; a task starts after it has been unchanged for 2 minutes, so the user can finish typing or add a screenshot from the phone]

Then say in two or three lines what you will build, the permission level, and that each run uses the user's own agent subscription or API key. Wait for a yes.

## 2. The file contract

Read `references/task-format.md`. In short:

- A task is a markdown checkbox line: `- [ ] text`, `- [/] text` (in progress), `- [x]` (done), `- [-]` (cancelled). Indented lines directly under it are its notes, part of the task. A nested checkbox is a task of its own.
- At most one flag, at the very start of the text: `🤖` `⚠️` `👀` `🛑` set by the automation, `🚩` set by the user to mean important. Setting a flag replaces the old one.
- A task is **new** when it is `[/]`, has no agent flag, has no `#manual` or `#noai` tag, and wasn't already in progress when the project was enabled.
- Under a task the automation adds one session line, so the user can find the session again: `🤖 session: <harness> <session-id>`. Agent notes go on that line or on new lines starting with `🤖:`. Lines that don't start with 🤖 are the user's.

## 3. The watcher

Write one small, self-contained program. Python 3 standard library is a good default: no installs, runs on macOS, Linux and Windows. Keep its config, state and logs outside the notes folder, as plain JSON and text files. Subcommands:

| Command | Does |
|---|---|
| `enable <project>` | Start watching a project. Record the keys of its current `[/]` tasks so they are ignored. Enabling twice changes nothing. |
| `disable <project>` | Stop watching. |
| `status` | Each project, its open session, and every tracked task with its flag. |
| `tick [-v]` | One pass. The scheduler runs this every interval. |
| `install` / `uninstall` | Add or remove the scheduler entry. |

Each tick, per enabled project:

1. **Read** the task file. Skip the project for this tick if the file is missing or not yet downloaded by the sync client.
2. **Apply the agent's reports** from `<repo>/.ai-tasks/reports.jsonl` (see "How the agent reports back" below). Keep a byte offset in state and apply only complete new lines past it; a last line that doesn't parse yet is still being written. Empty the file and reset the offset only when no session for that repo is running.
3. **Find new tasks.** Track each by a key (`references/task-format.md`). A new task starts only after its title and notes have stayed unchanged for the settle time.
4. **Dispatch.** If the project has no open session, start one with every settled new task. If a session is open and idle, send the new tasks into it as the next turn. If it is running, keep them queued for when it ends. Write the session line and 🤖 under each task as soon as it is handed over.
5. **Sync flags.** A task's flag is whatever the agent last reported for it. A task in the current turn that the agent hasn't reported on follows the session: 🤖 while it runs; 👀 if the run ended normally; 🛑 if it crashed or was stopped, with the note "session ended (exit code N), log: <path>" placed like any agent note. Tasks from earlier turns keep their flag.
6. **Answers.** When the user adds lines under a ⚠️ task (lines not starting with 🤖, below the agent's question) and they have settled, send them to the same session as the next turn and set 🤖. Collect every answered task of the project into one turn.
7. **Close.** A task the user closed (`[x]`, `[-]`, or moved back to `[ ]`) stops being tracked; its flag stays as history. After a rename, keep the old key as an alias of the task, so a report the agent writes with the key it was given still lands. A report about a task that is no longer tracked is logged and dropped. A session with no open tasks is done; the next new task starts a fresh one.

Rules that keep the user's notes safe:

- **Never rewrite the file from memory.** Re-read it immediately before every write, change only the lines you mean to, and if the file changed between your read and your write, start over (the loop is in `references/task-format.md`). The user may be editing from a phone while sync runs.
- Find a task to change by its key or its session line, never by line number.
- Never tick a checkbox, never delete a line, never touch a task that isn't tracked.
- If a sync conflict drops a flag, put it back on the next tick rather than starting the task again.
- Take a lock so two ticks never run at once (a lock file created exclusively works on every OS), and log every handover and flag change.

## 4. Starting the agent

Read `references/harnesses.md` for the exact commands per harness. In every case:

- Run in the project's repo (or its worktree), in the background, so the tick returns at once. Store the process id, the session id, the turn's task keys and the log path in state.
- Use the permission level the user chose. Never use a "skip all permissions" mode unless they asked for it by name, and then only inside a worktree or sandbox.
- Copy the task's attached images into `<repo>/.ai-tasks/attachments/` and give the agent those paths, so it can read them without access outside the repo.
- Add `.ai-tasks/` to the repo's exclude file, so none of this shows up in git. `git rev-parse --git-path info/exclude` gives its path, in a worktree too.
- Send the message from `references/prompt.md`. Every message, including later turns of the same session, carries the "How to report" block with that turn's task keys.

### How the agent reports back

The agent reports by appending one JSON line per update to `<repo>/.ai-tasks/reports.jsonl`:

```json
{"task": "my-app:3f6c2a1e8d", "state": "review", "note": "Done. Check the login screen on a small phone."}
```

`state` is `review`, `failed` or `input`; `note` is optional. A file inside the repo is something every harness can write at its normal permission level, with its own file tools or a shell `echo >>`, so reporting never depends on extra permissions or a sandbox exception. An earlier design had the agent run the watcher's own `mark` command instead; under Claude Code's `acceptEdits` mode that command was refused, and the task only reached 👀 when the session ended.

Headless agents can't ask questions. They report `input` with the question as the note and move on. The user answers under the task, and step 6 above resumes the session.

When the harness has no command line (an IDE agent such as Antigravity or Cursor), use inbox mode from `references/harnesses.md`: the watcher writes the batch to an inbox file in the repo and notifies the user, and the agent reports through the same file when the user opens it.

## 5. Schedule it

- **macOS:** a LaunchAgent with `StartInterval` 60 running `tick`. If the notes are in iCloud Drive and the log shows `Operation not permitted`, macOS is blocking the background interpreter from iCloud: give that interpreter Full Disk Access, or run the tick through an app binary that already has access to the folder.
- **Linux:** a systemd user timer (or cron) every minute.
- **Windows:** Task Scheduler, repeating every minute.
- A tick with nothing new must cost no tokens: only read files and compare.

## 6. Check it before handing over

Do all of these on a throwaway task in a real project, and show the user the result:

- [ ] A `[/]` task that existed before `enable` is ignored.
- [ ] A new `[/]` task starts only after the settle time, gets 🤖 and the session line, and the agent session opens in the right repo.
- [ ] Two tasks set to in progress together go to one session (when batching per project).
- [ ] The agent's `review` report shows 👀 and the note on the right task, written by the agent's own report, not by the end-of-session fallback. Nothing else in the file changed (diff it).
- [ ] An `input` report shows ⚠️; writing an answer under the task resumes the same session, and the agent's next report lands.
- [ ] Renaming a tracked task's title doesn't start it again; closing a task stops tracking; a `#manual` task is never picked up.
- [ ] Editing the file between the watcher's read and its write loses nothing.
- [ ] `install` schedules the tick and `uninstall` removes it.

Then give the user a short how-to: write a task, set it to in progress, watch the flag, answer under ⚠️, review, close. To run a task again: remove its flag and session line, set it to `[ ]`, wait a tick, set it back to `[/]`. `status` and the log file are where to look when nothing happens.

## Works well with

Any markdown editor shows the flags as emoji. The TODO.md app for iPhone and Mac (todo-md.devviy.com) reads the same files and shows these flags as coloured chips, so the user can hand over tasks and follow them from their phone.
