# Card 14 — Durable Local Session Log: Primary-Source Findings

Research for the Card 14 lesson ("session entries, storage, tree/path replay,
branches, compaction, and recovery" — see
`learning-records/0016-refines-mission-toward-local-recovery.md`). Every
substantive claim is cited to an official spec, doc, or the Tau source itself.

Per the learner's own course direction, this is deliberately **local-first**:
the external references below are chosen because they are *directly useful* to a
single-machine, single-process coding agent (SQLite's own append-only WAL and
atomic-commit essays, the JSON Lines file format, Git's branch/ancestry model,
and LLM-provider context compaction). No WebSocket, multi-tenant, or distributed
platform material is included except one clearly-labeled one-line note.

As in the Card 13 reference file, two things are deliberately kept separate:

1. **Tau's current implementation** — what the repo actually does today (an
   append-only JSONL session log with parent-pointer branching, replay-based
   crash recovery, and `CompactionEntry`/`BranchSummaryEntry` summarization).
2. **External design guidance** — what primary sources say about durable
   append-only logs, crash recovery, DAG/branch ancestry, and context
   compaction, mapped to the same concerns.

---

## 1. Tau's current model

### 1.1 Append-only JSONL session log

- The durable unit is a `SessionEntry`, a discriminated union of entry types
  (`MessageEntry`, `ModelChangeEntry`, `ThinkingLevelChangeEntry`,
  `CompactionEntry`, `BranchSummaryEntry`, `LabelEntry`, `LeafEntry`,
  `SessionInfoEntry`, `CustomEntry`), each inheriting
  `BaseSessionEntry` with `id: str`, `parent_id: str | None`, and
  `timestamp: float` (`src/tau_agent/session/entries.py:25-32, 103-114`).
- Persistence is a plain JSONL file. `JsonlSessionStorage.append` opens the
  file in **append mode** and writes one serialized entry per line:

  ```python
  with self.path.open("a", encoding="utf-8") as file:
      file.write(entry_to_json_line(entry))
  ```
  (`src/tau_agent/session/storage.py:30-34`). `read_all` treats a missing file
  as an empty session and splits only on `"\n"` — deliberately not
  `str.splitlines()` — because U+2028 and friends can appear unescaped inside
  JSON string values (`src/tau_agent/session/storage.py:36-42`).
- Serialization is canonical Pi wire shape via a pydantic `TypeAdapter`
  (`src/tau_agent/session/jsonl.py:19-21`). Deserialization **migrates old
  persisted Tau-v1 messages** at the persistence boundary so user history stays
  readable across format changes (`src/tau_agent/session/jsonl.py:45-111`).

### 1.2 Crash recovery = replay the log into state

- `CodingSession.load` calls `storage.read_all()` and then replays the entries
  into a `SessionState`:

  ```python
  entries = await config.storage.read_all()
  ...
  linear_state = SessionState.from_entries(entries)
  latest_leaf = _latest_leaf_entry(entries)
  state = (
      SessionState.from_entries(entries, leaf_id=latest_leaf.entry_id)
      if latest_leaf is not None
      else linear_state
  )
  ```
  (`src/tau_coding/session.py:297-323`). So "recovery" today means: re-open the
  file, re-run the reducer. A file truncated at a line boundary simply yields a
  decode error for that line (`SessionJsonlError`, `jsonl.py:15-17, 24-32`).
- Dangling parent pointers are tolerated at load time: `_detach_missing_parents`
  rewrites any `parent_id` that does not resolve to an id in the log to `None`
  (`src/tau_coding/session.py:2154-2162, 315`). This is how imported/partial
  history is normalized, and it also bounds the damage of a crash that left an
  orphaned entry whose parent never landed.
- Interrupted tool calls are **repaired on load**: `_persist_loaded_interrupted_tool_repairs`
  appends the synthetic tool results the harness needs so providers do not
  reject a resumed transcript (`src/tau_coding/session.py:1728-1763`).

### 1.3 Durability level today — and what is missing

- The only durability mechanism is the POSIX append mode of the file handle.
  There is **no `fsync`/`flush`, no atomic-rename write, and no checksum**
  anywhere in `src/tau_agent/session/` (verified: no `fsync`/`flush`/`os.replace`
  calls in the session package).
