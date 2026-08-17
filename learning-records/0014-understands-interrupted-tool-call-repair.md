# Understands interrupted tool-call repair as protocol compensation

The learner completed Card 11 (Reliability). They correctly selected the repair strategy for an orphan tool call: append one `ToolResultMessage` whose `tool_call_id` matches the unresolved `ToolCall.id` and whose `is_error` is true, rather than deleting history or automatically rerunning the tool.

They initially assumed that the tool had completed before the result was lost. This was corrected using the uncertainty window: from an orphan call alone, the host cannot know whether execution never started, stopped midway, or completed its external side effect before the result was persisted. The learner then correctly distinguished repair from re-execution and explained that a deployment must not be retried by default. The precise reason was refined from general danger to possible duplicate side effects under uncertain completion.

Implications: the learner understands repair as an idempotent compensating message that restores protocol completeness and replayability. Later state/durability work should revisit the separate requirements for safe automatic retries: idempotency keys, durable execution records, and status reconciliation.
