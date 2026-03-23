# ADR-005: Write-Ahead Trade Pattern

**Date:** 2025-03
**Status:** Accepted

## Context

When placing an order on BingX, two operations must succeed:
1. BingX API call (creates the position)
2. DB write (records it in our system)

If the BingX call succeeds but the DB write fails, we have a "ghost position" — a real open trade that our system doesn't know about. This is a critical risk.

## Decision

Write a `PENDING` trade row to the DB **before** calling BingX, using `clientOrderId = trade.id` for idempotency.

```
1. Generate UUID for trade
2. INSERT trade (status=PENDING) → commit to DB
3. Call BingX: POST /order with clientOrderId = trade.id
4. On success: UPDATE trade SET status=OPEN, bingxOrderId=xxx
5. On failure: status stays PENDING → reconciliation job handles it
```

## Consequences

**Positive:**
- No ghost positions — trade is always recorded before execution
- Idempotency — retrying BingX call with same clientOrderId is safe (BingX deduplicates)
- Reconciliation job can always find and resolve PENDING trades

**Negative:**
- Adds one extra DB write per trade (negligible overhead)
- PENDING state adds complexity to state machine
- Reconciliation job must run reliably
