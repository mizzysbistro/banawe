# Cloud-sync sale test matrix (planning only)

## Status
Planning checklist only. These are proposed tests, not tests already run. Do not connect them to production. Use an isolated Supabase project with synthetic data and clearly marked test receipt numbers.

## Gate 0 — resolve business rules
Before asserting final expected totals, the owner should confirm the rules listed in `current-pos-business-rules-audit.md`, especially discount stacking, SC/PWD overlap, points/credit treatment on refunds, allowed payment methods, and insufficient-stock behavior. Tax/receipt treatment must be reviewed with the shop's accountant or tax authority.

## Calculation test cases
Use a deterministic test menu and customer balances. Store the inputs and expected line-level outputs in each test fixture.

- [ ] VAT-inclusive regular sale: verify subtotal, VAT-exclusive sales, 12% VAT, total and rounding.
- [ ] Item discount: verify it is applied per unit and cannot exceed the line amount.
- [ ] Fixed discount and percentage discount: verify bounds, approval requirements and reason audit.
- [ ] Neighbor Discount: verify only eligible All Day Breakfast lines qualify and the maximum is ₱100.
- [ ] Senior/PWD, one seat: verify VAT-exclusive base, 20% discount, exempt amount and receipt breakdown.
- [ ] Multiple SC/PWD entries and overlapping seat/whole-ticket selections: verify the owner-approved allocation rule.
- [ ] Points redemption: test exactly 50 points, fewer than 50 points, and a sale below ₱25 remaining.
- [ ] Zero-rated sale: verify eligibility evidence is required and the server recomputes tax treatment.
- [ ] Store credit: test zero, partial credit, credit equal to total, and credit above total.
- [ ] Rewards: verify 1 point per ₱20, Gold threshold and 2% cashback according to the confirmed rules.
- [ ] Mixed payments: exact balance, multiple methods, cash overpayment/change, non-cash overpayment rejection, invalid amounts and missing/duplicate references.
- [ ] Rounding boundaries: values ending in fractions of a cent and multiple discounted lines; confirm totals reconcile to payments.

## Transaction and concurrency tests
- [ ] Two devices sell the final unit of the same item at the same time: only one succeeds; stock never goes negative.
- [ ] Seven or more devices complete sales concurrently: no lost sales, duplicated sale IDs, duplicated receipt numbers, or overwritten orders.
- [ ] Retry the same sale request after a simulated timeout: the original result is returned; no duplicate stock, payment, points, credit or receipt changes.
- [ ] Submit two different sales simultaneously: each gets its own unique server-assigned receipt number from the single shared sequence.
- [ ] Force a database error halfway through sale completion: every change rolls back together.
- [ ] Disconnect/reconnect and retry an offline queued sale: no silent duplicate or stale overwrite; require clear conflict behavior if safe completion is impossible.
- [ ] Two staff attempt to refund the same order concurrently: one refund record and one stock restoration only.
- [ ] Repeat the same refund request: no second refund or second stock restoration.
- [ ] Admin-only operations with an invalid, expired, or insufficient role credential are rejected server-side.
- [ ] Shared role PIN audit logs identify the role/register but do not claim to identify an individual staff member.
- [ ] Receipt series exhaustion: last allowed number succeeds; the next sale is blocked without wrapping or inventing a number.
- [ ] Price or menu configuration changes during checkout: server applies a defined version/price rule and records what was actually charged.
- [ ] Shift close while another register is recording a sale: the transaction follows the approved shift boundary and reconciliation rule.

## Recovery and rollout tests
- [ ] Cloud feature flag is OFF by default on a new install and after sign-out.
- [ ] Local-only selling still works with cloud disabled.
- [ ] A failed cloud operation preserves a recoverable local draft and gives a clear status; it must not show a sale as completed unless the server confirms it.
- [ ] Export and restore a backup into a separate test environment.
- [ ] Confirm there is no service-role key or other secret in browser assets.
- [ ] Review database grants, RLS, function permissions and audit retention before enabling a pilot.
- [ ] Complete two-device pilot before seven-device concurrency testing.
- [ ] Require explicit owner approval before merging, applying production SQL, enabling cloud sales, or deploying.

## Pass criteria
All transaction invariants must hold under retries and concurrency: each confirmed sale is recorded exactly once; payments reconcile to amount due plus change; stock never goes negative unless an explicitly approved policy allows it; customer points/credit change exactly once; refunds and stock restoration happen exactly once; receipt numbers are unique and inside the configured series; no device can overwrite another device’s business records with a stale snapshot.


## Full and partial refund tests (planning only)
- [ ] Full refund reverses all refundable payments, restores all sold quantities exactly once, and reverses the original sale's reward points and Gold cashback/store credit exactly once.
- [ ] Partial refund of one line/quantity returns only the eligible amount for that line using the original sale's recorded pricing and discount/tax allocation.
- [ ] Multiple partial refunds cannot cumulatively exceed any original line quantity or the remaining refundable payment amount.
- [ ] Mixed-payment refunds follow the approved allocation policy and never refund more than the amount paid; ambiguous payment cases route to Admin review.
- [ ] Partial reward reversals use deterministic rounding; total points/cashback reversed across partial refunds never exceeds the original award.
- [ ] If rewards to be reversed have already been spent/redeemed, the refund is held for Admin review and displays the original award, amount used and proposed reversal.
- [ ] Admin approval/rejection is audited; rejected or failed refunds leave payment, stock and reward balances unchanged.
- [ ] Repeating or concurrently submitting the same full/partial refund does not duplicate payment adjustments, stock restoration or reward reversals.
- [ ] Failure at any step rolls back the entire refund transaction.


## Refund allocation and transaction-integrity tests (design follow-up)
- [ ] Cash overpayment is recorded separately from change; refund ceiling uses net cash contribution, not cash handed over.
- [ ] Partial refunds of discounted, SC/PWD, zero-rated, or points/store-credit-assisted sales use immutable sale-time line allocations and reconcile to the original sale.
- [ ] Order-level discount and cents-rounding allocations are deterministic; the sum across all lines equals the recorded original discount/tax totals.
- [ ] Mixed-tender refund allocations sum to the approved refund amount and never exceed each original tender's remaining refundable contribution.
- [ ] Store credit used and points redeemed on the original sale are restored proportionally under the approved policy; earned points and Gold cashback are reversed without exceeding the original awards.
- [ ] A provider refund that is pending, failed, or uncertain is not displayed as successfully refunded and can be reconciled without duplicate payout.
- [ ] Refund transaction failure leaves refund status, payment allocations, stock movements, customer ledger and audit state unchanged.
