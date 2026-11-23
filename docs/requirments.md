# PayLink Requirements Pack

**Members & roles:**
  Requirements & Product Lead: `Mohamed Khaled`
  Quality & DevOps Lead: `Hossam Abdul Menem`
  Architecture Lead: `Amira Fawaz & Yussef Mahmoud`
 **Repo:** `https://github.com/hossamkhero/PayLink`
 **Tech stack:** ` Python 3 + FastAPI + PostgreSQL 16`
 **Date:** `[2026-11-23]`

---

## 1. Problem Summary

PayLink is an Egyptian bill-payment and wallet company. Customers pay utility bills from a digital wallet. PayLink earns a service fee per payment and pays commission to agents.

Because PayLink holds customers' money, correctness is a regulatory obligation, not a quality goal. A double charge, a charge for a payment the biller never received, or a negative balance is a failure of trust.

The new backend must therefore be correct with money. Every movement is a recorded transaction, and balances are derived, never overwritten. It must be extensible, so that new billers are added without touching existing modules. It must stay reliable when things go wrong. Timeouts, refusals, duplicate callbacks and provider outages must never lose or invent money. And it must handle month-end load, the busiest period, without breaking.

---

## 2. Stakeholders

| Stakeholder | Interest | Influence |
|---|---|---|
| **Customer** | Pay bills fast, see accurate balance, get notified, get refunds on failures | High |
| **Agent (shop owner)** | Earn commission, see their wallet grow, terminal must not block | High |
| **Biller (ElectroCo, AquaCo, TopUpNet, NetCo)** | Receive correct payments, get settlement-matched records | High |
| **PayLink Support** | Find any payment by reference, see full history, block compromised accounts | Medium |
| **PayLink Finance / Reconciliation** | Daily totals, zero unresolved discrepancies after settlement | High |
| **PayLink Admin** | Configure billers, fees, limits without redeploy | Medium |
| **Regulator / Auditor** | Every money movement is auditable (who, when, why) | High |
| **Engineering team** | Extend with new billers safely, test without a real biller | High |

---

## 3. User Stories

### 3.0 Story Index

| ID | Title | Role | Priority | FR | HC |
|---|---|---|---|---|---|
| US-01 | Customer registration | Customer | Must | FR-1 | — |
| US-02 | Upgrade to verified | Customer | Should | FR-2 | — |
| US-03 | View balance & history | Customer | Must | FR-3 | — |
| US-04 | Top up wallet | Customer | Must | FR-4 | — |
| US-05 | Balances from ledger | Auditor | Must | FR-5 | — |
| US-06 | Browse billers | Customer | Must | FR-7 | — |
| US-07 | Admin manages billers | Admin | Must | FR-6 | — |
| US-08 | New biller = adapter only | Developer | Must | §3.3 | HC8 |
| US-09 | Bill inquiry preview | Customer | Must | FR-8 | — |
| US-10 | Pay a bill | Customer | Must | FR-9 | — |
| US-11 | Idempotent submission | Customer | Must | FR-10 | HC1 |
| US-12 | State machine enforced | System | Must | FR-11 | — |
| US-13 | Hold & release | Customer | Must | FR-12 | HC3 |
| US-14 | Timeout → Unknown | Customer | Must | FR-13 | HC2 |
| US-15 | Async callbacks | System | Must | FR-14 | HC6 |
| US-16 | 3 billers, 3 protocols | System | Must | FR-15 | — |
| US-17 | Fee calculation | System | Must | FR-16 | — |
| US-18 | Agent pays + commission | Agent | Should | FR-17 | — |
| US-19 | Over-limit rejection | Customer | Must | FR-18 | HC4 |
| US-20 | Velocity block | Risk officer | Should | FR-19 | — |
| US-21 | Support search by ref | Support | Must | FR-20 | — |
| US-22 | SMS + push | Customer | Must | FR-22 | HC5 |
| US-23 | Notification retry | System | Should | FR-23 | HC5 |
| US-24 | Audit log | Auditor | Must | FR-25 | — |
| US-25 | Nightly reconciliation | Finance | Should | FR-26 | HC7 |

