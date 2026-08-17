# Card 13 — Streaming: Primary-Source Findings

Research for the Card 13 lesson ("provider stream → stable client event contract:
slow consumers, backpressure, disconnect, replay"). Every substantive claim is
cited to an official spec, doc, or the Tau source itself.

Two things are deliberately kept separate in this file:

1. **Tau's current in-process model** — what the repo actually does today (an async
   generator driven by a sequential in-process listener fan-out, with no transport
   buffer, no disconnect, and no event-stream replay).
2. **Distributed transport semantics** — how production event systems (AnyIO bounded
   streams, SSE, WebSocket, Kafka) solve slow consumers, backpressure, disconnect
   recovery, and offset/event-ID replay.

---

## 1. Tau's current in-process streaming model

### 1.1 The whole pipeline is pull-based async generators

- `run_agent_loop` is an `async def` that `yield`s `AgentEvent` objects, typed
  `AsyncIterator[AgentEvent]`
  (`src/tau_agent/loop.py:44-58`). Calling it returns an async generator; nothing
  executes until a consumer drives it with `async for` (PEP 525, §2 below).
- Provider streaming is the same shape: the `ModelProvider` protocol exposes
  `stream_response(...) -> AsyncIterator[AssistantMessageEvent]`
  (`src/tau_agent/provider.py:19-32`). The loop consumes it with `async for event
  in source` (`src/tau_agent/loop.py:206`).
- `AgentHarness._run` drives the loop generator: `async for event in run_agent_loop(...)`
  then `await self._notify(event); yield event`
  (`src/tau_agent/harness.py:161-184`). The harness re-exposes its own
  `AsyncIterator[AgentEvent]` (`prompt` / `continue_`, `harness.py:146-159`).

Net effect: production advances **one step per consumer request**. There is no
buffering anywhere in the chain, so the producer can never run ahead of the
consumer. This is the pull-based model.

### 1.2 `AgentHarness._notify` — sequential in-process fan-out with zero-buffer backpressure

```python
async def _notify(self, event: AgentEvent) -> None:
    for listener in list(self._listeners):
        result = listener(event)
        if isawaitable(result):
            await result
```
(`src/tau_agent/harness.py:192-196`)

- Listeners are plain callables registered in-process via `subscribe()`
  (`harness.py:107-114`). There is no queue, no ring buffer, no drop policy.
- `_notify` **awaits each listener inline before `_run` yields the event**
  (`harness.py:183-184`). A slow or blocking listener stalls the whole run: the
  next event cannot be produced until every listener has finished with the current
  one. The learner's own Card 10 record states the propagation order exactly:
  "loop `yield` → harness `await _notify` → harness `yield` → downstream requests
  the next event," and that "a slow listener delays production of the next event"
  (`learning-records/0013-understands-loop-harness-architecture-split.md`).
- Consequences worth naming explicitly: (a) **no backpressure buffer** — a fast
  producer is coupled 1:1 to the slowest listener; (b) **no disconnect** — a
  listener that stops being consumed simply blocks forever; (c) **no replay** — the
  live `AgentEvent` stream is transient; there is no offset or ID to resume from.

### 1.3 What durability Tau does have — and what it is not

- Session persistence is an **append-only JSONL log** of `SessionEntry` rows, each
  with `id`, `parent_id`, and `timestamp`
  (`src/tau_agent/session/entries.py:15-33, 25-33`; `JsonlSessionStorage.append`
  opens in `"a"` mode, `src/tau_agent/session/storage.py:24-34`).
- `SessionState.from_entries(...)` **replays** entries into a transcript, and
  `path_to_entry(...)` replays only the root-to-leaf path for a branch
  (`src/tau_agent/session/memory.py:36-58`; `src/tau_agent/session/tree.py:22-40`).