- Concretely, each `append` is a single buffered `file.write(...)` of one
  complete line. This gives a *crash-window* property (each line is written by
  one write call, so a process crash mid-write is less likely to produce a
  half-line) but **no power-loss durability**: nothing syncs the page cache to
  stable storage, so an OS/hardware crash can drop the tail of the log. This is
  exactly the gap the SQLite sources in §3–4 reason about.

### 1.4 Branching histories: parent pointers form a tree/DAG

- Entries are linked by `parent_id` into a tree (a DAG with a single parent per
  node). The active branch is reconstructed by walking parent pointers from a
  leaf back to the root:

  ```python
  def path_to_entry(entries, leaf_id):
      # walk current_id = entry.parent_id until None, then reverse
  ```
  (`src/tau_agent/session/tree.py:22-40`). The walk detects cycles
  (`tree.py:31`) and missing parents (`tree.py:35`), and duplicate ids are
  rejected (`tree.py:12-19`).
- **The branch pointer is itself an append-only entry**: a `LeafEntry`
  (`entries.py:79-83`) is appended after each new message and after each
  branch/compaction move. `_latest_leaf_entry` scans the log from the end and
  takes the last `LeafEntry` as the active branch
  (`src/tau_coding/session.py:2173-2177`), then `SessionState.from_entries`
  replays only the root-to-leaf path for that leaf (`memory.py:36-58`). Because
  the leaf pointer is *in the same append-only log*, moving between branches is
  just appending a new leaf — the old branch's entries are never deleted.
- `_persist_messages_since` writes, for each completed harness message, a
  `MessageEntry(parent_id=self._last_parent_id, ...)` followed by a
  `LeafEntry(parent_id=entry.id, entry_id=entry.id)` so the current branch is
  observable while a run is still active
  (`src/tau_coding/session.py:1765-1785`).
- `branch_to_entry(entry_id, summarize=False, ...)` moves the active leaf to a
  previous branchable entry. With `summarize=True` it first summarizes the
  messages that will be abandoned and appends a **`BranchSummaryEntry`**
  (`entries.py:64-69`; `session.py:524-543`), then appends a new `LeafEntry`
  pointing at the branch point and re-replays state
  (`session.py:548-553`). The harness is rebuilt from the replayed path
  (`session.py:553`).
- Full-tree rendering (for the tree picker / export) builds a children-by-parent
  map and walks it iteratively (depth-first, cycle-`expanded` guarded) rather
  than recursively, so deep sessions cannot blow the recursion limit
  (`src/tau_coding/session.py:2214-2251`).

### 1.5 Context compaction/summarization

- A `CompactionEntry` stores `summary: str` and `replaces_entry_ids: list[str]`
  (`entries.py:56-61`). During replay, `_apply_compaction` removes the listed
  message ids from the active transcript and inserts **one** summary
  `UserMessage` in their place (`memory.py:106-125`), formatted as
  `"Previous conversation summary:\n<summary>"` (`memory.py:128-129`). The
  compacted-away entries **remain in the log** — compaction only affects the
  replayed active path.
- Compaction is triggered by estimated context size. `_maybe_auto_compact`
  fires when the estimate exceeds `auto_compact_token_threshold`
  (`session.py:1961-1974`), which is either configured or derived from the
  model context window minus a reserve (`context_window.py:166-170`).
  `_recent_preserving_compaction_plan` keeps the most recent
  `DEFAULT_COMPACTION_KEEP_RECENT_TOKENS` (20 000) tokens and summarizes
  everything older (`session.py:2037-2055`). Manual compaction
  (`session.compact`) summarizes the whole active context
  (`session.py:1410-1421`).
