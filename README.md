# Stripe information handoff for development teams

This guide helps **clients / account owners** gather everything needed for payment integration, webhooks, reporting, and ongoing operations via the Stripe API in one place. Dashboard paths follow Stripe’s official docs; Stripe occasionally renames or moves sections—if something differs, use the Dashboard search bar (`/` shortcut) to find the section by name.

---

## 1. Test mode vs Live mode

| Mode | Use |
|------|-----|
| **Test** | Development, CI, manual testing without real money |
| **Live** | Real customer transactions |

Use the **Test mode / Live mode** toggle at the top of the Dashboard. **API keys and webhook signing secrets are separate for Test and Live.** Provide both sets per the checklist below unless your engineering team explicitly asks for only one.

---

## 2. Typical handoff checklist

The client should send these through a **secure channel** (not plain email or public chat).

### A) API keys

| Typical env var | Description | Key prefix |
|-----------------|-------------|------------|
| Publishable key | Frontend / Elements / client-side Checkout; safe to expose in the browser | `pk_test_…` / `pk_live_…` |
| Secret key | Backend only; **confidential**; broad API access (subject to account capabilities) | `sk_test_…` / `sk_live_…` |

**Dashboard path:**  
[Developers → API keys](https://dashboard.stripe.com/test/apikeys)  
(In Live mode, same path with the Live toggle on.)

### B) Webhook signing secret

Each webhook **endpoint** has its own secret (`whsec_…`). You need it to verify that events actually came from Stripe.

**Path:**  
[Developers → Webhooks](https://dashboard.stripe.com/test/webhooks) → select the endpoint → **Signing secret**.

Your engineers should give you the final webhook URL to register (for example `https://api.example.com/api/webhook/stripe`). For local development, [Stripe CLI](https://docs.stripe.com/stripe-cli) can forward events; that flow uses a **different** signing secret (`whsec_…` from the CLI).

### C) Webhook events

When creating or editing an endpoint, select at least the events your application depends on. For a typical Checkout flow, these are often relevant (confirm exact names with your engineering team):

- `checkout.session.completed`
- If you need deeper tracking: `payment_intent.succeeded`, `charge.succeeded`, etc.

---

## 3. Enabling payment methods (cards, iDEAL, PayPal, …)

Many integrations—including **Stripe Checkout** without manually listing `payment_method_types`—**read enabled methods from Dashboard settings**.

**Main path:**  
[Settings → Payment methods](https://dashboard.stripe.com/settings/payment_methods)

### Important notes

- **Dynamic payment methods:** Stripe recommends managing preferences on this page so eligible methods are shown based on currency, country, and other factors. See [Dynamic payment methods](https://docs.stripe.com/payments/payment-methods/dynamic-payment-methods).
- **Cards:** Usually on by default; multi-currency / regional behavior still depends on account settings and risk controls.
- **iDEAL:** Common for **EUR** and eligible customers (e.g. Netherlands). iDEAL must be enabled under **Payment methods**, and the checkout **currency** must align with EUR where required. Region/currency details: [Payment method support](https://docs.stripe.com/payments/payment-methods/payment-method-support).
- **PayPal via Stripe:** Besides a PayPal account, you turn it on in Stripe and connect PayPal when prompted. Steps: [Activate PayPal payments](https://docs.stripe.com/payments/paypal/activate).  
  **Geographic eligibility:** Not available in all countries (Stripe’s docs maintain the current list).
- New methods may require **business verification, risk review, or extra documentation**; until complete, they may not appear at Checkout.

---

## 4. Reducing dependence on the client for Dashboard tasks

Three common approaches (combinable):

### A) Standard secret key (`sk_…`)

- **Benefit:** Full programmatic API access for reporting, refunds, transaction lookup, Customer management, etc.
- **Risk:** Large blast radius if leaked. Store only on servers and in a secrets manager.

### B) Restricted API key (`rk_…`)

Stripe lets you assign **Read / Write / None** per resource; prefixes are `rk_test_` / `rk_live_`. Reporting often needs **read** on Charges, PaymentIntents, Customers, Balance, Payouts, etc.; financial actions need **write** where appropriate. See [Restricted API keys](https://docs.stripe.com/keys/restricted-api-keys).

Use this when you want least-privilege access scoped to the project.

### C) Invite team members with appropriate roles

If engineers sometimes need the **Dashboard** (webhooks, error logs, settings):

**Path:**  
[Settings → Team and security → Team](https://dashboard.stripe.com/settings/team) (or the equivalent in your account settings.)

Pick roles that match the need (read-only vs developer vs admin—exact labels are on Stripe’s page). Some work can be done in the Dashboard without sharing the secret key; **API automation still requires a server-side key.**

---

## 5. Account readiness for reporting and future operations

Depending on your agreement, the client or your team should confirm:

- **Business profile and verification:** Incomplete verification can limit volume or certain payment methods.
- **Company details and bank account for payouts:** Financial sections of the Dashboard (bank accounts, etc.).
- **Statement descriptor / name on card** if branding matters.
- **Roles and MFA** on the Stripe account (account security = money security).

---

## 6. Monitoring and debugging inside Stripe

- **Events:** [Developers → Events](https://dashboard.stripe.com/test/events) — inspect payloads and event chains.
- **API request logs:** Via **Workbench** (Stripe enables Workbench by default for many new accounts) or the Developers area; see [Developers Dashboard / Workbench](https://docs.stripe.com/development/dashboard) and [Developers settings](https://dashboard.stripe.com/settings/developers).

If the API returns an error, the **Request ID** in Stripe’s response is useful for support and debugging.

---

## 7. Security essentials (short list for the client)

- Treat the secret key and `whsec_` like banking credentials—do not commit them to git, paste them in public tickets, or share via screenshots.
- Prefer **separate keys or environments** for staging vs production to limit blast radius.
- Stripe supports **rotating** keys; after rotation, revoke the old key on servers.

---

## 8. One-page template for the engineering team

The client can fill in:

```
Stripe account (business name on Dashboard):
Client technical contact:

--- Test mode ---
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_SECRET_KEY=sk_test_...   (or rk_test_... if using a restricted key)
STRIPE_WEBHOOK_SECRET=whsec_...   (for the endpoint matching the engineering team’s URL)
Webhook URL registered in Test:

--- Live mode ---
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_SECRET_KEY=sk_live_...   (or rk_live_...)
STRIPE_WEBHOOK_SECRET=whsec_...
Webhook URL registered in Live:

Webhook events subscribed:

Payment methods enabled under Settings → Payment methods (summary):
e.g. cards, iDEAL, …

PayPal via Stripe: enabled / not enabled — if enabled, settlement preference (Stripe balance vs PayPal) per your choice:

Invite engineering emails to Stripe with role … : yes / no (emails):
```

---

## Official references (maintained by Stripe)

- Manage API keys in the Dashboard: https://docs.stripe.com/development/dashboard/manage-api-keys  
- Restricted API keys: https://docs.stripe.com/keys/restricted-api-keys  
- Webhooks: https://docs.stripe.com/webhooks  
- Payment methods and Dashboard: https://docs.stripe.com/payments/payment-methods/integration-options  

If your codebase uses `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, and `STRIPE_WEBHOOK_SECRET`, use those same names in the server `.env` file.
