# TODO.md AI Tasks

One to-do list for you and your AI agent. It works on plain markdown tasks, the same ones the [TODO.md app](https://todo-md.devviy.com) shows on iPhone and Mac, so you can use that app or any markdown editor.

Write a task in your notes, mark it in progress, and a couple of minutes later an AI coding agent starts on it in the right project. The task itself shows you where it is:

| Flag | Means |
|---|---|
| 🤖 | the agent is working on it |
| ⚠️ | it needs an answer from you (write it under the task) |
| 👀 | done, ready for your review |
| 🛑 | failed or blocked, with a note why |

You review the work and tick the task off yourself.

```markdown
- [/] 👀 Fix the login button on small phones
    It overlaps the logo below 375 px.
    🤖 session: claude 3f6c2a1e-…: Fixed in LoginView. Check it on an iPhone SE.
```

This is not an app you install. It's a skill: instructions your own AI agent reads to build the automation for your computer, your notes and your agent.

## What you need

- Your tasks in markdown files: an Obsidian vault, or any folder of `.md` notes. One file per project.
- An AI coding agent: Claude Code, Codex, Gemini CLI, opencode, or another one that runs from a command line. Editor-only agents (Antigravity, Cursor and similar) work too, through an inbox file.
- macOS, Linux or Windows, on a computer that stays on while you're away if you want work to happen then.

## Install the skill

```bash
git clone https://github.com/andriyvl/todo-md-ai-tasks.git
cp -r todo-md-ai-tasks/todo-md-ai-tasks ~/.claude/skills/    # or your agent's folder from the table
```

Copy the inner `todo-md-ai-tasks` folder into your agent's skills folder:

| Agent | Folder |
|---|---|
| Claude Code | `~/.claude/skills/` |
| Codex | `~/.codex/skills/` |
| Gemini CLI | `~/.gemini/skills/` |
| opencode | `~/.config/opencode/skills/` |
| Others | `~/.agents/skills/`, or the folder your agent's docs name |

Then ask your agent: **"Set up TODO.md AI tasks for me."**

If your agent doesn't load skills, point it at the file instead: "Read `todo-md-ai-tasks/SKILL.md` and follow it."

The agent asks a few questions (where your notes are, which file goes with which repo, which agent runs the work, how much it may do without asking), says what it will build, and waits for your yes. Then it writes a small watcher, schedules it every minute, and tests it with you on a throwaway task.

## Using it

1. Write a task in a project's file: `- [ ] Add a dark mode toggle`. Add notes or screenshots under it if you like.
2. Mark it in progress: `- [/]`. Two minutes later the agent has it.
3. Watch the flag. On ⚠️, write your answer on a new line under the task; the agent continues.
4. On 👀, review the work, then close the task: `- [x]`.

Keep a task for yourself with `#manual` in its text. Flag one as important with 🚩 at the start.

## Good to know

- The watcher reads files and compares them; it costs nothing until a task is new. Each agent session uses your own subscription or API key.
- The agent works without asking for every step, at the permission level you choose during setup. It's told never to push, deploy or tick your checkboxes, but don't give it credentials it doesn't need.
- The TODO.md app for iPhone and Mac reads the same markdown files and shows these flags as coloured chips, so you can hand over tasks and follow them from your phone: [todo-md.devviy.com](https://todo-md.devviy.com).

Made by Andriy Viychuk.
