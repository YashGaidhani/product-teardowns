# Cross-Border Payments: A Product Manager's Teardown
## Flow Architecture, API Design Gaps, and the $150B Friction Problem

**Author:** Yash Gaidhani | PGDM General Management, XLRI Jamshedpur 2026  
**Published:** August 2026  
**Context:** Independent product analysis. No proprietary data used.  
**Audience:** Product managers, fintech strategists, and payment architects.

---

## Why This Matters

Global cross-border payment flows exceeded $150 trillion in 2023.
The cost of moving that money: $120B+ in transaction fees annually.
The average transaction failure rate on international corridors: 3-5%.
The average settlement time for a B2B international wire: 2-5 business days.

For context: a Bangalore engineer transferring $5,000 to a US vendor
waits longer, pays more, and has less visibility into that transaction
than they would ordering a Swiggy meal.

This is not a technology problem. The pipes exist.
SWIFT has been operational since 1973.
This is a **product problem** — fragmented rails, opaque APIs,
and payment initiation portals designed for compliance officers,
not users.

This teardown covers three things:
1. How cross-border payment flows actually work (the full stack)
2. Where the APIs break and why
3. What a better payment initiation portal looks like

---

## PART 1: THE CROSS-BORDER PAYMENT FLOW — WHAT ACTUALLY HAPPENS

Most diagrams show: Sender → Bank → Receiver.
That is missing 11 steps.

Here is what actually happens when a company in Mumbai
sends $50,000 to a supplier in Frankfurt:

### Step 1: Payment Initiation
The treasurer logs into their payment initiation portal
(a corporate banking portal — HSBC HSBCnet, Citi CitiDirect,
or an ERP-integrated gateway like SAP Multi-Bank Connectivity).

They input:
- Beneficiary IBAN (International Bank Account Number)
- BIC/SWIFT code of the recipient bank
- Payment amount and currency (USD, EUR, or source INR)
- Payment reference and purpose code (mandatory for RBI compliance in India)
- Value date (when funds should arrive, not when initiated)

**Product gap #1 here:** Most portals validate IBAN format only.
They do not validate that the IBAN exists, that the BIC matches
the IBAN country, or that the beneficiary name matches
the account. Errors surface 24-48 hours later as
failed transactions with opaque error codes.

---

### Step 2: FX Conversion and Rate Lock

The initiating bank (say HDFC India) checks whether the
payment is in INR or foreign currency.

If INR → USD: the bank executes an FX conversion.
The rate offered is the bank's internal treasury rate —
typically 0.5-2% above the mid-market rate (interbank rate).

The customer either:
a) Accepts the spot rate at time of initiation
b) Uses a pre-booked forward contract (locked rate)
c) Uses an FX sweep if they hold a foreign currency account

**Product gap #2:** FX transparency is near-zero on most portals.
The customer sees "₹41,23,500 debited" but cannot see
what exchange rate was applied, what the mid-market rate was,
or what the bank's margin was. Wise and Airwallex built
billion-dollar businesses by showing this one number.

---

### Step 3: Compliance and Screening

Before the payment leaves the initiating bank, it passes
through a mandatory compliance stack:

**AML screening:** Beneficiary name and account checked against
OFAC (US Treasury), UN sanctions lists, EU consolidated list,
and the bank's own watchlists. This is automated via
screening engines (Fircosoft, Actimize, Oracle FCCM).

**Purpose of payment:** RBI mandates purpose codes for all
outward remittances from India under FEMA regulations.
Wrong or missing purpose code = payment held or returned.

**Transaction monitoring:** ML models flag anomalies —
unusual amount for this sender, unusual beneficiary country,
payment at 2am, etc.

**KYC/KYB verification:** Is the beneficiary already verified
in the bank's system? First-time beneficiaries often trigger
manual review queues — adding 1-3 days.

**Product gap #3:** Compliance is a black box to the user.
"Payment under review" with no ETA, no reason, no action required
from the user. In reality, one missing document would resolve it
in 10 minutes — but the portal doesn't tell you what's missing.

