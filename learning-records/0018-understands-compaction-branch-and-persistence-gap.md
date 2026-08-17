---
name: understands-compaction-branch-and-persistence-gap
description: Understands automatic/overflow compaction (keep-recent + leaf move), branch-vs-compaction difference, and the harness-yield→persistence timing gap
metadata:
  type: project
---

The learner closed the Card 13→14 gap and now understands the full local durability chain: harness yields → CodingSession consumes → `_persist_messages_since` snapshots `harness.messages[persisted_count:]` → append `MessageEntry`+`LeafEntry` pairs to JSONL. They correctly identified the **append-lag**: a message enters `harness.messages` one boundary after its own `MessageEndEvent` (assistant appended at `loop.py:132` after `_assistant_events` returns; tool result appended at `loop.py:155` after its end event), so persistence writes the *previous* boundary's messages, never the current one. `session.py:1606`'s post-run persist is required, not defensive.

**Why:** Distinguishing "state.messages is the replay output" from "JSONL is the durable truth" is the core mental model for local recovery interviews.

**How to apply:** Learner confirmed three summary statements:
1. Auto/overflow compaction appends one `CompactionEntry` (with `replaces_entry_ids`) + moves leaf onto it; keeps ~tail 20k tokens (`DEFAULT_COMPACTION_KEEP_RECENT_TOKENS=20000`), trigger at `context_window - 16384` reserve.
2. Compaction updates `state.messages` during replay (`_apply_compaction` at `memory.py:106`), but real messages stay on disk and parent chain — never cut; branch *does* cut the chain (leaf to an old entry), compaction never does.
3. `tau_agent.session` provides portable primitives (`entries.py`, `storage.py` Protocol, `jsonl.py`, `tree.py:path_to_entry`, `memory.py:SessionState.from_entries`); real persistence + state assembly live in `tau_coding/session.py:CodingSession`.

Also covered: incremental compaction (`_split_previous_compaction_summary` strips the `"Previous conversation summary:\n"` prefix, puts old summary in `<previous-summary>` tags, uses `UPDATE_SUMMARIZATION_PROMPT`); CompactionEntry renders as a leading `UserMessage` in provider context; branching to pre/post-compaction nodes bypasses vs. passes through the compaction transform.

Lessons regenerated on 2026-08-11 to match this discussion: `0014-map-local-session-code.html` (added persist snapshot model + append-lag timing table), `0015-recover-session-from-jsonl.html` (added file model: one session = one JSONL, one message = two entries, LeafEntry-as-selector), `0016-branch-without-rewriting-history.html` (added branch-vs-compaction chain-cut comparison), and new `0017-compact-without-losing-history.html` (trigger strategies, keep-recent 20k + boundary correction, replay replace, incremental compaction).

Next: branch/compaction are understood — ready to move toward the Card 15 secondary-development exercise. See [[refines-mission-toward-local-recovery]] and [[understands-local-streaming-event-path]].
