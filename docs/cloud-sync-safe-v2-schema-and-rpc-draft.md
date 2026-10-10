# Cloud Sync Safe-v2 — schema and server transaction draft

## Status: review draft only — not executable or deployed

This is a design draft for review on branch cloud-sync-safe-v2. It is not a migration and has not been run against any database. Do not paste it into the Supabase SQL Editor or apply it to production. The existing app, Supabase schema/data, and deployment remain unchanged.

**Safety gate:** the current whole-snapshot sync is not safe for multi-register use. Keep cloud sync OFF for multiple registers until a server-authoritative transaction implementation passes isolated tests.

## Proposed policy defaults (not yet owner-approved)

These defaults make the design concrete, but remain proposals until explicitly confirmed:

1. **Partial item refunds:** use sale-time recorded line allocations, never current menu prices or settings.
2. **Order-level discounts:** at sale time, allocate each eligible order-level discount proportionally across eligible lines, round to cents, and assign residual cents by a stable documented line ordering. Line allocations must sum exactly to the original discount.
3. **Mixed tenders:** allocate net bill contribution deterministically at sale time. A refund reverses the original line's remaining tender allocations and cannot exceed any tender's remaining refundable balance. If allocations are missing/ambiguous or provider reversal needs a manual action, hold for Admin review.
4. **Cash change:** only the amount applied to the bill after change is refundable; not the gross cash handed over.
5. **Points/store credit redeemed on the sale:** restore the proportional amount attributable to refunded lines using stored line allocations and deterministic rounding.
6. **Rewards earned:** reverse only the points and Gold cashback/store credit actually recorded at sale time. Allocate awards to lines at sale time; cumulative reversals cannot exceed the original award.
7. **Already-spent rewards:** do not finalize automatically. Present the proposed reversal and amount already consumed for Admin review; no stock, payment, or ledger effects occur until approved.
8. **External payments:** a database row alone does not reverse card/GCash/Maya payment. Track provider/manual refund status and references; never report success while status is pending or uncertain.
9. **Refund permission:** assume Admin is the manager-level approver based on the owner's earlier confirmation, but re-confirm mapping against actual Cashier, Admin, and Kitchen roles before implementation.

## Logical schema

Prefer normalized records over a shared JSON snapshot. Exact names and data types must be finalized in a real migration after reviewing the existing menu/customer model.

| Table | Important fields and constraints |
|---|---|
| registers | Stable register UUID, display name, active flag, timestamps. |
| role_credentials | Role, strong salted PIN hash, active flag, credential version, updated timestamp. Never store raw PINs. Failed-attempt throttling must be server-controlled. |
| role_sessions | Opaque session/token hash, role, register, expiry, revoked-at; server code only can create/revoke/read it. |
| menu_items | Stable item UUID, name snapshot source, active/sellable flag, current price and stock-tracking mode. |
| inventory_balances | Item UUID primary key, on-hand quantity and version; nonnegative constraint for tracked inventory. |
| sales | Sale UUID, unique request UUID, register/shift IDs, receipt series/number, customer ID, calculation summary, status and timestamps. Unique receipt-series + number. |
| sale_lines | Sale ID, product ID/name snapshot, quantity sold, modifiers, unit price, discount/tax/zero-rating allocations, final payable amount, allocated points/cashback/store-credit and points-redemption values. Store immutable sale-time allocations. |
| sale_payments | Sale ID, method, amount tendered, change, net contribution, provider reference/status, amount refunded. Net contribution cannot be negative. Unique external reference where appropriate. |
| refunds | Unique request UUID, original sale ID, status, reason, requester/approver role sessions, amount and timestamps. |
| refund_lines | Refund ID, original sale-line ID, quantity and original allocated amount reversed. Cumulative refunded quantity cannot exceed sold quantity. |
| refund_payment_allocations | Refund ID, original sale-payment ID, amount, provider/manual reference and status. Cannot exceed remaining refundable contribution. |
| inventory_movements | Append-only movement, item, signed quantity, source type/ID and unique operation key. |
| customer_ledger | Append-only points/credit/cashback deltas, customer, ledger kind, original sale/refund reference and unique source-operation key. |
| shifts / shift_cash_movements | Server-recognized register shift and cash pay-in/out/reconciliation records. |
| idempotency_requests | Request UUID, operation, payload hash, status, result ID and safe response metadata. Same UUID with different payload is rejected. |
| receipt_series | Series ID, authorized start/end and next number, active flag. Verify official series before production; never wrap on exhaustion. |
| audit_events | Append-only action, verified role, register, reason, approval decision, source record and timestamps. Shared role PINs do not establish individual identity. |

### Constraints and data rules