- The summary is model-generated with a structured prompt:
  `SUMMARIZATION_SYSTEM_PROMPT` (`context_window.py:26-32`) plus either a
  first-time `SUMMARIZATION_PROMPT` (Goal / Constraints / Progress / Key
  Decisions / Next Steps / Critical Context, `context_window.py:34-60`) or an
  `UPDATE_SUMMARIZATION_PROMPT` when a previous compaction summary is detected
  (`context_window.py:62-93, 204-224, 268-281`). The summarizer runs as a
  tool-less provider call via `_generate_compaction_summary`
  (`session.py:1976-2007`). If the model call fails or returns nothing, a
  **deterministic fallback** `summarize_messages_for_compaction` builds a plain
  line-per-message digest (`context_window.py:193-201`), and branch summaries
  likewise fall back to it (`session.py:2009-2026`).

### 1.6 Session index (metadata sidecar)

- In addition to the per-session log, `SessionManager` keeps an append-only
  metadata index at `~/.tau/sessions/index.jsonl` (`SessionRecordModel` fields:
  id, path, cwd, model, provider_name, title, created_at, updated_at)
  (`src/tau_coding/session_manager.py:17-92`). This is the resume list
  (`tau --session <id>`, `tau sessions`), separate from the transcript log.

---

## 2. JSON Lines: the file format Tau's log is written in (official spec)

Source: JSON Lines, https://jsonlines.org/

- "The JSON Lines format has three requirements": (1) UTF-8 encoding with no
  BOM; (2) "one JSON value per line" — "The most common values will be objects
  or arrays, but any JSON value is permitted"; (3) the line terminator is `'\n'`
  ("`'\r\n'` is also supported because surrounding white space is implicitly
  ignored when parsing JSON values").
- "Including a line terminator after every JSON value makes generating and
  concatenating JSON Lines files easier." (Tau's `entry_to_json_line` appends
  exactly one `"\n"` per entry, `jsonl.py:19-21`.)
- Fit for logs: "JSON Lines is a convenient format for storing structured data
  that may be processed one record at a time," "It's a great format for log
  files," and "It's also a flexible format for passing messages between
  cooperating processes." This is the spec-level justification for an
  append-only transcript: each entry is self-contained on its own line, so a
  reader can process entries incrementally and an appender can append without
  rewriting earlier bytes.

---

## 3. SQLite WAL: append-only change log with automatic crash recovery (official docs)

Source: SQLite — Write-Ahead Logging, https://www.sqlite.org/wal.html

The WAL essay is the canonical primary source for *why an append-only log
survives crashes*. It is directly applicable to Tau's JSONL because the
mechanism — commit = append a record, recovery = replay the log — is the same
shape at a higher level.

- **Append-only commit:** "A **COMMIT** occurs when a special record indicating
  a commit is appended to the WAL. Thus a COMMIT can happen without ever writing
  to the original database." And "Writers merely append new content to the end
  of the WAL file" (§"How WAL Works", §2.2). Tau's analog: every durable state
  change is an appended `SessionEntry` line; there is no in-place edit.
- **Crash recovery:** "If the last connection to a database crashed, then the
  first new connection to open the database will start a recovery process"
  (§"Sometimes Queries Return SQLITE_BUSY In WAL Mode"). Recovery is automatic
  and happens on next open — the same "reopen and replay" posture Tau takes in
  `CodingSession.load` (§1.2).
- **The log is part of the durable state:** "If a database file is separated
  from its WAL file, then transactions that were previously committed to the
  database might be lost" (§"The WAL File"). Tau's equivalent invariant: the
  JSONL *is* the state; there is no separate main file to lose, but the tail of
  the log is the only record of the newest entries.
- **Durability is a sync-level choice:** "syncing the content to the disk is
  not required, as long as the application is willing to sacrifice durability
  following a power loss or hard reboot." Writers "sync the WAL on every
  transaction commit if [PRAGMA synchronous] is set to FULL but omit this sync
  if [PRAGMA synchronous] is set to NORMAL" (§2.3). This is the vocabulary for
  Tau's current no-fsync state: it is roughly "synchronous = off" today (§1.3).
- **Concurrency (context, not requirement):** "readers do not block writers and
  a writer does not block readers" (§"Overview"). Tau is single-process so it
  does not need WAL's concurrency; the takeaway is only that append-only logs
  are designed so readers and appenders need not coordinate on a shared
  mutating file.

---

## 4. SQLite Atomic Commit: what "durable" means and what fsync buys (official docs)

