# SQL Write Hotspot per LLM Request

> Branch: `scr-260513` · Date: 2026-05-13

## Overview

The credit/balance deduction system uses a **two-phase model**: Pre-Consume (reserve quota before calling upstream) → Settle (true-up after response). This document maps every SQL write that fires during a single LLM relay request and identifies hotspots.

---

## 1. Request Lifecycle — Per-Request SQL Writes

```
                     ONE LLM REQUEST — 6–7 SQL WRITES
 ───────────────────────────────────────────────────────────────────────►

  ┌─── PHASE 1: PRE-CONSUME (sync, before upstream call) ───┐
  │                                                           │
  │  WRITE 1:  UPDATE tokens                                 │  ← HOTTEST: every request
  │             SET remain_quota = remain_quota - ?,           │    hits tokens table
  │                 used_quota   = used_quota   + ?            │
  │             model/token.go:424   decreaseTokenQuota()     │
  │                                                           │
  │  WRITE 2:  UPDATE users    (wallet funding)               │  ← OR
  │             SET quota = quota - ?                          │
  │             model/user.go:928    decreaseUserQuota()       │
  │          OR UPDATE subscriptions (subscription funding)    │
  │             SET amount_used = amount_used + ?              │
  │             model/subscription.go:970                      │
  │                                                           │
  │  If trust-bypass applies (user quota > trustQuota &&     │
  │  token unlimited/high-quota): both writes SKIPPED.        │
  │  service/billing_session.go:190–193                        │
  └───────────────────────────────────────────────────────────┘
                        │
                        ▼
              ┌─────────────────────┐
              │  Upstream LLM Call  │  (100ms–30s, blocking)
              └─────────────────────┘
                        │
                        ▼
  ┌─── PHASE 2: SETTLEMENT (sync, after response) ──────────┐
  │                                                           │
  │  WRITE 3:  UPDATE users   (wallet: delta adjust)         │  ← same row as write 2
  │             SET quota = quota ± Δ                          │    → 2 hits/row
  │             model/user.go:928/903                          │
  │          OR UPDATE subscriptions                           │
  │             SET amount_used = amount_used + Δ              │
  │             model/subscription.go:1182                     │
  │                                                           │
  │  WRITE 4:  UPDATE tokens  (delta adjust)                  │  ← same row as write 1
  │             SET remain_quota = remain_quota ± Δ,           │    → 2 hits/row
  │                 used_quota   = used_quota   ∓ Δ            │
  │             model/token.go:424/394                         │
  │                                                           │
  │  WRITE 5:  UPDATE users    (aggregate stats)              │
  │             SET used_quota = used_quota + ?,               │
  │                 request_count = request_count + 1          │
  │             model/user.go:962                              │
  │                                                           │
  │  WRITE 6:  UPDATE channels (channel stats)                │
  │             SET used_quota = used_quota + ?                │
  │             model/channel.go:805                           │
  │                                                           │
  │  WRITE 7:  INSERT INTO logs   (consume log, LOG_DB)       │  ← separate DB
  │             model/log.go:249    RecordConsumeLog()         │    optional (LogConsumeEnabled)
  └───────────────────────────────────────────────────────────┘

  ┌─── PHASE 3: REFUND (async goroutine, only on error) ─────┐
  │                                                           │
  │  WRITE 8:  UPDATE users    SET quota = quota + ?          │  ← gopool.Go
  │             model/user.go:903                              │
  │  WRITE 9:  UPDATE tokens   SET remain_quota = remain_quota + ?, │
  │                              used_quota = used_quota - ?   │
  │             model/token.go:394                             │
  └───────────────────────────────────────────────────────────┘
```

---

## 2. Sync vs Async Breakdown

| Phase | DB Write | Blocks LLM Relay? | Mechanism |
|-------|----------|:-----------------:|-----------|
| **PreConsume** (write 1, 2) | `UPDATE tokens` + `UPDATE users` | **YES** | Sync — must succeed before upstream call |
| **Settle** (write 3–7) | Quota delta + stats + log | **YES** | Sync — happens in response path before client receives data |
| **Refund** (write 8, 9) | Quota + token refund | **NO** | Async — `gopool.Go` goroutine; error response sent first |
| **Redis cache** | `DECR` / `INCR` | **NO** | Always async via `gopool.Go` |