- This is durable **state** replay, not **event-stream** replay: it reconstructs the
  conversation from an append-only log, but it is not a live transport with consumer
  offsets, per-subscriber cursor, or incremental resume of the in-flight event
  stream. It is the right *state-side* analog to mention (compare Kafka §6), but it
  must not be conflated with transport replay.

---

## 2. Python async generators: pull-based flow control (official spec)

Source: PEP 525 — Asynchronous Generators, https://peps.python.org/pep-0525/

- An async generator is `async def` containing `yield`; calling it returns an async
  generator object that implements the async iteration protocol from PEP 492.
- The protocol is **consumer-driven**: `__anext__()` "returns an awaitable, that
  performs one asynchronous generator iteration when awaited," so the body advances
  one step **per consumer request**. There is no producer push unless you call
  `asend(val)` / `athrow(...)`, which exist explicitly "to push data and throw
  exceptions into asynchronous generators."
- Finalization is cooperative: `aclose()` "throws a `GeneratorExit` exception into
  the generator"; event loops hook GC via `sys.set_asyncgen_hooks()` and asyncio
  schedules `create_task(gen.aclose())`; `loop.shutdown_asyncgens()` "will schedule
  all currently open asynchronous generators to close."
- `yield` inside a `finally` is a `RuntimeError`.

This is the normative basis for why Tau's stream is pull-based: `async for` over
`run_agent_loop` advances the loop only when the harness asks for the next event.

### Bounded queues for backpressure (official docs)

Source: asyncio.Queue, https://docs.python.org/3/library/asyncio-queue.html

- `maxsize` semantics: "If *maxsize* is less than or equal to zero, the queue size
  is infinite. If it is an integer greater than 0, then `await put()` blocks when
  the queue reaches *maxsize* until an item is removed by `get()`."
- Empty behavior: "Remove and return an item from the queue. If queue is empty,
  wait until an item is available."
- Non-blocking variants raise: `get_nowait()` raises `QueueEmpty` if nothing is
  immediately available; `put_nowait(item)` raises `QueueFull` if no slot is free.
- This is the canonical bounded-buffer backpressure primitive: a full buffer makes
  the **producer** wait on the **consumer**, which is exactly what Tau's inline
  `await _notify(...)` approximates (the harness waits on the slowest listener)
  but without any buffer between listener and producer.

---

## 3. AnyIO bounded memory object streams / backpressure (official docs)

Sources:
- AnyIO streams guide, https://anyio.readthedocs.io/en/stable/streams.html
- AnyIO API reference, https://anyio.readthedocs.io/en/stable/api.html

- `create_memory_object_stream()` returns "a pair of object streams: one for
  sending, one for receiving" (`MemoryObjectSendStream` / `MemoryObjectReceiveStream`).
  API signature: `create_memory_object_stream(max_buffer_size: float = 0, item_type=None)`.
- `max_buffer_size` is "the number of items held in the buffer until `send()` starts
  blocking." With the default `0`, "send() will block until there's another task
  that calls receive()" — i.e., a synchronous handoff/rendezvous. "It is also
  possible to have an unbounded buffer by passing `math.inf` as the buffer size but
  this is not recommended."
- Backpressure is therefore **sender-side blocking**: a full buffer suspends the
  sender until a receiver consumes, which throttles the producer to the consumer's
  rate — the same coupling Tau gets for free from inline awaits, but now decoupled
  through an explicit buffer between tasks.
- Endpoint lifecycle: streams "can be cloned by calling the `clone()` method"; each
  clone closes separately, and "the send stream will start raising
  `BrokenResourceError` only when both receive streams have been closed."
  `MemoryObjectReceiveStream.receive()` raises `EndOfStream` "if this stream has
  been closed from the other end" and `ClosedResourceError` if explicitly closed.

Relevance: AnyIO memory object streams are the standard in-process building block
for a **decoupled producer/consumer with explicit bounded-buffer backpressure** — a
natural upgrade path from Tau's current listener fan-out, and the same shape a
distributed transport would take.

