# Cloud Sync Safe-v2 — refund transaction design (draft)

## Status and safety boundary

This document is a design proposal on `cloud-sync-safe-v2` only. It does not change the POS runtime, Supabase schema/data, SQL, functions, or production deployment. Do not apply migrations or enable cloud sync from this document alone. Keep the existing whole-snapshot sync OFF for multi-register use.

## Goal

Support both full-order and partial refunds without allowing duplicate refunds, excess payment returns, incorrect stock, or incorrect loyalty balances across concurrent registers.

## Recommended record model

Use normalized records with server-owned values. Exact names are illustrative and require schema review.

- `sales`: immutable sale header with id, request/idempotency UUID, register, shift, receipt-series/number, customer, totals, payment-due amount, reward points earned/redeemed, cashback/store-credit awarded, store credit used, status and timestamps.
- `sale_lines`: immutable original line snapshots: product ID/name, modifier, quantity sold, unit price, item discount, allocated order/neighbor discount, SC/PWD and zero-rated/tax allocation, final refundable gross/net amount, and original reward-allocation basis. Store the final per-unit/line allocations at sale time; do not recompute from current menu configuration.
- `sale_payments`: one row per tender entry, including method, gross tender received, change attributed to that tender, net contribution to the bill, external reference, and amount already refunded. For cash overpayment, refundable paid amount is the net contribution after change, not the cash handed over.
- `refunds`: refund ID, unique request UUID, original sale ID, full/partial kind, reason, requested/approved status, verified Admin role/session, register, timestamps, amount, and audit metadata.
- `refund_lines`: original sale-line ID, quantity refunded by this refund, and the stored allocated price/discount/tax amount being reversed.
- `refund_payment_allocations`: each refund's amount allocated to a specific original payment row/method, including external refund reference/status where supported.
- `inventory_movements`: immutable stock ledger entry tied to sale or refund, product, signed quantity, source line, and unique operation key.
- `customer_ledger`: append-only entries for points earned/reversed/restored and store credit earned/reversed/restored/used; include source sale/refund and unique operation keys.
- `idempotency_requests`: unique request UUID plus operation type, result reference and safe response metadata.
- `audit_events`: verified role/session, register, action, reason, before/after summaries, approval/rejection, and timestamps. Do not claim an individual identity when a shared role PIN cannot establish one.
- `receipt_series`: server-controlled series/limits; refunds reference the original receipt and never allocate a new sale receipt number.

Do not make browser-supplied totals, stock balances, refund eligibility, role names, reward balances, or payment balances authoritative.

## Refund transaction

Implement one trusted server/database operation (conceptually `process_refund`) that performs all validation and writes in one database transaction:

1. Verify the server-issued session and permission. Refunds require the configured manager-level permission (currently expected to be Admin, pending confirmation of role mapping).
2. Require a persisted client-generated idempotency UUID. A retry with the same UUID returns the same result; reusing it with a different payload is rejected.
3. Lock the original sale, its lines, payment rows, relevant customer ledger/balance rows, and stock rows in a consistent order to serialize simultaneous refunds.
4. Reject unknown, cancelled, unpaid, or otherwise ineligible sales. Check that requested quantities are positive and that sold quantity minus all completed/pending approved refunds remains sufficient for every line.
5. Derive the refundable value from immutable sale-time line allocations. A partial refund cannot exceed remaining refundable value for each line or the remaining net paid amount. Current menu prices and discount rules are never consulted.
6. Produce a preview showing line quantities, item/discount/tax breakdown, proposed tender allocations, inventory changes, reward changes, and any Admin review requirement. Do not commit if review is required and not approved.
7. Allocate refund money only against the remaining refundable amount of the original tenders. Never refund more than a payment's net bill contribution. External tenders (Card/GCash/Maya) require the supported provider/manual workflow and reference; do not claim that a POS record itself reverses a provider payment. If the required tender allocation cannot be made safely, hold for Admin review.
8. Restore only the refunded quantities to inventory, using one uniquely keyed movement per refund line.
9. Append customer-ledger entries: reverse the original earned points and Gold cashback/store credit attributable to the refunded portion; restore the portion of points redeemed on the original sale; and re-credit any store credit used as tender on the refunded portion. Do not mutate historical ledger entries.
10. If reversing earned points/cashback would consume rewards already spent/redeemed, stop before committing and require Admin review. Show original award, amount already used, proposed reversal and the resulting balance; record approval/rejection and reason. Never silently create a negative balance.
11. Save refund header/lines, payment allocations, stock movements, reward ledger entries, audit events and final order status atomically. Any validation or write failure rolls back all effects.
12. Mark the sale fully refunded only when its remaining refundable lines/payment balance are zero; otherwise retain a partially refunded status. Do not allow later refunds beyond remaining quantities/value.

