---
title: "Affiliate Commission & Payout System"
date: 2026-04-15
draft: false
tags:
  - "Systems Design"
  - "Financial Flows"
  - "Data Integrity"
  - "Concurrency"
  - "Laravel"
decription: ""
toc:

---

## 1. Problem Context

Affiliate systems are easy to describe at the UI level.

Partners get a referral code, see referred users, and eventually request a payout.

That is not the hard part.

The harder problem is **financial correctness**:

- When does a referral become a real earning?
- When does that earning become withdrawable?
- How do you prevent duplicate commissions under retries or repeated invoice events?
- How do you prevent duplicate or premature payouts?
- How do you move money without partial state updates?

The objective was not simply to add an affiliate dashboard.

It was to design a commission and payout system that **separates attribution, earning, approval, and disbursement** so the platform could support affiliates without weakening trust in the money flow.

---

## 2. Product Surface In Brief

The product surface had two sides:

### Affiliate Surface

- Referral code and onboarding flow
- Referred-user visibility
- Earnings summary
- Commission history
- Payout request flow
- Payout-method management

### Admin Surface

- Affiliate summary and referred-user visibility
- Commission review
- Payout approval and payment actions
- Commission-rate updates
- Affiliate activation / deactivation

This product layer mattered, but it was mostly a shell around the real problem:

**modeling the money lifecycle with clear states and safe transitions.**

---

## 3. Referral Attribution Model

The first decision was to treat attribution as a one-time identity event.

When a referred user signs up, the platform resolves the referrer and stores that relationship once. Commission logic reads from that persisted attribution later rather than trying to resolve referral identity during invoice processing.

That separation matters because it:

- Reduces attribution ambiguity
- Keeps earning logic independent from signup logic
- Prevents payout calculations from depending on fragile runtime referral lookups

Design principle:

> Resolve referral identity once. Compute money from stored attribution later.

This keeps the system simpler to reason about and makes downstream money events more deterministic.

---

## 4. Commission Creation Model

Commissions are created from paid invoice activity.

A key engineering decision was to defer commission creation until the surrounding invoice work commits successfully. In practice, that means commission side effects only happen after the invoice state is durable.

That protects the system from a common class of bugs:

- invoice state rolls back
- commission was already created
- earnings drift away from billing reality

Commission calculation itself was modeled explicitly:

- Money is represented in integer cents
- Rates are represented in basis points
- Eligible invoice items are filtered by commission basis

The system supports two earning models:

- `SUBSCRIPTION`
- `TRANSACTION_FEE`

That split is important because it makes revenue policy explicit in code. Instead of a vague “commission rate,” the system can decide which invoice components actually participate in earnings.

This gives the commission engine a clear contract:

- read billing events
- select eligible revenue
- compute commission deterministically
- persist one earning record for one invoice

---

## 5. Approval And Eligibility Layer

The next important decision was **not** to make all earnings immediately withdrawable.

The system introduces a separate approval layer between commission creation and payout availability.

Behaviorally, the model is:

- commissions can be created
- commissions can remain pending
- approved commissions become payout-eligible
- only approved value contributes to available payout balance

This matters because “earned” and “withdrawable” are not the same thing.

For subscription-style commissions, the system applies a waiting period before earnings become approved. A scheduled approval path promotes eligible pending commissions after the configured delay.

That creates a safer money model:

- raw earning events are recorded early
- approval is delayed intentionally
- payout availability is derived from approved earnings only

The result is a much cleaner settlement boundary.

Instead of a mutable “wallet” number drifting independently, the platform can explain available payout as:

> approved commissions minus historical paid-out amount

That is simpler, more auditable, and easier to defend.

---

## 6. Idempotency And Duplicate Prevention

Invoice-driven systems are naturally exposed to retries, repeated updates, and replayed events.

Because of that, duplicate prevention had to be part of the design rather than an afterthought.

The strongest safeguard is the commission idempotency boundary:

- the service checks whether an invoice already produced a commission
- the database also enforces uniqueness at the invoice level

That combination is important.

App-level checks are useful, but schema-level guarantees are what protect the system when concurrency or retry behavior gets messy.

This turns commission creation into a more reliable contract:

- one paid invoice
- one commission record
- no silent duplication under reprocessing

For a money-related feature, this is one of the highest-value engineering decisions in the whole system.

---

## 7. Payout Request Safety

Payout requests were designed as a transactional path, not as a simple insert.

When an affiliate requests a payout, the system:

