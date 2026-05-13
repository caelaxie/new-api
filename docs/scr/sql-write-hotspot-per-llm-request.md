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
