# PIN-only staff authentication for cloud sync (design draft)

## Scope and safety
- Branch: `cloud-sync-safe-v2`.
- Design only: no runtime app code, database, or production deployment is changed by this document.
- Staff continue using the existing POS PIN screen. Do not require staff email/password entry at the register.
- The existing whole-snapshot cloud sync must remain OFF for multi-register use until server-side transactions and authorization are implemented and tested.

## What the current browser code does
- Staff records and default four-digit PINs are embedded in `index.html`.
- `key()` compares the entered PIN against the in-memory `staff` list.
- `loadPins()` reads changed PINs from browser `localStorage`; `savePins()` writes them there.
- The browser assigns `me` after a match, and role checks run in client-side JavaScript.
- These checks can improve the user experience but cannot be trusted by the database as proof of staff identity or role. A browser user can inspect/change JavaScript and local storage.

## Proposed model: paired register + server-verified staff PIN
1. **Pair each register once.** An administrator creates a short-lived, one-use pairing code through a protected setup flow. The code is consumed server-side and the server registers that device. Avoid relying on a public device UUID as a secret.
2. **Store staff credentials server-side.** Store a per-staff salted password hash (never plaintext PINs) and the staff role/status in protected database tables. Do not seed production credentials from the current hardcoded demo PINs. Require administrators to set new PINs during setup.
3. **Limit four-digit PIN risk.** Enforce persistent server-side rate limits by both device and staff/account, increasing cooldown after failures and logging lockouts. Four-digit PINs are low entropy, so a PIN alone should not be treated as a strong remote credential.
4. **Create a short-lived server session.** After successful PIN verification on a paired register, return a revocable, expiring session credential. Store it using the safest supported browser mechanism for this static app; do not put service-role keys or database admin credentials in the browser. Rotate/revoke sessions on PIN reset, staff deactivation, or device unpairing.
5. **Authorize every business operation on the server.** The server derives staff ID, role, and register ID from the verified session; it must not trust those fields if supplied in the request. Admin-only operations, refunds, stock edits, and shift actions need explicit server-side permission checks.
6. **Use transactional operations.** The authenticated server session calls the database transaction for sale completion, payment recording, refund, stock movement, and shift changes. Each operation has an idempotency key and returns the canonical saved result.
7. **Keep local-only mode available.** Until the new path is tested, keep existing local persistence. Cloud operations must be explicitly opt-in for test devices, with visible error handling and a recovery export.

## Important trade-off
A four-digit PIN can be guessed if an attacker can repeatedly reach the verification endpoint. Pairing, strict persistent throttling, short-lived sessions, audit logging, and secure credential storage reduce the risk, but the business should consider longer PINs or a second factor for administrators. Do not silently assume the current local PIN screen is secure server authentication.

## Decisions required before implementation
- Who can pair/unpair a register and reset staff PINs?
- Is the Admin role allowed to create/deactivate staff, and are there separate manager permissions?
- How many registers should be paired, and should an offline register be allowed to record a sale locally while disconnected?
- What session expiry/idle-lock policy is acceptable? Preserve the existing five-minute local auto-lock behavior unless explicitly redesigned.
- Are staff IDs/names/roles and current PINs merely demo data? Treat current embedded PINs as demo credentials and rotate them before any real deployment.
- What is the authorized receipt-number series and who can configure it? Receipt allocation must be server-side and constrained to the configured valid series.

## First implementation milestone
Implement and test only the isolated authentication slice first: pair a test register, verify a test staff PIN, reject an invalid PIN, enforce persistent rate limits across app reloads, revoke a session, and prove that a cashier session cannot invoke admin-only actions. Do not connect this milestone to production sales or modify the current POS runtime until reviewed.
