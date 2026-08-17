# Understands the provider-neutral model contract

The learner correctly identified that a provider receives the model, system prompt, message history, tools, and cancellation signal; returns a unified asynchronous assistant-event stream; and converts vendor-specific streaming chunks inside the adapter. This demonstrates understanding of the dependency-inversion seam between `run_agent_loop` and concrete model providers.
