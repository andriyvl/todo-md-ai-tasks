# Running the agent: per harness

Flags change between versions. Before writing the adapter, run the harness's `--help` (and the help of its exec or resume subcommand) and use what the installed version actually offers. Then run one real start and one real resume by hand and look at the output, so you know where the session id comes from.

Every adapter needs the same four things:

| Need | Why |
|---|---|
| Start a session non-interactively, in a given folder, with a prompt | step 3 of the tick |
| Get the session id | to resume it and to write the session line |
| Resume that session with a new prompt | new tasks for an open session, the user's answers |
| Know whether it is running, finished or failed | the flags |

Run each start or resume in the background (a detached child process), write its output to a log file per session, and keep the process id in state. Running: the process is alive. Finished: it exited with code 0. Failed: any other exit code, or it was killed. Never resume a session while a process for it is still running; queue the new prompt and send it when the process ends.

Pass the prompt through stdin or a file when the harness allows it, so long prompts and special characters survive.

## Claude Code

```bash
# start: choose the session id yourself, so you know it before the run ends
claude -p --session-id "$UUID" --permission-mode acceptEdits --output-format json "$PROMPT"
# resume
claude -p --resume "$UUID" --permission-mode acceptEdits --output-format json "$REPLY"
```

- Run with the repo as the working directory.
- `--permission-mode acceptEdits` lets it edit files in the repo, which is all it needs to report back (`.vault-tasks/reports.jsonl`). Shell commands it isn't allowed to run are refused rather than asked about, so allow the project's own commands with `--allowedTools` (for example `"Bash(npm test:*)" "Bash(git diff:*)"`) instead of jumping to `bypassPermissions`.
- Tested: start with `--session-id`, then `--resume` with the same id, continues the same conversation and returns the same `session_id` in the JSON.
- The user can open the same session interactively with `claude --resume <id>`. The Claude Code desktop app reads skills from the same `~/.claude/skills/` folder, so the user can run this setup from there too; the watcher still starts the agent through the `claude` CLI.

## Codex CLI

```bash
# start
codex exec --json -C "$REPO" -s workspace-write "$PROMPT"
# resume (run it with the repo as the working directory)
codex exec resume "$SESSION_ID" "$REPLY"
```

- With `--json`, the output is a stream of JSON events; the session (thread) id is in the first event. Check the field name on the installed version.
- `-s workspace-write` lets it write inside the repo. `--skip-git-repo-check` is needed outside a git repo. `-o <file>` saves the last message, handy for the log.
- The user can continue it interactively with `codex resume <id>`.

## Gemini CLI

```bash
# start
gemini -p "$PROMPT" --approval-mode auto_edit -o json
# resume
gemini --resume latest -p "$REPLY" --approval-mode auto_edit -o json
```

- Sessions are stored per project folder, and `--resume` takes `latest` or an index from `gemini --list-sessions`. If the installed version can't resume by id, the user's own sessions in that folder can get in the way: resume by index from `--list-sessions` right after the start, or start a fresh session for each turn and include the task's notes (which hold the whole conversation) in the prompt.
- `--approval-mode`: `auto_edit` approves edits; `yolo` approves everything.

## opencode

```bash
# start
opencode run --dir "$REPO" --format json "$PROMPT"
# resume
opencode run --dir "$REPO" --session "$SESSION_ID" --format json "$REPLY"
```

- The session id is in the JSON events. Permissions come from the opencode config (`permission` settings), not from flags.

## Other command-line agents

Any agent that can (1) run a prompt without a terminal UI and (2) continue a named session works. Fill the four needs from its docs. If it can't resume, start a fresh session per turn and put the task's notes in the prompt: the notes under the task are the conversation.

## Orchestrator apps with a CLI

Some desktop apps that run agents (multi-agent managers, agent "inboxes") have their own CLI to create a thread in a project and post a message to it, and report the thread's state (running, waiting for input, done). Prefer that when the user lives in such an app: the session line can then be a link that opens the thread on desktop or phone, and "waiting for input" comes straight from the app instead of from an `input` report. Check that the app opens the thread in the project's own folder: when the repo sits inside a folder the app already knows as a project, it may pick the parent folder, so register the repo as its own project first.

## IDE-only agents: inbox mode

When the harness only runs inside an editor (Antigravity, Cursor, VS Code agent modes and similar) and has no command line to start it:

- The watcher writes the batch to `<repo>/.agent-inbox.md` (the same message as `prompt.md`) and sets each task to 🤖 with the session line `🤖 session: inbox <date>`.
- It notifies the user (macOS `osascript -e 'display notification "…" with title "Vault tasks"'`, `notify-send` on Linux, or a push service they already use).
- The repo's agent rules file (for example `AGENTS.md`, or the harness's own rules or workflows) gets one line: "When `.agent-inbox.md` exists, work through it first, then delete it."
- The agent reports through `.vault-tasks/reports.jsonl` exactly like a CLI agent, so the flags come back the same way. Without a running process, the watcher can't see "running" or "crashed": tasks stay 🤖 until the agent reports on them.

First check whether the IDE ships a command-line launcher that can start an agent chat with a prompt; if it can, treat it like a command-line agent instead.

## Permissions and safety

- Headless agents act without asking. Match the permission level to what the user agreed to, and prefer allowing specific commands over allowing everything.
- With a worktree per session (`git worktree add ../<repo>-ai-<key> -b ai/<key>`), the agent can't disturb the user's own checkout, and the user merges what they like.
- The prompt forbids pushing, deploying and touching production, but a prompt is not a guarantee: don't give the agent credentials it doesn't need.
- Every session costs tokens on the user's subscription or API key. The watcher itself must cost nothing when no task is new.