---

## 3. Hotspot Ranking

| Rank | Table | Writes/req | Rows Hit | Contention Risk |
|:----:|-------|:---------:|----------|:---------------:|
| **1** | `tokens` | **2** | same row | **HIGH** — reserve + settle to same row, hit every request |
| **2** | `users` (wallet) | **2–3** | same row (quota col) + stats | **HIGH** — `quota` column gets 2 updates per request |
| **3** | `subscriptions` | **2** | same row | **MEDIUM** — `amount_used` reserve + settle |
| **4** | `channels` | **1** | shared row | **MEDIUM** — all users on same channel contend on one row |
| **5** | `users` (stats) | **1** | same row | **LOW** — `used_quota`, `request_count` are counters |
| **6** | `logs` | **1** | new row | **LOW** — append-only INSERT to separate `LOG_DB` |

### Bottleneck Detail

The `tokens` and `users` tables both receive **2 UPDATEs to the same row**, separated by the upstream LLM call:

```
Time ─────────────────────────────────────────────────────────►

  UPDATE tokens SET remain -= 100     (lock held ~1ms)
         │
         │  ▲ gap = upstream latency (100ms – 30s for LLM)
         │  │ row is UNLOCKED during this time
         ▼
  UPDATE tokens SET remain ±= Δ       (lock held ~1ms)

  Same row, same columns, two separate transactions.
  Result: 2× write amplification on the hottest tables.
```

Row-level locks are brief (~1ms), so lock-wait contention is low. The primary concern is **doubled write volume** on these tables.

---

## 4. Mitigations

### 4.1 Batch Update Mode

When `common.BatchUpdateEnabled` is `true`, quota writes (1–4) skip the immediate DB `UPDATE` and queue to an in-memory buffer instead:

- `model/user.go:920` — `addNewRecord(BatchUpdateTypeUserQuota, id, -quota)`
- `model/token.go:418` — `addNewRecord(BatchUpdateTypeTokenQuota, id, -quota)`

The buffer is flushed periodically, merging multiple increments into a single `UPDATE`. This eliminates the per-request DB round-trip for quota writes but defers visibility.

**Trade-off**: lower per-request DB load at the cost of stale quota reads from DB (Redis cache remains real-time).

### 4.2 Trust Bypass

When a user's balance exceeds the trust threshold and their token is unlimited or high-quota, both pre-consume writes (1, 2) are **completely skipped**:

- `service/billing_session.go:190–193`

This avoids unnecessary DB writes for trusted/high-balance users.

### 4.3 Separate LOG_DB

Consume logs (write 7) are written to a dedicated `LOG_DB` connection, isolating append-heavy log writes from the operational DB that handles quota mutations.

---

## 5. Call Chain Reference

```
controller/relay.go:Relay()
  ├─ relay/helper/price.go:ModelPriceHelper()        [calc estimated cost]
  ├─ service/billing.go:PreConsumeBilling()          [WRITE 1 + 2: reserve]
  │    └─ service/billing_session.go:preConsume()
  │         ├─ model/token.go:DecreaseTokenQuota()   [WRITE 1]
  │         └─ service/funding_source.go:PreConsume()
  │              ├─ model/user.go:DecreaseUserQuota() [WRITE 2 - wallet]
  │              └─ model/subscription.go:PreConsumeUserSubscription() [WRITE 2 - sub]
  │
  ├─ relayHandler() → TextHelper()                  [upstream relay]
  │    └─ relay/channel/openai/adaptor.go:DoResponse()
  │
  ├─ service/text_quota.go:PostTextConsumeQuota()   [WRITE 3–7: settle]
  │    ├─ service/billing.go:SettleBilling()        [WRITE 3 + 4: delta]
  │    │    └─ service/billing_session.go:Settle()
  │    │         ├─ funding.Settle(delta)           [WRITE 3]
  │    │         └─ model/token.go:Decrease/IncreaseTokenQuota() [WRITE 4]
  │    ├─ model/user.go:UpdateUserUsedQuotaAndRequestCount()  [WRITE 5]
  │    ├─ model/channel.go:UpdateChannelUsedQuota()           [WRITE 6]
  │    └─ model/log.go:RecordConsumeLog()                     [WRITE 7]
  │
  └─ defer: billing_session.go:Refund()             [WRITE 8 + 9: async]
       └─ gopool.Go → funding.Refund() + token.Refund()
```

