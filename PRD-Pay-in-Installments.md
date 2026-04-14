# PRD: Pay in Installments (Splitit Integration)

**Author:** Product Team
**Date:** April 14, 2026
**Status:** Draft
**Stakeholders:** Engineering, Design, Finance, Splitit Partner Team

---

## 1. Overview

Vibrant Wellness will integrate Splitit's white-label installment payment service into both the Provider Portal and the Patient Portal. This allows patients to split their order payments into 2, 3, or 4 equal monthly installments using their existing credit card — with zero interest and no additional fees.

Splitit's model is fundamentally different from traditional BNPL (Klarna, Afterpay): it does not issue new loans or require credit checks. Instead, it places an authorization hold on the patient's existing credit card for the full amount and charges installments monthly. Vibrant receives full payment upfront from Splitit regardless of the patient's installment schedule.

---

## 2. Goals

- **Reduce payment friction:** Give patients a flexible way to pay for high-value test orders without committing to the full amount at once.
- **Increase conversion:** Patients who hesitate at checkout due to cost may convert with an installment option.
- **Zero additional risk:** Splitit's authorization-hold model means Vibrant bears no default risk.
- **Minimal engineering lift:** Leverage Splitit's Hosted Form Redirect — Splitit hosts the entire payment page, Vibrant only needs to integrate via API.

---

## 3. Scope

### In Scope

- Add "Pay in Installments" option to **Provider Portal** (within the Patient Pay Now checkout flow)
- Add "Pay in Installments" option to **Patient Portal** (as a top-level payment method alongside Credit Card, BNPL, and Google Pay)
- Installment plan options: **2, 3, or 4 monthly payments**
- Integration method: **Splitit Hosted Form Redirect**
- Refund handling via Splitit Merchant Hub / API
- Optional "Powered by Splitit" branding

### Out of Scope

- Provider Office Bill — no installment option for this payment method
- Installment plans beyond 4 months
- Custom payment page hosted by Vibrant (Splitit hosts the payment form entirely)
- Splitit Shopper Portal customization — patients use Splitit's default portal for plan management

---

## 4. User Flows

### 4.1 Provider Portal — Patient Pay Now

**Context:** The provider has already selected "Patient Pay Now" as the payment method on the previous (Place Order) step. The checkout page shows the payment form.

1. Provider sees two sub-tabs under Payment Method: **"Pay in Full"** (default, existing credit card form) and **"Pay in Installments"** (new).
2. Provider selects "Pay in Installments."
3. The credit card form is replaced by an installment picker showing 2, 3, and 4-month options with calculated per-month amounts.
4. Provider selects a plan (e.g., 3 payments of $205.00).
5. A summary confirms the breakdown: per-month amount × number of payments = total. Copy emphasizes: first payment today, no interest, no hidden fees.
6. A notice explains the patient will be redirected to a secure page to enter card details.
7. Provider clicks **"Set Up 3-Month Plan · $205.00/mo"** (CTA updates dynamically).
8. Backend calls Splitit Initiate API with the order amount, selected number of installments, and redirect URLs.
9. Patient is redirected to Splitit's hosted payment page to enter credit card information.
10. After completion, patient is redirected back to Vibrant's confirmation page.
11. Backend calls Splitit VerifyAuthorization API to confirm payment success.
12. Order is confirmed. Vibrant receives full payment from Splitit.

### 4.2 Patient Portal — Patient Pay Later

**Context:** The patient has received an order from their provider and is viewing the payment page in the Patient Portal.

1. Patient sees four top-level payment method options: **Credit / Debit Card**, **Pay in Installments** (new), **Buy Now Pay Later**, and **Google Pay**.
2. "Pay in Installments" is positioned second (high visibility), with a "NEW" badge during the launch period.
3. Patient selects "Pay in Installments."
4. An installment picker expands below, showing 2, 3, and 4-month options with per-month amounts.
5. Patient selects a plan.
6. Summary and secure redirect notice are displayed.
7. Patient clicks **"Set Up 3-Month Plan · $221.67/mo"**.
8. Same backend flow as Provider Portal: Initiate API → redirect to Splitit → return to Vibrant → VerifyAuthorization.
9. Order payment is confirmed.

---

## 5. Technical Architecture

### 5.1 Integration Method

**Splitit Hosted Form Redirect** — the simplest integration path. Vibrant never touches credit card data; Splitit handles all PCI-sensitive operations.

