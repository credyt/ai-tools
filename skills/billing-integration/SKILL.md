---
name: billing-integration
description: Wire Credyt billing into your application code. Adds customer creation to registration, usage event tracking, balance checks, cost tracking, and billing portal links. Run after /credyt:billing-setup and /credyt:billing-verification. Use when the user wants to add Credyt to their app, integrate billing, send events from code, show balances, or add a billing page.
---

# Credyt Integrate

Help the user wire Credyt into their application code. This skill works with the user's actual codebase — reading their existing code and adding Credyt integration in the right places.

The full integration guide is at [docs.credyt.ai/ai-integration.md](https://docs.credyt.ai/ai-integration.md). SDK references with code examples are at [docs.credyt.ai/sdk](https://docs.credyt.ai/sdk).

## Understand the codebase first

Before writing any code, understand what the user has:

> "Let me look at your project to understand your stack and where billing should plug in."

Check for:
- Language and framework (Node/Express, Next.js, Python/FastAPI, etc.)
- Authentication setup (how users sign up and log in)
- Where the billable activities happen in their code
- Existing environment variable patterns

## SDK or direct HTTP?

Based on the user's stack, recommend the appropriate integration approach. Default to the SDK — it handles auth, serialisation, and error shaping for you:

- **TypeScript/Node.js** → `@credyt/api-client` (`npm install @credyt/api-client`)
- **Python** → `credyt-api` (`pip install credyt-api`)
- **Other language or preference for raw HTTP** → direct API calls with the `X-CREDYT-API-KEY` header

Ask if they'd prefer to use the SDK or hand-roll the HTTP calls. For TypeScript and Python users, lead with the SDK. The full SDK reference with examples for all three approaches is at [docs.credyt.ai/sdk](https://docs.credyt.ai/sdk).

## Integration areas

Walk through each area. Not all will apply to every user — ask which ones they need.

### 1. API key setup

The Credyt API key must be stored securely on the server side — never in code that runs in the browser.

Add `CREDYT_API_KEY` to their environment variables alongside their other secrets, then initialise the client once and reuse it across requests.

**TypeScript:**
```typescript
import { CredytApiClient } from "@credyt/api-client";
const client = new CredytApiClient({ key: process.env.CREDYT_API_KEY! });
```

**Python:**
```python
from corehttp.credentials import ServiceKeyCredential
from credytapi import CredytApiClient

client = CredytApiClient(
    credential=ServiceKeyCredential(key=os.getenv("CREDYT_API_KEY")),
)
```

**Direct HTTP:** Attach `X-CREDYT-API-KEY: <key>` to every request. Show them how to create a helper function or configured HTTP client instance that sets this header automatically.

### 2. Customer creation (registration flow)

When a new user signs up in their app, create a matching Credyt customer. Find their registration/signup handler and add customer creation after successful account creation.

Key points:
- Use `externalId` / `external_id` to link the Credyt customer to their app's user ID
- Subscribe the customer to the relevant products during creation
- Store the Credyt customer ID in their database alongside the user record
- Handle the case where the customer already exists (409/422 — look up by external_id instead)

**SDK method:** `client.customers.create({ name, externalId, email, subscriptions })` (TS) or `client.customers.create(body={...})` (Python).

**Recurring fixed fees — pending subscriptions**

If the product uses a recurring fixed fee (e.g. $20/month), the customer must pay upfront before their subscription activates. In this case the API returns a `pending` status rather than activating immediately.

Set `returnUrl` / `return_url`, `failureUrl` / `failure_url`, and `redirectTo` / `redirect_to` on the subscription so Credyt knows where to send the customer after payment — before you ever redirect them anywhere:

- `returnUrl` — where to send the customer after successful payment (e.g. `https://yourapp.com/account`)
- `failureUrl` — where to send them if payment fails (e.g. `https://yourapp.com/callbacks/payment-failed`)
- `redirectTo` — set to `"return_url"` so the customer lands back on your site instead of staying in the Credyt billing portal (this is the default, but set it explicitly)

When the response status is `pending`:
- Check the `required_actions` array for an action with `type: "payment"` and extract its `redirect_url`
- Redirect the customer to that URL — Credyt will handle the payment form and route them to your `returnUrl` or `failureUrl` automatically
- Do not activate the user's account yet — store it as pending in your database until payment is confirmed
- If the redirect link expires before the customer completes payment, fetch the customer by their Credyt ID to get a refreshed link

Once the customer pays, Credyt fires a `subscription.activated` webhook. Listen for this event on your backend and use it to activate the user's account.

### 3. Usage event tracking

Find where the billable activities happen in their code and add event submission after each one. Each event needs:

- A unique ID (UUID) so the same event can't be billed twice
- The correct `eventType` / `event_type` matching the product configuration
- A timestamp of when it happened (`occurredAt` / `occurred_at`)
- Any data fields needed for pricing (volume fields, dimensions)

**SDK method:** `client.events.sendUsage(customerId, { events })` (TS) or `client.events.send_usage(body={ customer_id, events })` (Python).

For **volume-based** products, the event data must include the volume field (e.g., `total_tokens: 1500`).
For **dimensional** products, include the dimension values (e.g., `model: "gpt-4"`).

### 4. Cost tracking (if they set up vendors)

If the user set up vendors in `/credyt:billing-setup`, add cost data to usage events. Each event can include a `costs` array with the vendor ID, the amount it cost, and the currency.

This is typically added right after the billable action completes, when the cost is known (e.g., after receiving the response from an AI API that includes token counts).

> "Even if you're not charging users yet, attaching costs to every event lets Credyt calculate your unit economics so you can make pricing decisions based on real data."

### 5. Balance checks (pre-action gating)

Before expensive operations, check the customer's wallet balance. If insufficient, block the action and prompt the user to top up.

**SDK method:** `client.wallets.customerWalletOps.getCustomerWallet(customerId)` to fetch the full wallet, or `client.wallets.customerWalletOps.getAccount(customerId, "accountName:ASSET")` to check a specific account balance.

Find where billable actions are initiated (API routes, button handlers, etc.) and add a balance check before the action runs. Return a clear message if the balance is too low.

Show them how to estimate the cost of the upcoming action and compare it against the available balance.

### 6. Billing portal / top-up UI

Help users add funds through Credyt's billing portal. This is the simplest way to handle payments — Credyt hosts the page, handles Stripe, and redirects back to their app.

Add a "Billing" or "Add funds" link/button in their app's settings or account page. When clicked, the backend creates a billing portal session and redirects the user to the URL.

**SDK method:** `client.billingPortal.createPortalSession({ customerId, returnUrl, failureUrl })` (TS) or the equivalent Python method.

Key points:
- Portal sessions expire after 10 minutes
- Set a `returnUrl` / `return_url` for where to send users after they're done
- Set a `failureUrl` / `failure_url` for payment failures

### 7. Balance display

Show the user's current balance in the app UI. Fetch from the wallet endpoint and display the available amount.

**SDK method:** `client.wallets.customerWalletOps.getCustomerWallet(customerId)`.

Consider where this fits in their app — sidebar, header, account page — and add it there.

## Implementation approach

Don't dump all the code at once. Work through each area one at a time:

1. Start with API key setup and customer creation — these are foundational
2. Then add usage event tracking — this is the core billing integration
3. Add balance checks to gate expensive operations
4. Add billing portal and balance display for the user-facing pieces
5. Add cost tracking last if applicable

After each piece, suggest they test it:

> "Try creating a new account in your app and check the Credyt dashboard — you should see a new customer appear. Then we'll move on to usage tracking."

## Reference

For detailed code examples, error handling patterns, and advanced topics (hybrid billing, refunds, auto top-up), point the user to:

- **SDK reference** (TypeScript & Python): [docs.credyt.ai/sdk](https://docs.credyt.ai/sdk)
- **Full integration guide**: [docs.credyt.ai/ai-integration.md](https://docs.credyt.ai/ai-integration.md)
- **API reference**: [docs.credyt.ai](https://docs.credyt.ai)
- **Examples**: [github.com/credyt/learn](https://github.com/credyt/learn)
