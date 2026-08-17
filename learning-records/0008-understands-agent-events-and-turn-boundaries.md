# Understands agent event layering and the turn causal boundary

The learner completed Card 05 (agent events) by first predicting the coupling from direct UI/logging in the loop, then classifying all 10 events into the four layers (agent, turn, message, tool). They correctly predicted the full event set and independently identified that tool results produce their own MessageStart/MessageEnd pair, but initially missed the two delta events (MessageUpdateEvent, ToolExecutionUpdateEvent).

The strongest learning signal was a three-attempt correction on the turn causal boundary. The learner first said TurnEndEvent.message is "the tool call and its result", then realized message is the assistant request (ToolCall) while tool_results is the result list, and finally nailed the retrieval check: an all-tool-call turn has ToolCall blocks in message.content and one ToolResultMessage per call in tool_results. They also connected the abstract event hierarchy to real usage: one AgentStartEvent = one prompt()/continue_() run, turns = internal model-call iterations within that run, and multiple runs share the persistent message history. The learner explicitly asked to map events onto actual tau usage, which grounded the run/turn distinction.

They correctly reasoned that MessageUpdateEvent exists because of streaming (delta events carry cumulative partials plus the nested raw provider event), and that cut points between thinking/tool-call/text come from the provider event type discriminator plus content_index.

Implications: run/turn/message/tool layering is now solid enough to proceed to Cards 06–09 (simple turn, tool cycle, control flow, harness) without re-teaching event semantics. The event stream can now serve as a reference point for the harness interplay lessons ahead.