---

### 3.1 Accounts & Wallet

#### US-01 — Customer registration
**As a** new visitor, **I want** to register with PayLink, **so that** I can hold a wallet and pay bills.
**Priority:** Must · **Source:** FR-1

**Acceptance criteria**
- **Given** a new visitor, **When** they submit a valid name, phone and email, **Then** a customer is created with `verificationLevel = basic` and a wallet with `available = 0`, `held = 0`, and a `LEDGER_OPEN` entry exists.
- **Given** an existing phone number, **When** registration is attempted, **Then** the request fails with `409 PHONE_ALREADY_REGISTERED`.

#### US-02 — Upgrade to verified
**As a** customer, **I want** to upgrade my verification level, **so that** my daily payment limit rises from 5,000 to 30,000 EGP.
**Priority:** Should · **Source:** FR-2

**Acceptance criteria**
- **Given** a basic customer, **When** an admin approves their submitted verification documents, **Then** `verificationLevel = verified` and the daily limit is 30,000 EGP effective immediately.
- **Given** a verified customer, **When** a 20,000 EGP payment is submitted within a day whose prior total is 0, **Then** the payment is not rejected by the daily-limit rule.

#### US-03 — View wallet balance and history
**As a** customer, **I want** to see my available and held balances plus my transaction history, **so that** I trust what the app tells me.
**Priority:** Must · **Source:** FR-3

**Acceptance criteria**
- **Given** a wallet with 3 completed payments and 1 held payment, **When** I open the wallet page, **Then** I see `available`, `held`, and a paginated list of ledger entries newest-first.
- **Given** the wallet is queried, **When** the response is generated, **Then** `available` and `held` are **computed from the ledger**, not read from a mutable balance column.

#### US-04 — Top up the wallet
**As a** customer, **I want** to top up my wallet by simulated card or via an agent in cash, **so that** I have funds to pay bills.
**Priority:** Must · **Source:** FR-4

**Acceptance criteria**
- **Given** a customer, **When** they top up 500 EGP by card, **Then** a `TOPUP` ledger entry is posted and `available` increases by exactly 500.00.
- **Given** an agent, **When** they top up a customer's wallet in cash, **Then** the agent's own cash-in ledger entry is recorded and the customer's wallet is credited.

#### US-05 — Balances are derived from the ledger
**As a** PayLink auditor, **I want** every balance change to be a ledger entry, **so that** the balance is always reconstructible and never drifts.
**Priority:** Must · **Source:** FR-5

**Acceptance criteria**
- **Given** any wallet, **When** its `available + held` is compared to the sum of its ledger entries, **Then** the two are equal at all times.
- **Given** an admin attempts a direct balance update, **When** the operation is submitted, **Then** it is rejected; only ledger-append operations are exposed.

---

### 3.2 Biller Catalog

#### US-06 — Browse billers and required inputs
**As a** customer, **I want** to see which billers are available and which input each requires (meter no., phone no., etc.), **so that** I know what to enter before paying.
**Priority:** Must · **Source:** FR-7

**Acceptance criteria**
- **Given** three active billers, **When** the customer opens the billers page, **Then** each biller shows its `requiredFields` array (e.g. `["meterNumber"]`).
- **Given** a disabled biller, **When** the list is fetched, **Then** it does not appear in the customer view.

#### US-07 — Admin manages billers and services
**As a** PayLink admin, **I want** to add, edit and disable billers and their services, **so that** the platform can be extended without a redeploy.
**Priority:** Must · **Source:** FR-6

**Acceptance criteria**
- **Given** a new biller payload, **When** the admin submits it, **Then** the biller is persisted, and if active it appears in the customer list within the next read.
- **Given** an active biller with in-flight payments, **When** it is disabled, **Then** new payments to that biller are rejected but in-flight payments continue to their terminal state.

#### US-08 — Add a new biller without touching existing modules
**As a** PayLink developer, **I want** to add a new biller by adding an adapter and configuration only, **so that** existing modules stay untouched and regressions are impossible.
**Priority:** Must · **Source:** NFR — Modifiability (NFR-4)

