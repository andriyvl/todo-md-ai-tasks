# Task file format

One markdown file per project. Anything else in the file (headings, prose, other lists) is left alone.

## Example

```markdown
# My app

- [ ] Add a dark mode toggle to the settings screen
- [/] 🤖 Fix the login button on small phones
    It overlaps the logo below 375 px.
    ![[login-bug.png]]
    🤖 session: claude 3f6c2a1e-8d4b-4c1e-9a77-2b1d5e0c9f10
- [/] ⚠️ Add Stripe checkout
    🤖 session: claude 3f6c2a1e-8d4b-4c1e-9a77-2b1d5e0c9f10: Test or live keys? I found both in .env.example.
    Test keys for now.
- [/] 👀 Write the README install section
    🤖 session: claude 3f6c2a1e-8d4b-4c1e-9a77-2b1d5e0c9f10: Added install steps for macOS and Linux. Windows untested.
- [ ] Call the client about the logo #manual
- [x] 👀 Set up the landing page
```

## Lines

- **Task line:** `^(?P<indent>[ \t]*)(?P<bullet>[-*+]|\d+[.)]) \[(?P<status>.)\] (?P<text>.*)$`
- **Status:** ` ` open, `/` in progress, `x` done, `-` cancelled. Treat any other character as not in progress.
- **Notes (the task's body):** the lines after the task that are blank or indented deeper than it, up to the next line at the same or a shallower indent. Trailing blank lines are not part of it. Measure indents with tabs as 4 spaces. A nested checkbox inside the notes is also a task of its own; it travels with its parent's notes.
- **Flag:** `^(🤖|⚠️?|👀|🛑|🚩)[ \t]*` at the start of the text. Accept ⚠ with and without the variation selector, since some editors drop it. One flag per task; a new flag replaces the old one.
- **Opt out:** `#manual` or `#noai` anywhere in the text keeps the task from the agent. So does an agent flag the user typed themselves.
- **Important:** 🚩 is the user's. List 🚩 tasks first in the agent's message and say they're important. Once the agent starts, its flag replaces 🚩.
- **Whose line:** a note line whose text starts with 🤖 (`🤖 session:`, `🤖:`) was written by the automation. Every other note line is the user's.

## Task key

A task needs an identity that survives a re-read and small edits:

1. Take the text without the flag.
2. Remove inline fields `[name:: value]` and, if the user has the Obsidian Tasks plugin, its date and priority emoji with their values (📅 ⏳ 🛫 ➕ ✅ ❌ ⏰ 🔔 ⏫ 🔼 🔽 🔺 ⏬, and 🔁 to the end of the line). Keep `#tags`.
3. Collapse whitespace, trim, lowercase, normalise Unicode (NFC).
4. Hash it (for example the first 10 hex characters of SHA-1).
5. Two tasks with the same title in one file: give the second and later ones a suffix (`<hash>-2`, `<hash>-3`) in file order.

When the user edits a tracked task's title, the key changes. Find the task again by its session line (the session id is in it), and when several tasks share that session, pick the one whose title is closest to the old one. Store the new key.

For the settle check, hash the title and the user's note lines together, so a task waits while the user is still adding notes or pictures.

## What the automation writes

- **Flag:** replace the flag at the start of the text, keep the rest of the line byte for byte.
- **Session line:** at handover, one line at the end of the task's notes, indented like its first note line (or one level deeper than the task when it has none): `🤖 session: <harness> <session-id>`. For a harness with a web or app link, a markdown link is nicer: `[🤖 session](https://...)`.
- **Agent note:** while the session line is still the last non-blank line of the task's notes, the first note goes on it after a colon. Otherwise the note gets its own new line at the end of the notes: `🤖: <note>`. That way the user's answers and the agent's notes read in order, like a conversation. Don't write the same note twice.
- **Images:** resolve `![[name.png]]` against the vault's attachment folder (Obsidian keeps it in `.obsidian/app.json` as `attachmentFolderPath`; `./` means next to the note) and `![](relative/path.png)` against the note's folder. Copy them into the repo's `.ai-tasks/attachments/` for the agent.
- **Cloud files:** a file that isn't downloaded yet (on iCloud Drive a `.<name>.icloud` placeholder next to it, or a size of 0) is skipped for this tick. On macOS, `brctl download <path>` asks iCloud to fetch it.

## Safe editing

```text
for attempt in 1..3:
    text = read(file)
    new = change(text)          # find the task by key or session id, never by line number
    if new is None or new == text:
        return                  # nothing to do
    if read(file) == text:      # nobody edited it meanwhile
        write(file, new)        # in place, UTF-8, keeping the file's line endings
        return
    sleep 0.5 s                 # someone edited it: start over with the new text
log "gave up, file kept changing" and leave the change for the next tick
```

Write the file in place rather than replacing it with a renamed temp file; some sync clients treat a replaced file as a delete plus a new file. Never write a file you failed to parse.
