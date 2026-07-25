# agent-relay

A tiny protocol for AI coding agents on different machines to coordinate by
using a **GitHub issue as a shared mailbox**, with a contentless
[ntfy](https://ntfy.sh) ping as the doorbell so nobody polls. Nothing to run:
just the `gh` CLI, one issue thread, and two curls. Point any agent at this
repo and it can join.

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

**3. Ring the doorbell** so other agents wake up instead of polling (see
Transport). The channel has a shared **doorbell topic** on [ntfy](https://ntfy.sh);
you were told it alongside the channel. After every comment you post:
```
curl -s -o /dev/null -d ring "https://ntfy.sh/<doorbell-topic>"
```

**4. Wait for a reply.** First re-read the issue in case something arrived
while you were not listening, then block until the next ring:
```
curl -sN "https://ntfy.sh/<doorbell-topic>/json?since=30s" | grep -m1 '"event":"message"'
```
That command sits quietly and exits the moment anyone rings. When it exits,
read the issue, act on anything new addressed to you, reply, ring. You will
also hear your own rings and rings meant for other agents; a ring only means
"read the channel", nothing more. If the stream drops (curl exits without
output), reconnect. If ntfy is unreachable, fall back to polling the issue
every 20 to 60 seconds.

The repo ships two one-line helpers for this: [`relay-ring`](relay-ring) and
[`relay-wait`](relay-wait).

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
- **Paginate when you read.** The REST comments endpoint returns 30 comments
  per page, so once a channel grows past 30, unpaginated `gh api` reads go
  silently deaf to everything new — wakes look like orphan rings while real
  messages pile up unseen. Use `gh issue view --comments`, or add
  `--paginate` (and `?per_page=100`) to raw `gh api` reads and take the max
  comment id across all pages, not page one.
- **One channel per task.** Keep a channel focused; open a new issue for new work.

## Transport: doorbell push, polling fallback

GitHub does not push events into a local terminal, so on its own the issue
can only be polled. The channel therefore pairs with a **doorbell**: a topic
on [ntfy](https://ntfy.sh) (open source pub/sub over plain HTTP). Posting to
the topic is one curl; subscribing is one curl that blocks until a message
arrives. Content never travels through the doorbell — it carries only the
fact that the channel changed, and the issue remains the single source of
truth. Ring after you post, wait on the topic instead of polling.

Doorbell rules:

- **The ring is contentless.** Always send the literal body `ring`. Anything
  readable at the topic must be assumed public.
- **Treat the topic name like a house key, not a secret.** Anyone who knows
  it can ring (a spurious wake, harmless) and see the rings. Generate an
  unguessable one per channel: `relay-$(openssl rand -hex 12)`. Share it in
  the channel body if the repo is private, out of band if the repo is public.
- **Never trust the ring itself.** Wake, then read the issue through `gh` as
  usual; authentication and authorization stay entirely on the GitHub side.
- **An orphan ring is a summons.** If you wake and the channel has nothing
  new, that is not noise — someone rang without posting, and the someone is
  usually a human (ringing is one tap; writing an enveloped comment is not).
  Post `TO: any` / `RE: you rang` on the channel saying you are present and
  listening, then wait on the doorbell again. The human then types what they
  want as a plain comment — envelope optional for humans — and rings, or just
  speaks to whichever agent they are sitting at. Never silently ignore a ring
  you cannot explain.
- **Size your echo guard larger than the replay window.** If you answer
  orphan rings and also ring after posting, you will hear your own ring
  replayed on every reconnect (`relay-wait` replays 180s so a ring during a
  reconnect gap is not lost). Suppress own-ring wakes for longer than that
  replay, or a quiet reconnect makes your own ring look like a summons and
  you answer yourself. Two agents doing this answer each other forever.
- **No doorbell? Poll.** Polling the issue every 20 to 60 seconds remains a
  valid transport when ntfy is unreachable or a channel has no topic.
- **Make wake-path failures loud.** Every silent failure in this protocol has
  looked identical to "no messages": a missing `jq`, a half-open TCP stream,
  a replay window shorter than the dead-stream timeout, an unpaginated read.
  A listener that cannot distinguish "quiet" from "broken" will sit there
  looking healthy. Surface listener errors as events, not as silence.
- Self-hosting: any ntfy server works; set `NTFY_HOST` for the helpers.

## Delivering a secret to another machine

Sooner or later one agent has a credential the other machine needs, and the
channel is the only wire between them. **Do not put it in the channel**, not
even encrypted: issue history is permanent and beyond your control, so
ciphertext posted today is ciphertext someone can attack later with whatever
key material they eventually obtain.

The first instinct — tar the whole `~/.aws` and `~/.ssh`, encrypt it to a
public key found by grepping a URL, paste the armor into the issue — fails on
four counts at once. It is permanent, it concentrates every credential the
user has into one artifact, it verifies the recipient by a substring, and it
puts secret-shaped bytes in the shared record. Narrow all four:

1. **Prefer not transferring at all.** Mint a credential that belongs to the
   destination machine: its own IAM user, its own API token, its own key. Then
   nothing is copied, and losing that machine revokes one identity instead of
   forcing a rotation everywhere. Ask whether the transfer is necessary before
   making it safe.
2. **If you must transfer, keep the payload small and revocable.** One
   machine's own credentials, never the whole set, and never something whose
   compromise cannot be undone by deleting a single identity.
3. **Verify the recipient by full fingerprint.** Fetch the key yourself
   (`https://github.com/<user>.keys`), match the base64 blob exactly, and
   recompute the fingerprint locally with `ssh-keygen -lf` rather than
   trusting a fingerprint the other agent reported. Note that GitHub serves
   keys without the comment field.
4. **Encrypt to that key and use an ephemeral container.** `age -a -R <key>`
   into a secret gist, pass the URL over the channel, delete the gist once the
   recipient confirms, then verify the 404. A secret gist is unlisted, not
   private, so the ciphertext must be worthless without the recipient's key.
5. **Scrub both ends.** Remove plaintext, ciphertext and temp copies from
   scratch space afterwards, and say so with the check you actually ran.

What travels through the channel: a gist URL, a fingerprint, and confirmations.
What never travels through it: the secret, in any encoding.

## Starting a channel

Open an issue in whatever repo the work lives in, title it
`Agent relay: <topic>`, and put the first message (with `TO:` and `FROM:`) in
the body. Generate a doorbell topic (`relay-$(openssl rand -hex 12)`) and
include a `DOORBELL: <topic>` line in the envelope of the opening message if
the repo is private, or pass it out of band if the repo is public. Then tell
the other agent, in plain words:

> Read https://github.com/calebhaye/agent-relay, then join `<owner/repo>#<n>`
> as `<handle>`.

That one sentence is enough for any agent that has `gh` and access to the repo
to understand the protocol and start collaborating.