---

### Step 4: Message Formatting and Network Routing

Once cleared, the payment is formatted as a financial message
and injected into the interbank network.

**SWIFT (Society for Worldwide Interbank Financial Telecommunication)**
is still the dominant network for B2B cross-border.
The payment becomes a SWIFT MT103 message (or ISO 20022
MX pacs.008 message on newer rails).

An MT103 contains:
- Field 20: Transaction reference number
- Field 32A: Value date, currency, amount
- Field 50K: Ordering customer (sender)
- Field 59: Beneficiary customer
- Field 70: Remittance information (payment reference)
- Field 71A: Who pays the charges (OUR / SHA / BEN)

**Routing:** The payment does not go directly from HDFC India
to Deutsche Bank Frankfurt. It travels through correspondent banks —
intermediary banks that have pre-established Nostro/Vostro
account relationships.

A typical corridor (India → Germany):
HDFC India → JPMorgan New York (correspondent) →
Deutsche Bank Frankfurt → Beneficiary account

Each hop deducts a fee ($5-25 per hop).
Each hop can hold the payment for compliance screening.
The sender has no visibility into which hops their
payment is currently at.

**Product gap #4:** SWIFT GPI (Global Payments Innovation)
was introduced to solve this — it adds a Unique End-to-End
Transaction Reference (UETR) that theoretically enables
real-time tracking. But only 50% of SWIFT member banks
have fully implemented GPI tracker integration into
their customer-facing portals. The pipe exists.
The product layer doesn't.

---

### Step 5: Nostro/Vostro Settlement

Correspondent banking runs on Nostro/Vostro accounts:

**Nostro account:** "Our money, held at your bank."
HDFC maintains a USD Nostro account at JPMorgan NY.
When HDFC sends a USD payment, it debits its own Nostro.