Source: SQLite — Atomic Commit In SQLite, https://www.sqlite.org/atomiccommit.html

- **Atomicity definition:** "Atomic commit means that either all database
  changes within a single transaction occur or none of them occur." And:
  "SQLite has the important property that transactions appear to be atomic even
  if the transaction is interrupted by an operating system crash or power
  failure" (§"Introduction").
- **The journal-existence test (the core crash-recovery trick):** "After the
  database changes are all safely on the mass storage device, the rollback
  journal file is deleted. This is the instant where the transaction commits. If
  a power failure or system crash occurs prior to this point, then recovery
  processes ... make it appear as if no changes were ever made to the database
  file. If a power failure or system crash occurs after the rollback journal is
  deleted, then it appears as if all changes have been written to disk"
  (§3.11). "A hot journal is a rollback journal that needs to be played back in
  order to restore the database to a sane state" (§4.2).
- **Why fsync exists:** SQLite "assumes that the operating system will buffer
  writes and that a write request will return before data has actually been
  stored in the mass storage device" and that writes "will be reordered," so it
  "does a 'flush' or 'fsync' operation at key points" (§"Hardware Assumptions").
  Two flush points matter — flushing the journal before writing changes (§3.7)
  and flushing the database changes themselves (§3.10), which "is a critical
  step to ensure that the database will survive a power loss without damage."
- **Relevance to Tau:** the journal is *separate file existence*; Tau's JSONL
  log is *line existence*. Both are "did the record durably land" questions.
  The precise lesson for Card 14: an append + buffered write is not a durable
  commit until the bytes are synced to stable storage; `fsync` is the explicit
  sync barrier that buys the "survive power loss" guarantee Tau currently
  foregoes. (One-line note: SQLite also locks files to serialize writers — §3.2,
  §3.8 — which is a multi-process concern Tau's single-process design does not
  need, consistent with the local-first framing.)

---

## 5. Git: branches, ancestry, and the DAG (official docs)

Git is the reference model for *branching histories*. Tau's parent-pointer tree
is a simplified single-parent version of Git's commit DAG; the official docs
supply the exact ancestry vocabulary.

### 5.1 A branch is a movable pointer; commits form a DAG via parent pointers

Source: Pro Git — Branches in a Nutshell, https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell

- "A branch in Git is simply a lightweight movable pointer to one of these
  commits." A branch "is actually a simple file that contains the 40 character
  SHA-1 checksum of the commit it points to, so branches are cheap to create and
  destroy."
- Commits link into a graph: each commit object contains "pointers to the commit
  or commits that directly came before this commit (its parent or parents): zero
  parents for the initial commit, one parent for a normal commit, and multiple
  parents for a commit that results from a merge of two or more branches."
- **Diverging histories** are the essence of branching: "You created and
  switched to a branch, did some work on it, and then switched back to your main
  branch and did other work. Both of those changes are isolated in separate
  branches." `HEAD` "is a pointer to the local branch you're currently on."
- Direct mapping to Tau: `LeafEntry` is Tau's movable branch pointer
  (append-only, §1.4); `parent_id` is Tau's single parent pointer; a
  `branch_to_entry` move is Tau's equivalent of `git checkout` + `git branch`
  to an old commit. The key structural difference: Git allows *merge commits*
  with multiple parents (a true DAG); Tau's entries have exactly one parent, so
  Tau's history is a *tree*, not a general DAG.

### 5.2 Ancestry and merge-base (official docs)

Sources: git-merge-base, https://git-scm.com/docs/git-merge-base; gitrevisions, https://git-scm.com/docs/gitrevisions

- **Reachability is the definition of ancestry:** "Given two commits _A_ and _B_,
  `git merge-base A B` will output a commit which is reachable from both _A_ and
  _B_ through the parent relationship" (git-merge-base, §"DISCUSSION"). "A
  commit's reachable set is the commit itself and the commits in its ancestry
  chain" (gitrevisions, §"SPECIFYING RANGES").
