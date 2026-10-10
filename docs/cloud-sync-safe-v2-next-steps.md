# Cloud sync safe-v2 implementation checklist

## Branch and release safety
- Work only on `cloud-sync-safe-v2`.
- Keep GitHub Pages production deployment on the current production branch.
- Never apply a development SQL file to the production Supabase project.
- Never place a Supabase service-role/secret key in browser code.
- Do not enable the existing whole-snapshot cloud sync for multiple registers.

## Verified current behavior
- `index.html` stores the app's operational state in a local snapshot.
- Cloud sync writes the complete snapshot to `public.pos_data` using an unconditional upsert keyed by `user_id`.
- A periodic sync checks for remote changes, but simultaneous devices can overwrite one another.
- The `finish()` client function decrements stock and increments local order/receipt counters before recording the sale.
- Therefore, the current cloud sync is not safe for seven concurrent registers.

## Implementation approach
1. Map every operation that changes money or stock: sale completion, payment recording, refunds/voids, stock adjustments, shift open/close and cash reconciliation.
2. Define a normalized server-side transaction model. Every write must be authenticated and checked by PostgreSQL in one transaction.
3. Add unique idempotency keys for sale/payment/refund requests so retries cannot double-charge or double-record an operation.
4. Allocate receipt/order numbers on the server under a database constraint/lock, not from a per-device counter.
5. Validate and decrement inventory atomically on the server; reject a sale if stock is insufficient.
6. Keep an operation ledger and return the canonical saved result to each device. Do not use last-write-wins for business data.
7. Add a feature flag that defaults to OFF and require an explicit opt-in on each test device.
8. Keep the existing local persistence and add conflict/recovery export before any migration away from snapshot sync.

## Required validation before production
- Static review of every changed file and SQL policy.
- Run migration and tests only against an isolated, non-production database.
- Verify concurrent sales, receipt uniqueness, repeated payment/refund requests, stock contention, offline retries, app restart, shifts and reconciliation.
- Test with two devices first; then seven or more devices under concurrent load.
- Confirm recovery/rollback instructions and backup restoration.
- Production rollout requires explicit approval after the tests pass.

## Current status
This file is planning only. No production database migration has been applied, and the existing POS runtime code has not been changed by this checklist. Do not merge or deploy based on this file alone.
