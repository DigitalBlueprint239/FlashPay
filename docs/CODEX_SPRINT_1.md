# FlashPay Sprint 1 — Codex Build Authority

## Mission
Build the first working FlashPay product shell for the U.S. → Mexico corridor.

FlashPay should feel like Cash App to the customer, but this sprint uses ONLY mocked/sandbox financial providers. No production money movement is allowed.

## Product definition
A bilingual mobile-first remittance application where a U.S. sender can:
1. Create an account.
2. Complete mock KYC.
3. Add a Mexican recipient.
4. Enter USD amount.
5. Receive a USD→MXN quote.
6. Review fee, exchange rate, recipient amount, and quote expiry.
7. Confirm a simulated transfer.
8. Track that transfer through a controlled state machine.
9. See transaction history.

The recipient does not need a FlashPay account in Sprint 1.

## Architecture
- Next.js latest stable App Router
- TypeScript strict mode
- Supabase PostgreSQL + Supabase Auth
- Tailwind CSS
- Zod validation
- Server-side domain/service/repository boundaries
- Vitest or Jest for unit/integration tests
- Playwright for E2E
- GitHub Actions CI
- Vercel-ready
- English + Spanish i18n from the beginning

Suggested structure:
- app/
- components/
- domain/
- services/
- providers/
- repositories/
- ledger/
- compliance/
- lib/
- tests/
- supabase/migrations/

Do not put financial business logic directly inside React components or route handlers.

## Sprint 1 scope

### FP-101 Authentication
Implement:
- sign up
- sign in
- sign out
- password reset
- email verification state
- protected app routes

Use Supabase Auth. Do not store passwords in application tables.

### FP-102 Sender Profile
Fields:
- full legal name
- date of birth
- U.S. address
- phone
- preferred language

Do not store full SSNs, raw ID documents, or biometric images.

### FP-103 KYC Provider Abstraction
Create an interface such as:
- startVerification()
- getVerificationStatus()
- handleWebhook()

Implement MockKycProvider only.

Supported internal states:
- not_started
- pending
- needs_information
- under_review
- verified
- rejected
- restricted

No production KYC provider is integrated in this sprint.

### FP-104 Recipient Management
Recipient:
- legal name
- nickname optional
- CLABE
- bank name/institution optional
- active/archive state

Validate CLABE format and checksum if practical.
Mask CLABE in UI after entry.
A valid CLABE format does NOT imply account ownership.

### FP-105 Quote Engine
Provider-independent quote interface.

Input:
- source currency USD
- destination currency MXN
- send amount in USD minor units
- recipient
- funding method placeholder

Output:
- quote_id
- principal_usd_minor
- fee_usd_minor
- fx_rate as decimal-safe representation
- amount_mxn_minor
- expires_at
- provider_quote_reference
- disclosure_version

Use a MockFxProvider.

The quote must expire and an expired quote cannot create a transfer.

### FP-106 Send Flow UI
Screens:
Recipient → Amount → Quote → Review → Confirm

Display:
- You send
- FlashPay fee
- exchange rate
- recipient gets
- estimated delivery placeholder
- quote expiration

Both English and Spanish.

### FP-107 Transfer Creation
POST-style server operation must:
- require authenticated user
- require valid non-expired quote
- snapshot recipient details used for the transfer
- accept an idempotency key
- create only one transfer for repeated requests with same key
- store monetary values in minor units
- never use JS floating point for financial calculations

### FP-108 Transaction State Machine
Implement explicit server-owned transitions.

Primary path:
created
→ quote_confirmed
→ compliance_pending
→ funding_pending
→ funded
→ payout_queued
→ payout_processing
→ delivered

Exception/terminal states:
requires_review
funding_failed
payout_failed
cancel_requested
cancelled
refund_pending
refunded
rejected

Illegal transitions must fail.
The browser may never directly mutate transfer status.

### FP-109 Ledger Foundation
Do NOT use users.balance.

Create append-only:
- ledger_accounts
- ledger_entries

Use integer minor units.

At minimum support simulated entries for:
- funding pending
- settlement receivable
- partner payable
- fees revenue
- refunds payable

Every simulated transfer should produce deterministic balanced ledger entries.

### FP-110 Event/Audit Model
Create:
- transfer_events
- audit_events
- webhook_events

Transfer state changes append an event.

Financial/admin mutations append an audit event.

### FP-111 Transaction History
User sees:
- recipient
- USD sent
- MXN recipient amount
- fee
- status
- created time
- reference

Transaction detail shows a timeline.

### FP-112 Bilingual UX
All customer-facing copy must use translation keys.

Initial locales:
- en-US
- es-MX

No hardcoded production UI strings in core customer screens.

### FP-113 Provider Abstractions
Create interfaces for:
- IdentityProvider
- FxQuoteProvider
- FundingProvider
- PayoutProvider
- NotificationProvider

This sprint implements mocks only.

### FP-114 Mock End-to-End Orchestrator
A simulated transfer should advance through the complete happy path without real banking:
KYC verified
→ quote
→ confirm
→ mock funding
→ mock payout
→ delivered

Support deterministic failure simulation for:
- funding failure
- payout failure
- KYC review
- expired quote

### FP-115 RLS and Authorization
Enable RLS.