- **Best common ancestor:** "One common ancestor is *better* than another common
  ancestor if the latter is an ancestor of the former. A common ancestor that
  does not have any better common ancestor is a *best common ancestor*, i.e. a
  *merge base*" (git-merge-base, §"DESCRIPTION"). "When the history involves
  criss-cross merges, there can be more than one *best* common ancestor"
  (git-merge-base, §"DISCUSSION").
- **Range semantics** (gitrevisions, §"Dotted Range Notations"): `r1..r2`
  = "commits that are reachable from r2 excluding those that are reachable from
  r1"; `r1...r2` is "the set of commits that are reachable from either one of
  `r1` or `r2` but not from both," and `A...B` is defined in terms of
  `git merge-base --all` — this is the "what is unique to each side of a branch
  split" query.
- **Relevance to Tau:** `path_to_entry(entries, leaf_id)` computes the
  root-to-leaf ancestry chain for a leaf, and `_messages_after_entry_on_active_path`
  (`session.py:2293-2312`) finds the messages unique to the abandoned side of a
  branch — the local, single-parent analog of `A...B`'s symmetric difference.
  There is no merge in Tau, so there is no `merge-base`; the branch point is
  simply the shared prefix of two root-to-leaf paths.

---

## 6. LLM provider context management: compaction and context engineering (official docs/research)

### 6.1 Anthropic: server-side context compaction

Source: Anthropic (Claude) docs — Compaction, https://platform.claude.com/docs/en/build-with-claude/compaction

- **Purpose:** "Compaction extends the effective context length for long-running
  conversations and tasks by automatically summarizing older context when
  approaching the context window limit. It also keeps the active context small:
  as a conversation grows, response quality degrades, so compaction replaces
  older content with a concise summary."
- **Mechanism:** "When compaction is enabled, Claude automatically summarizes
  your conversation when it reaches the configured token threshold": it detects
  the trigger, "Generates a summary of the current conversation," "Creates a
  `compaction` block containing the summary," then "Continues the response with
  the compacted context." On subsequent requests, "the API automatically drops
  all content blocks prior to the `compaction` block, continuing the
  conversation from the summary." Enabled by adding the `compact_20260112`
  strategy to `context_management.edits` (beta header `compact-2026-01-12`),
  with a token `trigger` (default 150 000, min 50 000) and optional custom
  `instructions` (which **replace** the default summary prompt).
- **Tradeoffs (what compaction loses):** the default summary prompt is written
  so the summary "provide[s] continuity ... where the raw history above may not
  be accessible and will be replaced with this summary." "Compaction requires an
  additional sampling step, which contributes to rate limits and billing." And
  "A long-running conversation might result in multiple compactions" — each
  compaction summarizes an already-summarized history, so fidelity degrades
  iteratively.
- **Direct mapping to Tau:** Tau's `CompactionEntry` is the same "summary block
  that replaces earlier context on replay" idea (§1.5), but local and
  *explicit*: `replaces_entry_ids` names exactly which entries are replaced, the
  raw entries stay in the JSONL, and the trigger is Tau's own token estimate
  rather than a server-side threshold. Anthropic's default summary prompt is
  conceptually identical to Tau's "Previous conversation summary:" framing
  (`memory.py:128-129`), and its "preserve state/next steps/learnings" guidance
  matches Tau's structured Goal/Progress/Decisions prompt
  (`context_window.py:34-60`).

### 6.2 Anthropic engineering: context rot and local techniques for agents

Source: Anthropic Engineering — Effective context engineering for AI agents, https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

- **Why compact at all ("context rot"):** "the model's ability to accurately
  recall information from that context decreases" as token count grows; "Context,
  therefore, must be treated as a finite resource with diminishing marginal
  returns"; "These factors create a performance gradient rather than a hard
  cliff." This is the *quality* rationale behind Tau's auto-compaction
  (`_maybe_auto_compact`).
