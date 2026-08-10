# Delícias da Jade — Payment Gateway Integration Plan

Status: **"Em breve" mode is LIVE.** The payment section shows the gateway UI with
manual payment active: MB Way tab has the number + copy + reference (pay directly in
the MB Way app → Jade's account) and a dashed "Em breve" box with the future online
flow; Cartão and PayPal tabs are dimmed + disabled with "em breve" tags (they need the
merchant account); IBAN tab works as before. `GATEWAY_CONFIG.active` is `false` and the
"Continuar para o pagamento" buttons are removed from the panels — `GATEWAY_PAY()` and
the `#gwOv` modal are still in the script, ready to wire.

## 1. Goal (Shelton's brief)

Client finishes the order → a payment gateway page opens → for MB Way the client types
their phone number → payment push lands in the MB Way app → approves. Same idea for
card (Visa/Mastercard), PayPal, and later Revolut. Price is fixed from the order
builder before the gateway opens.

## 2. Hard prerequisites (no way around these)

1. **Merchant identity for Jade / Delícias da Jade**: NIF (empresa/ENI or recibo verde)
   + business bank account. PSP onboarding (KYC) needs this. **Jade does NOT have this
   yet (2026-08-10)** — this is the gating item.
2. **HTTPS hosting** for the site + a tiny backend. Free options: Netlify / Vercel /
   Cloudflare Pages (+ their serverless functions). Site is currently a single HTML
   file on `localhost:8779`; hosting is still undecided (ask Shelton).
3. **PSP account** (below) — approval takes days–weeks; start early.

## 3. PSP recommendation

- **Primary: ifthenpay** (ifthenpay.com) — Portuguese, no monthly fees, does
  **MB WAY payment requests (client types phone → app push)**, Visa/Mastercard,
  Multibanco refs, Apple/Google Pay. Has a **Shopify app** (community-verified) → the
  integration survives the Shopify move.
- **Alternative: Easypay** (easypay.pt) — same MB Way phone flow ("o cliente recebe na
  APP MB Way"), developer-friendly API, Shopify config exists too.
- **PayPal**: separate PayPal Business account + hosted checkout button (not bundled
  with the PT PSPs). Simple.
- **Revolut Pay**: separate — Revolut Business's own gateway (payments via Revolut).
  Defer to phase 2; note Revolut's gateway is a full merchant account too.
- Verify current fee tables on each PSP site before choosing (transaction % + MB Way
  per-request cost). No commitment needed from us until Jade signs up.

## 4. Architecture

```
Static site (index.html) ──POST /api/pay {method, amount, order}──▶ Backend (serverless fn)
        │                                                            │ calls PSP API
        │ ◀──────────── redirect to PSP hosted checkout ────────────┤
        │   (MB Way: phone field · card: secure fields · PayPal)     │
        │                                                            │
        └───────────── webhook POST /api/confirm ◀── PSP notifies ───┘
                          → notify Jade (WhatsApp/email)
```

- **Backend**: one serverless function set (free tier). Two endpoints:
  - `POST /api/pay` — validate amount (server-side, never trust client), call PSP
    create-payment, return `{ url }` → site redirects.
  - `POST /api/confirm` — PSP webhook; verify signature; notify Jade
    (reuse the WhatsApp bridge / email) with order + amount.
- **Secrets**: PSP API keys + webhook secret live ONLY in backend env vars. Never in
  index.html, never committed.
- **PCI**: card data never touches our server — client types into the PSP hosted page.
- **Site-side**: when the merchant account is approved: (1) re-enable the Cartão/PayPal
  tabs (remove `disabled` + the `tab-soon` tags), (2) restore a "Continuar para o
  pagamento" button in each panel calling `GATEWAY_PAY(method)` (the function, the
  `#gwOv` modal, and the `/api/pay` fetch are already in the script), (3) hide the
  `#gwNote` banner and the dashed `gw-soon` boxes, (4) set
  `GATEWAY_CONFIG = { active: true, apiBase: 'https://…' }`.
  `GATEWAY_PAY(method)` posts `{method, amount, currency:'EUR', order}` and redirects
  to `data.url`.

## 5. Payment methods matrix (phase 1)

| Method | Provider | Flow on gateway page |
|---|---|---|
| MB Way | ifthenpay / Easypay | type phone → approve in app |
| Visa / Mastercard | same PSP | card fields + 3-D Secure |
| Multibanco (optional, free for clients) | same PSP | reference + ATM/app payment |
| PayPal | PayPal Business | login → confirm |
| Revolut Pay | Revolut Business | phase 2 |

## 6. Execution order (when approved)

1. Jade registers with chosen PSP (+ PayPal if wanted) → credentials.
2. Decide hosting (recommend Netlify or Cloudflare Pages) + move the site there (HTTPS).
3. Build the backend (2 endpoints) with the PSP's sandbox keys.
4. Test in PSP sandbox: create payment → MB Way push → webhook → notification to Jade.
5. Flip `GATEWAY_CONFIG.active = true` + real keys (backend env).
6. Keep the WhatsApp confirm flow as fallback for manual/IBAN and edge cases.
7. Refund/chargeback handling per PSP docs; keep a simple orders log (backend).

## 7. Shopify migration (later)

- The PSP chosen above has a Shopify app → payment methods carry over.
- When the site moves to Shopify, the order builder maps to Shopify cart/checkout;
  this static site becomes a menu/showcase. Keep `PAYMENTS-PLAN.md` updated with what
  was actually used (PSP, backend host) so the migration is a checklist, not a hunt.

## 8. Security & compliance notes

- Amount is re-validated server-side at `POST /api/pay` (compute from the order payload
  server-side too — don't trust the client's number).
- Webhook signature verification (PSP provides a secret) — reject unsigned callbacks.
- Rate-limit `/api/pay`; keep an order id per payment for reconciliation.
- No card data, no phone numbers stored by the site; logs keep only order id + status.
