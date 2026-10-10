# Cloud sync safe-v2 — sale transaction contract (draft)

## Safety boundary
- This is a design document on `cloud-sync-safe-v2` only.
- It does not change `index.html`, GitHub Pages, Supabase schema, or live data.
- Do not deploy a function or apply SQL until this contract is reviewed and tested against an isolated database.
- Keep the existing snapshot sync disabled for multi-register use.
- Cloud sales must remain OFF until the authentication, transaction, and concurrency tests pass.

## Current staff sign-in model
The shop has no Supabase login for staff. Staff use PINs in the POS, and multiple staff members may share a role PIN (for example, Cashier).

Implications:
- Treat a shared PIN as proof only that the operator knows the role credential; it does not identify the individual person using it.
- A sale audit record may record the verified role and register, but must not claim reliable individual cashier attribution when a role PIN is shared.
- Keep the familiar POS PIN screen, but verify the credential in a trusted server-side endpoint before issuing a short-lived, limited-purpose session/token. Do not trust a role name sent by the browser.
- Store only a strong salted hash of each role PIN server-side; never store or log the raw PIN. A four-digit PIN is weak, so enforce server-side rate limits, increasing delays/temporary lockouts, and audit failed attempts. Do not reveal whether a particular PIN/role exists.
- Restrict each role to explicit permissions. For example, Cashier can complete ordinary sales; refunds, stock adjustments, price overrides, and shift corrections should require the configured manager permission.
- PIN changes and role deactivation must take effect centrally across all registers. Do not use browser local storage as the source of truth for cloud authorization.
- If individual staff accountability is needed later, introduce individual staff PINs or a separate staff selection/identification step; do not infer identity from a shared role PIN.

## Why define the contract first
The current `finish()` operation changes several pieces of business state at once: it reduces menu stock, adjusts customer points/store credit, records the order and its payment list, and increments local order/official-receipt counters. A safe server operation must commit these related changes together or commit none of them.

## Proposed first operation
One authorized request, `complete_sale`, represents a fully paid ticket. The database transaction should:
1. Validate a server-issued session/token and that its verified role is authorized to complete sales for this shop/register.
2. Check a client-generated, persisted idempotency UUID. Repeating the same request UUID must return the original saved result without charging, decrementing stock, or allocating another receipt.
3. Validate ticket lines against the canonical menu/product records and validate allowed quantities, prices, modifiers, discounts, tax treatment, and payment methods.
4. Recalculate totals on the trusted server/database from approved rules; do not trust browser-supplied totals as authoritative.
5. Lock/update the affected stock rows and reject the whole sale if any item lacks sufficient stock. No item may become negative.
6. Validate the payments sum against the amount due, including change and store-credit rules; enforce payment-reference uniqueness where applicable.
7. Apply eligible customer credit/points changes exactly once.
8. Allocate the official receipt/order number on the server under a database constraint/lock, honoring the shop's configured authorized series. Never invent or extend the permitted series.
9. Save the order, line items, payments, stock movements, customer ledger entries, and audit event in the same transaction.
10. Return the canonical saved order, receipt number, and updated values to the device.

## Request shape (concept only)
- `request_id`: UUID generated once for this sale and retained unchanged for retries.
- `ticket_id`: local ticket identifier for traceability, not a globally unique receipt number.
- `register_id`: stable register/device identifier.
- `staff_role_session`: server-issued short-lived credential; do not trust a browser-supplied role string.
- `lines`: product IDs, quantities, approved modifiers/options, and any permitted seat/course details.
- `discount/tax/customer-credit inputs`: identifiers and selections only where possible; server recomputes eligibility and totals.
- `payments`: method, amount, and external reference when relevant.
- `shift_id`: currently open server-recognized shift.
- `note`: optional bounded text.

The browser should not be allowed to submit an authoritative stock balance, receipt number, total, customer balance, points balance, or staff role.

## Current POS behaviors that must be preserved or explicitly redesigned
- Menu stock is currently held in `menu[].s`; product identity is `menu[].id`; current price is `menu[].p`.
- `calc(t)` includes VAT-inclusive pricing, SC/PWD discount allocation, item discounts, percentage/fixed discounts, breakfast discount, points redemption, zero-rated sales, and store credit. Each rule needs tests before the server becomes authoritative.
- `finish()` records the selected payment list, change, customer points/credit changes, order, shift link, and OR number.
- Refund currently marks an order refunded and restores its line quantities. A server-side refund must be idempotent and must not restore stock twice.
- Stock adjustments, voids, shift open/close, cash pay-ins/pay-outs, reconciliation, and refund flows also need server-side transactions before full multi-register operation is safe.

## Suggested database responsibilities
Use normalized records rather than one shared JSON snapshot:
- shop/register/staff-role authorization and PIN verifier;
- menu items and stock quantity;
- sales and sale lines;
- payments and unique external payment references where required;
- stock-movement ledger;
- customer account ledger for credit/points;
- shifts and shift cash movements;
- refunds/voids;
- audit events, including verified role and register;
- rate-limit/failed-attempt records;
- idempotency requests/results;
- receipt-series configuration and allocation.

Exact table names, constraints, grants, and policies remain to be designed. All tables in exposed schemas need deliberate RLS and grants. Do not expose a service-role/secret key in the browser.

## Acceptance tests for this first slice
- An invalid PIN cannot obtain a server-issued role session.
- Repeated PIN failures trigger the configured server-side throttling/temporary lockout.
- A cashier-role session cannot perform manager-only actions.
- A deactivated/changed role PIN stops working across all registers after the intended session-expiry/revocation window.
- Two registers sell the last unit at the same time: at most one sale succeeds.
- Two simultaneous sales receive distinct authorized receipt numbers.
- Retrying the same request after a timeout returns the same order/receipt and changes stock only once.
- A payment reference cannot be accepted twice where uniqueness is required.
- Underpayment, invalid discounts, unauthorized roles, closed shifts, or insufficient stock reject the whole transaction.
- A failed transaction leaves no partial order, payment, points, stock, or receipt allocation.
- Refund retries restore stock at most once.
- Existing local-only mode still works while cloud sync remains opt-in and OFF by default.

## Next gate
Before writing migration SQL, map the exact existing PIN/role behavior and manager permissions, confirm the authorized receipt series rules, and write test cases for every `calc(t)` rule. Then draft the schema and transaction in this branch only. No production changes without explicit approval.
