# Understands the simple-turn exit path and run-delta semantics

The learner completed Card 06 (simple turn) by predicting the loop exit correctly on the first try: for a pure-text answer, has_more_tools becomes False, the inner `while has_more_tools or pending` loop exits, then break terminates the run. After one correction they understood follow-up semantics: a follow-up does not append events to the current turn but returns to the outer `while True`, opening a brand-new turn within the same run (first_turn flag prevents a duplicate initial TurnStartEvent). They also correctly identified the early-return path for error/aborted stop_reason that bypasses break.

The sharpest new insight this card: `AgentEndEvent.messages` carries `new_messages` — the delta of messages newly added during this run (assistant message, tool results, prompts) — not the full persistent history owned by the harness. This established the delta-vs-full-state distinction: new_messages is per-run recording state, harness._messages is the durable conversation state. This maps directly to the state & durability card (Card 14) later.

Implications: turn boundaries and exit paths are solid. Ready to move into Card 07 (tool cycle: how a tool result returns to model context), where the learner already has deep runtime context from the earlier tool-execution deep-dive.