**Vostro account:** "Your money, held at our bank."
JPMorgan holds HDFC's USD balance — from JPMorgan's perspective
this is a Vostro (foreign bank's money in our custody).

Settlement is the actual movement of funds between these accounts.
This happens on net positions, not transaction by transaction.
Multiple payments between the same bank pair net out and
settle once per day through central bank systems
(Fedwire in US, TARGET2 in Eurozone, RTGS in India).

This is why cross-border payments appear slow —
the message travels in minutes, but the actual fund
settlement happens in overnight batch cycles.

**Product gap #5:** The customer experiences "pending" for
2-3 days because settlement is batched, not because
the technology is slow. No portal explains this
to the customer. They assume something is wrong.

---

### Step 6: Last-Mile Credit

Deutsche Bank Frankfurt receives the MT103 and credits
the beneficiary's account.

Potential failure points at this stage:
- Account name mismatch (Confirmation of Payee not universal)
- Dormant account
- Account type mismatch (current vs. savings)
- Beneficiary bank not accepting this currency

When it fails here: the payment is returned to the sender
via SWIFT MT103 Return message. Return takes 3-7 days.
The sender gets a debit to their account 3-7 days later
with a vague reference. No explanation. No breakdown
of what fees were deducted during the failed journey.

**Product gap #6 (the most expensive one):** Failed payment
recovery is entirely manual in most banks. No automated
notification, no reason code surfaced to the user,
no one-click resubmission with corrections.

---

## PART 2: THE API LAYER — WHERE PRODUCTS BREAK

Modern cross-border payment APIs fall into three categories.

### Category 1: Bank Direct APIs (HDFC SmartHub, ICICI API Banking)

These are corporate treasury APIs that allow ERP systems
to initiate payments programmatically — no portal login required.

Typical endpoints:

POST /api/v1/payments/international
GET /api/v1/payments/{transactionId}/status
GET /api/v1/fx/rates?from=INR&to=USD
POST /api/v1/beneficiaries



**What they do well:**
- Straight-through processing (STP) — payment initiated from SAP,
  no human intervention needed
- Bulk file upload (ISO 20022 XML pain.001 format) for
  high-volume payroll and vendor payments
- Webhook callbacks on status change

**Critical API design gaps:**

Gap 1 — Error response quality is poor.
A failed payment returns:
```json
{
  "status": "FAILED",
  "errorCode": "PAYMENT_REJECTED",
  "message": "Transaction rejected"
}
```

Compare to what a good API should return:
```json
{
  "status": "FAILED",
  "errorCode": "BENEFICIARY_ACCOUNT_MISMATCH",
  "message": "Account name provided does not match bank records",
  "resolution": "Verify beneficiary account name with recipient",
  "retryable": true,
  "suggestedAction": "Update beneficiary record and resubmit",
  "supportRef": "TXN-2026-084729"
}
```

The second version costs the same to build.
The first version generates a support ticket.
The second resolves itself. Most banks ship the first.

Gap 2 — Status polling vs. webhooks.
Most bank APIs require the client to poll GET /status
every 30 seconds to check payment status.
This creates thousands of wasted API calls per day.
The fix is webhook-first design with polling as fallback.
Fewer than 30% of Indian bank corporate APIs offer
production-grade webhooks.

Gap 3 — FX rate API and payment API are decoupled.
To execute a payment at a specific FX rate, you must:
1. Call the FX rate API to get a quote
2. Note the quote ID and expiry (usually 30-60 seconds)
3. Call the payment API with the quote ID
4. Hope both calls succeed within the window

Rate expiry mid-transaction is a common failure mode
in high-volume treasury systems. The fix is atomic FX+payment
transactions — a single API call that locks rate
and initiates payment together. This is how Airwallex,
Wise Business, and Currencycloud have done it.
Traditional banks have not.

---

### Category 2: Fintech Overlay APIs (Wise Business, Airwallex, Currencycloud)

These sit on top of banking rails and expose a clean,
developer-first API layer with better UX primitives.

**Wise Business API strengths:**
- Real exchange rate vs. bank rate shown side by side
- Estimated arrival time with confidence interval (not just "2-5 days")
- Confirmation of Payee before sending (account name verified)
- Transparent fee breakdown before payment confirmation

**Airwallex strengths:**
- Multi-currency wallets (hold USD, EUR, GBP, AUD, SGD in one account)
- Local payment rails where available
  (ACH in US, SEPA in Europe, FPS in UK, UPI-linked in India)
- Payout API with 30+ local rails, auto-selects fastest/cheapest

**Their shared limitation:**
Correspondent banking still applies for corridors
without local rail coverage.
Neither Wise nor Airwallex has solved
the last-mile problem in emerging markets
(Nigeria, Bangladesh, Pakistan, sub-Saharan Africa).

---

### Category 3: ISO 20022 — The Standard Rewriting Everything

ISO 20022 is the new global financial messaging standard
replacing SWIFT MT messages (MT103, MT202, etc.).

The shift matters for product managers because
ISO 20022 carries significantly more structured data:

MT103 remittance field: 140 characters, unstructured text.
ISO 20022 pacs.008: unlimited structured remittance data,
mandatory creditor/debtor LEI, purpose codes, tax IDs.

This structured data enables:
- Automated invoice reconciliation (payment carries enough
  data to match itself to the invoice without human intervention)
- Richer fraud detection (structured beneficiary data
  is easier to screen than free-text fields)
- Regulatory reporting automation

SWIFT has mandated full ISO 20022 migration by November 2025.
Most large banks are compliant or in migration.
Corporate portals have not caught up — they still expose
unstructured fields to users instead of leveraging
the richer data ISO 20022 carries.

**Product opportunity:** The first corporate banking portal
that uses ISO 20022 structured data to auto-reconcile
payments against uploaded invoices — without any
user input — eliminates an entire back-office function
for treasury teams.

---

## PART 3: PAYMENT INITIATION PORTALS — A PRODUCT TEARDOWN

Three portals evaluated: HSBC HSBCnet, Citi CitiDirect BE,
and Airwallex Business Portal. Evaluated against five criteria.

### Criteria 1: Pre-Payment Validation
Does the portal validate beneficiary details before
the payment is submitted, not after?

HSBCnet: Validates IBAN format only. No account name check.  
CitiDirect: Validates BIC/SWIFT code against SWIFT directory.
No account name check.  
Airwallex: Confirms account name via Confirmation of Payee (CoP)
for UK GBP payments. No CoP for other corridors.  

**Industry-wide gap:** Account name verification before
payment is not universal. Australia mandated it (PayID).
UK implemented CoP in 2020. India has not mandated it
for cross-border. The fraud and error rate is directly
related to this gap.

**PM opportunity:** Build a universal pre-payment
validation service that checks:
1. IBAN checksum validity
2. BIC/SWIFT existence in SWIFT directory
3. Account name fuzzy match against bank records
4. Sanction screening pre-submission (not in batch)
Run as a separate API call before payment initiation.
Show the result to the user before they click Send.

---

### Criteria 2: FX Transparency

HSBCnet: Exchange rate shown only after submission
confirmation. No mid-market rate shown.
No explanation of bank margin.  
CitiDirect: Rate shown at confirmation step but labelled
as "Indicative" — actual rate applied may differ.  
Airwallex: Rate shown at initiation with mid-market
rate, bank rate, and fee in three separate lines.
Rate locked for 30 seconds with countdown timer.
If timer expires, new rate is fetched and user confirms again.

**Winner clearly: Airwallex.**
This is purely a product decision, not a technology
constraint. HSBCnet and CitiDirect have access to the
same FX rates. They chose not to show them transparently.

---

### Criteria 3: Payment Status Visibility

HSBCnet: "Processing" → "Sent" → "Completed."
Three status states. No correspondent bank tracking.
No estimated arrival time.

CitiDirect: Similar. SWIFT GPI tracker is available
as a separate module (CitiDirect GPI) but not integrated
into the main payment flow. User must actively navigate
to it. Most users do not know it exists.

Airwallex: Real-time status with estimated arrival
confidence interval ("Expected arrival: Aug 20-22").
In-portal notification when payment is credited.
Email to sender and optionally to beneficiary.

**PM framework for payment status UX:**

A payment status should answer four questions at all times:
1. Where is the money right now? (In transit, with correspondent, at destination bank)
2. When will it arrive? (Specific date estimate, not "2-5 days")
3. Is anything wrong? (Yes/No + what action is required)
4. What do I do if something goes wrong? (One-click support access)

None of the three portals fully answer all four.

---

### Criteria 4: Error Recovery UX

**The test:** Submit a payment with a deliberately incorrect
beneficiary account number. Measure what happens.

HSBCnet: Payment submitted, enters processing.
Failed notification received by email 48 hours later:
"Transaction REF-XXXX has been returned by the
beneficiary bank." No reason code. No suggested action.
Funds returned 3-5 business days later with fees deducted.
Net loss: 5 days + $35 return fee + staff time to investigate.

CitiDirect: Similar experience. Return notification
slightly more detailed ("Account not found at
beneficiary institution") but no guided resolution.

Airwallex: For payment corridors with CoP enabled,
the error is caught at submission — before the payment
is sent. "Account name does not match records.
Received: 'Acme Corp Ltd.' Expected: 'Acme Corporation.'
Proceed anyway or update beneficiary?"

**The product lesson:**
Shifting error detection from post-sending (48 hours later)
to pre-sending (at initiation) reduces cost by 100x.
The technology difference is a single API call
to the destination bank's name verification service.
The product difference is whether you surface that
API call to the user at the right moment.

---

### Criteria 5: Bulk Payment UX

For treasury teams sending 50-500 payments daily
(payroll, vendor settlements, partner payouts):

HSBCnet: Supports bulk SWIFT file upload (MT101 format).
File format is rigid — any deviation from template
causes full-file rejection. No preview before upload.
Error log is a raw text file with line numbers.

CitiDirect: Similar. Additional support for NACHA files
for US ACH payments. Same rigid validation.

Airwallex: CSV template for bulk upload with
column-by-column validation on upload —
errors highlighted by row and column before submission.
Ability to fix errors in-portal without re-uploading
the entire file. Individual retry of failed rows
without cancelling successful ones.

**The PM framing:**
Bulk payment UX is B2B product design at its hardest —
the user is a finance professional doing repetitive,
high-stakes work. Errors are expensive. Time is scarce.
The portal that treats the finance team as the customer
(rather than a compliance constraint) wins the renewal.

---

## PART 4: THREE PRODUCT OPPORTUNITIES I WOULD OWN AS PM

### Opportunity 1: Pre-Payment Intelligence Layer
**What:** An API-first validation service that runs
before every cross-border payment — verifying beneficiary
account, checking sanctions, estimating arrival time,
and flagging purpose code errors — all in under 3 seconds.

**Why now:** ISO 20022 migration is making structured
beneficiary data available at scale for the first time.
Banks have the data. The product layer doesn't exist.

**North Star metric:** % of cross-border payments
that complete without human intervention or exception handling.

---

### Opportunity 2: FX Transparency Dashboard for Treasury
**What:** A real-time FX comparison embedded in the
payment portal that shows the bank rate, mid-market rate,
bank margin in basis points, and savings vs. forward contract.

**Why it hasn't been built:** Banks profit from FX opacity.
Fintechs who built this (Wise) captured $1B+ in revenue
by being the transparent alternative.
As regulation moves toward mandatory rate transparency
(EU IFR Article 59, India FEMA liberalisation),
incumbents will need this before it is mandated.

**North Star metric:** Customer FX rate captured vs.
mid-market rate (the smaller the spread, the better
the product is working for the customer).

---

### Opportunity 3: Intelligent Payment Return Handler
**What:** When a cross-border payment is returned,
an automated system that:
1. Parses the SWIFT return message for reason code
2. Matches it to the original payment record
3. Classifies the error (wrong account, sanctions, dormant)
4. Routes to self-serve resolution (user corrects and resubmits)
   or automated resolution (bank corrects and retries)
5. Notifies sender and beneficiary with plain-English
   explanation and next steps
6. Tracks SLA for refund of returned funds

**Why it matters:** Industry return rate is 3-5%.
For a bank processing $10B/month in cross-border,
that is $300-500M/month in failed transactions
handled manually. Automating return handling
at even 60% self-serve resolution rate
eliminates significant operational cost
and dramatically improves customer experience.

**North Star metric:** % of returned payments resolved
without a support ticket.

---

## SUMMARY: THE PRODUCT LENS ON CROSS-BORDER PAYMENTS

| Layer | Current State | The Gap | Product Fix |
|---|---|---|---|
| Pre-payment validation | Format check only | No account name verification | CoP API integration |
| FX transparency | Opaque, post-facto | Margin hidden from user | Real-time rate comparison UI |
| Payment tracking | 3 binary states | No correspondent visibility | GPI tracker integration |
| Error messaging | Codes, no context | User doesn't know what to do | Structured error + guided resolution |
| Return handling | Manual, 5+ days | No automated recovery | Intelligent return classifier |
| Bulk UX | File upload, all-or-nothing | One error kills the batch | Row-level validation and retry |

Cross-border payments are not slow because the technology
is old. SWIFT processes millions of messages daily in seconds.

They are slow, expensive, and opaque because the
**product layer between the rails and the user
has not been rebuilt with the user as the customer.**

Every gap listed above is a product management problem,
not an engineering problem. The APIs exist.
The regulatory frameworks are aligning.
The window for whoever builds the best product layer
is open right now.

---

*Yash Gaidhani is a Manager Consulting Expert at CGI India
(XLRI Jamshedpur 2026) focused on enterprise AI product
strategy. This analysis is based entirely on public
information, published research, and first-principles
reasoning. No proprietary or client-confidential data
is referenced.*

*Connect: linkedin.com/in/yashgaidhani-xlri*  
*Portfolio: github.com/yashgaidhani*
