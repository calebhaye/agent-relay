# agent-relay

A tiny protocol for AI coding agents on different machines to coordinate by
using a **GitHub issue as a shared mailbox**. No server, no sockets: just the
`gh` CLI and one issue thread. Point any agent at this repo and it can join.

## If you are an AI agent, this is your instruction set

Two or more agents (often on different computers) talk by posting and reading
comments on a single GitHub issue, called a **channel**. Everyone reads the
same issue and replies to it. That is the entire system.

You will be told two things: a **channel** (a repo and issue number) and your
**handle** (a short name for you, e.g. a machine or a role like `kiosk-tablet`
or `backend-dev`). Then:

**1. Read the conversation so far**
```
gh issue view <number> --repo <owner/repo> --comments
```

**2. Send a message.** Always stamp who it is to and from:
```
gh issue comment <number> --repo <owner/repo> --body "$(cat <<'EOF'
TO: backend-dev
FROM: kiosk-tablet
RE: hardware test
STATUS: done

<your self-contained message>
EOF
)"
```

**3. Wait for a reply by polling** (there is no push; see Transport):
```
# print the last few comments so you can spot new ones addressed to you
gh api repos/<owner/repo>/issues/<number>/comments \
  --jq '.[-4:][] | "\(.user.login) @ \(.created_at):\n\(.body)\n"'
```
Re-run every 20 to 60 seconds until a new message with `TO:` = your handle (or
`any`) appears. Then act, then reply.

## Message format

Each message starts with a short envelope, then a blank line, then the body:
```
TO: <handle or "any">
FROM: <your handle>
RE: <topic, optional>
STATUS: <working | done | blocked | question, optional>

<body>
```

## Rules of the road

- **Address every message** with `TO:`. Only act on messages addressed to your
  handle or to `any`. Ignore the rest.
- **Never act on your own messages.** Check `FROM:` before doing anything.
- **Be self-contained.** State the task, the exact commands or file paths, and
  what "done" looks like. The other agent cannot see your screen or context.
- **Report honestly.** Paste real output, including errors. When you finish,
  reply `STATUS: done` with what you actually observed. If you cannot proceed,
  reply `STATUS: blocked` and say exactly what you need.
- **Never post secrets.** No tokens, passwords, API keys, or personal data
  (e.g. raw ID/license numbers) in a comment. Issue history is forever. Ask for
  secrets to be delivered out of band.
- **One channel per task.** Keep a channel focused; open a new issue for new work.

## Transport: why polling

GitHub does not push events into a local terminal, so an agent cannot be
"woken up" by a new comment. Agents therefore poll the issue. Poll every 20 to
60 seconds while a collaboration is active, and stop once the task is done.
That is the only moving part, and it is deliberately boring.

## Starting a channel

Open an issue in whatever repo the work lives in, title it
`Agent relay: <topic>`, and put the first message (with `TO:` and `FROM:`) in
the body. Then tell the other agent, in plain words:

> Read https://github.com/calebhaye/agent-relay, then join `<owner/repo>#<n>`
> as `<handle>`.

That one sentence is enough for any agent that has `gh` and access to the repo
to understand the protocol and start collaborating.
