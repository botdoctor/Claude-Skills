---
name: stripe-payments
description: |
  Integrate Stripe payments into any application. Framework-agnostic patterns for Node.js, Python, and other backends. Use this skill when:
  (1) Setting up Stripe Checkout, Payment Intents, or embedded payment forms
  (2) Implementing subscription billing with plans, trials, upgrades, or downgrades
  (3) Building usage-based or metered billing (pay-per-use, credit/token systems)
  (4) Handling Stripe webhooks and event processing
  (5) Setting up Stripe Customer Portal for self-service billing
  (6) Managing invoices, refunds, or payment disputes
  (7) Implementing a credit/token purchase and consumption system
  (8) Configuring Stripe API keys, webhook secrets, or test mode
  (9) Any task involving Stripe payment processing, billing, or subscription management
---

# Stripe Payments Integration

## Integration Decision Tree

```
What payment flow do you need?
├── Redirect to Stripe-hosted page → Checkout Sessions
├── Embedded form in your UI       → Payment Intents + Stripe Elements
├── Recurring billing               → Subscriptions (via Checkout or API)
├── Pay-per-use / metered           → Usage-Based Billing
├── Credit/token system             → Credits System (custom + Stripe)
└── Self-service billing management → Customer Portal
```

## Core Concepts

### API Authentication

```
# Environment variables (NEVER hardcode)
STRIPE_SECRET_KEY=sk_test_...       # Server-side only
STRIPE_PUBLISHABLE_KEY=pk_test_...  # Client-side safe
STRIPE_WEBHOOK_SECRET=whsec_...     # Webhook signature verification
```

- **Secret key** (`sk_`): Server-side only. Never expose to client.
- **Publishable key** (`pk_`): Safe for client-side. Used in Stripe.js/Elements.
- **Webhook secret** (`whsec_`): Verify webhook signatures. Per-endpoint.

### Key Stripe Objects

| Object | Purpose | Created Via |
|--------|---------|-------------|
| `Customer` | Represents a user. Stores payment methods, subscriptions. | `stripe.customers.create()` |
| `Product` | What you sell (e.g., "Pro Plan"). | Dashboard or API |
| `Price` | How much and how often (e.g., $20/month). | Dashboard or API |
| `Checkout Session` | Hosted payment page. | `stripe.checkout.sessions.create()` |
| `PaymentIntent` | Tracks a single payment lifecycle. | `stripe.paymentIntents.create()` |
| `Subscription` | Recurring billing relationship. | `stripe.subscriptions.create()` |
| `Invoice` | Bill for subscription period or one-off. | Auto-generated or API |
| `Webhook Event` | Async notification of state changes. | Stripe sends to your endpoint |

### Stripe Object Relationships

```
Customer
├── PaymentMethod (card, bank, etc.)
├── Subscription
│   ├── Price → Product
│   ├── Invoice (per billing period)
│   │   └── PaymentIntent (charge attempt)
│   └── Usage Records (if metered)
└── Checkout Session (creates Subscription or PaymentIntent)
```

## Implementation Workflow

### 1. Initial Setup

1. Create Stripe account and get API keys
2. Set environment variables (`STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_WEBHOOK_SECRET`)
3. Install Stripe SDK (`stripe` for Python, `stripe` npm package for Node.js)
4. Create Products and Prices in Stripe Dashboard (or via API)

### 2. Create Customers

Always create a Stripe Customer for each user. Store `stripe_customer_id` in your database:

```python
# Python
customer = stripe.Customer.create(
    email=user.email,
    metadata={"user_id": str(user.id)}
)
# Save customer.id to your database
```

```javascript
// Node.js
const customer = await stripe.customers.create({
  email: user.email,
  metadata: { user_id: user.id },
});
// Save customer.id to your database
```

### 3. Choose Integration Path

| Need | Reference |
|------|-----------|
| Accept payments (one-time or first subscription) | [checkout-and-payments.md](references/checkout-and-payments.md) |
| Recurring billing with plan management | [subscriptions.md](references/subscriptions.md) |
| Handle Stripe events server-side | [webhooks.md](references/webhooks.md) |
| Metered billing or credit/token systems | [usage-and-credits.md](references/usage-and-credits.md) |
| Let users manage their own billing | [customer-portal.md](references/customer-portal.md) |
| Security, testing, and error handling | [security-and-testing.md](references/security-and-testing.md) |

### 4. Webhook Setup (Required)

Webhooks are not optional. Stripe is async - payments confirm, subscriptions renew, and cards fail asynchronously. Always implement webhook handling. See [webhooks.md](references/webhooks.md).

## Common Patterns

### Store Stripe IDs in Your Database

```sql
-- Users table (add these columns)
stripe_customer_id    TEXT UNIQUE
stripe_subscription_id TEXT
subscription_status    TEXT  -- 'active', 'past_due', 'canceled', etc.
plan_tier              TEXT  -- 'free', 'basic', 'pro'
credits_balance        INTEGER DEFAULT 0
```

### Check Subscription Status (Middleware Pattern)

```python
# Python pseudocode
def require_active_subscription(user):
    if user.subscription_status not in ("active", "trialing"):
        raise HTTPException(403, "Active subscription required")
```

```javascript
// Node.js pseudocode
function requireSubscription(req, res, next) {
  if (!["active", "trialing"].includes(req.user.subscriptionStatus)) {
    return res.status(403).json({ error: "Active subscription required" });
  }
  next();
}
```

### Idempotency

Always use idempotency keys for create operations to prevent duplicate charges:

```python
stripe.PaymentIntent.create(
    amount=2000,
    currency="usd",
    idempotency_key=f"payment_{order_id}"
)
```

## Error Handling

```python
try:
    result = stripe.SomeResource.create(...)
except stripe.error.CardError as e:
    # Card declined - show user-friendly message
    handle_card_error(e)
except stripe.error.RateLimitError:
    # Too many requests - retry with backoff
    retry_with_backoff()
except stripe.error.InvalidRequestError as e:
    # Invalid parameters - log and fix
    log_error(e)
except stripe.error.AuthenticationError:
    # API key issue - alert ops
    alert_ops("Stripe auth failed")
except stripe.error.StripeError as e:
    # Generic Stripe error
    handle_generic_error(e)
```
