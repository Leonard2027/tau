# Tau Durable Session Glossary

本页是 Lessons 14–17 的本地会话机制速查表。它只描述 Tau 当前的
**single-machine / append-only session** 实现，不把进程内消费者称为网络客户端，也不把
remote replay、multi-tenant transport 或分布式一致性纳入当前主线。

- [Lesson 14：代码地图与 persist snapshot](../lessons/0014-map-local-session-code.html)
- [Lesson 15：JSONL 恢复](../lessons/0015-recover-session-from-jsonl.html)
- [Lesson 16：分支](../lessons/0016-branch-without-rewriting-history.html)
- [Lesson 17：压缩](../lessons/0017-compact-without-losing-history.html)
- [Card 14 一手资料整理](card14-durable-session-log-primary-sources.md)

## 1. 四个不要混淆的“位置”

| 术语 | 精确定义 | 不要误解成 | 主来源 |
|---|---|---|---|
| Runtime transcript | `AgentHarness.messages` 中当前进程使用的消息列表；loop 会在运行过程中向它追加完整消息。 | 已经持久化的事实；消息可能已经出现在内存里但还未写入 JSONL。 | [`loop.py:44–168`](../src/tau_agent/loop.py), [`harness.py:146–190`](../src/tau_agent/harness.py) |
| `persisted_count` | 单次 `CodingSession.prompt()` 内的增量持久化游标。`_persist_messages_since()` 读取 `harness.messages[persisted_count:]`，返回推进后的计数。 | JSONL entry ID、leaf ID 或全局 durable offset；它只是当前 prompt 调用中的消息列表索引。 | [`session.py:1561, 1577–1580, 1606, 1765–1785`](../src/tau_coding/session.py) |
| `_last_parent_id` | 追加游标：下一条新 durable entry 应使用的 `parent_id`。persist、branch、compaction 后都会更新。 | “最新写入行”的通用 ID；`LeafEntry` 本身通常不是后续消息的 parent。 | [`session.py:1765–1785, 503–553, 2060–2081`](../src/tau_coding/session.py) |
| Active replay anchor | 最新 `LeafEntry.entry_id` 选择的 entry；load/replay 从这里沿 `parent_id` 回到 root。`SessionState.active_leaf_id` 保存的也是这个 anchor ID。 | 最新 `LeafEntry` 自己的 `id`，也不是文件最后一行的任意 entry。 | [`entries.py:79–83`](../src/tau_agent/session/entries.py), [`session.py:317–323, 2173–2177`](../src/tau_coding/session.py), [`memory.py:36–80`](../src/tau_agent/session/memory.py) |

一句话区分：

> `persisted_count` 决定“内存消息从哪里继续写”，`_last_parent_id` 决定“新 entry
> 接到哪里”，active replay anchor 决定“恢复时选择哪条历史”。

## 2. Append-only entry 模型

### `BaseSessionEntry`

所有 durable entry 的公共字段：

- `id`：entry 自身的稳定标识；
- `parent_id`：当前 entry 的父节点；
- `timestamp`：创建时间。

主来源：[`entries.py:25–32`](../src/tau_agent/session/entries.py)

### 九种 `SessionEntry`