## Allocation and rounding rules proposed for implementation

These rules need approval before code is written:

- **Line value:** Freeze each line's final allocated amounts at sale time after item discount, order/neighbor discount, SC/PWD treatment, zero-rating/tax calculation, and other eligible discounts. For partial quantity refunds, calculate from the original per-unit allocation; assign any rounding remainder deterministically to the final refundable quantity so that all refunds together cannot exceed the original line amount.
- **Order-level discounts:** Allocate the order-level fixed/percentage/neighbor discount across eligible lines in proportion to their eligible pre-discount amounts, round to cents, and assign residual cents deterministically. Respect eligibility rules; do not allocate the Neighbor discount to non-breakfast lines or SC/PWD amounts outside their eligible lines. This allocation method is a recommendation, not a confirmed business rule.
- **Mixed tenders:** At sale time, allocate the bill's net tender contributions across sale lines in proportion to their final payable amounts, with deterministic cent rounding. For a refund, reverse the selected line's remaining tender allocations, capped by each original tender's refundable balance. This is more auditable than guessing a new tender method at refund time. The owner must approve this method; if unavailable or the tender/provider cannot be reversed, hold for Admin review.
- **Cash change:** Store both amount handed over and change. The refundable cash contribution is the amount applied to the bill after change, not the full cash tendered.
- **Store credit and points redemption:** Treat store credit used and points redeemed as separate sale-tender/discount components in the sale record. A refund should restore the proportional store credit used and points redeemed for the refunded lines, in addition to reversing rewards earned from the sale. Define the points restoration rounding rule and test it before rollout.
- **Points earned:** Current behavior awards floor(sale total / ₱20). Save the actual award and line allocation at sale time. Partial refunds reverse the allocated points; total reversals may not exceed the original earned points.
- **Gold cashback:** Current behavior awards 2% of sale total when the customer is Gold at the time of sale. Save the actual amount and allocation at sale time; do not re-evaluate current tier. Partial refunds reverse the allocated cashback/store credit; total reversal may not exceed the original award.
- **Admin review:** If any amount that would be reversed has already been consumed, require explicit Admin decision before finalizing. If rejected, no refund, stock movement, payment allocation or reward change is committed.

## Required concurrency and failure tests

- Two registers submit the same refund UUID: exactly one refund is created and both receive the same result.
- Two different refund requests concurrently target the same remaining line quantity: combined refunded quantity cannot exceed quantity sold.
- Two concurrent refunds cannot exceed the remaining net refundable balance of any tender.
- Full refund after one or more partial refunds refunds only the remaining quantity/value.
- Partial refund after full refund is rejected.
- Cash overpayment/change never inflates the refundable cash amount.
- Mixed-tender allocations sum exactly to the refundable amount, never exceed any original tender, and cents reconcile after repeated partial refunds.
- Discounts, SC/PWD, zero-rated tax and points/store-credit redemption are apportioned from original snapshots and reconcile to the original sale.
- Stock, payments, reward ledger, refund status and audit event all commit together or none do.
- Already-spent rewards trigger Admin review; approval/rejection is audited and duplicate approvals/retries do not duplicate effects.
- Provider refund failure or uncertain external status remains pending/reconcilable; it must not be reported as successfully refunded.
- Receipt numbers are not reallocated by refunds; the original receipt remains traceable.
- Test data is clearly marked and isolated from official receipt series and production data.

## Decisions still needed before implementation

1. Approve/revise proportional mixed-tender allocation and confirm how card/GCash/Maya refunds are actually performed.
2. Approve/revise proportional order-discount allocation and cent-rounding rules.
3. Confirm whether points redeemed on a refunded sale are restored proportionally, and approve rounding.
4. Confirm Admin is the manager-level role allowed to approve/complete refunds.
5. Verify the authorized receipt series and applicable receipt/refund-document requirements with the owner and accountant/tax adviser before live rollout.

## Explicit non-changes

No runtime code, Supabase schema, SQL migration, RLS/grants, server function, production data, receipt configuration, or deployment has been changed. Existing whole-snapshot sync remains unsafe for multi-register use and must stay off until the new transaction model is implemented and tested in isolation.
