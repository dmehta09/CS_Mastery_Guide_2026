# Read Receipts

How to implement "read" indicators at scale—who read which message and when—without overwhelming storage and write throughput.

---

## Case Study 1: Write Amplification from Read Receipts

### Problem

Every time a user opens a conversation and reads messages, you want to record "read at time T" for each message. With 1 billion users and tens of billions of messages read per day, that's an enormous number of writes. A naive design (one write per message per read) can exceed your database's write capacity and dominate cost.

**Context**: WhatsApp, Messenger, Slack, Discord all face this—read receipts at billion-scale create more writes than the messages themselves

**In simple terms**: Imagine a library where every time someone looks at a page, the library has to stamp "read by person X at time T" on that page. With millions of readers, the stamping alone would overwhelm the library. Read receipts are the same: the "read" event is more frequent than the "message sent" event, so the write load from reads can dwarf the load from messages.

### Quick Answer

Reduce write amplification: don't write one row per message read. Use "last read pointer" (e.g. "user A has read up to message_id X" or "timestamp T") so one write covers many messages. Batch or throttle updates (e.g. at most one write per user per conversation per N seconds). Store only what you need to display (e.g. "read by" and "read at" for last read message, not every message).

### Detailed Explanation

#### Why This Happens (Root Cause)

**Naive model:**
- User opens chat, sees 50 messages.
- System writes 50 read-receipt records (user_id, message_id, read_at).
- 100M users open chats 10 times/day, 50 messages each → 50B writes/day just for read receipts.

**Root cause**: Treating "read" as a per-message event multiplies writes by the number of messages in view. At scale, that exceeds typical write capacity and cost.

#### The Solution Approach

**High-level strategy:**
- Model "read" as a position in the conversation, not a set of per-message events.
- One write per user per conversation (or per N seconds): "User A read up to message_id M (or timestamp T)."
- To show "read by" on a message: compare message_id (or timestamp) with each participant's last-read position. If message_id ≤ last_read_id, show as read.

**Last-read pointer (conceptual):**
```
Conversation: msg_1, msg_2, ..., msg_100
User A opens chat, reads up to msg_95.
Store: last_read[A, conversation] = msg_95 (one write)
Display: For each message, "read" if message_id ≤ last_read[reader, conversation]
```

**Batching and throttling:**
- Don't write on every scroll. Write when user leaves conversation, or when they pass a threshold (e.g. 10 new messages read), or at most once every 30 seconds.
- Reduces writes further (e.g. 1 write per session instead of 50).

**Architecture flow:**
```
User reads messages
      │
      ▼
Client: "I read up to message_id 95"
      │
      ▼
Server: Upsert last_read(user_id, conversation_id) = 95
      │ (one write, or batched with other updates)
      ▼
Other clients: "Who read message 90?" → Compare 90 <= last_read[user] for each user
```

**Key Design Decisions:**
- **Pointer semantics**: Last-read message_id or timestamp. Timestamp is simpler for time-ordered feeds; message_id is explicit and avoids clock skew.
- **Write frequency**: Throttle (e.g. max 1 update per 30s per user per conversation) to cap write rate.
- **Storage**: One row per (user, conversation) for last-read position, not one row per (user, message). Dramatically fewer rows and writes.

#### Production Results

| Metric | Per-message receipts | Last-read pointer + throttle | Improvement |
|--------|----------------------|------------------------------|-------------|
| Writes per "open conversation" | 50 | 1 (or 0 if no change) | 50x reduction |
| Daily write volume (receipts) | 50B | ~1B | ~50x reduction |
| Storage for receipts | O(messages × readers) | O(conversations × participants) | Orders of magnitude |
| Display accuracy | Per-message exact | "Read up to here" (consistent) | Acceptable for UX |

**Key insight**: Read receipts at scale are a write-amplification problem. Last-read pointer plus throttling keeps write volume and storage manageable while still answering "has this been read?"

**Lessons Learned**:
- Design for write volume first; reads are easier to scale.
- Last-read pointer is the standard pattern (WhatsApp, Messenger, iMessage conceptually similar).
- Throttle and batch; real-time to the second is rarely required for "read" UX.
- Store only what you need to render; avoid storing every (user, message) read event.

---

## Case Study 2: Consistency and Ordering of Read Receipts

### Problem

User A reads message 10, then message 20. You send two updates: "read up to 10" and "read up to 20." They can arrive out of order at the server or at other clients. If "read up to 20" is processed before "read up to 10," you might incorrectly show "read up to 10" and overwrite the newer state.