- locks the affiliate row
- computes available balance from approved earnings
- checks the minimum payout threshold
- rejects requests above the available balance
- verifies there is no existing pending payout
- creates the payout request only if all checks pass

This matters because payout requests are exactly where concurrency bugs become expensive.

Without locking, two requests submitted close together can both see the same balance and create duplicate cashout requests.

The service-level protections are backed by a schema-level invariant:

- only one pending payout is allowed per affiliate

That is a strong design choice because it combines:

- business-rule enforcement in the service
- race protection in the database

The system does not merely hope duplicate payout requests are unlikely. It makes them structurally harder to create.

---

## 8. Payout Method And Data Handling

Payout-method handling is easy to under-design.

If payout details are treated like ordinary profile data, historical payout requests can become hard to trust, and sensitive financial information can leak too broadly.

The implementation avoided that by separating:

- current payout methods
- historical payout requests

Each payout request stores a **snapshot** of the selected payout method at request time.

That means later edits to payout details do not rewrite payout history.

This is a subtle but important decision.

It improves:

- historical accuracy
- operator confidence during payout review
- resilience against profile edits after a payout request is created

Sensitive payout details are also handled carefully:

- payout-method data is stored encrypted
- affiliate-facing views use masked banking details
- the system maintains exactly one default payout method per affiliate

Together, these decisions improve both privacy and operational clarity.

---

## 9. Final Disbursement Flow

The strongest technical safeguard in the system is the final pay step.

When an approved payout is disbursed, the platform performs the money-state update atomically:

- the payout request is locked
- the affiliate aggregate is locked
- the historical paid-out total is incremented
- the payout status is moved to `PAID`
- the state transition happens inside a single transaction

This is the core trust-preserving part of the design.

If the process updates only one side of that state, the system can drift into impossible conditions:

- payout marked paid but aggregate not updated
- aggregate updated but payout still looks unpaid
- concurrent pay attempts racing against each other

By treating payment as one atomic unit of work, the system reduces the chance of partial updates and duplicate disbursement.

That is the centerpiece engineering decision in the case study.

---

## 10. Privileged Control And Operational Flow

Money movement still requires an operational control surface.

The platform includes a privileged admin path for:

- reviewing affiliate payouts
- approving or rejecting payout requests
- paying approved payouts
- rejecting pending commissions
- updating commission rates

That layer is intentionally separated from the affiliate self-service surface.

Admin actions are protected through a signed control path, and payout requests generate operational notifications so the payout lifecycle is visible outside the affiliate dashboard itself.

This creates an important product characteristic:

- affiliates can request
- admins can review
- payment remains a deliberate privileged action

That separation is often the difference between a useful affiliate feature and a trustworthy payout system.

---

## 11. Engineering Decisions And Tradeoffs

Several design choices shaped the system:

**Why integer cents and basis points instead of floating-point money math?**  
To reduce precision errors and make financial calculations deterministic.

**Why create commissions from invoice events?**  
Because billing events are the most defensible trigger for earnings.

**Why separate earning, approval, and disbursement?**  
Because not every earning should become immediately withdrawable.

**Why use both service checks and schema constraints?**  
Because money systems need protection against both ordinary validation failures and concurrent race conditions.

**Why snapshot payout methods?**  
Because historical payout requests should not change when a user edits banking details later.

**Why keep payment as an explicit admin step?**  
Because disbursement is the highest-risk state transition in the flow.

These choices add some complexity, but they produce a far more trustworthy system than a flat “commission balance” implementation.

---

## 12. Reliability And Future Hardening

The system already has several reliability-oriented properties:

- state separation between earnings and payouts
- transactional payout request creation
- schema-backed duplicate prevention
- delayed approval before withdrawal
- encrypted handling of payout details
- atomic final disbursement updates

Future hardening would focus on:

- deeper auditability for privileged money actions
- even stricter state-transition rules around payout review
- richer operational visibility around payout notifications and reconciliation workflows

Those are worthwhile extensions, but the core system already demonstrates the main design goal:

> money should move through explicit states, with deliberate approval boundaries and strong duplicate-prevention safeguards.

---

## 13. Outcome

The result was not just an affiliate feature.

It was a structured commission and payout system with:

- durable referral attribution
- deterministic commission creation
- explicit approval timing
- safe payout eligibility rules
- concurrency-aware payout requests
- atomic disbursement updates

That combination made the feature more than a dashboard.

It turned it into a platform capability that could support affiliates while preserving correctness in the money flow.
