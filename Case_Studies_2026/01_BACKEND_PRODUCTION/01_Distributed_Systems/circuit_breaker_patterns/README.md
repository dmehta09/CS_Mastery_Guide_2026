# Circuit Breaker Patterns

How to stop cascading failures when a downstream service is slow or failing, and give it time to recover.

---

## Case Study 1: Cascading Failure from One Slow Service

### Problem

Your API calls a recommendation service. The recommendation service slows down (e.g. 10s response time). Your API threads block waiting for it. Thread pool fills up. Your API stops responding to all requests—including ones that don't use recommendations. One bad dependency takes down the whole system.

**Context**: Classic microservices failure mode - Netflix experienced this and built Hystrix to address it

**In simple terms**: Imagine a power strip where one faulty appliance shorts the circuit. Without a circuit breaker, the whole strip (and maybe the room) goes dark. A circuit breaker trips so only that circuit fails; the rest keep working. In software, the circuit breaker "trips" so one failing service doesn't exhaust your threads and take down your service.

### Quick Answer

Use a circuit breaker: wrap calls to the downstream service. If failures or slow responses exceed a threshold, "open" the circuit—stop sending new requests for a period and return a fallback (cached value, default, or error). After a timeout, try again ("half-open"); if that succeeds, "close" the circuit and resume normal traffic.

### Detailed Explanation

#### Why This Happens (Root Cause)

Without a circuit breaker:

```
Recommendation Service: 10s latency (overloaded or bug)
    ↓
Your API: 100 threads, all blocked waiting on recommendation service
    ↓
Thread pool exhausted in 10 seconds (10 req/sec × 10s = 100 threads)
    ↓
All incoming requests (search, profile, cart) get "503 no threads"
    ↓
Full outage from one dependency
```

**Root cause**: Synchronous calls to a slow or failing service consume resources (threads, connections) until there are none left. Failure propagates to every caller of your service.

#### The Solution Approach

**High-level strategy:**
- Wrap every call to the risky dependency in a "circuit."
- Track success/failure (and optionally latency) in a sliding window.
- When failure rate (or error count) exceeds a threshold, open the circuit: stop calling the dependency and return fallback immediately.
- After a "sleep" window, allow one or a few test requests (half-open). If they succeed, close the circuit; if they fail, stay open.

**Architecture Decision: Circuit Breaker States**

**Three states:**
- **Closed**: Normal. Requests go to the downstream service. Failures are counted.
- **Open**: Circuit tripped. Requests do not go to the downstream service; they get fallback (or fast-fail). After a configured time, move to half-open.
- **Half-open**: Allow a limited number of requests through. If they succeed, close the circuit. If they fail, open again.

**How it works conceptually:**
1. Closed: 20 requests in last 10s, 12 failed → failure rate 60% > 50% threshold → open circuit.
2. Open: All requests return fallback (e.g. "Recommendations temporarily unavailable") for 30 seconds.
3. After 30s: Half-open. Next 3 requests go to recommendation service. All 3 succeed → close circuit. Any failure → open again for another 30s.

**Netflix Hystrix (conceptual):**
- Default: open after 20 requests in 10s window with >50% failure.
- Fallback: developer-provided (static list, cache, error message).
- Half-open: after 5s (configurable) allow one request; success closes, failure reopens.

**Architecture Flow:**
```
Request → Circuit closed? → Call downstream → Success → Return result
                        → Failure → Count; threshold? → Open circuit
         → Circuit open? → Return fallback (no call)
         → After sleep   → Half-open → Test request → Success → Close
                                                    → Failure → Open again
```

**Key Design Decisions:**
- **Threshold**: Failure percentage (e.g. 50%) or count (e.g. 5 in 10s). Lower = trip sooner, more fallbacks; higher = more traffic to failing service.
- **Window**: Sliding or rolling window for metrics. 10s is common.
- **Fallback**: Must be safe (no downstream call). Cached data, empty list, or "try again later" message.
- **Half-open test**: One or few requests; success closes, failure reopens. Prevents stampede when dependency recovers.

#### Production Results

| Metric | Before (No Breaker) | After (Circuit Breaker) | Improvement |
|--------|---------------------|-------------------------|-------------|
| Full outages from one dependency | 4/year | 0/year | Eliminated |
| P99 latency (when dependency slow) | 30s (timeout) | 50ms (fallback) | 600x faster |
| Thread pool exhaustion incidents | 12/month | 0/month | Eliminated |
| Dependency recovery time (no traffic) | N/A | Minutes (breaker opens, load drops) | Service can recover |

**Key insight**: The circuit breaker protects your service from a bad dependency and protects the dependency from continued load while it's failing—giving it time to recover.