---

## 4. WHATWG Server-Sent Events: reconnect semantics, `id` / `Last-Event-ID` (official spec)

Source: HTML Standard §9.2 (Server-Sent Events),
https://html.spec.whatwg.org/multipage/server-sent-events.html

- **`id` field (§9.2.6 Interpreting an event stream):** a field named `id` "set[s]
  the last event ID buffer to the field value" (NULL-containing values are
  ignored). The buffer persists, so the last ID applies to all subsequent events
  until the next `id` field. It is exposed to the application as
  `MessageEvent.lastEventId`.
- **`Last-Event-ID` request header (§9.2.4):** during **reconnection only**, "if the
  last event ID string is not empty, ... set the `Last-Event-ID` HTTP header" to its
  value. The server uses this to skip ahead and resend only the missed events. This
  is the spec's built-in **resume/replay marker**.
- **Reestablish the connection (§9.2.3 Processing model):** on a network error, or
  when the response body simply ends, the user agent reconnects: it queues a task
  that sets `readyState` to `CONNECTING` and fires an `error` event, waits the
  "reconnection time" (with optional exponential backoff), then re-fetches,
  attaching `Last-Event-ID` if set. **Failure is terminal:** "if res's status is not
  200, or if res's Content-Type is not text/event-stream, then fail the connection"
  — failing means `readyState` becomes `CLOSED`, an `error` fires, and "it does not
  attempt to reconnect." (The non-normative intro notes HTTP 204 No Content can
  tell a client to stop reconnecting.)
- **`retry` field (§9.2.6):** a `retry:` field of ASCII digits sets the stream's
  "reconnection time"; the initial value "must be an implementation-defined value,
  probably in the region of a few seconds."
- **Event types (§9.2.2 / §9.2.3):** `open` (on successful connection), `message`
  (default; the `event:` field can override the type), `error` (on entering
  `CONNECTING` or on `CLOSED` failure).

Relevance: SSE is the canonical **server-to-client, one-way** event transport with
reconnect-and-replay built into the spec via `id` + `Last-Event-ID`. It answers the
Card 13 question "how does a client resume after a disconnect" at the protocol
level: the server must honor `Last-Event-ID` and replay from that point. Tau's
in-process listeners have no equivalent — a listener that misses events has nothing
to resume from.

---

## 5. WebSocket RFC 6455: disconnect/closure semantics, no built-in replay (official spec)

Source: RFC 6455, The WebSocket Protocol, https://www.rfc-editor.org/rfc/rfc6455.html

- **Closing handshake (§5.5.1):** either peer sends a Close control frame (opcode
  0x8). "The application MUST NOT send any more data frames after sending a Close
  frame." On receiving a Close when it has not sent one, "the endpoint MUST send a
  Close frame in response" (and SHOULD do so "as soon as practical"). It is "safe
  for both peers to initiate this handshake simultaneously" (§1.4).
- **Connection teardown (§5.5.1, §7.1.1):** an endpoint "considers the WebSocket
  connection closed and MUST close the underlying TCP connection" only after both
  sending and receiving a Close. "The server MUST close the underlying TCP
  connection immediately"; the client "SHOULD wait for the server to close the
  connection." §7.1.1: "To *Close the WebSocket Connection*, an endpoint closes the
  underlying TCP connection."
- **Abnormal closure / 1006 (§7.4.1):** 1006 is a **reserved** code: "1006 is a
  reserved value and MUST NOT be set in a Close frame by an endpoint. It is
  designated for use in applications expecting a status code to indicate that the
  connection was closed abnormally, e.g., without sending or receiving a Close
  control frame." It is never transmitted on the wire — the local stack synthesizes
  it when the TCP connection dies without a closing handshake. Related codes: 1000
  "indicates a normal closure, meaning that the purpose for which the connection
  was established has been fulfilled"; 1001 "indicates that an endpoint is 'going
  away', such as a server going down or a browser having navigated away from a
  page"; 1002 is protocol error.
