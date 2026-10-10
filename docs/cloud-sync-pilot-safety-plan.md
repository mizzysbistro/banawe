# Mizzy's Bistro POS cloud-sync pilot — safety plan

## Current status
- This is a development-only branch: `cloud-sync-pilot`.
- The existing `index.html` is intentionally unchanged at this stage.
- Do not merge or deploy this branch to GitHub Pages yet.
- Do not apply the prepared SQL migration to the production Supabase project yet.

## Why we are taking this in stages
The current POS stores orders, inventory, refunds, shifts, audit records, and receipt counters in one JSON snapshot (`public.pos_data`). A compare-and-swap version check can detect a stale write, but it does not safely merge two simultaneous sales. It also cannot, by itself, guarantee unique receipt numbers or transactional stock checks across seven or more registers.

## Required pilot gates
1. Use a separate development Supabase project or isolated database branch; never test writes against the live POS data.
2. Make cloud sync opt-in per device. A device must not silently sync just because an old login session exists.
3. Keep local persistence and provide a downloadable recovery backup when a conflict is detected.
4. Use server-side transactional operations for sales, payments, refunds, stock updates, shifts, and receipt numbering before treating this as safe for 7+ registers.
5. Verify authentication and row-level security. Never put a Supabase service-role or secret key in browser code.
6. Test on two devices first: simultaneous sales, repeated payment references, duplicate refund attempts, stock contention, receipt uniqueness, network loss/recovery, and reload persistence.
7. Only after all tests pass, expand to seven or more devices and prepare a reviewed production deployment.

## Production stop conditions
Do not deploy if any test can produce a lost sale, duplicate receipt number, duplicate refund, negative stock, silent stale overwrite, or unreconciled offline sale. Do not revoke existing production write paths until all deployed clients have been updated together.