**Lessons Learned**:
- Put a circuit breaker on every call to an external or shared service (DB, HTTP, cache if critical).
- Fallback must not call another failing service (avoid fallback chains that cascade).
- Tune threshold and window so you don't trip on normal blips but do trip before thread exhaustion.
- Monitor open-circuit rate; high rate means dependency is unhealthy or threshold too sensitive.

---

## Case Study 2: Choosing Fallback Strategy

### Problem

When the circuit is open, what should you return? Empty list? Cached data? Error message? Wrong choice can hurt UX or business (e.g. showing stale prices, or no recommendations at all).

**Context**: Design decision - affects user experience and correctness

**In simple terms**: When the power is out, do you show a candle (degraded but useful), a "power out" sign (honest), or pretend the lights are on (stale data that might be wrong)? Fallback is the same: degraded, honest, or cached—each with trade-offs.

### Quick Answer

Prefer fallbacks that are safe and clearly degraded: static or cached data when freshness is not critical, or a clear "unavailable" when correctness matters (e.g. pricing, inventory). Never fall back to another unreliable service; keep fallback logic simple and fast.

### Detailed Explanation

#### Why This Matters

**Fallback options:**
- **Static default**: e.g. "Recommendations unavailable." Safe, no stale data risk; poor UX.
- **Cached data**: e.g. last successful response. Good UX if data is not time-sensitive; risk of showing stale or wrong data (e.g. prices).
- **Degraded response**: e.g. fewer recommendations, or from a different (reliable) source. Balance of UX and correctness.
- **Error with retry**: e.g. 503 + Retry-After. Client can retry; no silent wrong data.

**Risks:**
- Stale cache: wrong price, wrong availability → complaints or trust issues.
- Fallback that calls another service: that service can fail too → cascade.
- Complex fallback: bugs or latency in fallback path → circuit breaker doesn't simplify the system.

#### The Solution Approach

**High-level strategy:**
- Classify dependency: Is freshness critical (payment, inventory) or not (recommendations, non-critical UI)?
- For non-critical: cache or static list is often fine; add TTL or "last updated" if shown in UI.
- For critical: prefer "unavailable" or 503; avoid showing stale critical data.
- Never chain fallbacks to other remote services without their own breakers and timeouts.

**Decision table:**

| Dependency type | Fallback preference | Rationale |
|-----------------|---------------------|-----------|
| Recommendations | Cache or empty list | Stale recommendations better than nothing |
| Pricing / inventory | Error (503) or "unavailable" | Stale price can cause wrong charges |
| User profile (non-critical) | Cache with TTL | Slight staleness acceptable |
| Search (non-critical) | Cached popular queries or error | Depends on product |

**Key Design Decisions:**
- **Cache TTL for fallback**: If using cache, keep TTL short (e.g. 1–5 min) so open circuit doesn't serve very old data for long.
- **Expose circuit state to clients (optional)**: e.g. header or field "degraded: recommendations from cache" so UI can show a notice.
- **Metrics**: Track fallback rate and cache hit rate for fallback; alert if circuit is open too long.

#### Production Results

| Strategy | User impact | Correctness risk | Operational complexity |
|----------|-------------|------------------|-------------------------|
| Static "unavailable" | Clear, no surprise | None | Low |
| Cached data | Good for non-critical | Stale data risk | Medium (cache invalidation) |
| 503 + Retry-After | Client can retry | None | Low |
| Fallback to other API | Can cascade | High | High |

**Key insight**: Fallback is part of your contract with users. Prefer safe and honest (error or short-TTL cache) over "clever" fallbacks that risk wrong data.

**Lessons Learned**:
- Document fallback behavior in API docs and runbooks.
- For money or inventory, do not show stale data as success; use error or explicit "unavailable."
- Test fallback path in load tests (force circuit open) so it doesn't become a hidden failure path.

---

## Case Study 3: Tuning Thresholds and Half-Open

### Problem

Circuit opens too often on normal blips (false positives) or stays closed until the system is already overloaded (false negatives). Half-open allows too many requests and overwhelms a recovering service, or allows too few and keeps circuit open too long.

**Context**: Operational tuning - Netflix Hystrix defaults (e.g. 20 requests, 50% failure, 10s window) are starting points, not one-size-fits-all

**In simple terms**: A home circuit breaker that trips when you run the microwave and the toaster (too sensitive) is annoying. One that never trips until the wiring smokes (not sensitive enough) is dangerous. You need the right threshold and the right "try again" policy.

### Quick Answer

Tune based on traffic and dependency behavior: higher volume = larger request count threshold; stricter SLA = lower failure percentage. Half-open: start with 1 request; if dependency is slow to recover, use a small batch (e.g. 3) and longer sleep window before half-open.

### Detailed Explanation