## 6. Key Files

| File | Line | Role |
|------|:----:|------|
| `controller/relay.go` | 68 | Entry point, orchestrates full lifecycle |
| `relay/helper/price.go` | 67 | Computes estimated quota cost |
| `service/billing.go` | 19 | PreConsumeBilling entry |
| `service/billing.go` | 34 | SettleBilling entry |
| `service/billing_session.go` | 186 | Core preConsume logic |
| `service/billing_session.go` | 41 | Core settle logic |
| `service/billing_session.go` | 82 | Async refund logic |
| `service/funding_source.go` | 14 | Wallet & Subscription funding interface |
| `model/user.go` | 910 | `DecreaseUserQuota` (users table write) |
| `model/user.go` | 903 | `IncreaseUserQuota` (refund) |
| `model/user.go` | 962 | `UpdateUserUsedQuotaAndRequestCount` (stats) |
| `model/token.go` | 405 | `DecreaseTokenQuota` (tokens table write) |
| `model/token.go` | 394 | `IncreaseTokenQuota` (refund) |
| `model/subscription.go` | 970 | `PreConsumeUserSubscription` |
| `model/subscription.go` | 1182 | `PostConsumeUserSubscriptionDelta` |
| `model/channel.go` | 805 | `UpdateChannelUsedQuota` |
| `model/log.go` | 207 | `RecordConsumeLog` |

---

## 7. Design Evaluation Under Heavy LLM Routing Load

### Overall Verdict: Fragile Under Concurrency

The two-phase pre-consume → settle model is architecturally sound (correctness-first reservation pattern), but the wallet-funding implementation has concrete correctness and throughput issues under load.

---

### 7.1 Non-Idempotent Wallet Refund — Silent Credit Loss

```
service/funding_source.go:57-64:

func (w *WalletFunding) Refund() error {
    // IncreaseUserQuota 是 quota += N 的非幂等操作，
    // 不能重试，否则会多退额度。
    // 订阅的 RefundSubscriptionPreConsume 有 requestId 幂等保护所以可以重试。
    return model.IncreaseUserQuota(w.userId, w.consumed, false)
}
```

The code itself documents the problem. The refund runs in a `gopool.Go` goroutine (`billing_session.go:106`) with **no retry, no idempotency key**. Under heavy load with DB timeouts or connection pool exhaustion, a failed refund means the user's credits are permanently lost.

The subscription path (`RefundSubscriptionPreConsume`) protects against this with `requestId`-based dedup within a transaction. The wallet path has no equivalent.

```
┌─── credit loss scenario under load ──────────────┐
│                                                    │
│  1. Pre-consume: user.quota -= 100  (SYNC, OK)     │
│  2. Upstream call: TIMEOUT                         │
│  3. defer: billing.Refund()                        │
│     → gopool.Go(                                   │
│         IncreaseUserQuota(userId, 100, false)       │
│       )                                            │
│  4. Client gets error response                     │
│  5. Refund goroutine: DB UPDATE fails (pool full)  │
│     → NO RETRY, NO LOG ALERT                       │
│     → user lost 100 credits permanently            │
│                                                    │
└────────────────────────────────────────────────────┘
```

**Severity**: HIGH — silent credit loss under DB instability.  
**Fix complexity**: LOW — add `requestId`-based idempotency to wallet refunds, same as subscriptions.

---

### 7.2 Double Write Amplification on Hottest Tables

Every request does two writes to the same row on `tokens` and `users`:

```
  UPDATE tokens SET remain -= 100    ← pre-consume
         ... upstream call (1–30s) ...
  UPDATE tokens SET remain += 20     ← settle (delta = -80)
```

