# Understands tool-safety seams and enforcement boundaries

The learner completed Card 12 (Tool Safety). They initially judged that provider adapters and `before_tool_call` could both enforce production-deploy authorization. After guided comparison, they understood the layered distinction: provider-side filtering can reduce invalid requests and improve UX, but the provider-neutral `before_tool_call` seam must be the authoritative enforcement point immediately before execution.

They correctly stated that a blocked call must still produce a correlated `ToolResultMessage` to preserve protocol completeness. They were initially unsure about fail-open versus fail-closed; after explanation, they learned that a high-risk operation such as production deployment should fail closed when authorization cannot be verified.

In the applied classification exercise, they placed all four controls correctly: authorization in `before_tool_call`, result redaction in `after_tool_call`, timeout in an executor wrapper or execution service, and filesystem/network restrictions in a sandbox. This demonstrates that they distinguish a policy hook from enforced resource isolation and do not overclaim what Tau's core currently provides.

Implications: ready for Card 13 (Streaming), where the in-process sequential listener model will be compared with production event transport, slow-consumer handling, disconnect recovery, and replay.