#### Why This Matters

**Too sensitive (trips too easily):**
- Brief network blip or one bad deployment → circuit opens.
- Users see fallback when dependency would have recovered in seconds.
- High open-circuit rate; dependency gets no load and looks "healthy" in metrics while callers see fallback.

**Not sensitive enough:**
- Circuit stays closed until many requests have failed and threads are already blocked.
- Cascading failure still happens; circuit breaker doesn't help in time.

**Half-open:**
- Too aggressive: 100 requests allowed in half-open → dependency still recovering → all fail → circuit reopens; recovery delayed.
- Too conservative: 1 request every 60s → circuit takes forever to close; long period of fallback.

#### The Solution Approach

**Threshold tuning:**
- **Request volume threshold**: Minimum requests in window before considering failure rate. Prevents opening on 1 failure out of 2 requests (50% but tiny sample). Hystrix default 20 is reasonable; for low-traffic services, use lower (e.g. 5).
- **Failure percentage**: 50% = open when more than half fail. For strict SLAs use 30–40%; for resilient dependencies use 60–70%.
- **Window size**: 10s sliding window is common. Shorter = faster reaction but more sensitive to spikes; longer = smoother but slower to open.

**Half-open tuning:**
- **Sleep window**: How long circuit stays open before half-open. 5–30s typical. Longer gives dependency more recovery time; shorter gets back to normal faster if it was a brief blip.
- **Test requests in half-open**: 1 is safest (no stampede). Some implementations allow 3–5; ensure dependency can handle that load when still recovering.

**Key Design Decisions:**
- **Per-dependency settings**: Recommendation service vs payment service may need different thresholds (e.g. stricter for payment).
- **Observe and iterate**: Use dashboards for open rate, half-open success rate, and dependency latency; adjust every few weeks if needed.
- **Document defaults**: So on-call and new developers know why values are what they are.

#### Production Results

| Tuning | Before | After | Effect |
|--------|--------|-------|--------|
| Request threshold 5 → 20 | False opens 15/day | 2/day | Fewer unnecessary fallbacks |
| Failure % 50% → 40% (payment) | Late trip 2 incidents | 0 | Circuit opens before cascade |
| Half-open: 1 request | Recovery 30–60s | 30–60s | Predictable |
| Sleep window 5s → 15s | Reopen rate 30% | 10% | Dependency has time to recover |

**Key insight**: Defaults are a starting point. Tune per dependency and traffic; monitor and adjust so the breaker protects without over-tripping.

**Lessons Learned**:
- Start with conservative defaults (e.g. 20 requests, 50%, 10s window, 1 half-open request).
- Tighten (lower failure %, lower volume) for critical dependencies; loosen for best-effort.
- Alert on circuit open rate and half-open success rate; investigate if open rate is high or half-open always fails.

---

## Common Mistakes

1. **Mistake**: No circuit breaker on external or shared dependencies
   **Why it's wrong**: One slow or failing dependency can exhaust threads and take down your service.
   **Instead**: Wrap every call to another service (or critical DB/cache) in a circuit breaker.

2. **Mistake**: Fallback that calls another remote service
   **Why it's wrong**: Fallback dependency can fail too; failure cascades.
   **Instead**: Use static response, short-TTL cache, or error; keep fallback local and fast.

3. **Mistake**: Using same threshold for all dependencies
   **Why it's wrong**: Payment and recommendations have different criticality and traffic.
   **Instead**: Stricter (trip sooner) for critical path; more lenient for best-effort.

4. **Mistake**: Ignoring circuit state in monitoring
   **Why it's wrong**: Can't tell if fallbacks are due to real dependency issues or misconfiguration.
   **Instead**: Dashboard and alerts for open rate, half-open success, and dependency latency.

5. **Mistake**: Half-open sends full traffic
   **Why it's wrong**: Recovering service can be overwhelmed again and fail; circuit reopens.
   **Instead**: Allow 1 or a small number of test requests in half-open; only then close circuit.

---

## Related Patterns

- **[Rate Limiting](../../04_API_Design/rate_limiting/README.md)** - Protect your own API from overload
- **[Idempotency Patterns](../../04_API_Design/idempotency_patterns/README.md)** - Safe retries when circuit reopens
- **[Distributed Transactions](./distributed_transactions/README.md)** - Saga and compensations when downstream fails
- **[Cache Consistency](../cache_consistency/README.md)** - Using cache as fallback source

---

## Research Resources

See [RESOURCES.md](./RESOURCES.md) for:
- Engineering blog posts (Netflix Hystrix, resilience patterns)
- Conference talks (QCon, AWS re:Invent)
- Documentation (Resilience4j, Spring Cloud Circuit Breaker)
- References on circuit breaker pattern
