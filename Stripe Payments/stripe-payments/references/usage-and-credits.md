# Usage-Based Billing & Credits System

## Table of Contents
- [Metered Subscriptions](#metered-subscriptions)
- [Credit/Token System](#credittoken-system)
- [Hybrid: Subscription + Credits](#hybrid-subscription--credits)
- [Usage Tracking Patterns](#usage-tracking-patterns)

---

## Metered Subscriptions

Charge customers based on actual usage each billing period using Stripe's built-in metering.

### Setup: Create Metered Price

```python
product = stripe.Product.create(name="API Requests")

price = stripe.Price.create(
    product=product.id,
    currency="usd",
    recurring={
        "interval": "month",
        "usage_type": "metered",           # Metered billing
        "aggregate_usage": "sum",          # Sum all usage in period
    },
    unit_amount=1,  # $0.01 per unit (1 cent)
    # Or use tiers:
    # billing_scheme="tiered",
    # tiers_mode="graduated",
    # tiers=[
    #     {"up_to": 1000, "unit_amount": 10},    # First 1000: $0.10/unit
    #     {"up_to": 10000, "unit_amount": 5},     # Next 9000: $0.05/unit
    #     {"up_to": "inf", "unit_amount": 2},     # Above 10000: $0.02/unit
    # ],
)
```

### Create Metered Subscription

```python
subscription = stripe.Subscription.create(
    customer=stripe_customer_id,
    items=[{"price": "price_metered_api"}],
)
# Save subscription.items.data[0].id as subscription_item_id
```

### Report Usage

```python
# Report usage throughout the billing period
stripe.SubscriptionItem.create_usage_record(
    subscription_item_id,
    quantity=100,                          # 100 units consumed
    timestamp=int(time.time()),
    action="increment",                    # Add to total (vs "set")
    idempotency_key=f"usage_{request_id}", # Prevent duplicates
)
```

```javascript
await stripe.subscriptionItems.createUsageRecord(
  subscriptionItemId,
  {
    quantity: 100,
    timestamp: Math.floor(Date.now() / 1000),
    action: "increment",
  },
  { idempotencyKey: `usage_${requestId}` }
);
```

### Retrieve Current Usage

```python
summary = stripe.SubscriptionItem.list_usage_record_summaries(
    subscription_item_id,
    limit=1,
)
current_usage = summary.data[0].total_usage
```

---

## Credit/Token System

A credit system where users purchase credits upfront and consume them. More flexible than metered billing - you control the experience entirely.

### Database Schema

```sql
-- Add to users table
credits_balance      INTEGER DEFAULT 0
lifetime_credits     INTEGER DEFAULT 0  -- Total ever purchased

-- Credit transactions log
CREATE TABLE credit_transactions (
    id                SERIAL PRIMARY KEY,
    user_id           TEXT NOT NULL,
    amount            INTEGER NOT NULL,     -- Positive = add, negative = deduct
    balance_after     INTEGER NOT NULL,
    transaction_type  TEXT NOT NULL,         -- 'purchase', 'usage', 'refund', 'bonus', 'expire'
    description       TEXT,
    stripe_payment_id TEXT,                  -- Links to Stripe payment
    metadata          JSONB DEFAULT '{}',
    created_at        TIMESTAMP DEFAULT NOW()
);
```

### Credit Packages (Products in Stripe)

```python
# Create credit packages in Stripe
packages = [
    {"name": "100 Credits", "credits": 100, "amount": 999},      # $9.99
    {"name": "500 Credits", "credits": 500, "amount": 3999},      # $39.99
    {"name": "2000 Credits", "credits": 2000, "amount": 9999},    # $99.99
]

for pkg in packages:
    product = stripe.Product.create(
        name=pkg["name"],
        metadata={"credits": str(pkg["credits"])},
    )
    stripe.Price.create(
        product=product.id,
        unit_amount=pkg["amount"],
        currency="usd",
    )
```

### Purchase Credits (Checkout)

```python
def create_credit_purchase_session(user, price_id, credits_amount):
    session = stripe.checkout.Session.create(
        customer=user.stripe_customer_id,
        mode="payment",
        line_items=[{"price": price_id, "quantity": 1}],
        success_url="https://yourapp.com/credits/success?session_id={CHECKOUT_SESSION_ID}",
        cancel_url="https://yourapp.com/credits",
        metadata={
            "user_id": str(user.id),
            "credits": str(credits_amount),
            "type": "credit_purchase",
        },
    )
    return session.url
```

### Provision Credits (Webhook)

```python
async def handle_checkout_completed(session):
    metadata = session.get("metadata", {})

    if metadata.get("type") == "credit_purchase":
        user_id = metadata["user_id"]
        credits = int(metadata["credits"])
        payment_intent = session["payment_intent"]

        # Idempotency: check if already provisioned
        if await db.is_payment_processed(payment_intent):
            return

        # Add credits
        await add_credits(
            user_id=user_id,
            amount=credits,
            transaction_type="purchase",
            description=f"Purchased {credits} credits",
            stripe_payment_id=payment_intent,
        )
```

### Credit Operations

```python
async def add_credits(user_id, amount, transaction_type, description, stripe_payment_id=None):
    """Add credits to user balance."""
    # Atomic update
    new_balance = await db.execute(
        """UPDATE users SET credits_balance = credits_balance + :amount,
           lifetime_credits = lifetime_credits + :amount
           WHERE id = :user_id RETURNING credits_balance""",
        {"amount": amount, "user_id": user_id}
    )

    # Log transaction
    await db.insert("credit_transactions", {
        "user_id": user_id,
        "amount": amount,
        "balance_after": new_balance,
        "transaction_type": transaction_type,
        "description": description,
        "stripe_payment_id": stripe_payment_id,
    })

    return new_balance


async def deduct_credits(user_id, amount, description):
    """Deduct credits. Raises InsufficientCreditsError if not enough."""
    # Atomic check-and-deduct (prevents race conditions)
    result = await db.execute(
        """UPDATE users SET credits_balance = credits_balance - :amount
           WHERE id = :user_id AND credits_balance >= :amount
           RETURNING credits_balance""",
        {"amount": amount, "user_id": user_id}
    )

    if result is None:
        raise InsufficientCreditsError(f"Need {amount} credits, insufficient balance")

    new_balance = result

    await db.insert("credit_transactions", {
        "user_id": user_id,
        "amount": -amount,
        "balance_after": new_balance,
        "transaction_type": "usage",
        "description": description,
    })

    return new_balance


async def get_credit_balance(user_id):
    """Get current credit balance."""
    return await db.fetchval(
        "SELECT credits_balance FROM users WHERE id = :user_id",
        {"user_id": user_id}
    )
```

### Credit Check Middleware

```python
# Python (FastAPI example)
async def require_credits(user, cost: int):
    """Check user has enough credits before proceeding."""
    balance = await get_credit_balance(user.id)
    if balance < cost:
        raise HTTPException(
            402,
            detail={
                "error": "insufficient_credits",
                "required": cost,
                "balance": balance,
                "purchase_url": "/credits",
            }
        )
```

```javascript
// Node.js
async function requireCredits(userId, cost) {
  const balance = await db.getCreditBalance(userId);
  if (balance < cost) {
    throw new ApiError(402, {
      error: "insufficient_credits",
      required: cost,
      balance,
    });
  }
}
```

### Usage Flow

```python
# In your API endpoint that costs credits
@app.post("/api/generate")
async def generate(request: Request, user = Depends(get_current_user)):
    cost = 5  # This operation costs 5 credits

    # 1. Check credits
    await require_credits(user, cost)

    # 2. Perform the work
    result = await do_generation(request.data)

    # 3. Deduct credits (only after success)
    await deduct_credits(
        user_id=user.id,
        amount=cost,
        description=f"Generation: {request.data.get('type')}",
    )

    return {"result": result, "credits_remaining": await get_credit_balance(user.id)}
```

---

## Hybrid: Subscription + Credits

Subscription tiers include a monthly credit allocation. Users can purchase additional credits.

### Monthly Credit Allocation

```python
PLAN_CREDITS = {
    "free": 10,
    "basic": 100,
    "pro": 500,
    "enterprise": 5000,
}

async def handle_invoice_paid(invoice):
    """Provision monthly credits when subscription renews."""
    customer_id = invoice["customer"]
    subscription_id = invoice.get("subscription")

    if not subscription_id:
        return

    user = await db.get_user_by_stripe_customer(customer_id)
    credits = PLAN_CREDITS.get(user.plan_tier, 0)

    if credits > 0:
        await add_credits(
            user_id=user.id,
            amount=credits,
            transaction_type="subscription_allocation",
            description=f"Monthly {user.plan_tier} plan allocation ({credits} credits)",
            stripe_payment_id=invoice["id"],
        )
```

### Top-Up Credits (Beyond Allocation)

```python
# Same credit purchase flow as above, but mark as top-up
async def create_topup_session(user, credits_amount, price_id):
    session = stripe.checkout.Session.create(
        customer=user.stripe_customer_id,
        mode="payment",
        line_items=[{"price": price_id, "quantity": 1}],
        metadata={
            "type": "credit_topup",
            "credits": str(credits_amount),
            "user_id": str(user.id),
        },
        success_url="...",
        cancel_url="...",
    )
    return session.url
```

---

## Usage Tracking Patterns

### Track Costs Per Feature

```python
FEATURE_COSTS = {
    "basic_search": 1,
    "advanced_search": 5,
    "export_report": 10,
    "ai_analysis": 25,
}

async def use_feature(user_id, feature_name):
    cost = FEATURE_COSTS.get(feature_name)
    if cost is None:
        raise ValueError(f"Unknown feature: {feature_name}")

    await require_credits(user_id, cost)
    result = await execute_feature(feature_name)
    await deduct_credits(user_id, cost, f"Feature: {feature_name}")
    return result
```

### Usage Analytics

```python
async def get_usage_summary(user_id, start_date, end_date):
    """Get credit usage summary for a period."""
    return await db.fetchall("""
        SELECT
            transaction_type,
            description,
            SUM(ABS(amount)) as total_used,
            COUNT(*) as transaction_count
        FROM credit_transactions
        WHERE user_id = :user_id
          AND amount < 0
          AND created_at BETWEEN :start AND :end
        GROUP BY transaction_type, description
        ORDER BY total_used DESC
    """, {"user_id": user_id, "start": start_date, "end": end_date})
```

### Low Balance Notifications

```python
async def check_low_balance(user_id):
    balance = await get_credit_balance(user_id)
    threshold = 10  # Notify when below 10 credits

    if balance <= threshold and balance > 0:
        await send_low_balance_email(user_id, balance)
    elif balance <= 0:
        await send_out_of_credits_email(user_id)
```