### 5.2 API Flow

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Vibrant   │         │   Splitit   │         │   Patient   │
│   Backend   │         │     API     │         │   Browser   │
└──────┬──────┘         └──────┬──────┘         └──────┬──────┘
       │                       │                       │
       │  1. POST /token       │                       │
       │──────────────────────>│                       │
       │  Bearer token         │                       │
       │<──────────────────────│                       │
       │                       │                       │
       │  2. POST /initiate    │                       │
       │  (amount, # of        │                       │
       │   installments,       │                       │
       │   redirect URLs)      │                       │
       │──────────────────────>│                       │
       │  CheckoutUrl          │                       │
       │<──────────────────────│                       │
       │                       │                       │
       │  3. Redirect patient  │                       │
       │─────────────────────────────────────────────>│
       │                       │                       │
       │                       │  4. Patient enters    │
       │                       │     card details      │
       │                       │<──────────────────────│
       │                       │                       │
       │                       │  5. Redirect back     │
       │                       │     (success/fail/    │
       │                       │      cancel URL)      │
       │<─────────────────────────────────────────────│
       │                       │                       │
       │  6. POST /verify      │                       │
       │──────────────────────>│                       │
       │  Authorization result │                       │
       │<──────────────────────│                       │
       │                       │                       │
       │  7. Show confirmation │                       │
       │─────────────────────────────────────────────>│
```

### 5.3 Key API Calls

| Step | API Endpoint | Purpose |
|------|-------------|---------|
| Auth | `POST /api/v3/oauth/token` | Get bearer token |
| Initiate | `POST /api/v3/installment-plans/initiate` | Create plan, get `CheckoutUrl` |
| Verify | `POST /api/v3/installment-plans/verify-authorization` | Confirm payment after redirect |
| Refund | `POST /api/v3/installment-plans/refund` | Process full or partial refund |

### 5.4 Initiate API — Key Parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| `Amount` | Order total | e.g., 614.99 |
| `NumberOfInstallments` | 2, 3, or 4 | Selected by user before redirect |
| `CurrencyCode` | USD | |
| `RedirectUrls.Succeeded` | `https://.../payment/success/{ipn}/{ref-order-number}` | Includes plan number and order ref |
| `RedirectUrls.Failed` | `https://.../payment/failed/{ipn}/{ref-order-number}` | |
| `RedirectUrls.Canceled` | `https://.../payment/canceled/{ipn}/{ref-order-number}` | |
| `EventsEndpoints.CreateSucceeded` | Webhook URL | Backup confirmation if redirect fails |

### 5.5 Important Technical Notes

- The `CheckoutUrl` returned by Initiate is a one-time-use URL, valid for 1 hour by default.
- Always call VerifyAuthorization before marking an order as paid — this prevents spoofed success URLs.
- Set up the `CreateSucceeded` webhook as a fallback for cases where the redirect back fails (e.g., network issues).
- Splitit supports Visa and Mastercard globally; Amex/Discover support depends on merchant configuration.

---

## 6. Refund & Cancellation Flow

### 6.1 How Refunds Work

Since Vibrant receives the full order amount upfront from Splitit, the refund flow involves returning funds to Splitit, who then refunds the patient:

1. Patient contacts Vibrant requesting a refund.
2. Vibrant approves per internal refund policy.
3. Vibrant initiates refund via Splitit Merchant Hub or Refund API (full or partial amount).
4. Vibrant returns the refund amount to Splitit.
5. Splitit processes the refund to the patient's credit card (up to 5 business days).
6. Splitit cancels any remaining installment charges and closes the plan.
7. Patient receives email confirmation from Splitit.

**Note:** Splitit no longer supports a standalone "cancel plan" action. To close out a plan, a full refund must be issued.

### 6.2 Pending Confirmation from Splitit

The exact mechanism of step 4 (how Vibrant returns funds to Splitit — automatic deduction via API vs. separate settlement) needs to be confirmed with the Splitit team.

---

## 7. Failed Payments & Default Handling

### 7.1 Failed Monthly Charge

This is handled entirely by Splitit — no action required from Vibrant:

1. If an installment charge is declined, Splitit emails the patient.
2. Patient has a **7-day grace period** to resolve the issue.
3. Patient can update their credit card via the **Splitit Shopper Portal** (shopper.splitit.com).
4. Splitit continues retrying the charge during the grace period.

### 7.2 Unresolved Failure (Default)

5. If the charge still fails after 7 days, Splitit may charge the **full remaining balance** using the last successful authorization.
6. This terminates the installment plan early.

### 7.3 Risk to Vibrant

**None.** Splitit authorizes the full purchase amount on the patient's card at checkout. If a patient cannot pay, Splitit handles collection. If the patient cancels their card entirely, Splitit collects from the issuing bank. Vibrant has already received full payment and carries zero default risk.

---

## 8. UI Specifications

### 8.1 Provider Portal (Patient Pay Now)