In practice, the estimate is usually close to actual usage (delta is small). So the first write is 80% wasted — you subtract 100 only to add 20 back. Under heavy load, this is **2× needless DB I/O** on the tables with the most concurrent writes.

A single-write alternative:
```
  SELECT quota FROM users WHERE id = ?  ← read
  IF quota >= estimated THEN
      call upstream
      UPDATE users SET quota -= actual  ← single write
  ELSE reject
```

**Severity**: MEDIUM — wastes DB throughput; row locks are brief (~1ms) so lock-wait is rarely the bottleneck.  
**Fix complexity**: MEDIUM — requires a validate-then-single-write approach with an optimistic check or a short-lived advisory lock.

---

### 7.3 Pre-Consume Holds Quota for Entire Upstream Latency

```
     ┌─── quota reserved ───────────────────────────────────┐
     │                                                       │
  reserve                                               settle
     │◄────────── upstream call (1–30s for streaming) ──────►│
     │                                                       │
     │  Other concurrent requests see reduced balance         │
     │  → false "insufficient quota" rejections              │
     └───────────────────────────────────────────────────────┘
```

Under high concurrency with long-running streaming LLM calls, the **sum of reserved quota across all in-flight requests** can far exceed actual usage. This causes false rejections: the user has enough real balance, but it's all temporarily held.

Example scenario:
- User has 10,000 quota total
- 100 concurrent streaming requests, each pre-consumes 200 quota
- Total reserved: 20,000 → all subsequent requests see 0 balance → rejected
- Actual settle (after streams complete): 100 × 120 = 12,000 → user had room

**Severity**: MEDIUM — degrades availability under concurrency, not correctness. Self-corrects as streams complete.  
**Fix complexity**: MEDIUM — consider optimistic concurrency (validate, don't reserve), or apply a reservation multiplier (e.g., pre-consume 50% of estimate), or set a quota-hold timeout that auto-releases stale reservations.

---

### 7.4 No Transaction Across Token + Funding Pre-Consume

```
service/billing_session.go:199-222:

1. PreConsumeTokenQuota()     ← UPDATE tokens   (SUCCEEDS)
2. funding.PreConsume()        ← UPDATE users    (FAILS: subscription quota insufficient)
   → rollback: IncreaseTokenQuota()  ← THIRD write to undo step 1
```

The two pre-consume writes are separate DB operations, not wrapped in a single transaction. Under heavy load, step 2 failures become more frequent (subscription exhaustion, DB contention), triggering an **extra compensatory write** — making the write count 3× in the failure path.

It's also technically possible (though unlikely) for the rollback write to fail, leaving a partial debit on the token without a corresponding funding debit.

**Severity**: LOW-MEDIUM — only impacts failure paths, but those become more frequent under load.  
**Fix complexity**: LOW — wrap both writes in a single DB transaction so they atomically succeed or rollback.

---

### 7.5 Summary

| Issue | Impact Under Load | DB writes affected | Fix complexity |
|-------|------------------|:-----------------:|:---:|
| **Non-idempotent wallet refund** | Silent credit loss | Write 8–9 (refund path) | Low — add `requestId` dedup |
| **2× write to same row** | Wasted DB throughput | Writes 1–4 | Medium — validate-then-single-write |
| **Quota held during upstream latency** | False "insufficient" rejections | Writes 1–2 | Medium — optimistic / reservation multiplier |
| **No txn across token+funding** | Compensatory writes, partial debit risk | Writes 1–2 (failure path) | Low — wrap in DB transaction |
| **Async refund with no retry** | Permanent credit loss on DB blip | Writes 8–9 | Low — retry with backoff |

### 7.6 What Works Well

- **Subscription path**: well-designed with `requestId` idempotency, proper transactions, and retry logic.
- **Trust bypass**: correctly avoids unnecessary writes for high-balance users.
- **Batch update mode**: practical mitigation for write amplification when eventual consistency is acceptable.
<!--- **Two-phase model**: semantically correct — the design philosophy is sound, only the wallet implementation needs hardening.-->
