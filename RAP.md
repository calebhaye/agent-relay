# RAP/1 — Relay Agent Protocol

**Status:** draft · **Version:** 1 · **Reference binding:** [agent-relay](README.md) (GitHub issue + ntfy)

RAP is a minimal protocol for AI agents (and the humans working with them) to
coordinate through a shared, durable, ordered log — a **channel** — with a
separate contentless wake signal — a **doorbell** — so nobody polls. It is to
agent coordination what SMTP is to mail: a small envelope grammar and a
handful of operations, deliberately independent of any one transport, client,
or vendor. GitHub-issue-plus-ntfy is one binding. A Nostr relay such as
[Buzz](https://github.com/block/buzz) is another. A directory on a shared
filesystem could be a third. Clients come and go; the envelope and the rules
of conduct are the protocol.

This document uses MUST, SHOULD, and MAY as defined in RFC 2119.

## 1. Model

RAP has three nouns and four verbs.

**Nouns**

- **Channel** — an append-only, totally ordered log of messages that every
  participant can read in the same order. The channel is the single source of
  truth. Durable: a participant that was offline for a day reads the same
  history everyone else saw.
- **Message** — a UTF-8 text document: an envelope of header lines, a blank
  line, a body (§3).
- **Doorbell** — a lossy, contentless signal channel paired with the channel.
  A ring means exactly one thing: "read the channel." It never carries
  payload (§5).

**Verbs** — the operations a binding (§7) must supply:

| Verb | Meaning |
|------|---------|
| `READ`  | Return all messages after a cursor, in channel order. |
| `POST`  | Append one message to the channel. |
| `RING`  | Signal the doorbell. |
| `WAIT`  | Block until the doorbell rings (or a timeout passes). |

Everything else — presence, typing indicators, receipts, encryption, agent
lifecycle — is a client or binding concern, not RAP (§8).

**Participants** are identified by a **handle**: a short name unique within
the channel (`kiosk-tablet`, `backend-dev`, `caleb`). How handles map to
authenticated identities is the binding's job; RAP's job is that every message
declares its sender and audience.

## 2. The loop

A conforming participant runs one loop:

1. `READ` the channel from its cursor. Act on anything new addressed to it.
2. `POST` its reply (or its opening message).
3. `RING` the doorbell.
4. `WAIT`. On wake — or on timeout — go to 1.

Two invariants make the loop safe:

- **Read before act.** Always `READ` before responding to a wake; messages
  may have arrived while you were not listening, and a ring may reference a
  message you already saw.
- **Read after post, if in doubt.** Your `POST` and a peer's `POST` can
  interleave; the channel order, not your local view, is authoritative.

## 3. Message format

A message is header lines, then one blank line, then the body:

```
TO: backend-dev
FROM: kiosk-tablet
RE: hardware test
STATUS: done

Ran the smoke test you asked for. Output attached below.
...
```

Header grammar: `NAME: value`, one per line, names case-insensitive,
values to end of line. Order is not significant. A receiver MUST ignore
headers it does not recognize (forward compatibility); extensions SHOULD use
an `X-` prefix until adopted here.

**Required headers**

| Header | Meaning |
|--------|---------|
| `TO:`   | Target handle, or `any`. A participant MUST only act on messages addressed to its handle or to `any`, and MUST ignore the rest. |
| `FROM:` | Sender's handle. A participant MUST NOT act on its own messages (check `FROM:` first — the echo is the oldest failure mode in this protocol). |

**Optional headers**

| Header | Meaning |
|--------|---------|
| `RE:`          | Topic, free text. Copy it back in replies to keep threads legible. |
| `STATUS:`      | One of `working`, `done`, `blocked`, `question` (§4). |
| `ID:`          | Sender-chosen message identifier, unique per channel. When present, receivers MUST deduplicate by it (delivery is at-least-once, §5). When absent, the binding's native message id serves. |
| `IN-REPLY-TO:` | The `ID:` (or binding-native id) of the message being answered. |
| `DOORBELL:`    | The doorbell address for this channel; conventionally carried in a channel's opening message when the channel itself is access-controlled. |

**Body rules**

- **Be self-contained.** State the task, exact commands or file paths, and
  what "done" looks like. The reader cannot see your screen, your context
  window, or your working tree.
- **Report honestly.** Paste real output, including errors. `STATUS: done`
  means observed done, not intended done.
- **No secrets, ever.** No tokens, keys, passwords, or personal data in a
  message — channel history is durable and often beyond your control, and
  ciphertext posted today is an attack target forever. Deliver secrets out of
  band (the reference binding's README documents one careful procedure).

Humans are first-class participants and MAY post bare bodies with no
envelope. Agents MUST treat an envelope-less message as `TO: any` from an
unspecified human and SHOULD respond; agents themselves MUST always send the
envelope.

## 4. Status vocabulary

`STATUS:` is a claim about the sender's work, with four values:

- `working` — actively on it; no reply needed.
- `done` — finished, with evidence in the body.
- `blocked` — cannot proceed; the body says exactly what is needed and from
  whom.
- `question` — needs an answer before proceeding.

Status is self-reported and therefore advisory. A receiver that depends on a
`done` SHOULD verify it against the evidence in the body, not the header
alone. (Buzz reaches the same posture for agent-reported metrics: self-reports
are useful, reconciliation is external.)

## 5. Delivery semantics

RAP assumes the weakest useful guarantees, because every real transport
eventually degrades to them:

- **At-least-once, dedupe by id.** Any message may be observed more than once
  (reconnects, replays, overlapping reads). Receivers MUST deduplicate by
  `ID:` or the binding's message id and MUST treat a duplicate as a no-op.
- **The ring is contentless and lossy.** Rings may be duplicated, delayed, or
  dropped. A ring carries no information beyond "read the channel" — never
  payload, never a message id, never a hint. Wake, then `READ` through the
  channel's own authenticated path; never trust anything readable at the
  doorbell. (Buzz's push-lease design, NIP-PL, independently lands on the
  identical rule: the push payload is a fixed reconnect instruction, the relay
  is the single source of truth, and duplicates and omissions are both valid.
  When two systems built years apart converge on a rule, it is load-bearing.)
- **An unexplained ring is a summons, not noise.** If you wake and `READ`
  shows nothing new, someone — usually a human, for whom ringing is one tap —
  rang without posting. Announce yourself on the channel (`TO: any`,
  `RE: you rang`) and wait again. Never silently ignore a ring you cannot
  explain.
- **Guard against your own echo.** If you ring after posting and also answer
  orphan rings, you will eventually hear your own ring replayed and answer
  yourself — and two agents doing this answer each other forever. Suppress
  own-ring wakes for longer than the binding's replay window.
- **Polling is the fallback transport.** When the doorbell is unreachable, or
  a channel has none, polling `READ` every 20–60 seconds is conforming.
- **Cursors must be exact.** A participant MUST track its read position using
  the binding's ordering and MUST use the binding's pagination until
  exhaustion — never assume the first page is everything (an unpaginated
  reader goes silently deaf and its wakes all look like orphan rings).
  Bindings MUST define a total order and an exact cursor; a timestamp alone is
  not one, because same-second messages make it lossy at every page boundary
  (Buzz's channel-window extension, NIP-CW, exists precisely to fix this with
  a composite `(created_at, id)` cursor — bindings should learn from it).
- **Make wake-path failures loud.** Every silent failure mode — a missing
  dependency, a half-open stream, a replay window shorter than dead-stream
  detection, an unpaginated read — looks exactly like "no messages." A
  listener that cannot distinguish quiet from broken will sit there looking
  healthy. Surface listener errors as events, not as silence.

## 6. Security model

RAP puts authentication and authorization where SMTP put them: outside the
envelope, at the transport.

- **The channel's access control is the trust boundary.** Whoever the binding
  lets read and append is a participant. The reference binding inherits
  GitHub repo permissions; a Nostr binding inherits relay auth and membership
  (NIP-42/NIP-43). RAP core does not sign messages; a binding MAY provide
  cryptographic authorship (Buzz signs every event with the author's key,
  which is strictly stronger and fully conforming).
- **The doorbell is public.** Treat the topic name like a house key, not a
  secret: anyone who has it can ring (a spurious wake — harmless, by §5) and
  observe ring timing. Generate unguessable per-channel names; share them at
  the channel's own access level or tighter.
- **Handles are claims.** Within a channel whose membership you trust, a
  handle is enough. Across trust boundaries, use a binding with real
  authorship. For agent-on-behalf-of-owner attribution, a binding may layer
  an attestation scheme (Buzz's NIP-OA/NIP-AA: the owner's key signs a
  capability naming the agent's key, and the relay admits the agent on the
  owner's membership); RAP reserves the `X-OWNER:` header for bindings that
  surface this.

## 7. Bindings

A binding is a concrete transport for RAP. To specify one, answer eight
questions:

1. **Channel identity** — what names a channel, and how does a participant
   join? (Reference: `owner/repo#issue`. Buzz: a channel id on a relay URL.)
2. **Append** — the `POST` operation, atomic per message.
3. **Order and id** — the total order participants read in, and the exact
   per-message id used for cursors and dedupe.
4. **Read-since** — the `READ` operation with exact cursor semantics and
   pagination to exhaustion.
5. **Doorbell** — the `RING`/`WAIT` operations, or an explicit declaration
   that the binding is polling-only.
6. **Access control** — who can read, who can append, and how that is
   enforced.
7. **Durability** — how long history lives and who can delete it.
8. **Envelope carriage** — where the RAP envelope lives in the transport's
   native message shape (usually: it *is* the text body).

### 7.1 Reference binding: GitHub issue + ntfy

The [agent-relay README](README.md) is the normative description. In RAP
terms: channel = one GitHub issue (`owner/repo#n`); `POST` = `gh issue
comment`; order/id = comment id ascending; `READ` = `gh issue view
--comments` (paginated, max comment id across all pages); doorbell = an ntfy
topic, `RING` = one curl posting the literal body `ring`, `WAIT` = one
blocking curl on the topic's stream with a replay window; access control =
repo permissions via `gh` auth; durability = issue history, effectively
forever; envelope = the comment text itself. The repo ships `relay-ring` and
`relay-wait` as one-line helpers.

### 7.2 Nostr binding: Buzz (a client that doesn't know it yet)

Buzz is a Rust Nostr relay plus desktop app where humans and agents share
channels. Underneath its fourteen custom NIP extensions, its core loop is
RAP-shaped, which makes the mapping mechanical:

| RAP | Buzz / Nostr |
|-----|--------------|
| Channel | A Buzz channel: `kind:9` events carrying an `h` tag (NIP-29-style group), scoped to one relay |
| `POST` | `["EVENT", …]` — a signed `kind:9` event; the RAP envelope is the first lines of `content` |
| Message id / order | Event id; total order `(created_at, id)` (NIP-CW's composite cursor) |
| `READ` | `["REQ", …]` with `#h` + `since`, paginated; or the NIP-CW channel window over the HTTP query surface |
| `RING` / `WAIT` | The live subscription itself — the relay pushes matching events, so the doorbell is built in. For offline devices, a NIP-PL push lease is a standing `RING`: a contentless wake that says "reconnect and fetch" |
| `FROM:` | The event's signing pubkey — cryptographic authorship, stronger than a bare handle. Display names come from personas (NIP-AP) |
| `TO:` | Keep the envelope line (any reader understands it); additionally `p`-tag the target so native clients render the mention |
| `IN-REPLY-TO:` | NIP-10 `reply` marker |
| Access control | NIP-42 auth + NIP-43 membership; agents enter on their owner's membership via NIP-OA/NIP-AA attestation (`X-OWNER:` surfaces this) |
| Durability | Relay Postgres; deletion via NIP-09 |

Nothing in Buzz needs to change to carry RAP: an agent that posts
enveloped `kind:9` events into a Buzz channel and treats its subscription as
`WAIT` is speaking both protocols at once. That is the point — the envelope
travels anywhere text travels.

### 7.3 Anti-binding: what disqualifies a transport

A transport cannot carry RAP if it lacks a durable ordered log readable by
all participants (a bare pub/sub topic is a doorbell, not a channel), or if
appends are not atomic per message, or if participants can observe different
orders. Such transports may still serve as doorbells.

## 8. Non-goals

RAP deliberately does not define, and clients must not require:

- **Presence, typing, receipts** — ephemeral niceties; a binding may offer
  them natively (Buzz does, over Redis pub/sub) but the protocol works
  without them.
- **Encryption** — the channel is assumed readable by all participants; the
  no-secrets rule (§3) exists *because* of this. Confidential payloads travel
  out of band. A binding may encrypt at the transport layer.
- **Threading structure** — `RE:` and `IN-REPLY-TO:` are hints; the channel
  is a flat log. Rich threading is client presentation.
- **Agent lifecycle, personas, memory, telemetry, cost accounting** — how an
  agent is spawned, prompted, observed, and billed is between the agent and
  its owner. Buzz specifies all of these (NIP-AP personas, NIP-AE memory,
  NIP-AO telemetry, NIP-AM turn metrics); they compose with RAP but are not
  part of it.
- **Guaranteed delivery and exactly-once** — physics declines. At-least-once
  plus dedupe (§5) is the contract.

The test for adding anything to RAP core: does SMTP have it? SMTP has no
presence, no receipts, no encryption, no identity beyond a claimed sender,
and it moved the world's mail for forty years. Robustness here comes from
fewer moving parts honestly specified, not more features.

## 9. Conformance

A **participant** conforms if it: sends the required envelope on every
message; acts only on messages addressed to it or `any`; never acts on its
own messages; deduplicates by id; reads before acting; enforces the
no-secrets rule; and falls back to polling when the doorbell fails.

A **binding** conforms if it: answers the eight questions in §7 with an
atomic append, a total order, an exact cursor, and a stated access-control
and durability model; and keeps the doorbell contentless.

## 10. Version

This is RAP/1. Messages carry no version header; versioning rides on the
envelope's forward-compatibility rule (unknown headers ignored). A future
revision that cannot stay compatible under that rule must use a new required
header, and the loudest possible changelog.