**Acceptance criteria**
- **Given** the existing codebase, **When** a new biller is added, **Then** the diff touches only `integrations/<newbiller>/` and a configuration file, with no changes to `payments/`, `wallet/`, `fees/`, or `notifications/`.
- **Given** the new biller's adapter implements the common interface, **When** a payment is submitted, **Then** the payment pipeline routes to it without conditional branches in the payment service.

---

### 3.3 Payment Processing

#### US-09 — Bill inquiry (amount, fee, total preview)
**As a** customer, **I want** to run a bill inquiry that shows the amount due, PayLink's fee, and the total, **so that** I know exactly what I'll pay before confirming.
**Priority:** Must · **Source:** FR-8

**Acceptance criteria**
- **Given** a valid meter number at ElectroCo, **When** the customer runs inquiry, **Then** the response contains `amountDue`, `serviceFee`, `total`, and a short-lived `inquiryId`.
- **Given** a bill of 350.00 EGP for electricity during no promotion, **When** inquiry runs, **Then** `serviceFee = 2.50` and `total = 352.50`.

#### US-10 — Pay a bill from the wallet
**As a** customer, **I want** to pay a bill from my wallet, **so that** I don't queue at the biller's office.
**Priority:** Must · **Source:** FR-9

**Acceptance criteria**
- **Given** an inquiry with total 352.50 and available balance 500.00, **When** I confirm the payment, **Then** 352.50 moves to `held`, the biller is called, and on confirmation the hold is converted to a final charge and the fee is credited to fee income.
- **Given** confirmation, **When** the payment completes, **Then** an SMS and a push notification are queued (see US-22).

#### US-11 — Idempotent payment submission
**As a** customer, **I want** retried submissions of the same payment to charge me only once, **so that** a slow network does not double-charge me.
**Priority:** Must · **Source:** FR-10

**Acceptance criteria**
- **Given** a payment request with idempotency key `K`, **When** an identical request with the same `K` arrives, **Then** exactly one payment exists, one charge is posted, and both responses return the same `paymentId` and status.
- **Given** a request with a **different** `K` but otherwise identical fields, **When** it is submitted, **Then** a second payment is created (deduplication is by key, not by payload).
- **Given** the first request is still in flight, **When** the duplicate arrives, **Then** the duplicate returns the same in-flight `paymentId` (not a new payment).

#### US-12 — Payment life cycle is enforced by a state machine
**As a** system, **I want** payments to move only through the transitions defined in §2.5, **so that** no illegal state is possible.
**Priority:** Must · **Source:** FR-11

**Acceptance criteria**
- **Given** a payment in `Completed`, **When** any transition is attempted, **Then** it is rejected (final state).
- **Given** a payment in `Unknown`, **When** a "refund" is attempted directly, **Then** it is rejected; `Unknown → Refunded` is illegal, only `Failed → Refunded` is allowed.
- **Given** any transition, **When** it is applied, **Then** a `payment_state_history` row is written with `from`, `to`, `event`, `timestamp`.

#### US-13 — Hold before biller call; release on failure
**As a** customer, **I want** my money held (not charged) until the biller confirms, and released if the biller refuses, **so that** a biller outage does not cost me money.
**Priority:** Must · **Source:** FR-12

**Acceptance criteria**
- **Given** a payment that reaches `SentToBiller`, **When** the state is inspected, **Then** the amount is in `held`, not in `available`.
- **Given** AquaCo returns an error code, **When** the adapter maps it to a refusal, **Then** the payment transitions `SentToBiller → Failed → Refunded`, the hold is released back to `available`, and an SMS with a human-readable reason is queued.

#### US-14 — Biller timeout becomes Unknown and is retried
**As a** customer, **I want** a slow biller to leave my payment in a "processing" state (not failed, not refunded), **so that** I don't lose money if the biller actually received it.
**Priority:** Must · **Source:** FR-13