| Entry | 回答的问题 | Replay 行为 | 对应课程 |
|---|---|---|---|
| `MessageEntry` | 发生了哪条 canonical user / assistant / tool-result 消息？ | 加入 message rows。 | [14](../lessons/0014-map-local-session-code.html), [15](../lessons/0015-recover-session-from-jsonl.html) |
| `ModelChangeEntry` | 当前选择哪个 model？ | 后出现的值覆盖前值。 | [14](../lessons/0014-map-local-session-code.html) |
| `ThinkingLevelChangeEntry` | 当前 thinking level 是什么？ | 后出现的值覆盖前值。 | [14](../lessons/0014-map-local-session-code.html) |
| `CompactionEntry` | 哪些旧 message rows 在 replay 时由哪段摘要替换？ | `_apply_compaction()` 使用 `replaces_entry_ids` 替换上下文输出。 | [17](../lessons/0017-compact-without-losing-history.html) |
| `BranchSummaryEntry` | 回到旧分支点时，是否保留被放弃尾部的摘要？ | 还原成一条以 `The following is a summary of a branch...` 开头、并用 `<summary>` 包裹内容的 `UserMessage`。 | [16](../lessons/0016-branch-without-rewriting-history.html) |
| `LabelEntry` | 当前 session label 是什么？ | 后出现的 label 覆盖前值。 | [15](../lessons/0015-recover-session-from-jsonl.html) |
| `LeafEntry` | 当前选择哪一个 replay anchor？ | 将 active anchor 更新为 `entry_id`。 | [15](../lessons/0015-recover-session-from-jsonl.html), [16](../lessons/0016-branch-without-rewriting-history.html) |
| `SessionInfoEntry` | 这个 session 的 root metadata 是什么？ | 恢复 cwd/title/created-at 等 session 信息。 | [14](../lessons/0014-map-local-session-code.html), [15](../lessons/0015-recover-session-from-jsonl.html) |
| `CustomEntry` | extension/application 还追加了什么 namespaced 数据？ | 收集到 `custom_entries`，不自动变成消息。 | [14](../lessons/0014-map-local-session-code.html) |

定义与 discriminated union：[`entries.py:35–114`](../src/tau_agent/session/entries.py)。

### `MessageEntry` vs `LeafEntry`

- **`MessageEntry` 回答“发生过什么”。**
- **`LeafEntry` 回答“当前选择进行到哪里”。**

普通消息持久化时，Tau 依次 append：

```text
MessageEntry(parent_id=_last_parent_id, message=message)
LeafEntry(parent_id=message_entry.id, entry_id=message_entry.id)
```

因此“一条消息 = 两行”描述的是当前 `_persist_messages_since()` 的写入协议；消息内容和
active-history 选择是两种不同事实。主来源：
[`session.py:1765–1785`](../src/tau_coding/session.py)。

## 3. Persistence boundary 与 append-lag

### Persistence trigger

`CodingSession.prompt()` 每次看到完整 `MessageEndEvent` 时，都会调用
`_persist_messages_since(persisted_count)`。这个 event 是**检查边界**，不是当次写入对象。

### Backlog snapshot

真正的数据来源是：

```python
new_messages = self._harness.messages[persisted_count:]
```

所以 persist 的精确定义是：

> 把 runtime transcript 中已经 append、但游标尚未覆盖的 backlog 批量写入。

主来源：[`session.py:1577–1580, 1765–1785`](../src/tau_coding/session.py)。

### Append-lag

assistant 和 tool-result 都在自己的 `MessageEndEvent` 被消费后，才 append 到
`messages`：

- assistant：end event 在 `loop.py:213/217`，append 在 `loop.py:132`；
- tool result：end event 在 `loop.py:282`，append 在 `loop.py:155`。

因此某条消息通常在**后一个事件边界**才被持久化。run 结束后的
`session.py:1606` final persist 负责捕捉没有后继边界的最后一条消息；它是协议必要步骤，
不是可删除的保险代码。