- **Compaction is the first lever:** "Compaction typically serves as the first
  lever in context engineering to drive better long-term coherence." During
  summarization "The agent preserves architectural decisions, unresolved bugs,
  and implementation details" — precisely what Tau's
  `UPDATE_SUMMARIZATION_PROMPT` asks for ("PRESERVE exact file paths, function
  names, and error messages").
- **Complementary local techniques (relevant to Card 14's local-first scope):**
  - *Structured note-taking / external memory:* "the agent regularly writes notes
    persisted to memory outside of the context window," providing "persistent
    memory with minimal overhead." This is the pattern behind Tau's `CustomEntry`
    namespace (`entries.py:95-100`) and general session files.
  - *Just-in-time context loading:* agents should "dynamically load data into
    context at runtime using tools" rather than pre-loading everything.
  - *Tool-result clearing:* "one of the safest lightest touch forms of compaction
    is tool result clearing" — old raw tool outputs are often not needed again.
- **Relevance to Tau:** these are design criteria, not Tau features. Card 14 can
  note that Tau's `CompactionEntry.replaces_entry_ids` gives it the seam to do
  "tool-result-only clearing" later, because the log keeps every entry and only
  the replay decides what is included.

### 6.3 OpenAI: compaction in the Responses API

Source: OpenAI Platform docs — Compaction guide, https://developers.openai.com/api/docs/guides/compaction

- **Definition:** "To support long-running interactions, you can use compaction
  to reduce context size while preserving state needed for subsequent turns.
  Compaction helps you balance quality, cost, and latency as conversations grow."
- **Two triggers:** server-side via `context_management.compact_threshold` on a
  Responses create request ("when the rendered token count crosses the configured
  threshold, the server runs server-side compaction"), or explicit via the
  `POST /v1/responses/compact` endpoint ("fully stateless").
- **The compaction item:** "The returned compaction item carries forward key
  prior state and reasoning into the next run using fewer tokens. It is opaque
  and not intended to be human-interpretable." For latency, "you can drop items
  that came before the most recent compaction item to keep requests smaller" —
  the OpenAI analog of Tau's `replaces_entry_ids` pruning the *sent* transcript
  while the raw log is retained.
- **Relevance to Tau:** same conceptual move as Anthropic §6.1 — an explicit
  compaction boundary after which older history is dropped from the active
  context. The "opaque compaction item" is the *server-side* equivalent of Tau's
  human-readable `CompactionEntry.summary`; Tau's version is transparent and
  inspectable because the JSONL keeps the raw entries.

---

## 7. Tau today vs. external design guidance: side-by-side

| Concern | Tau today (local, from source) | External guidance (primary sources) |
|---|---|---|
| Log format | Append-only JSONL, one `SessionEntry` per line (`storage.py:30-34`) | JSON Lines: one JSON value per line, `\n`-terminated, made for log files (JSON Lines, §2) |
| Commit model | Appending a line is the state change; no in-place edits | SQLite WAL: "A COMMIT occurs when a special record indicating a commit is appended to the WAL" (§3) |
| Crash recovery | Reopen + replay `read_all()` → `SessionState.from_entries`; missing file = empty session (`session.py:296-323`); dangling parents detached (`session.py:2154-2162`); interrupted tool calls repaired on load (`session.py:1728-1763`) | WAL auto-recovers on next open (SQLite WAL §3); journal-existence test decides committed-vs-rolled-back (SQLite Atomic Commit §4) |
| Durability / fsync | None — buffered `write`, no `fsync`/flush/atomic rename (§1.3) | Durability is a sync choice: `synchronous=FULL` syncs per commit; NORMAL omits it (SQLite WAL §3); two fsync points define the atomic commit moment (SQLite Atomic Commit §4) |
| History structure | Tree via single `parent_id`; `path_to_entry` root-to-leaf (`tree.py:22-40`); branch pointer is an appended `LeafEntry` (`session.py:1765-1785, 2173-2177`) | Git: commits with parent pointers form a DAG; branch = movable pointer; `HEAD` = current branch (Pro Git §5.1) |
| Branch operation | `branch_to_entry` moves the active leaf, optionally appending a `BranchSummaryEntry` (`session.py:503-567`) | `git checkout`/`git switch` to an old commit; `A...B` = symmetric difference, `merge-base` = best common ancestor (Pro Git §5.1, gitrevisions/merge-base §5.2) |
| What's unique to a branch | `_messages_after_entry_on_active_path` — messages after a branch point on the active path (`session.py:2293-2312`) | `r1...r2` — commits reachable from either but not both (gitrevisions §5.2) |
| Compaction entry | `CompactionEntry` with `summary` + `replaces_entry_ids`; replaces entries **only during replay**; raw entries retained (`entries.py:56-61`, `memory.py:106-125`) | `compaction` block replaces content before it; older messages dropped from active context (Anthropic §6.1); prune items before the compaction item (OpenAI §6.3) |
| Compaction trigger | Local token estimate vs. threshold; keep-recent-tokens plan (`session.py:1961-1974, 2037-2055`) | Server-side token threshold (`compact_20260112` trigger; OpenAI `compact_threshold`) |
| Summarizer | Model call with structured Goal/Progress/Decisions prompt + deterministic fallback (`context_window.py:26-106, 193-224`) | Anthropic default summary prompt = continuity notes (state, next steps, learnings) (§6.1); context engineering: preserve decisions/bugs/implementation details (§6.2) |
| What compaction costs | Model sampling cost, plus quality loss from summarizing already-summarized history (both true in Tau and in the providers) | "Requires an additional sampling step, which contributes to rate limits and billing"; repeated compaction degrades fidelity (Anthropic §6.1) |
| Session listing | Sidecar metadata `index.jsonl` (`session_manager.py:17-92`) | (No external analog needed; a local convenience) |

---

## 8. Suggested Card 14 exercise thesis (local-first)

The course map frames Card 14's interview exercise around "session entries、
storage、tree/path replay、branches、compaction、recovery"
(`reference/architecture-course-map.html`; `learning-records/0016-refines-mission-toward-local-recovery.md`).

A defensible one-line thesis the learner can build on:

> Tau's durability story is **an append-only JSONL log whose own replay is the
> crash-recovery mechanism**: every state change is a `SessionEntry` line with a
> `parent_id`, the active branch is the last appended `LeafEntry`, and a session
> is restored by replaying the root-to-leaf path of that leaf. This gives Tau
> SQLite-WAL-style "commit by append, recover by replay" semantics for free,
> except for one explicit gap: **there is no fsync**, so Tau has crash-window
> safety but not power-loss durability (SQLite's `synchronous=OFF` case).
> Branching is Git's model reduced to a tree (single parent per entry; `LeafEntry`
> is Tau's branch pointer; `A...B` becomes "messages after the branch point on
> the active path"). Compaction is a provider-style summary block that replaces
> named entries *during replay only* — so the raw log stays intact and only the
> sent transcript shrinks. A local upgrade path is: add an `fsync`-after-append
> policy (per-commit FULL vs checkpoint-only NORMAL), keep the leaf-pointer
> scheme, and expose compaction as pure replay semantics on top of the immutable
> log.

---

## Source index

| Topic | Source | URL |
|---|---|---|
| JSON Lines file format | JSON Lines spec | https://jsonlines.org/ |
| Append-only log + crash recovery | SQLite Write-Ahead Logging | https://www.sqlite.org/wal.html |
| Atomic commit / durability / fsync | SQLite Atomic Commit In SQLite | https://www.sqlite.org/atomiccommit.html |
| Branches as pointers / commit DAG | Pro Git — Branches in a Nutshell | https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell |
| Common ancestor / ancestry | git-merge-base | https://git-scm.com/docs/git-merge-base |
| Reachability / range syntax | gitrevisions | https://git-scm.com/docs/gitrevisions |
| Anthropic server-side compaction | Claude docs — Compaction | https://platform.claude.com/docs/en/build-with-claude/compaction |
| Anthropic context engineering | Anthropic Engineering blog | https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents |
| OpenAI compaction | OpenAI Platform docs | https://developers.openai.com/api/docs/guides/compaction |
| Tau session log / entries | Repo source | `src/tau_agent/session/entries.py`, `storage.py`, `jsonl.py`, `tree.py`, `memory.py` |
| Tau session orchestration | Repo source | `src/tau_coding/session.py`, `context_window.py`, `session_manager.py` |
| Course trajectory | Repo note | `learning-records/0016-refines-mission-toward-local-recovery.md` |