**Location:** Checkout page → Payment Method section

| Element | Specification |
|---------|--------------|
| Sub-tabs | "Pay in Full" (default) and "Pay in Installments" (new, with NEW badge at launch) |
| Installment picker | 3 cards showing 2, 3, 4 payment options with per-month amount |
| Summary block | "{per-month} per month × {n} payments = {total} total" + "First payment collected today. No interest, no hidden fees." |
| Redirect notice | "The patient will be taken to a secure payment page to enter their card details and confirm the plan. Once complete, they'll return here automatically." |
| CTA button | Dynamic: "Set Up {n}-Month Plan · ${amount}/mo" |
| Powered by | Optional "Powered by Splitit" text below the section |

### 8.2 Patient Portal (Patient Pay Later)

**Location:** Payment page → Payment Method section (top-level option)

| Element | Specification |
|---------|--------------|
| Position | 2nd option (after Credit/Debit Card, before Buy Now Pay Later) |
| Card label | "Pay in Installments" with NEW badge at launch |
| Installment picker | Same as Provider Portal |
| Summary block | "{per-month} per month × {n} payments = {total} total" + "Your first payment is collected today. The rest are charged monthly — no interest, no hidden fees." |
| Redirect notice | "You'll be taken to a secure page to enter your card details and confirm the plan. Once done, you'll return here automatically." |
| CTA button | Dynamic: "Set Up {n}-Month Plan · ${amount}/mo" |

### 8.3 Design Notes

- Provider Portal follows existing Vibrant Wellness teal brand palette (#00b4b4).
- Patient Portal follows existing navy brand palette (#1b3a4b). No teal on this portal.
- "Pay in Installments" is a **top-level** payment method in the Patient Portal, not nested under Buy Now Pay Later. Rationale: Splitit's model (existing credit card, no new loans) is fundamentally different from BNPL; nesting it reduces discoverability and creates confusion.

---

## 9. Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Installment adoption rate | 10-15% of eligible orders within 3 months | Orders using installments / total orders |
| Checkout conversion lift | +3-5% on orders > $500 | A/B test against control (no installment option) |
| Installment completion rate | > 95% | Plans completed without default / total plans |
| Refund rate (installment orders) | ≤ baseline refund rate | Installment refunds / installment orders vs. overall |

---

## 10. Launch Plan

### Phase 1 — Sandbox Integration (Week 1-2)
- Set up Splitit developer account and sandbox credentials
- Implement Initiate → Redirect → VerifyAuthorization flow in staging
- Build installment picker UI on both portals
- Test with Splitit sandbox test cards

### Phase 2 — Internal QA (Week 3)
- End-to-end testing: happy path, failed payments, cancellations
- Verify refund flow via Merchant Hub
- Confirm webhook fallback works
- Cross-browser and mobile testing

### Phase 3 — Soft Launch (Week 4)
- Enable for a small subset of provider accounts
- Monitor conversion, error rates, and Splitit webhook reliability
- Collect feedback from providers and patients

### Phase 4 — General Availability (Week 5+)
- Roll out to all accounts
- Remove NEW badge after ~4 weeks
- Monitor success metrics

---

## 11. Open Questions

| # | Question | Status |
|---|----------|--------|
| 1 | Exact refund settlement mechanism — when Vibrant refunds via API, is the money automatically deducted from Vibrant's next Splitit payout, or is there a separate settlement process? | Pending — asked Splitit team via Slack |
| 2 | Can the Splitit hosted payment page be styled (colors, logo) to match Vibrant branding, or is it fully Splitit-controlled? | Pending |
| 3 | What is the minimum order amount eligible for installments? | To be defined |
| 4 | Should we show installment options only above a certain order value threshold (e.g., > $200)? | To be decided by Product |
| 5 | Do we want to link patients to the Splitit Shopper Portal from our Patient Portal for plan management, or keep the experiences fully separate? | To be decided |

---

## 12. References

- [Splitit Developer Docs — Getting Started](https://developers.splitit.com/getting-started/adding-splitit)
- [Hosted Form Redirect Integration Guide](https://developers.splitit.com/checkout-solutions/hosted-forms/hosted-form-redirect)
- [Installment Plan Options](https://developers.splitit.com/getting-started/plan-options)
- [Shopper Portal Guide](https://developers.splitit.com/shopper-portal)
- [Splitit Support — Refunds](https://support.splitit.com/hc/en-us/articles/360010801639)
- [Splitit Support — Failed Payments](https://support.splitit.com/hc/en-us/articles/21721998969106)
- Provider Portal Prototype: `vibrant-installment-prototype.html`
- Patient Portal Prototype: `patient-portal-installment-prototype.html`