- **No built-in replay (§1.5):** the design philosophy is that WebSocket is "as
  close to just exposing raw TCP to script as possible." There is **no message ID,
  no acknowledgement, no resume/Last-Event-ID, and no retransmission across
  reconnects** in the protocol. Message numbering exists only within a single
  connection (§5.4 fragmentation is for unknown-size messages, not recovery).
  Recovery is the application's job: after an abnormal closure the client must
  reconnect on its own and the server must re-establish context (e.g., by an
  application-level resume token, offset, or full re-request).

Relevance: WebSocket gives a **bidirectional, ordered, in-connection** stream with a
clean closing handshake and explicit "abnormal closure" reporting — but **zero
replay support**. This is the sharpest contrast with SSE §4 (which has `Last-Event-ID`
built in) and Kafka §6 (offset-based replay built in). For Tau Card 13: a
WebSocket-hosted agent stream would need application-level replay (e.g., resume
from the last delivered event ID) on top of the protocol.

---

## 6. Durable event streams: Kafka — offset/event-ID replay (official docs)

Sources:
- Apache Kafka Introduction, https://kafka.apache.org/intro
- Apache Kafka `KafkaConsumer` javadoc, https://kafka.apache.org/23/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html

Kafka is the canonical example of a **durable, replayable** event stream, and the
official docs give precise language for the offset/event-ID concept.

- **Append-only, ordered, re-readable (§"How Kafka works", Intro):**
  "When a new event is published to a topic, it is actually appended to one of the
  topic's partitions," and a consumer "will always read that partition's events in
  exactly the same order as they were written." "Unlike traditional messaging
  systems, events are not deleted after consumption" — they "can be read as often
  as needed."
- **Retention (Intro):** "you define for how long Kafka should retain your events
  through a per-topic configuration setting, after which old events will be
  discarded." Retention is **independent of consumption** — this is what makes
  replay possible at all.
- **Offset = position/ID (KafkaConsumer javadoc):** "Kafka maintains a numerical
  offset for each record in a partition. This offset acts as a unique identifier of
  a record within that partition, and also denotes the position of the consumer in
  the partition." "The position of the consumer gives the offset of the next record
  that will be given out."
- **Commit → resume (KafkaConsumer javadoc):** "The committed position is the last
  offset that has been stored securely. Should the process fail and restart, this
  is the offset that the consumer will recover to." This is exactly the durable
  cursor SSE's `Last-Event-ID` approximates at a coarser granularity.
- **Replay/seek (KafkaConsumer javadoc):** "Kafka allows specifying the position
  using `seek(TopicPartition, long)` to specify the new position. This means a
  consumer can re-consume older records, or skip to the most recent records." This
  is the authoritative mechanism for replaying from an arbitrary event-ID/offset.
- **Consumer groups / rebalancing (KafkaConsumer javadoc):** "All consumer instances
  sharing the same `group.id` will be part of the same consumer group." "Kafka will
  deliver each message in the subscribed topics to one process in each consumer
  group," "balancing the partitions between all members in the consumer group so
  that each partition is assigned to exactly one consumer in the group." "If a
  process fails, the partitions assigned to it will be reassigned to other
  consumers in the same group" — "This is known as rebalancing the group."

Relevance: offset = durable per-record ID, commit = stored cursor, seek = explicit
replay, group rebalancing = handoff on consumer failure. This is the vocabulary for
"durable event stream with event-ID replay" that the Card 13 exercise asks the
learner to reason about. Tau's `SessionEntry.id` (append-only log, §1.3) is a
rudimentary local analog, but it is used for state reconstruction, not for
per-consumer transport cursor/replay.

---

## 7. Tau in-process vs. distributed transport: side-by-side