Tests must prove:
- User A cannot read User B profile.
- User A cannot read User B recipients.
- User A cannot read User B transfers.
- User cannot create ledger entries directly.
- User cannot mutate KYC result directly.
- User cannot mutate transfer status directly.

### FP-116 XRPL Sandbox Foundation
Include XRPL as an isolated experimental provider, NOT as the remittance rail.

Requirements:
- XRPL Testnet only.
- Build server-only XRPL adapter.
- Never return or log a seed/private key.
- Do not store raw seed in browser/localStorage/database.
- Key-management interface must be abstracted so production can later use KMS/HSM/custody.
- Ability to create/fund a testnet wallet only through safe server-side test utilities.
- Record public XRPL address and testnet transaction hashes when used.

This is architecture groundwork, not customer custody.

### FP-117 FPC Test Token Sandbox
FPC is rewards/utility sandbox only.

Requirements:
- XRPL Testnet only.
- Do not represent FPC as USD or MXN.
- Do not use FPC to satisfy a remittance obligation.
- Implement a mock/test rewards service capable of crediting FPC for a simulated event.
- Keep FPC rewards separate from the fiat ledger.
- Document issuer/trustline assumptions.
- No production issuance.

### FP-118 CI
On every PR run:
- install
- lint
- TypeScript check
- unit tests
- integration tests
- Playwright E2E
- production build

Fail the PR if any required check fails.

## Required database tables
At minimum:
- profiles
- identity_verifications
- recipients
- quotes
- transfers
- transfer_events
- ledger_accounts
- ledger_entries
- provider_transactions
- webhook_events
- audit_events
- fpc_reward_events
- xrpl_accounts (public metadata only; no secret material)

Add timestamps, foreign keys, constraints, indexes and RLS policies.

## Financial invariants
1. Money amounts are stored as integer minor units.
2. Fiat ledger entries must balance.
3. Transfer status is server-controlled.
4. Duplicate client retries do not create duplicate transfers.
5. Duplicate webhook/event processing is harmless.
6. Quote values cannot change after transfer confirmation.
7. Recipient data used for a transfer is snapshotted.
8. FPC accounting is separate from fiat accounting.
9. No raw private keys, bank credentials, card data, IDs or SSNs in logs.

## UX direction
Simple, trustworthy, mobile-first.
The app should feel closer to Cash App than to a crypto exchange.

Avoid:
- crypto jargon on the primary send flow
- trading screens
- token prices
- DeFi
- staking
- charts
- complicated dashboards

Primary home actions:
- Send money
- Recipients
- Activity

## Explicitly out of scope
Do not implement:
- production ACH
- production debit cards
- production SPEI
- real-money custody
- production KYC
- real FPC issuance
- XRP remittance payments
- stablecoin settlement
- cash pickup
- OXXO
- merchant acquiring
- staking
- DeFi
- trading
- cards
- Jamaica/Haiti/DR corridors
- referrals with monetary value
- production admin refunds

## Security
- Secrets only server-side.
- Supabase service role never exposed to browser.
- Secure env validation.
- Rate-limit sensitive endpoints.
- Validate all inputs.
- Sanitize/redact logs.
- No private keys in source control.
- Add .env* to .gitignore.
- Use secure headers.
- Document threat assumptions.

## Tests required before Sprint 1 is accepted
1. New user signup/login.
2. Mock KYC verification.
3. Add valid recipient.
4. Reject invalid CLABE.
5. Generate USD→MXN quote.
6. Expired quote rejected.
7. Confirm transfer.
8. Same idempotency key returns same transfer.
9. Happy-path mock transfer reaches delivered.
10. Funding failure path.
11. Payout failure path.
12. Unauthorized cross-user reads blocked.
13. Direct transfer-status mutation blocked.
14. Fiat ledger balances.
15. FPC ledger remains separate.
16. XRPL test utility never exposes secret in API response/log.
17. English/Spanish core send flow.
18. Production build succeeds.

## Definition of Done
Sprint 1 is complete when a fresh test user can:
sign up
→ complete mock KYC
→ add a Mexican recipient
→ receive a mock USD/MXN quote
→ review exact fee/rate/recipient amount
→ confirm a simulated transfer
→ watch it advance to delivered
→ view it in activity/history

and CI + security/RLS/financial invariant tests all pass.

## Codex execution rules
- Inspect the repository before modifying it.
- Preserve LICENSE.
- Treat the existing ZIP as legacy reference only; do not build on it unless its contents are intentionally reviewed.
- Do not invent provider credentials.
- Do not create production payment integrations.
- Do not expose secrets.
- Do not merge directly to main.
- Work only on codex/sprint-1-foundation or child branches.
- Commit in logical increments.
- Keep README updated with local setup and architecture.
- Add DECISIONS.md for architectural decisions/assumptions.
- Add .env.example with placeholders only.
- If a requirement is ambiguous, prefer the safest mocked implementation and document the assumption instead of expanding scope.

## Final Codex report
When complete, report:
- files added/changed
- architecture implemented
- migrations created
- tests and exact results
- known gaps
- security findings
- money-flow assumptions
- XRPL/FPC sandbox status
- preview/deployment status
- recommended Sprint 2 work
