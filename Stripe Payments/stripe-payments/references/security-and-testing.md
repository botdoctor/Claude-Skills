# Security, Testing & Error Handling

## Table of Contents
- [API Key Security](#api-key-security)
- [Webhook Security](#webhook-security)
- [PCI Compliance](#pci-compliance)
- [Error Handling](#error-handling)
- [Testing](#testing)
- [Going Live Checklist](#going-live-checklist)

---

## API Key Security

### Key Types

| Key Prefix | Type | Visibility | Use |
|------------|------|------------|-----|
| `sk_test_` | Test Secret | Server only | Development/testing |
| `sk_live_` | Live Secret | Server only | Production |
| `pk_test_` | Test Publishable | Client safe | Frontend (test) |
| `pk_live_` | Live Publishable | Client safe | Frontend (production) |
| `rk_test_` / `rk_live_` | Restricted | Server only | Limited permissions |
| `whsec_` | Webhook Secret | Server only | Signature verification |

### Rules

1. **Never hardcode keys** - Use environment variables
2. **Never expose secret keys** - Server-side only, never in client code, git, or logs
3. **Use restricted keys** when possible - Limit to only needed permissions
4. **Rotate keys** if compromised - Dashboard > Developers > API Keys > Roll Key
5. **Separate test and live keys** - Use environment-specific config

### Environment Setup

```bash
# .env (never commit this file)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# .env.production
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

```python
# Python
import os
import stripe

stripe.api_key = os.environ["STRIPE_SECRET_KEY"]
```

```javascript
// Node.js
const stripe = require("stripe")(process.env.STRIPE_SECRET_KEY);
```

### Restricted Keys

Create restricted keys with minimal permissions:

```
Dashboard > Developers > API Keys > Create restricted key
```

Example: A key that can only create Checkout Sessions and read Customers.

---

## Webhook Security

### Always Verify Signatures

```python
# CORRECT - verify signature
event = stripe.Webhook.construct_event(payload, sig_header, webhook_secret)

# WRONG - trusting raw input
event = json.loads(payload)  # NEVER do this
```

### Protect Webhook Endpoint

- Use HTTPS only
- Verify `Stripe-Signature` header on every request
- Return 200 quickly, process asynchronously
- Store processed event IDs to prevent replay
- Use separate webhook secrets per endpoint

### IP Allowlisting (Optional)

Stripe webhook IPs are published at: https://stripe.com/docs/ips

---

## PCI Compliance

### Using Checkout or Elements (SAQ A)

If you use Stripe Checkout (hosted) or Stripe Elements (embedded), card data never touches your server. This is the easiest PCI compliance path:

- **DO**: Use Checkout Sessions or Payment Intents with Elements
- **DO**: Use Stripe.js to tokenize cards client-side
- **DON'T**: Collect card numbers in your own form fields
- **DON'T**: Log or store raw card data anywhere
- **DON'T**: Pass card numbers through your server

### Content Security Policy

If using Stripe.js, allow these domains in CSP:

```
script-src: https://js.stripe.com
frame-src: https://js.stripe.com https://hooks.stripe.com
connect-src: https://api.stripe.com
```

---

## Error Handling

### Stripe Error Types

```python
import stripe

try:
    result = stripe.SomeResource.create(...)

except stripe.error.CardError as e:
    # Card was declined
    # e.code: "card_declined", "expired_card", "incorrect_cvc", etc.
    status = e.http_status  # 402
    body = e.json_body
    err = body.get("error", {})
    code = err.get("code")
    decline_code = err.get("decline_code")
    message = err.get("message")
    # Show user-friendly message

except stripe.error.RateLimitError:
    # Too many requests to Stripe (429)
    # Retry with exponential backoff

except stripe.error.InvalidRequestError as e:
    # Invalid parameters sent to Stripe (400/404)
    # Fix the request, don't retry
    # e.g., invalid customer ID, missing required field

except stripe.error.AuthenticationError:
    # API key is invalid (401)
    # Check environment configuration

except stripe.error.APIConnectionError:
    # Network communication with Stripe failed
    # Retry with backoff

except stripe.error.StripeError as e:
    # Generic Stripe error
    # Log and handle gracefully
```

### Node.js Error Handling

```javascript
try {
  const result = await stripe.someResource.create({...});
} catch (err) {
  switch (err.type) {
    case "StripeCardError":
      // Card declined
      console.log(`Decline code: ${err.decline_code}`);
      break;
    case "StripeRateLimitError":
      // Retry
      break;
    case "StripeInvalidRequestError":
      // Bad request
      break;
    case "StripeAuthenticationError":
      // Bad API key
      break;
    case "StripeAPIError":
      // Stripe server error, retry
      break;
    case "StripeConnectionError":
      // Network error, retry
      break;
  }
}
```

### User-Friendly Card Error Messages

```python
CARD_ERROR_MESSAGES = {
    "card_declined": "Your card was declined. Please try a different card.",
    "expired_card": "Your card has expired. Please update your card.",
    "incorrect_cvc": "The CVC code is incorrect. Please check and try again.",
    "processing_error": "A processing error occurred. Please try again.",
    "insufficient_funds": "Insufficient funds. Please try a different card.",
    "incorrect_number": "The card number is incorrect.",
}

def get_user_message(error_code):
    return CARD_ERROR_MESSAGES.get(
        error_code,
        "Payment failed. Please try again or use a different payment method."
    )
```

### Retry with Backoff

```python
import asyncio

async def stripe_with_retry(func, *args, max_retries=3, **kwargs):
    """Call Stripe API with exponential backoff on retryable errors."""
    for attempt in range(max_retries + 1):
        try:
            return await func(*args, **kwargs)
        except (stripe.error.RateLimitError, stripe.error.APIConnectionError):
            if attempt == max_retries:
                raise
            wait = (2 ** attempt) + (random.random() * 0.5)
            await asyncio.sleep(wait)
        except stripe.error.StripeError:
            raise  # Don't retry non-retryable errors
```

---

## Testing

### Test Mode

All `sk_test_` and `pk_test_` keys operate in test mode. No real charges.

### Test Card Numbers

| Card | Number | Behavior |
|------|--------|----------|
| Visa (success) | `4242 4242 4242 4242` | Always succeeds |
| Visa (decline) | `4000 0000 0000 0002` | Always declined |
| 3D Secure required | `4000 0025 0000 3155` | Requires authentication |
| Insufficient funds | `4000 0000 0000 9995` | Decline: insufficient funds |
| Expired card | `4000 0000 0000 0069` | Decline: expired card |
| CVC check fail | `4000 0000 0000 0127` | Decline: incorrect CVC |
| Attach to Customer | `4242 4242 4242 4242` | Works for subscriptions |

Use any future expiry date and any 3-digit CVC.

### Test Webhook Events (Stripe CLI)

```bash
# Listen for events locally
stripe listen --forward-to localhost:8000/webhooks/stripe

# Trigger specific events
stripe trigger checkout.session.completed
stripe trigger customer.subscription.created
stripe trigger customer.subscription.updated
stripe trigger invoice.paid
stripe trigger invoice.payment_failed
stripe trigger customer.subscription.deleted

# Trigger with specific data
stripe trigger payment_intent.succeeded --add payment_intent:metadata[order_id]=test123
```

### Automated Tests

```python
# Python test example
import stripe
import pytest

stripe.api_key = "sk_test_..."

@pytest.fixture
def test_customer():
    customer = stripe.Customer.create(
        email="test@example.com",
        metadata={"test": "true"},
    )
    yield customer
    stripe.Customer.delete(customer.id)

def test_create_checkout_session(test_customer):
    session = stripe.checkout.Session.create(
        customer=test_customer.id,
        mode="payment",
        line_items=[{
            "price_data": {
                "currency": "usd",
                "unit_amount": 1000,
                "product_data": {"name": "Test Product"},
            },
            "quantity": 1,
        }],
        success_url="https://example.com/success",
        cancel_url="https://example.com/cancel",
    )
    assert session.url is not None
    assert session.payment_status == "unpaid"

def test_create_subscription(test_customer):
    subscription = stripe.Subscription.create(
        customer=test_customer.id,
        items=[{"price": "price_test_monthly"}],
        trial_period_days=7,
    )
    assert subscription.status == "trialing"
    stripe.Subscription.cancel(subscription.id)
```

### Mock Webhook Events in Tests

```python
def test_webhook_handler():
    """Test webhook handler with mock event."""
    mock_event = {
        "type": "checkout.session.completed",
        "data": {
            "object": {
                "id": "cs_test_123",
                "customer": "cus_test_456",
                "mode": "subscription",
                "subscription": "sub_test_789",
                "payment_status": "paid",
                "metadata": {"user_id": "user_123"},
            }
        }
    }
    # Call handler directly (skip signature verification in tests)
    handle_stripe_event(mock_event)
    # Assert database was updated
```

---

## Going Live Checklist

### Before Going Live

- [ ] Switch to live API keys (`sk_live_`, `pk_live_`)
- [ ] Set up live webhook endpoint in Stripe Dashboard
- [ ] Use live webhook signing secret (`whsec_...`)
- [ ] Create Products and Prices in live mode (or copy from test)
- [ ] Verify webhook endpoint returns 200 for all subscribed events
- [ ] Test full purchase flow with real card (refund after)
- [ ] Enable HTTPS on all endpoints
- [ ] Set up error alerting for webhook failures
- [ ] Configure dunning (retry) settings in Dashboard > Settings > Billing
- [ ] Set up email receipts in Dashboard > Settings > Emails

### Environment Variables (Production)

```bash
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...  # Live endpoint secret
```

### Monitoring

- Dashboard > Developers > Webhooks: Check for failed deliveries
- Dashboard > Developers > Logs: API request logs
- Set up alerts for: failed webhooks, elevated error rates, payment failures
