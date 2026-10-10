# Current POS business-rules audit (planning only)

## Purpose and safety boundary
This document records behavior observed in the existing `index.html` so a future cloud implementation can preserve intended shop workflows. It is not a statement that the tax treatment or receipt wording is legally correct, and it is not authorization to change production.

- Branch: `cloud-sync-safe-v2`
- Reviewed source file SHA: `ef88cd40ae47d3febc082dd95fc90fc84d24310b`
- No runtime code, Supabase schema, function, production data, or GitHub Pages deployment is changed by this audit.
- Keep cloud sales and the existing whole-snapshot sync OFF for multi-register use until the transaction implementation passes isolated tests.

## Rules observed in the current app

### Prices, discounts and VAT calculations
- Menu item prices are treated as VAT-inclusive; the receipt says prices include 12% VAT.
- Subtotal is the sum of item price × quantity.
- Per-item discount is a peso amount per unit, bounded in the UI/calculation by the item price.
- Senior citizen/PWD discount is 20% of the VAT-exclusive value of the assigned eligible line(s). The app calculates VAT-exclusive value as gross ÷ 1.12, calculates the VAT portion as gross minus that value, and calculates the 20% discount from the VAT-exclusive amount.
- The app assigns lines to SC/PWD entries by seat. A whole-ticket entry claims lines not already claimed by an earlier entry; this order/overlap behavior must be reproduced deliberately or corrected only after owner approval.
- A normal discount can be a fixed peso amount or percentage. The UI requests a reason. Percentage values above 20% and fixed discounts above ₱300 require Admin approval; percentages above 100% are rejected. The calculation caps the combined normal and Neighbor discounts at the remaining gross amount.
- Neighbor Discount is fixed at up to ₱100 and only applies to All Day Breakfast lines, after per-item discounts.
- Points redemption is presented when a customer has at least 50 points. The calculation gives a discount of up to ₱25 and the sale completion subtracts 50 points.
- Zero-rated status is applied to the regular remaining balance after discounts and points; the current calculation removes VAT by dividing that balance by 1.12. SC/PWD amounts are handled separately.
- Store credit applied to a sale is capped at the lower of the customer’s available credit and the calculated total. The amount due is total minus applied credit.

### Rewards and customer balances
- Silver/Gold tier is based on the customer's current points: Gold at 200 points or more.
- A completed sale awards floor(total ÷ ₱20) points.
- A customer redeeming points loses 50 points at completion.
- A customer who is Gold when the sale is completed receives 2% of the sale total as store credit. The current code checks the tier before adding points earned on that same sale.
- Customer credit and points are currently changed in browser memory as part of sale completion, not in a server transaction.

### Payments
- The payment screen supports multiple payment entries.
- Non-cash payment entries are not allowed to exceed the remaining balance; cash can exceed it and the excess is returned as change.
- The stored sale includes payment method, amount, optional reference, and change.
- Server-side code must independently validate amounts, allowed methods, references, total paid, and change; it must not trust values calculated by the browser.
- A repeated request must not create a second sale/payment or consume a second receipt number.

### Inventory, sales, refunds and receipt numbers
- Sale completion reduces each sold item’s stock locally, clamping stock at zero. It does not reject a sale when the requested quantity exceeds available stock.
- A refund marks the order refunded and restores its line quantities to stock locally. The UI refuses to refund an already-refunded order, but this browser-only guard is not concurrency-safe.
- Manual stock changes and price changes are currently local writes.
- The current app increments order and OR counters on the device. These counters are not safe as the authority for multiple registers.
- Shift open/close, cash pay-ins/outs, shift totals and audit history are currently local snapshot state.
- The current app has Cashier, Admin and Kitchen roles with default PINs in the source. PINs can be changed in each browser’s local storage. A browser-side role or PIN check is not authoritative server authorization.

## Server-side requirements inferred from these observations
1. Treat the browser as an untrusted display/client. Recalculate sale totals on the server from authoritative prices, discounts, customer balances, tax flags, and validated quantities.
2. Use one database transaction for sale creation, payment validation, stock checks/decrements, points/credit changes, and shared receipt-number allocation.
3. Reject insufficient stock instead of clamping it to zero.
4. Use unique idempotency keys for sale, payment, refund and stock-adjustment requests.
5. Make refunds atomic and one-time, with a linked refund record and stock restoration exactly once.
6. Protect price changes, large discounts, refunds and stock adjustments with server-verified Admin authorization and an audit trail. A shared role PIN does not reliably identify an individual employee.
7. Store receipt series as explicit configuration with server-side limits. Current series values are test values, not confirmed authorized receipt numbers. Do not issue live receipts until the actual authorized series and applicable limits are verified with the shop’s accountant/tax authority.
8. Keep local-only use available and cloud writes disabled by default. Test in an isolated project, first with two devices and then seven or more.

## Items requiring owner/accountant confirmation before implementation
- Confirm that the observed discount stacking order is intentional: SC/PWD assignment, per-item discount, normal discount, Neighbor Discount, points, zero-rated treatment, then store credit.
- Confirm how overlapping SC/PWD seats/whole-ticket discounts should be resolved.
- Confirm whether insufficient-stock sales should always be rejected, including any authorized negative-stock workflow.
- Confirm which payment methods are allowed and which require reference numbers; confirm duplicate-reference policy.
- Confirm the intended rewards rules: 50 points = ₱25 discount, earn 1 point per ₱20 of sale total, Gold threshold of 200 points, and 2% Gold cashback.
- **Owner-confirmed:** support both full-order and partial refunds. Partial refunds must be limited to original sold lines/quantities and the remaining refundable payment amount; use original sale pricing/discount/tax allocations, restore only refunded stock, and reverse the original points and Gold cashback/store credit without exceeding the original awards. If rewards being reversed have already been spent/redeemed, hold the refund for Admin review. Record the decision and process all effects atomically and idempotently. The current app does not yet implement these safeguards.
- Confirm whether staff names should remain role-only in audit records or whether individual staff identities are needed. Shared role PINs cannot provide individual attribution.
- Confirm actual authorized receipt series and receipt issuance requirements before any live rollout.

## Explicit non-changes
No app runtime code, production database schema, RLS policy, SQL migration, server function, production data, or deployment was changed as part of this audit.