| Concern | Tau today (in-process) | Distributed transport |
|---|---|---|
| Production model | Pull-based async generators (`run_agent_loop`, `stream_response`) — producer advances per `__anext__` (PEP 525, §2) | Push with buffering; producer may run ahead up to buffer size (AnyIO §3, asyncio.Queue §2) |
| Slow consumer | `await self._notify(event)` blocks the whole run on the slowest listener (`harness.py:192-196`) | Sender blocks on a full buffer (backpressure) or drops per policy (AnyIO §3, asyncio.Queue §2) |
| Buffer between producer and consumer | None | Bounded memory object stream / queue (AnyIO `max_buffer_size`, asyncio `maxsize`) |
| Disconnect | Not a concept — listeners are in-process callables; no reconnection | SSE auto-reconnect with `Last-Event-ID` (§4); WebSocket app-level reconnect, 1006 abnormal closure (§5); Kafka consumer-group rebalance (§6) |
| Replay of missed events | None for the live stream; only whole-state reconstruction from the append-only session JSONL (§1.3) | SSE `id`/`Last-Event-ID` (§4); Kafka `seek()` to an offset (§6); WebSocket: application-layer only (§5) |
| Durable cursor | No consumer cursor; session entries have `id`/`parent_id` but no per-subscriber position (§1.3) | Kafka committed offset; SSE `last event ID string` |
| Multi-subscriber | N listeners, all awaited inline, 1:1 coupling to slowest | Each receive clone / consumer group member gets its own delivery and position (AnyIO §3, Kafka §6) |

---

## 8. Suggested Card 13 interview exercise (maps to the course's stated exercise)

The course map frames Card 13's interview exercise as: "讨论慢消费者、背压、断线与重放"
(discuss slow consumers, backpressure, disconnect, and replay),
`reference/architecture-course-map.html` (Card 13).

A defensible one-line thesis the learner can build on:

> Tau's current stream is a **pull-based in-process async generator with sequential
> listener fan-out** — that gives it natural backpressure (a slow listener stalls the
> producer) but **no buffer, no disconnect handling, and no replay**. Turning it into
> a stable client contract requires adding, in order: (1) a bounded buffer/queue with
> sender-side blocking (asyncio.Queue / AnyIO memory object streams) to decouple
> producer from the slowest consumer; (2) a durable event ID + client-supplied resume
> marker (SSE `id`/`Last-Event-ID`, or Kafka-style offsets) so a reconnecting consumer
> can resume where it left off; and (3) an explicit close/reconnect protocol (WebSocket
> closing handshake / 1006 abnormal closure) so the server can tell a normal end from a
> dropped connection and decide whether to replay.

---

## Source index

| Topic | Source | URL |
|---|---|---|
| Async generators (pull-based) | PEP 525 | https://peps.python.org/pep-0525/ |
| Bounded queue backpressure | asyncio.Queue docs | https://docs.python.org/3/library/asyncio-queue.html |
| AnyIO memory object streams | AnyIO streams guide | https://anyio.readthedocs.io/en/stable/streams.html |
| AnyIO API (`create_memory_object_stream`) | AnyIO API reference | https://anyio.readthedocs.io/en/stable/api.html |
| SSE reconnect / `id` / `Last-Event-ID` | WHATWG HTML Standard §9.2 | https://html.spec.whatwg.org/multipage/server-sent-events.html |
| WebSocket close / 1006 / no replay | RFC 6455 | https://www.rfc-editor.org/rfc/rfc6455.html |
| Kafka durable log / retention | Apache Kafka Intro | https://kafka.apache.org/intro |
| Kafka offset / commit / seek / groups | KafkaConsumer javadoc | https://kafka.apache.org/23/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html |
| Tau in-process model | Repo source | `src/tau_agent/harness.py`, `loop.py`, `provider.py`, `session/` |
| Tau learner record (Card 10/13 implication) | Repo note | `learning-records/0013-understands-loop-harness-architecture-split.md` |
