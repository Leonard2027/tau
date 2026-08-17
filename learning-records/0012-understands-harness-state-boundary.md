# Understands the stateless loop / stateful harness boundary

The learner completed Card 09 (AgentHarness) and with it the first part of the course (tau_agent core, Cards 01–09). They correctly predicted the three state categories the harness must own (lifecycle, queues, message history), and classified all five items correctly on the first pass using the "does the state survive across runs" criterion: turn counter/has_more_tools are loop-local; _running guard, steering/follow-up queues, and the cancellation token are harness-owned; appending assistant results to messages is a loop action while the message list itself is harness-held.

They connected this back to the earlier tool-execution deep-dive: the sequential `for call in calls` and the declarative execution_mode metadata are both evidence of the loop staying stateless, leaving parallelism policy to the host.

Implications: the first part of the course is complete. The learner now has the full mental model of the portable tau_agent core and is ready to pivot to Part 2 (Cards 10–16), the Agent backend/platform interview track. Card 10 (Loop vs Harness) frames the loop/harness split as an architectural-decision question the learner can now answer from the source they just studied.
