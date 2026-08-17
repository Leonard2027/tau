# Understands tool execution chain, runtime management, and concurrency model

The learner traced the full tool lifecycle: product-layer executor → AgentTool contract → AgentHarnessConfig → Harness passes to run_agent_loop → loop indexes by name and dispatches → AgentToolResult → loop wraps as ToolResultMessage → message history. They correctly identified that the actual file-reading logic belongs to `tau_coding`'s executor, not to Harness or loop.

After initial confusion about Harness vs loop responsibilities, the learner understood that Harness manages the run lifecycle (single-flight guard, cancellation token, queues, listeners) while loop performs tool lookup and sequential dispatch. They also grasped the cooperative cancellation model (flag-setting, not task.cancel()) and asked unprompted about parallelism, leading to the distinction between `execution_mode` as declarative metadata vs actual sequential `for call in calls` execution, and between loop-level scheduling vs tool-internal asyncio tasks (e.g. bash's process+cancel wait).

Implications: ready to study events and then the loop/harness interaction in depth (Cards 05–09), with enough runtime context to make those lessons concrete rather than abstract.