- Use UUID primary keys and foreign keys; choose a consistent exact-money representation (numeric decimal or integer minor units), never floating point.
- Use unique constraints for idempotency UUIDs, receipt-series + number, external payment references where applicable, and each sale/refund inventory or ledger source operation.
- Completed sale calculation snapshots are immutable. Corrections are linked transactions, not silent history rewrites.
- Store line-level allocations at sale time so refunds do not recalculate historical discounts, taxes or rewards.
- Decide whether quantities may be fractional before choosing quantity types/checks.
- A locally recorded provider refund is not proof that the provider completed it.

## Server transaction contract

Expose narrowly scoped operations through a trusted server-side endpoint/Edge Function, not arbitrary browser writes. The browser sends identifiers/selections and an idempotency UUID; the server recalculates and validates authoritative values.

### complete_sale

1. Verify the short-lived server-issued role session, register binding and permission.
2. Validate request UUID and payload hash; return the saved result for an identical retry, reject UUID reuse with a different payload.
3. Lock the open shift, inventory rows and receipt-series row in a consistent order.
4. Load canonical menu/customer/discount settings; validate lines, stock, payment methods and external references.
5. Calculate VAT-inclusive pricing, SC/PWD allocation, item/order/breakfast/Neighbor discounts, zero-rated sales, points redemption, store-credit use and Gold cashback on the server.
6. Allocate deterministic line-level discounts, tax, payable value, tender contribution, points and cashback; save them as sale-time snapshots.
7. Verify net payment contributions sum exactly to amount due and change is represented correctly.
8. Allocate one shared server receipt number under a row lock and unique constraint. Never accept the receipt number from the browser.
9. Write sale, lines, payments, stock movements/balances, customer ledger, idempotency result and audit event in one transaction.
10. Commit and return the canonical saved sale. Any failure rolls back all writes, including stock and receipt allocation.

### preview_refund

1. Verify role session and preview permission.
2. Load original sale and all completed/pending refund lines and payment allocations.
3. Calculate remaining quantity/value for each line and tender from immutable sale-time snapshots.
4. Compute proposed line refund, tender allocation, stock restoration and reward reversal with deterministic rounding.
5. If rewards were consumed or external tender handling is uncertain, return a review-required preview. Preview does not mutate stock, payments or customer balances.

### process_refund

1. Verify Admin/manager permission and unique idempotency UUID. Verify the approved preview matches the refund payload.
2. Lock original sale, lines, prior refunds, tender rows, customer ledger/balance rows and inventory rows in a consistent order.
3. Recompute eligibility and remaining quantities/amounts inside the transaction; never trust browser preview totals.
4. If review is required, require a recorded Admin decision and reason before financial/inventory effects. Rejection records an audit decision but no refund side effects.
5. Create refund header/lines and allocations against original payment methods, subject to remaining tender amounts and provider/manual workflow.
6. Append inventory movements and update tracked stock once. Append reward/credit ledger reversals/restorations using stored allocations and unique constraints.
7. If external provider status is pending/uncertain, keep refund pending and do not report it completed.
8. Set sale status to partially or fully refunded based on remaining refundable lines/payment amount.
9. Write idempotency result and audit event, then commit atomically. Any database error rolls back every side effect.

## Security model

- Never put a service-role/secret key in index.html or a browser bundle.
- Do not grant browser roles unrestricted INSERT/UPDATE/DELETE on sales, payments, inventory, refunds, customer ledger, receipt series, idempotency or role-session tables.
- Enable RLS on exposed tables with explicit least-privilege policies. Keep internal tables in a non-exposed schema where feasible.
- If a privileged database function is required, place it in a non-exposed schema, restrict EXECUTE grants, pin search_path, validate caller/session inside the function, and review its bypass-RLS behavior.
- Server-side PIN verification needs salted hashes, rate limiting, progressive delays/temporary lockouts and audit events.
- Use isolated test credentials/database and clearly labelled test receipt series. Do not reuse production secrets.

## Isolated test gates

1. Confirm business policies and inspect the current menu/customer/order model; capture test cases for all current calculation rules.
2. Use an isolated test project/database with no production customer/payment data. Do not use paid branches or add-ons without approval.
3. Implement schema/functions only in that isolated environment and review grants, RLS and security advisors.
4. Test rollback, idempotent retries, receipt locking, last-unit concurrency, payment allocation, tax/discount rounding, customer ledger and provider-pending cases.
5. Test with two independent devices, then simulate at least seven concurrent registers.
6. Keep cloud sales disabled by default and preserve local-only operation.
7. Do not merge/deploy or alter production until the owner explicitly approves after reviewing test results.

## Owner decisions to confirm before implementation

Please review the proposed defaults, especially:
- Proportional order-discount allocation across eligible lines.
- Proportional mixed-tender refund allocation against original tender contributions.
- Proportional restoration of redeemed points/store credit on partial refunds.
- Admin approval whenever rewards to reverse have already been spent.
- Actual external payment-provider workflow and receipt/refund documentation requirements.

No response should be interpreted as authorization to apply SQL or deploy code. This is design-only.
