# Subscription Management

## Table of Contents
- [Creating Subscriptions](#creating-subscriptions)
- [Subscription Lifecycle](#subscription-lifecycle)
- [Trials](#trials)
- [Plan Changes (Upgrades/Downgrades)](#plan-changes)
- [Cancellation](#cancellation)
- [Pausing Subscriptions](#pausing-subscriptions)
- [Subscription Statuses](#subscription-statuses)

---

## Creating Subscriptions

### Via Checkout (Recommended)

```python
session = stripe.checkout.Session.create(
    customer=stripe_customer_id,
    mode="subscription",
    line_items=[{"price": "price_pro_monthly", "quantity": 1}],
    success_url="https://yourapp.com/success?session_id={CHECKOUT_SESSION_ID}",
    cancel_url="https://yourapp.com/pricing",
)
```

### Via API (Custom UI)

Requires an attached PaymentMethod first:

```python
subscription = stripe.Subscription.create(
    customer=stripe_customer_id,
    items=[{"price": "price_pro_monthly"}],
    default_payment_method=payment_method_id,
    payment_behavior="default_incomplete",  # Requires confirmation
    expand=["latest_invoice.payment_intent"],
)
# Return subscription.latest_invoice.payment_intent.client_secret for frontend confirmation
```

```javascript
const subscription = await stripe.subscriptions.create({
  customer: stripeCustomerId,
  items: [{ price: "price_pro_monthly" }],
  default_payment_method: paymentMethodId,
  payment_behavior: "default_incomplete",
  expand: ["latest_invoice.payment_intent"],
});
```

---

## Subscription Lifecycle

```
Created → trialing (optional) → active → past_due → canceled/unpaid
                                   ↓
                               active (renewed each period)
```

### Key Webhook Events

| Event | Meaning | Action |
|-------|---------|--------|
| `customer.subscription.created` | New subscription | Store subscription ID, set plan tier |
| `customer.subscription.updated` | Plan change, status change | Update plan tier and status |
| `customer.subscription.deleted` | Subscription ended | Revoke access, set to free tier |
| `invoice.paid` | Successful renewal | Confirm active status |
| `invoice.payment_failed` | Renewal charge failed | Notify user, update status |

---

## Trials

### Free Trial with Card Upfront

```python
session = stripe.checkout.Session.create(
    customer=stripe_customer_id,
    mode="subscription",
    line_items=[{"price": "price_pro_monthly", "quantity": 1}],
    subscription_data={"trial_period_days": 14},
    success_url="...",
    cancel_url="...",
)
```

### Free Trial Without Card (API only)

```python
subscription = stripe.Subscription.create(
    customer=stripe_customer_id,
    items=[{"price": "price_pro_monthly"}],
    trial_period_days=14,
    payment_settings={
        "save_default_payment_method": "on_subscription",
    },
    trial_settings={
        "end_behavior": {"missing_payment_method": "cancel"},
    },
)
```

### Trial End Handling

Listen for `customer.subscription.trial_will_end` (fires 3 days before trial ends) to notify users.

---

## Plan Changes

### Upgrade (Immediate, Prorated)

```python
subscription = stripe.Subscription.retrieve(subscription_id)
stripe.Subscription.modify(
    subscription_id,
    items=[{
        "id": subscription["items"]["data"][0]["id"],  # Existing item ID
        "price": "price_enterprise_monthly",            # New price
    }],
    proration_behavior="create_prorations",  # Charge difference immediately
)
```

```javascript
const subscription = await stripe.subscriptions.retrieve(subscriptionId);
await stripe.subscriptions.update(subscriptionId, {
  items: [{
    id: subscription.items.data[0].id,
    price: "price_enterprise_monthly",
  }],
  proration_behavior: "create_prorations",
});
```

### Downgrade (End of Period)

```python
subscription = stripe.Subscription.retrieve(subscription_id)
stripe.Subscription.modify(
    subscription_id,
    items=[{
        "id": subscription["items"]["data"][0]["id"],
        "price": "price_basic_monthly",
    }],
    proration_behavior="none",  # No immediate credit
    # Change takes effect at next billing period via schedule:
)
```

### Schedule Plan Change for End of Period

```python
# Use Subscription Schedules for deferred changes
schedule = stripe.SubscriptionSchedule.create(
    from_subscription=subscription_id,
)
stripe.SubscriptionSchedule.modify(
    schedule.id,
    phases=[
        {
            "items": [{"price": "price_pro_monthly", "quantity": 1}],
            "start_date": schedule.phases[0]["start_date"],
            "end_date": schedule.phases[0]["end_date"],
        },
        {
            "items": [{"price": "price_basic_monthly", "quantity": 1}],
        },
    ],
)
```

---

## Cancellation

### Cancel at End of Period (Recommended)

```python
stripe.Subscription.modify(
    subscription_id,
    cancel_at_period_end=True,
)
# User keeps access until current period ends
# Webhook: customer.subscription.updated (cancel_at_period_end=True)
# Later: customer.subscription.deleted (when period actually ends)
```

### Cancel Immediately

```python
stripe.Subscription.cancel(subscription_id)
# Immediate cancellation, no refund by default
# Webhook: customer.subscription.deleted
```

### Cancel with Prorated Refund

```python
stripe.Subscription.cancel(
    subscription_id,
    prorate=True,  # Refund unused time
)
```

### Reactivate Canceled Subscription

If `cancel_at_period_end=True` and period hasn't ended:

```python
stripe.Subscription.modify(
    subscription_id,
    cancel_at_period_end=False,  # Undo cancellation
)
```

---

## Pausing Subscriptions

```python
# Pause collection (subscription stays active but no charges)
stripe.Subscription.modify(
    subscription_id,
    pause_collection={"behavior": "void"},  # or "keep_as_draft", "mark_uncollectible"
)

# Resume
stripe.Subscription.modify(
    subscription_id,
    pause_collection="",  # Clear pause
)
```

---

## Subscription Statuses

| Status | Meaning | User Access |
|--------|---------|-------------|
| `active` | Paying and current | Full access |
| `trialing` | In trial period | Full access |
| `past_due` | Payment failed, retrying | Grace period (your choice) |
| `canceled` | Subscription ended | Revoke access |
| `unpaid` | All retry attempts failed | Revoke access |
| `incomplete` | Initial payment pending | No access yet |
| `incomplete_expired` | Initial payment expired | No access |
| `paused` | Collection paused | Your choice |

### Status Check Pattern

```python
def get_user_access_level(user):
    """Determine access based on subscription status."""
    active_statuses = {"active", "trialing"}

    if user.subscription_status in active_statuses:
        return user.plan_tier  # "basic", "pro", "enterprise"

    if user.subscription_status == "past_due":
        return user.plan_tier  # Grace period - keep access

    return "free"  # No active subscription
```

---

## Product & Price Setup

### Create via API

```python
# Create Product
product = stripe.Product.create(
    name="Pro Plan",
    description="Full access with priority support",
    metadata={"tier": "pro"},
)

# Create Monthly Price
monthly_price = stripe.Price.create(
    product=product.id,
    unit_amount=2000,      # $20.00
    currency="usd",
    recurring={"interval": "month"},
    lookup_key="pro_monthly",  # Stable reference key
)

# Create Yearly Price (with discount)
yearly_price = stripe.Price.create(
    product=product.id,
    unit_amount=19200,     # $192.00 ($16/month equivalent)
    currency="usd",
    recurring={"interval": "year"},
    lookup_key="pro_yearly",
)
```

### Retrieve Prices by Lookup Key

```python
prices = stripe.Price.list(lookup_keys=["pro_monthly", "pro_yearly"])
```