详解：[Lesson 14 · append-lag](../lessons/0014-map-local-session-code.html#lag)。

## 4. Storage、文件与 resume index

| 术语 | 定义 | 边界 |
|---|---|---|
| `SessionStorage` | 可替换的 append/read port，仅定义 `append(entry)` 与 `read_all()`。 | 不负责选择 active branch 或 reduce state。 |
| `JsonlSessionStorage` | 本地实现：append mode 写一条 JSON line；读取时按文件顺序解析。 | 当前没有显式 `flush()` / `fsync()` / checksum。 |
| Session JSONL | 一个 coding session 的 durable transcript 文件；session 归属由文件路径决定。 | 不是 `SessionManager` 的索引文件。 |
| `SessionManager` | 创建、索引、列出和 resume session 的产品层目录；记录 session ID、transcript path、cwd、model、title、updated-at。 | `index.jsonl` 是 resume metadata sidecar，不是消息 transcript，也不执行 replay。 |

主来源：

- [`storage.py:12–42`](../src/tau_agent/session/storage.py)
- [`session_manager.py:15–94, 108–183`](../src/tau_coding/session_manager.py)
- [Lesson 15 · 文件模型](../lessons/0015-recover-session-from-jsonl.html#file)

## 5. Replay pipeline

### `path_to_entry(entries, leaf_id)`

从指定 anchor 开始，沿 `parent_id` 走回 root，再反转为 root-to-leaf 顺序。它会拒绝：

- duplicate entry IDs；
- cycle；
- missing entry。

主来源：[`tree.py:12–40`](../src/tau_agent/session/tree.py)。

### `SessionState.from_entries()`

纯 replay reducer。给定全部 entries 或指定 root-to-leaf path，重建：

- `messages`；
- model / thinking level / label；
- active replay anchor；
- session info / custom entries；
- compaction state 与 context entry IDs。

主来源：[`memory.py:21–103`](../src/tau_agent/session/memory.py)。

### Load sequence

```text
storage.read_all()
  → detach dangling parents
  → find latest LeafEntry
  → path_to_entry(active anchor)
  → SessionState.from_entries(...)
  → AgentHarness(messages=state.messages)
  → persist interrupted-tool repairs
  → attach extension listener
```

主来源：[`session.py:296–405`](../src/tau_coding/session.py)。

### `_detach_missing_parents`

load 时，如果一个 entry 的 `parent_id` 在文件中不存在，Tau 会复制该 entry 并把
`parent_id` 设为 `None`，把导入/部分历史从外部缺失 ancestry 上 detach。

这不是自动修复所有损坏日志；它只明确处理 dangling parent。
主来源：[`session.py:2154–2162`](../src/tau_coding/session.py)。

## 6. Branching vocabulary

### Branch

`branch_to_entry()` 通过 append 新 `LeafEntry`，让 active replay anchor 回到旧 entry。
旧 JSONL 行不被复制、删除或修改。

### Chain cut

“分支切断 parent chain”是 replay 语义：新 anchor 的 ancestry 不再经过被放弃的旧尾部。
它不表示磁盘中的旧 entries 被物理删除。

### Abandoned tail

分支点之后、原 active path 上不再被新 anchor 选中的 entries。它们仍在文件里，可用于审计
或将来再次分支，但不会进入当前 transcript。

### `BranchSummaryEntry`

当 `summarize=True` 且存在 abandoned messages 时，Tau 先 append 一条 branch summary，
再把 leaf 指到该 summary。它是“从被放弃分支带回摘要”，不是 context compaction。

主来源：[`session.py:503–567`](../src/tau_coding/session.py)；详解：
[Lesson 16](../lessons/0016-branch-without-rewriting-history.html)。

## 7. Compaction vocabulary

### `CompactionEntry`

一条 replay 指令，包含：

- `summary`：压缩摘要；
- `replaces_entry_ids`：在 active replay 中由摘要替换的 message entry IDs。

它的 `parent_id` 指向压缩前的最后 active entry，因此旧消息仍在 ancestry chain 上。

### No chain cut

Compaction append 新 entry 和 leaf，但不把 anchor 移回过去。旧 message entries 仍可沿
parent chain 回溯；只是在 reducer 输出 `state.messages` 时被摘要替换。

### `_apply_compaction()`

replay 时遍历 message rows：第一次遇到被替换 ID 时插入一条：

```text
UserMessage("Previous conversation summary:\n<summary>")
```

其余被替换 rows 不进入 active provider context，未被替换的近期消息继续保留。
主来源：[`memory.py:106–129`](../src/tau_agent/session/memory.py)。

### Keep-recent / reserve

- `DEFAULT_COMPACTION_KEEP_RECENT_TOKENS = 20_000`：自动和 overflow compaction 尽量保留的近期上下文。
- `DEFAULT_COMPACTION_RESERVE_TOKENS = 16_384`：从模型 context window 中预留的安全空间。
- 自动阈值：`context_window_tokens - 16_384`。

Tau 还会校正切点，避免从不干净的 user/tool-result 边界截断。
主来源：[`context_window.py:17–24, 166–170`](../src/tau_coding/context_window.py),
[`session.py:2037–2055, 2085–2123`](../src/tau_coding/session.py)。

### Incremental compaction

如果第一条消息是以 `Previous conversation summary:\n` 开头的 `UserMessage`：

1. `_split_previous_compaction_summary()` 剥离旧摘要；
2. 新消息放入 `<conversation>`；
3. 旧摘要单独放入 `<previous-summary>`；
4. 使用 `UPDATE_SUMMARIZATION_PROMPT`，而不是第一次压缩使用的 `SUMMARIZATION_PROMPT`。

主来源：[`context_window.py:24, 34–93, 204–224, 268–281`](../src/tau_coding/context_window.py)；
详解：[Lesson 17](../lessons/0017-compact-without-losing-history.html)。

## 8. Crash-window vocabulary

| 窗口 | 文件里可能有什么 | 正常 load 的结果 |
|---|---|---|
| Before `MessageEntry` | 完整消息可能只在内存；受 append-lag 影响，即使它自己的 end event 已出现，也可能尚未进入 `harness.messages`。 | 该消息不可从 JSONL 恢复。 |
| After `MessageEntry`, before `LeafEntry` | 新 message line 已存在，active selector 尚未前移。 | 新 message 通常不在最新 leaf 的 active path 上，成为可审计但未选中的 entry。 |
| After `LeafEntry` | message 和新的 active selector 均已 append。 | load 可从最新 leaf anchor 重放出新消息。 |
| OS / power loss | Python 写调用可能已返回，但当前实现没有显式 `fsync`。 | 不能承诺日志尾部已经进入稳定存储。 |

详解：[Lesson 15 · 崩溃窗口](../lessons/0015-recover-session-from-jsonl.html#window)。外部可靠性词汇对照见
[Card 14 primary-source findings](card14-durable-session-log-primary-sources.md)。

## 9. Interrupted-tool repair on load

`CodingSession.load()` 在建立 harness 后调用 `_persist_loaded_interrupted_tool_repairs()`：
如果 active transcript 中存在没有结果的 tool call，它会 append synthetic
`ToolResultMessage` 对应的 `MessageEntry`，再 append 新 `LeafEntry`，随后 replay 并重建
harness。

这使“内存中的协议修复”也成为 durable fact，避免下一次 resume 又回到悬空调用。
主来源：[`session.py:1728–1763`](../src/tau_coding/session.py)；机制背景见
[Lesson 11](../lessons/0011-repair-interrupted-tool-calls.html)。

## 10. Branch vs compaction 一页对比

| 维度 | Branch | Compaction |
|---|---|---|
| 目的 | 选择较早历史并从那里继续 | 缩短当前 provider context |
| 写入方式 | append `LeafEntry`；可选先 append `BranchSummaryEntry` | append `CompactionEntry` + `LeafEntry` |
| Active anchor | 移到过去的 entry 或新 branch summary | 移到新 compaction entry |
| Parent chain | **切断当前 ancestry**：旧尾部不再可达 | **不断链**：compaction parent 指向旧尾部 |
| 原始 entries | 保留在 JSONL | 保留在 JSONL |
| Replay 输出 | 只重放新 anchor 的 ancestry | 在完整 ancestry 上应用 `replaces_entry_ids` |

Canonical phrasing：

> Branch 改变“当前选择哪条历史”；compaction 改变“同一条历史怎样呈现给模型”。

## Scope boundary

本术语表到本地 JSONL、单进程 replay 和当前 Tau runtime 为止。若之后要讨论远程客户端、
event offset、多租户状态或分布式存储，应明确标为平台面试外推，并先重新定义 durability、
ordering、ownership 与 replay contract；不要把这些概念倒推成当前 Tau 已有行为。