**Acceptance criteria**
- **Given** ElectroCo does not answer within 10 seconds, **When** the timeout fires, **Then** the payment transitions `SentToBiller → Unknown`, the hold stays, and the customer is told the payment is being processed.
- **Given** a payment in `Unknown`, **When** the retry schedule runs, **Then** it is retried at +1, +5 and +15 minutes (3 attempts), and after that it waits for reconciliation.
- **Given** a payment in `Unknown`, **When** no automatic resolution occurs, **Then** it is **never** auto-refunded.

---

### 3.4 Biller Integration

#### US-15 — Asynchronous callbacks from TopUpNet
**As a** system, **I want** to accept TopUpNet's asynchronous callbacks and ignore duplicates, **so that** a double callback cannot complete a payment twice.
**Priority:** Must · **Source:** FR-14

**Acceptance criteria**
- **Given** a payment in `SentToBiller` with TopUpNet, **When** a `success` callback arrives, **Then** the payment → `Completed` and a callback log row is written.
- **Given** the same callback arrives a second time, **When** it is processed, **Then** the payment stays `Completed` (no second charge, no second commission) and the duplicate is logged with `ignored_duplicate`.
- **Given** a callback for an unknown reference, **When** it arrives, **Then** it is logged and returns `200` (to prevent provider retry storms) but is not applied.

#### US-16 — Support 3 billers with distinct protocols
**As a** PayLink system, **I want** to integrate ElectroCo (REST/JSON), AquaCo (SOAP/XML) and TopUpNet (REST/async callback), **so that** the customer sees a uniform "pay a bill" flow regardless of protocol.
**Priority:** Must · **Source:** FR-15

**Acceptance criteria**
- **Given** a payment to AquaCo above the bill amount, **When** the adapter receives the biller's rejection, **Then** the pipeline receives a typed `BillerRefused` result, not a raw HTTP/XML error.
- **Given** a payment to any of the three billers, **When** the payment service handles it, **Then** it sees only the common `BillerAdapter` interface with no `if (biller === 'aquaco')` branches.

---

### 3.5 Fees & Commissions

#### US-17 — Fee calculation (fixed, percentage, tiered, promotion)
**As a** PayLink system, **I want** to compute fees per the rules in §2.6, **so that** customers are charged correctly across all services and promotion periods.
**Priority:** Must · **Source:** FR-16

**Acceptance criteria**
- **Given** an electricity payment of 350.00, **When** the fee is computed, **Then** the result is 2.50.
- **Given** a mobile top-up of 20.00, **When** the fee is computed, **Then** the result is `max(1.00, 0.015 × 20) = 1.00`.
- **Given** an internet bill of 150.00, **When** the fee is computed, **Then** the result is 2.00; of 250.00, the result is 4.00.
- **Given** an active promotion for AquaCo (start ≤ now ≤ end), **When** a water payment is made, **Then** the fee is 0.00.

#### US-18 — Agent pays on behalf and earns commission
**As an** agent, **I want** to pay a bill on behalf of a customer and earn commission, **so that** I am compensated for using my terminal.
**Priority:** Should · **Source:** FR-17

**Acceptance criteria**
- **Given** an agent initiates a payment for a customer, **When** the payment completes, **Then** the agent's wallet is credited 40% of the service fee as a `COMMISSION` ledger entry.
- **Given** the payment does not reach `Completed`, **When** the payment ends in `Failed` or `Rejected`, **Then** no commission is credited.

---

### 3.6 Fraud Checks & Limits

#### US-19 — Reject over-limit payments
**As a** customer, **I want** payments above my single-payment limit or my daily limit to be rejected with a clear reason, **so that** I understand why and can act.
**Priority:** Must · **Source:** FR-18

**Acceptance criteria**
- **Given** a verified customer, **When** they submit a 10,001 EGP single payment, **Then** the payment → `Rejected` with reason `EXCEEDS_SINGLE_LIMIT` and no ledger entry is posted.
- **Given** a basic customer with 4,900 EGP already paid today, **When** they submit a 200 EGP payment, **Then** it is `Rejected` with reason `EXCEEDS_DAILY_LIMIT`.
- **Given** a wallet with 100 EGP, **When** two 80 EGP payments arrive concurrently, **Then** exactly one succeeds, the other is `Rejected` for insufficient funds, and `available` never goes negative.

