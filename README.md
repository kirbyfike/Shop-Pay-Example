# Shop Pay Wallet – Basic Example

A minimal, single-file example of adding the **Shop Pay** button and Shop Pay login (email recognition) to a custom, non-Shopify checkout. It uses the [Shop Pay Wallet JavaScript SDK](https://shopify.dev/docs/api/commerce-components/pay/javascript-sdk).

It shows the whole client-side flow:

- loading and configuring the SDK
- rendering the Shop Pay button and the email-recognition login
- building a payment request (line items, subtotal, tax, total)
- handling the session events (session creation, address changes, shipping changes, payment confirmation, completion and failure)

> ⚠️ This is a **frontend example only**. It calls two backend endpoints that you must build yourself, because the Storefront API calls must run on a server. Sketches of both are under [Build the backend](#4-build-the-backend).

---

## Contents

- [How it works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
  - [1. Configure your Shopify store](#1-configure-your-shopify-store)
  - [2. Clone and configure](#2-clone-and-configure)
  - [3. Run it locally](#3-run-it-locally)
  - [4. Build the backend](#4-build-the-backend)
- [Event reference](#event-reference)
- [Testing](#testing)
- [Going to production](#going-to-production)
- [Troubleshooting](#troubleshooting)
- [Resources](#resources)

---

## How it works

```
 Browser (index.html)                Your server                     Shopify
 ────────────────────                ───────────                     ───────
 Buyer clicks Shop Pay
   └─ "sessionrequested" ─────────▶  POST /api/shop-pay/session
                                       └─ shopPayPaymentRequest ───▶  Storefront API
                                          SessionCreate
   completeSessionRequest() ◀──────  { token, checkoutUrl, sourceIdentifier }

 Buyer picks address / shipping
   └─ "shippingaddresschanged"
   └─ "deliverymethodchanged"        (handled in the browser in this example;
      → return an updated request     in production, validate on your server)

 Buyer clicks "Pay now"
   └─ "paymentconfirmationrequested" ▶ POST /api/shop-pay/submit
                                       └─ shopPayPaymentRequest ───▶  Storefront API
                                          SessionSubmit
   completePaymentConfirmationRequest()

   └─ "paymentcomplete" ───▶ redirect to /thank-you
```

---

## Prerequisites

- A Shopify store with **Shop Pay Wallet** access (part of Commerce Components by Shopify). Per Shopify, the Shop Pay Wallet API can't be used on an existing store, so create a new store for this integration.
- **Shopify Payments** fully set up, with **test mode** turned on while you develop.
- The **Shop** sales channel installed on the store.
- Storefront API credentials for your backend (the Shop Pay Wallet plan includes a pre-configured custom app).
- Any static file server for local development (examples below use Node's `npx`, Python also works).

---

## Setup

### 1. Configure your Shopify store

These steps follow Shopify's [Shop Configuration Guide](https://shopify.dev/docs/api/commerce-components/pay/shop-configuration).

1. **Get your IDs.** In Shopify Admin, open the **Shop** sales channel. On its homepage, copy:
   - **Shop ID** (numeric). This becomes `SHOP_ID`.
   - **Shop Pay Client ID**. This becomes `CLIENT_ID`.

   > Don't confuse the Shop Pay Client ID with the client ID of a Shop app (that one is for customer sign-in).

2. **Allow your origins.** Go to **Shop** sales channel → **Domains allow list**. Add every origin where the SDK will load, with the scheme and **without a port**. For example:
   ```
   http://localhost
   https://your-tunnel.ngrok.app
   https://store.example.com
   ```
   These don't need DNS verification.

3. **Turn on test mode** in Shopify Payments so no real charges are made.

4. **Get Storefront API credentials** for your backend. Keep them on the server and never ship them in `index.html`.

### 2. Clone and configure

```bash
git clone https://github.com/kirbyfike/shop-pay-basic-example.git
cd shop-pay-basic-example
```

Open `index.html` and fill in the config block:

```js
const SHOP_ID   = 1;               // your numeric Shop ID
const CLIENT_ID = "[REPLACE-ME]";  // your Shop Pay Client ID

const SESSION_ENDPOINT = "/api/shop-pay/session";
const SUBMIT_ENDPOINT  = "/api/shop-pay/submit";
```

The example product ($25.00), the shipping options, and the delivery dates in `SHIPPING_OPTIONS` are all hard-coded placeholders. Update the dates to be in the future, or the shipping options may look stale.

### 3. Run it locally

Serve the folder from `localhost`. The page will **not** work when opened as a `file://` URL.

```bash
npx serve .
# or
python3 -m http.server 8080
```

Then open `http://localhost:3000` (or `:8080`).

With `debug: true` set, the SDK logs to the browser console. The button and login should render. Clicking the button will fail until your backend endpoints exist.

> To test on a real phone or share a link, use a tunnel (such as ngrok or Cloudflare Tunnel) and add the tunnel's origin to the **Domains allow list**.

### 4. Build the backend

The page calls two endpoints. Each one calls a Storefront API mutation from your server.

**Endpoint:** `https://{your-store}.myshopify.com/api/2026-10/graphql.json`
**Auth header:** `Shopify-Storefront-Private-Token: <your private Storefront token>`

> The mutation signatures below are for Storefront API **2026-10** and later. In earlier versions, `shopPayPaymentRequestSessionCreate` also requires a `paymentRequest` argument. Check the [GraphQL pre-payment docs](https://shopify.dev/docs/api/commerce-components/pay/graphql-pre-payment) for your version.

#### `POST /api/shop-pay/session` → `shopPayPaymentRequestSessionCreate`

```graphql
mutation shopPayPaymentRequestSessionCreate($sourceIdentifier: String!) {
  shopPayPaymentRequestSessionCreate(sourceIdentifier: $sourceIdentifier) {
    shopPayPaymentRequestSession {
      token
      sourceIdentifier
      checkoutUrl
    }
    userErrors { field message }
  }
}
```

`sourceIdentifier` is your own ID for this checkout (for example, your cart or order ID). It **must be unique across all orders** so that reconciliation works. Return `{ token, checkoutUrl, sourceIdentifier }` to the browser.

#### `POST /api/shop-pay/submit` → `shopPayPaymentRequestSessionSubmit`

```graphql
mutation shopPayPaymentRequestSessionSubmit(
  $token: String!
  $paymentRequest: ShopPayPaymentRequestInput!
  $idempotencyKey: String!
  $orderName: String
) {
  shopPayPaymentRequestSessionSubmit(
    token: $token
    paymentRequest: $paymentRequest
    idempotencyKey: $idempotencyKey
    orderName: $orderName
  ) {
    paymentRequestReceipt { token processingStatusType }
    userErrors { field message }
  }
}
```

Before you submit, **re-validate the final payment request on the server**: prices, tax, shipping, and inventory. Don't trust the totals the browser sends. Use a fresh UUID for `idempotencyKey`, and reuse it if you retry the same submission. If the mutation returns `userErrors`, return an `errors` field so the page shows the buyer an error.

#### Minimal Node.js sketch (Express)

This is untested scaffolding meant as a starting point. It isn't production code.

```js
import express from "express";
import crypto from "node:crypto";

const app = express();
app.use(express.json());
app.use(express.static(".")); // serves index.html

const STOREFRONT_URL = `https://${process.env.SHOP_DOMAIN}/api/2026-10/graphql.json`;

async function storefront(query, variables) {
  const res = await fetch(STOREFRONT_URL, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Shopify-Storefront-Private-Token": process.env.STOREFRONT_PRIVATE_TOKEN,
    },
    body: JSON.stringify({ query, variables }),
  });
  return res.json();
}

app.post("/api/shop-pay/session", async (req, res) => {
  const { data } = await storefront(SESSION_CREATE_MUTATION, {
    sourceIdentifier: `order-${crypto.randomUUID()}`, // use your own cart/order ID
  });
  const result = data.shopPayPaymentRequestSessionCreate;
  if (result.userErrors.length) return res.status(422).json({ errors: result.userErrors });
  res.json(result.shopPayPaymentRequestSession);
});

app.post("/api/shop-pay/submit", async (req, res) => {
  const { token, payment_request } = req.body;
  // TODO: re-validate payment_request (prices, tax, shipping, inventory) here
  const { data } = await storefront(SESSION_SUBMIT_MUTATION, {
    token,
    paymentRequest: payment_request,
    idempotencyKey: crypto.randomUUID(),
  });
  const result = data.shopPayPaymentRequestSessionSubmit;
  if (result.userErrors.length) return res.status(422).json({ errors: result.userErrors });
  res.json(result.paymentRequestReceipt);
});

app.listen(3000, () => console.log("http://localhost:3000"));
```

(`SESSION_CREATE_MUTATION` and `SESSION_SUBMIT_MUTATION` are the GraphQL strings above.)

---

## Event reference

These are the events `index.html` handles:

| Event | When it fires | What the example does |
| --- | --- | --- |
| `sessionrequested` | The buyer clicks the button or authenticates | POSTs to `SESSION_ENDPOINT`, then calls `completeSessionRequest()` |
| `shippingaddresschanged` | The buyer picks or changes an address | Offers shipping options for US addresses only, and returns an error for other countries |
| `deliverymethodchanged` | The buyer picks a shipping method | Adds the shipping line and recalculates the total |
| `paymentconfirmationrequested` | The buyer clicks **Pay now** | POSTs to `SUBMIT_ENDPOINT`, then calls `completePaymentConfirmationRequest()` |
| `paymentcomplete` | The payment succeeded | Closes the session and redirects to `/thank-you` |
| `paymentattemptfailed` | The payment was declined | Logs the reason and keeps the window open so the buyer can retry |
| `windowclosed` | The buyer closed the Shop Pay window | Logs it |

Every `complete*` call must return a **complete** updated payment request, and it must happen within **15 seconds**. See [Sessions and events](https://shopify.dev/docs/api/commerce-components/pay/sessions-and-events).

> The example turns off the discount field (`discountCodeField: false`) so it doesn't need a `discountcodechanged` handler. If you turn the field on, add that handler.

---

## Testing

- Keep Shopify Payments in **test mode**.
- Use the test cards and accounts in [Test Shop Pay](https://shopify.dev/docs/api/commerce-components/pay/test-shop-pay).
- Test these paths: a successful payment, a declined card, address changes (including a non-US address), changes to the shipping method, and closing the popup partway through.

---

## Going to production

- [ ] Remove `debug: true` from `configure()`.
- [ ] Add your production origins to the **Domains allow list**.
- [ ] Calculate tax and shipping on the **server**, not with the hard-coded values in this example.
- [ ] Allow these domains in your Content Security Policy: `cdn.shopify.com`, `cdn.shopifycloud.com`, `shop.app`, `pay.shopify.com`, `*.shopifysvc.com`, and `*.stripe.com`.
- [ ] Don't rely only on the client-side `paymentcomplete` event. Reconcile orders with webhooks (`ORDERS_CREATE`, `ORDER_TRANSACTIONS_CREATE`) and a reconciliation job. See [GraphQL post-payment](https://shopify.dev/docs/api/commerce-components/pay/graphql-post-payment).
- [ ] Build a real `/thank-you` page.
- [ ] Set up monitoring. See [Monitoring and resiliency](https://shopify.dev/docs/api/commerce-components/pay/monitoring).

---

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| The button or login doesn't render | Your origin isn't in the **Domains allow list**, `SHOP_ID` or `CLIENT_ID` is wrong, or you opened the page as a `file://` URL |
| `window.ShopPay` is undefined | The SDK `<script>` didn't load. Check that it's in `<head>` and that your CSP allows it |
| You can't find the Shop Pay configuration | The **Shop** sales channel isn't installed. Reinstall it from the Shopify App Store |
| The checkout window hangs, then errors | A `complete*` call wasn't made within 15 seconds, or your backend endpoint returned an error |

See also Shopify's [Troubleshooting guide](https://shopify.dev/docs/api/commerce-components/pay/troubleshooting-guide).

---

## Resources

- [Shop Pay Wallet overview](https://shopify.dev/docs/api/commerce-components/pay)
- [Shop configuration guide](https://shopify.dev/docs/api/commerce-components/pay/shop-configuration)
- [JavaScript SDK](https://shopify.dev/docs/api/commerce-components/pay/javascript-sdk)
- [Frontend API reference](https://shopify.dev/docs/api/commerce-components/pay/frontend-api-reference)
- [Sessions and events](https://shopify.dev/docs/api/commerce-components/pay/sessions-and-events)
- [Identify Shop users](https://shopify.dev/docs/api/commerce-components/pay/identify-shop-users)
- [Design guidelines](https://shopify.dev/docs/api/commerce-components/pay/design-guidelines)
