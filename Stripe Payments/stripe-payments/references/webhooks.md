# Webhook Handling

## Table of Contents
- [Webhook Endpoint Setup](#webhook-endpoint-setup)
- [Signature Verification](#signature-verification)
- [Essential Events](#essential-events)
- [Event Handler Patterns](#event-handler-patterns)
- [Idempotency and Reliability](#idempotency-and-reliability)
- [Testing Webhooks](#testing-webhooks)

---

## Webhook Endpoint Setup

### Python (FastAPI)

```python
from fastapi import FastAPI, Request, HTTPException
import stripe

app = FastAPI()
stripe.api_key = os.environ["STRIPE_SECRET_KEY"]
WEBHOOK_SECRET = os.environ["STRIPE_WEBHOOK_SECRET"]

@app.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    payload = await request.body()
    sig_header = request.headers.get("stripe-signature")

    try:
        event = stripe.Webhook.construct_event(payload, sig_header, WEBHOOK_SECRET)
    except ValueError:
        raise HTTPException(400, "Invalid payload")
    except stripe.error.SignatureVerificationError:
        raise HTTPException(400, "Invalid signature")

    # Route to handler
    await handle_stripe_event(event)

    return {"status": "ok"}
```

### Python (Flask)

```python
from flask import Flask, request, jsonify
import stripe

app = Flask(__name__)

@app.route("/webhooks/stripe", methods=["POST"])
def stripe_webhook():
    payload = request.get_data()
    sig_header = request.headers.get("Stripe-Signature")

    try:
        event = stripe.Webhook.construct_event(payload, sig_header, WEBHOOK_SECRET)
    except (ValueError, stripe.error.SignatureVerificationError):
        return "Invalid", 400

    handle_stripe_event(event)
    return jsonify({"status": "ok"}), 200
```

### Node.js (Express)

```javascript
const express = require("express");
const stripe = require("stripe")(process.env.STRIPE_SECRET_KEY);

// IMPORTANT: Use raw body parser for webhook endpoint
app.post(
  "/webhooks/stripe",
  express.raw({ type: "application/json" }),
  async (req, res) => {
    const sig = req.headers["stripe-signature"];

    let event;
    try {
      event = stripe.webhooks.constructEvent(
        req.body,
        sig,
        process.env.STRIPE_WEBHOOK_SECRET
      );
    } catch (err) {
      return res.status(400).send(`Webhook Error: ${err.message}`);
    }

    await handleStripeEvent(event);
    res.json({ received: true });
  }
);
```

### Next.js API Route

```javascript
// app/api/webhooks/stripe/route.js (App Router)
import { NextResponse } from "next/server";
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

export async function POST(req) {
  const body = await req.text();
  const sig = req.headers.get("stripe-signature");

  let event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET
    );
  } catch (err) {
    return NextResponse.json({ error: err.message }, { status: 400 });
  }

  await handleStripeEvent(event);
  return NextResponse.json({ received: true });
}
```

**Critical**: Webhook endpoints must receive the **raw request body** (not parsed JSON) for signature verification. In Express, use `express.raw()`. In Next.js, use `req.text()`. In FastAPI, use `request.body()`.

---

## Signature Verification

**Always verify webhook signatures.** Without verification, anyone can send fake events to your endpoint.

```python
# Verification happens via stripe.Webhook.construct_event()
# It checks:
# 1. Signature matches payload + secret
# 2. Timestamp is recent (prevents replay attacks)
event = stripe.Webhook.construct_event(
    payload,       # Raw request body (bytes)
    sig_header,    # Stripe-Signature header value
    webhook_secret # whsec_... from Stripe Dashboard
)
```

---

## Essential Events

### Payments

| Event | When | Action |
|-------|------|--------|
| `checkout.session.completed` | Customer completes Checkout | Fulfill order, provision access |
| `payment_intent.succeeded` | Payment confirmed | Record successful payment |
| `payment_intent.payment_failed` | Payment failed | Notify customer |

### Subscriptions

| Event | When | Action |
|-------|------|--------|
| `customer.subscription.created` | New subscription | Store sub ID, set plan tier |
| `customer.subscription.updated` | Plan change, status change | Update tier and status |
| `customer.subscription.deleted` | Subscription canceled/expired | Revoke access |
| `customer.subscription.trial_will_end` | 3 days before trial ends | Notify user |

### Billing

| Event | When | Action |
|-------|------|--------|
| `invoice.paid` | Invoice payment succeeded | Confirm access, provision credits |
| `invoice.payment_failed` | Invoice payment failed | Notify user, set past_due |
| `invoice.finalized` | Invoice ready for payment | Optional logging |

### Customer

| Event | When | Action |
|-------|------|--------|
| `customer.updated` | Customer details changed | Sync to your DB |
| `payment_method.attached` | New payment method added | Optional logging |

---

## Event Handler Patterns

### Python

```python
async def handle_stripe_event(event):
    event_type = event["type"]
    data = event["data"]["object"]

    handlers = {
        "checkout.session.completed": handle_checkout_completed,
        "customer.subscription.created": handle_subscription_created,
        "customer.subscription.updated": handle_subscription_updated,
        "customer.subscription.deleted": handle_subscription_deleted,
        "invoice.paid": handle_invoice_paid,
        "invoice.payment_failed": handle_invoice_payment_failed,
    }

    handler = handlers.get(event_type)
    if handler:
        await handler(data)


async def handle_checkout_completed(session):
    customer_id = session["customer"]
    mode = session["mode"]
    metadata = session.get("metadata", {})

    if mode == "subscription":
        subscription_id = session["subscription"]
        # Update user's subscription in your DB
        await db.update_user_subscription(
            stripe_customer_id=customer_id,
            stripe_subscription_id=subscription_id,
            status="active",
        )

    elif mode == "payment":
        # One-time payment - fulfill order
        payment_intent_id = session["payment_intent"]
        await fulfill_order(metadata.get("order_id"), payment_intent_id)


async def handle_subscription_updated(subscription):
    customer_id = subscription["customer"]
    status = subscription["status"]
    price_id = subscription["items"]["data"][0]["price"]["id"]

    # Map price to plan tier
    tier = price_to_tier(price_id)

    await db.update_user_subscription(
        stripe_customer_id=customer_id,
        subscription_status=status,
        plan_tier=tier,
        cancel_at_period_end=subscription.get("cancel_at_period_end", False),
    )


async def handle_subscription_deleted(subscription):
    customer_id = subscription["customer"]
    await db.update_user_subscription(
        stripe_customer_id=customer_id,
        subscription_status="canceled",
        plan_tier="free",
    )


async def handle_invoice_paid(invoice):
    customer_id = invoice["customer"]
    subscription_id = invoice.get("subscription")

    if subscription_id:
        # Subscription renewal - confirm active
        await db.update_user_subscription(
            stripe_customer_id=customer_id,
            subscription_status="active",
        )

    # If using credits system, provision credits here
    metadata = invoice.get("metadata", {})
    credits = metadata.get("credits")
    if credits:
        await db.add_credits(stripe_customer_id=customer_id, amount=int(credits))


async def handle_invoice_payment_failed(invoice):
    customer_id = invoice["customer"]
    await db.update_user_subscription(
        stripe_customer_id=customer_id,
        subscription_status="past_due",
    )
    await send_payment_failed_email(customer_id)
```

### Node.js

```javascript
async function handleStripeEvent(event) {
  const data = event.data.object;

  switch (event.type) {
    case "checkout.session.completed":
      await handleCheckoutCompleted(data);
      break;
    case "customer.subscription.created":
    case "customer.subscription.updated":
      await handleSubscriptionChange(data);
      break;
    case "customer.subscription.deleted":
      await handleSubscriptionDeleted(data);
      break;
    case "invoice.paid":
      await handleInvoicePaid(data);
      break;
    case "invoice.payment_failed":
      await handlePaymentFailed(data);
      break;
  }
}
```

---

## Idempotency and Reliability

### Stripe May Send Events Multiple Times

Always handle events idempotently:

```python
async def handle_checkout_completed(session):
    session_id = session["id"]

    # Check if already processed
    if await db.is_event_processed(session_id):
        return  # Already handled

    # Process the event
    await provision_access(session)

    # Mark as processed
    await db.mark_event_processed(session_id)
```

### Event Ordering

Events may arrive out of order. Always fetch the latest state from Stripe when needed:

```python
async def handle_subscription_updated(subscription):
    # Fetch latest state from Stripe (not the event payload)
    latest = stripe.Subscription.retrieve(subscription["id"])
    # Use latest.status, not subscription["status"]
```

### Return 200 Quickly

Stripe expects a 2xx response within 20 seconds. Do heavy processing in the background:

```python
@app.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    event = verify_and_construct(request)

    # Queue for background processing
    background_tasks.add_task(handle_stripe_event, event)

    return {"status": "ok"}  # Return immediately
```

---

## Testing Webhooks

### Stripe CLI (Local Development)

```bash
# Install Stripe CLI, then:
stripe listen --forward-to localhost:8000/webhooks/stripe

# Trigger specific events:
stripe trigger checkout.session.completed
stripe trigger invoice.paid
stripe trigger customer.subscription.updated
```

### Test Webhook Signing

```python
# Generate test event
payload = json.dumps({"type": "checkout.session.completed", ...})
timestamp = int(time.time())
signed_payload = f"{timestamp}.{payload}"
signature = hmac.new(webhook_secret.encode(), signed_payload.encode(), hashlib.sha256).hexdigest()
sig_header = f"t={timestamp},v1={signature}"
```

### Dashboard Setup

1. Go to Stripe Dashboard > Developers > Webhooks
2. Add endpoint URL (your production webhook URL)
3. Select events to listen for
4. Copy the signing secret (`whsec_...`) to your environment