#### US-20 — Velocity rule blocks the account
**As a** PayLink risk officer, **I want** more than 5 payments in 60 seconds from one account to block that account for 10 minutes, **so that** scripted abuse is stopped.
**Priority:** Should · **Source:** FR-19

**Acceptance criteria**
- **Given** 6 payments from one account within 60 seconds, **When** the 6th is evaluated, **Then** the account is blocked for 10 minutes and the 6th is `Rejected` with reason `VELOCITY_BLOCKED`.
- **Given** a blocked account, **When** a payment is submitted inside the block window, **Then** it is `Rejected` with reason `ACCOUNT_BLOCKED`.

---

### 3.7 Support

#### US-21 — Support searches a payment by reference and sees full history
**As a** support employee, **I want** to search a payment by its reference and see every state transition plus biller messages, **so that** I can answer customer tickets without escalating.
**Priority:** Must · **Source:** FR-20

**Acceptance criteria**
- **Given** a payment reference, **When** support searches it, **Then** the response contains the payment, its full `state_history`, the biller request/response logs, and the notification log.
- **Given** a reference that doesn't exist, **When** support searches, **Then** the response is `404` with a clear message.

---

### 3.8 Notifications

#### US-22 — SMS and push on completion and failure
**As a** customer, **I want** an SMS and a push notification when a payment completes or fails, **so that** I don't have to refresh the app to know what happened.
**Priority:** Must · **Source:** FR-22

**Acceptance criteria**
- **Given** a payment reaching `Completed`, **When** the notification step runs, **Then** one SMS and one push are queued, and a `notification_sent` row is logged for each.
- **Given** a payment reaching `Failed` via AquaCo refusal, **When** notifications are queued, **Then** the SMS includes a mapped human-readable reason (not the raw biller code).
- **Given** the SMS provider is down when the payment completes, **When** the send fails, **Then** the payment still reaches `Completed` and the SMS is retried independently (see US-23).

#### US-23 — Notification retry, then undelivered
**As a** PayLink system, **I want** failed notifications retried 3 times and then marked undelivered, **so that** a flaky provider is contained and observable.
**Priority:** Should · **Source:** FR-23

**Acceptance criteria**
- **Given** a failed SMS, **When** retries are exhausted (3 attempts), **Then** the row is marked `undelivered` and an alert metric is incremented.
- **Given** a failed push notification, **When** the SMS eventually succeeds, **Then** the push is still retried independently (no cross-channel blocking).

---

### 3.9 Audit & Reconciliation

#### US-24 — Audit log for money movements and admin actions
**As a** PayLink auditor, **I want** every money movement and every admin action recorded with who, when and why, **so that** the regulator can be satisfied and disputes are resolvable.
**Priority:** Must · **Source:** FR-25

**Acceptance criteria**
- **Given** any ledger entry, **When** it is queried, **Then** an associated audit row exists with `actorId`, `timestamp`, `reasonCode` and `correlationId`.
- **Given** any admin action (biller change, verification approval), **When** it occurs, **Then** an audit row is written before the change is acknowledged.

#### US-25 — Nightly reconciliation report
**As a** Finance team member, **I want** a nightly reconciliation report from each biller's settlement file, **so that** discrepancies are surfaced before they become customer complaints.
**Priority:** Should · **Source:** FR-26

**Acceptance criteria**
- **Given** a settlement file for ElectroCo, **When** reconciliation runs, **Then** every settlement row is matched to a PayLink payment by `billerReference` and `amount`, and mismatches are written to the discrepancy report.
- **Given** a payment PayLink has as `Failed` but the settlement file lists as received, **When** reconciliation runs, **Then** a discrepancy row is written with `expected=Failed, actual=Received` and status `open` for support review.
- **Given** a payment in the settlement file with no PayLink record, **When** reconciliation runs, **Then** an `orphan_settlement` row is written.

---
## 4. Non-Functional Requirements

