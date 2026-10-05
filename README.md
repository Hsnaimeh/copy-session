# copy-session

Copy a Claude Code conversation, or just the last response, to your clipboard.

```bash
copy-session -r      # the last response, on your clipboard
```

## Why this is not a slash command

Claude Code already has `/export`, which copies the whole conversation and
costs nothing - it runs in the client and never calls the model.

What it does not have is "just the last response", "just the code blocks", or
"every message that mentioned the migration". The obvious way to add those is a
custom slash command in `.claude/commands/`, and that is the wrong way: those
are *prompt* commands. They get sent to the model, so every run costs tokens,
which is the opposite of what a copy button should do.

So this is a plain shell script. It reads the transcript Claude Code already
writes to disk and pipes it to your clipboard. It never talks to Claude, never
sends a request, and costs nothing.

## Install

```bash
git clone https://github.com/Hsnaimeh/copy-session.git ~/.copy-session
echo 'export PATH="$HOME/.copy-session:$PATH"' >> ~/.zshrc   # or ~/.bashrc
exec $SHELL
```

Requires `python3` and a clipboard tool. It finds `pbcopy` (macOS), `wl-copy`
(Wayland), `xclip` or `xsel` (X11), or `clip.exe` (WSL). Without one, `-p` and
`-o` still work.

## Use

```bash
copy-session              # the whole conversation
copy-session -r           # just the last response
copy-session -u           # just the last thing you typed
copy-session -n 6         # the last 6 messages
copy-session -c           # the code blocks of the last response
copy-session -s "docker"  # every message matching a pattern
```

Choosing a session, or a different output:

```bash
copy-session -l             # list this project's sessions, newest first
copy-session -i 3 -r        # last response of the 3rd-newest session
copy-session -i <id> -r     # by session id, as -l prints it
copy-session -o notes.md    # write a file instead of copying
copy-session -p             # print to stdout (pipe it anywhere)
copy-session -t             # include tool calls and results
copy-session -h             # this, shorter
```

Every selection flag works with every mode, so `copy-session -i 2 -c -o
snippets.md` is the code blocks from the second-newest session, written to a
file.

## Which session it picks

The one you are in. Claude Code exports `CLAUDE_CODE_SESSION_ID`, and this
reads it.

That matters more than it sounds. With two Claude sessions open on the same
project, "newest file" is whichever one wrote last - so a plain `ls -t` hands
you the other session's answer, which is exactly what the first version of this
script did. An explicit `-i` always wins; without one, you get your own
conversation.

Outside Claude Code - a normal terminal, a script, a cron job - there is no
session id to read, so it falls back to the newest transcript for the current
directory.

## Where the transcripts come from

Claude Code already records every session as JSONL:

```
~/.claude/projects/<cwd-with-slashes-replaced>/<session-id>.jsonl
```

This reads those files and nothing else. It makes no network request and writes
nothing except the file you ask for with `-o`.

Worth knowing while you are in there: nothing prunes them. One project on the
machine this was written on held 50 sessions, the largest 81 MB. `copy-session
-l` prints the sizes.

## Output

Markdown, with each message under a `## user` or `## assistant` heading. The
single-message modes (`-r`, `-u`, `-c`) print the body alone, since a heading on
one message is just noise in a paste.

Tool calls and their results are left out unless you pass `-t`. They are most of
the bytes in a working session and almost never what you are trying to paste.
With `-t`, each call is one line naming the tool and its main argument, and each
result is truncated to 400 characters.

Thinking blocks are never included.

## When it finds nothing

It says so and exits non-zero, rather than copying something misleading:

```
$ copy-session -s "nothing-matches-this"
no message matches 'nothing-matches-this'

$ copy-session -c
the last response has no code blocks
```

A response made entirely of prose genuinely has no code blocks. That is an
answer, not a failure.

## A note on what ends up on your clipboard

A transcript is whatever you and Claude said, which on a work machine is source
code, file paths, hostnames, and sometimes a key you pasted in. `copy-session`
does not filter any of it. Check before you paste into a public issue.

## License

MIT