**Context**: Distributed systems issue—network reordering, retries, multiple devices

**In simple terms**: Like sending two text messages: "I'm at the store" and "I'm back home." If they arrive in the wrong order, someone might think you're still at the store. With read receipts, wrong order can make "read up to 20" be replaced by "read up to 10," so the receipt goes backward.

### Quick Answer

Make last-read updates monotonic: only accept an update if the new position is strictly after the stored position (e.g. higher message_id or later timestamp). Ignore or reject out-of-order updates. Optionally attach a sequence number or timestamp from the client so the server can discard stale updates.

### Detailed Explanation

#### Why This Matters

**Scenario:**
- Client sends: read_up_to(20), then read_up_to(10) (e.g. user scrolled back, or bug).
- Server processes read_up_to(10) after read_up_to(20).
- Stored value becomes 10; other users see "read up to 10" instead of 20.
- Receipt appears to go backward.

**Root cause**: No ordering guarantee on the wire; retries and multiple devices can produce out-of-order updates.

#### The Solution Approach

**High-level strategy:**
- Store one value per (user, conversation): last_read_message_id (or timestamp).
- On update: only set new value if new_position > stored_position (monotonic).
- If new_position <= stored_position, ignore or return success without writing (idempotent).

**Monotonic update (conceptual):**
```
Stored: last_read[user_A, conv_1] = 15
Incoming: read_up_to(20) → 20 > 15 → accept, store 20
Incoming: read_up_to(10) → 10 < 20 → reject or no-op, keep 20
```

**Optional: version or timestamp from client**
- Client sends (position, client_timestamp or sequence).
- Server rejects if client_timestamp is older than the timestamp that produced the current stored position.
- Helps with multiple devices (phone and desktop); only the latest update wins.

**Key Design Decisions:**
- **Monotonicity**: Enforce in the service layer or with a conditional write (e.g. "UPDATE ... SET last_read = ? WHERE ... AND (last_read IS NULL OR last_read < ?)").
- **Idempotency**: Duplicate "read up to 20" is safe (no change); no need for client to track "already sent."
- **Multiple devices**: Last-write-wins with monotonic position is usually enough; optional client timestamp for tie-breaking.

#### Production Results

| Metric | Before (no ordering) | After (monotonic) | Improvement |
|--------|------------------------|-------------------|-------------|
| "Receipt went backward" reports | 0.5% of sessions | 0% | Eliminated |
| Consistency (receipt always advances) | 99.5% | 100% | Correct behavior |
| Complexity | Low | Low (one comparison) | Minimal cost |

**Key insight**: Read receipts are a monotonic state: "read up to X" should only move forward. Enforcing that on the server avoids confusing UX and bugs from reordering.

**Lessons Learned**:
- Always enforce monotonicity for last-read (or similar) state.
- Use conditional writes or compare-and-set so concurrent updates don't overwrite with an older value.
- Document behavior for multiple devices (e.g. "latest read position across devices" or "per-device").

---

## Case Study 3: Delivering Read Receipts to Other Clients in Real Time

### Problem

When user B reads messages, user A (the sender) should see "read" quickly. That requires getting the updated read position from the server to A's client with low latency. At scale, that means fan-out (B's read update must reach A, and in group chats, many participants) and efficient use of connections and messaging.

**Context**: Real-time delivery—WhatsApp, Messenger, Slack all push read state to other participants

**In simple terms**: When someone reads your message, you see the "read" checkmark appear. That checkmark is a small update that has to reach you in real time, just like the message did. Doing that for billions of conversations means efficient push (WebSocket, long polling, or similar) and not broadcasting every read event to the whole world.

### Quick Answer

Push read receipt updates over the same real-time channel you use for messages (WebSocket, long polling, or mobile push). Only send to participants in that conversation. Send compact updates (e.g. "user B read up to message_id X") not full state. Optionally coalesce: one push per user per conversation per N seconds with the latest position.

### Detailed Explanation

#### Why This Matters

**Requirements:**
- Low latency: "Read" should appear within a second or two.
- Scale: Millions of concurrent connections; many conversations per user.
- Efficiency: Don't send redundant or overly large payloads.

**Challenges:**
- Finding the right connections: Given "user B read conv_1 up to msg_20," which connections need this? All other participants in conv_1 (e.g. user A).
- Fan-out in groups: In a 100-person group, one read might need to go to 99 other participants (or only to the message sender, depending on product).
- Coalescence: If B reads 10 messages in 2 seconds, send one "read up to X" or ten? One is enough if you send the latest.