| # | Quality | Metric / target | How measured |
|---|---|---|---|
| NFR-1 | Correctness | 0 double charges and 0 negative balances in the concurrency + duplicate test suite | Automated tests in CI; results in pipeline artefact |
| NFR-2 | Performance | 100 payments/sec sustained for 60 s against mock billers (p95 < 500 ms) | `k6` load-test job against `docker compose` stack |
| NFR-3 | Resilience | A biller outage does not reduce success rate for other billers by more than 1% | Chaos test: mock biller returns 500 for 60 s; measure other-biller success |
| NFR-4 | Modifiability | Adding a new biller touches only `integrations/<new>/` and config | CI check: `git diff --stat` on a sample "add biller" branch, assert no changes outside allowed paths |
| NFR-5 | Auditability | 100% of money movements and admin actions have an audit record with who / when / why | SQL assertion in CI: `count(ledger_entries) == count(audit where type='money')` |
| NFR-6 | Testability | ≥ 70% line coverage on domain packages (`wallet/`, `payments/`, `fees/`) | Coverage report published by pipeline; CI gate fails under 70% |


## 5. Ambiguities Found & Decisions Made

| # | Ambiguity | Decision | Rationale |
|---|---|---|---|
| A-1 | The brief says the fee is charged "per payment" but does not say when the fee is recognised if the payment fails | Fee is **only** posted on `Completed`; Failed / Rejected / Refunded payments produce no fee | Charging for a service not received is a regulatory risk in Egypt |
| A-2 | Timeout rule says "10 seconds for live billers" but does not say from when the clock starts | 10 s wall-clock from the moment the HTTP request is fully written to the socket | Matches what a biller's own SLA dashboard would measure; makes HC2 test deterministic |
| A-3 | Daily limit — measured on **initiated** or **completed** payments | Measured on all payments whose state is **not** `Rejected` | Prevents an attacker from multiplying limits by triggering refusals |
| A-4 | Timezone for the daily reset | `Africa/Cairo` | Business is Egyptian |
| A-5 | Velocity rule ("5 payments in 60 s → block 10 min") — do **rejected** payments count | **Yes** | Otherwise an attacker could probe with 1000 rejected attempts |
| A-6 | ElectroCo v2 moves amounts from pounds to piasters. Do we store both v1 and v2 | Store canonical amount as `DECIMAL(12,2)` EGP; the adapter converts to/from piasters at the boundary | Keeps the domain in one unit; v2 change touches only the adapter (§2.4 grading criterion) |
| A-7 | When a payment is in `Unknown` after all retries exhaust, what does the customer see, and is it ever auto-refunded | Shown as *"Processing — awaiting biller confirmation"*. Money stays in `held`, **never** auto-refunded | §2.5 is explicit: `Unknown` is resolved only by reconciliation, not by auto-refund |

---

## 6. Hard Case Acceptance Criterion Map

| Hard case | Where it lives | Will be tested by |
|---|---|---|
| HC1 Duplicate submission | US-11 AC1–AC3 | `tests/hard/hc1_idempotency.py` |
| HC2 ElectroCo 12 s timeout | US-14 AC1 | `tests/hard/hc2_timeout_unknown.py` |
| HC3 AquaCo refusal | US-13 AC2 | `tests/hard/hc3_aquaco_refusal.py` |
| HC4 Concurrent balance | US-19 AC3 | `tests/hard/hc4_concurrency.py` |
| HC5 SMS provider down | US-22 AC3 + US-23 AC1 | `tests/hard/hc5_sms_outage.py` |
| HC6 Duplicate TopUpNet callback | US-15 AC2 | `tests/hard/hc6_dup_callback.py` |
| HC7 Settlement vs Failed | US-25 AC2 | `tests/hard/hc7_reconciliation.py` |
| HC8 New biller add | US-08 AC1 | `tests/hard/hc8_add_biller.py` |
---

## 7. Out of Scope

- Real KYC / identity verification provider integration (manual admin approval for now).
- Real card acquirer integration (top-up is simulated).
- Multi-currency (EGP only, per brief).
- Mobile/UI polish — the brief states marks are for the backend.