#### The Solution Approach

**High-level strategy:**
- When last_read is updated, produce an event (e.g. "user B, conv_1, read up to msg_20").
- Resolve which users need this (e.g. conversation participants, or only senders of messages that are now "read").
- For each recipient, push the update on their active connection(s) (WebSocket, etc.).
- Coalesce: one update per (reader, conversation) with latest position; drop or merge older updates in the same time window.

**Who receives the update (product choice):**
- **1:1**: Often both participants see each other's read position.
- **Groups**: Either "read by" only for message senders (so only message author sees "read by X"), or all participants get "user X read up to Y" for presence/UI. Latter increases fan-out.

**Push path (conceptual):**
```
User B updates last_read(conv_1) = 20
      │
      ▼
Server: Persist; emit event (B, conv_1, 20)
      │
      ▼
Resolve: Participants of conv_1 = {A, B}; recipients = {A} (exclude B)
      │
      ▼
Push to A's connection(s): { type: "read_receipt", user: B, conversation: conv_1, up_to: 20 }
```

**Coalescence:**
- In a short window (e.g. 1–2s), if multiple updates for same (user, conversation) arrive, keep only the latest position and send one push per recipient.
- Reduces load on connections and clients.

**Key Design Decisions:**
- **Same channel as messages**: Reuse WebSocket or long-polling endpoint so no extra connection or protocol.
- **Compact payload**: Message ID or timestamp + user/conversation ID; avoid sending full conversation state.
- **Fan-out limit**: In large groups, consider not pushing every read to everyone (e.g. only to message senders, or aggregate "N people read").
- **Offline**: If A is offline, store last_read in DB; when A reconnects, include latest read state in sync or next fetch so UI can show it.

#### Production Results

| Metric | Before (polling) | After (push + coalesce) | Improvement |
|--------|------------------|--------------------------|-------------|
| Latency (read → sender sees it) | 5–30s | < 2s | 5–15x faster |
| Messages per "read session" | 1 per message | 1 per conversation (coalesced) | Large reduction |
| Connection load | High (frequent polls) | Low (one push per update) | Lower CPU and bandwidth |

**Key insight**: Read receipts are real-time state; they belong on the same push path as messages, with minimal payload and coalescence to avoid write and push amplification.

**Lessons Learned**:
- Push over existing real-time channel; don't add a separate receipt-only channel unless needed.
- Coalesce updates per (user, conversation) so one push carries the latest position.
- Define clearly who receives read updates (1:1 vs groups, senders only vs all) to control fan-out and product behavior.

---

## Common Mistakes

1. **Mistake**: Storing one row per (user, message) for read receipts
   **Why it's wrong**: Write and storage scale with messages × readers; unsustainable at scale.
   **Instead**: Store last-read position per (user, conversation); infer "read" by comparing message_id to that position.

2. **Mistake**: No monotonicity for last-read updates
   **Why it's wrong**: Out-of-order or retried updates can make receipt go backward.
   **Instead**: Only accept updates where new position > stored position; ignore or reject older updates.

3. **Mistake**: Writing on every scroll or every message view
   **Why it's wrong**: Write volume explodes; database and cost can't keep up.
   **Instead**: Throttle (e.g. at most one write per user per conversation per 30s) and/or write on leave or threshold.

4. **Mistake**: Pushing every read event to all participants in large groups
   **Why it's wrong**: Fan-out and connection load grow with group size.
   **Instead**: Limit who receives (e.g. only message senders) or coalesce and send one update per reader per conversation.

5. **Mistake**: Separate real-time channel for read receipts
   **Why it's wrong**: Extra connections and protocol complexity.
   **Instead**: Reuse the same WebSocket or long-poll channel as messages; send receipt as a small message type.

---

## Related Patterns

- **[WebSockets at Scale](../websocket_at_scale/README.md)** - Connection management and push at scale
- **[Real-Time Feeds](../real_time_feeds/README.md)** - Fan-out and push for feeds
- **[Idempotency Patterns](../../04_API_Design/idempotency_patterns/README.md)** - Safe retries for receipt updates
- **[Event-Driven Architecture](../../05_Event_Driven_Architecture/kafka_patterns.md)** - Using events to fan out read state (if needed)

---

## Research Resources

See [RESOURCES.md](./RESOURCES.md) for:
- Engineering blog posts (WhatsApp, Messenger scale)
- Talks and articles on messaging and read receipts at scale
- Documentation and references
