# Customer Portal & Invoices

## Table of Contents
- [Customer Portal Setup](#customer-portal-setup)
- [Creating Portal Sessions](#creating-portal-sessions)
- [Portal Configuration](#portal-configuration)
- [Invoices](#invoices)
- [Refunds](#refunds)

---

## Customer Portal Setup

Stripe Customer Portal is a hosted page where customers can manage their billing independently: update payment methods, change plans, cancel subscriptions, and view invoices.

### Enable in Dashboard

1. Go to Stripe Dashboard > Settings > Billing > Customer Portal
2. Configure allowed features:
   - Update payment methods
   - View invoice history
   - Cancel subscriptions
   - Switch plans
3. Set business name and branding

---

## Creating Portal Sessions

### Python

```python
portal_session = stripe.billing_portal.Session.create(
    customer=stripe_customer_id,
    return_url="https://yourapp.com/account",  # Where to return after
)
# Redirect user to portal_session.url
```

### Node.js

```javascript
const portalSession = await stripe.billingPortal.sessions.create({
  customer: stripeCustomerId,
  return_url: "https://yourapp.com/account",
});
// Redirect user to portalSession.url
```

### API Endpoint Pattern

```python
# Python (FastAPI)
@app.post("/api/billing/portal")
async def create_portal_session(user=Depends(get_current_user)):
    if not user.stripe_customer_id:
        raise HTTPException(400, "No billing account found")

    session = stripe.billing_portal.Session.create(
        customer=user.stripe_customer_id,
        return_url=f"{settings.app_url}/account",
    )
    return {"url": session.url}
```

```javascript
// Node.js (Express)
app.post("/api/billing/portal", requireAuth, async (req, res) => {
  const user = req.user;
  if (!user.stripeCustomerId) {
    return res.status(400).json({ error: "No billing account" });
  }

  const session = await stripe.billingPortal.sessions.create({
    customer: user.stripeCustomerId,
    return_url: `${process.env.APP_URL}/account`,
  });
  res.json({ url: session.url });
});
```

---

## Portal Configuration

### Programmatic Configuration

```python
configuration = stripe.billing_portal.Configuration.create(
    business_profile={
        "headline": "Manage your subscription",
        "privacy_policy_url": "https://yourapp.com/privacy",
        "terms_of_service_url": "https://yourapp.com/terms",
    },
    features={
        "customer_update": {
            "enabled": True,
            "allowed_updates": ["email", "address", "phone", "tax_id"],
        },
        "payment_method_update": {"enabled": True},
        "invoice_history": {"enabled": True},
        "subscription_cancel": {
            "enabled": True,
            "mode": "at_period_end",  # Cancel at end of period
            "cancellation_reason": {
                "enabled": True,
                "options": [
                    "too_expensive",
                    "missing_features",
                    "switched_service",
                    "unused",
                    "other",
                ],
            },
        },
        "subscription_update": {
            "enabled": True,
            "default_allowed_updates": ["price", "quantity"],
            "products": [
                {
                    "product": "prod_basic",
                    "prices": ["price_basic_monthly", "price_basic_yearly"],
                },
                {
                    "product": "prod_pro",
                    "prices": ["price_pro_monthly", "price_pro_yearly"],
                },
            ],
            "proration_behavior": "create_prorations",
        },
    },
)
```

### Use Configuration with Portal Session

```python
session = stripe.billing_portal.Session.create(
    customer=stripe_customer_id,
    return_url="https://yourapp.com/account",
    configuration=configuration.id,  # Use specific config
)
```

---

## Invoices

### List Customer Invoices

```python
invoices = stripe.Invoice.list(
    customer=stripe_customer_id,
    limit=10,
    status="paid",  # "paid", "open", "draft", "void", "uncollectible"
)

for invoice in invoices.auto_paging_iter():
    print(f"Invoice {invoice.number}: {invoice.amount_paid / 100} {invoice.currency}")
    print(f"  PDF: {invoice.invoice_pdf}")
    print(f"  Hosted URL: {invoice.hosted_invoice_url}")
```

### Create One-Off Invoice

```python
# Create invoice item
stripe.InvoiceItem.create(
    customer=stripe_customer_id,
    amount=5000,  # $50.00
    currency="usd",
    description="Custom development work",
)

# Create and finalize invoice
invoice = stripe.Invoice.create(
    customer=stripe_customer_id,
    collection_method="send_invoice",
    days_until_due=30,
    auto_advance=True,
)

# Send it
stripe.Invoice.send_invoice(invoice.id)
```

### Upcoming Invoice Preview

Show users what they'll be charged next:

```python
upcoming = stripe.Invoice.upcoming(
    customer=stripe_customer_id,
)
print(f"Next charge: ${upcoming.amount_due / 100} on {upcoming.period_end}")
```

### Preview Invoice After Plan Change

```python
# Show user what upgrade/downgrade will cost
upcoming = stripe.Invoice.upcoming(
    customer=stripe_customer_id,
    subscription=subscription_id,
    subscription_items=[{
        "id": subscription_item_id,
        "price": "price_pro_monthly",  # New price
    }],
    subscription_proration_behavior="create_prorations",
)
proration_amount = upcoming.amount_due  # What they'd pay now
```

---

## Refunds

### Full Refund

```python
refund = stripe.Refund.create(
    payment_intent=payment_intent_id,
    # charge=charge_id,  # Alternative: use charge ID
)
```

### Partial Refund

```python
refund = stripe.Refund.create(
    payment_intent=payment_intent_id,
    amount=500,  # Refund $5.00 of the charge
)
```

### Refund with Reason

```python
refund = stripe.Refund.create(
    payment_intent=payment_intent_id,
    reason="requested_by_customer",  # or "duplicate", "fraudulent"
)
```

### Webhook for Refunds

```python
async def handle_charge_refunded(charge):
    """Handle refund webhook."""
    customer_id = charge["customer"]
    refund_amount = charge["amount_refunded"]

    # If using credits, optionally deduct refunded credits
    # await deduct_credits(user_id, credits_to_remove, "Refund adjustment")

    await send_refund_confirmation_email(customer_id, refund_amount)
```

Listen for `charge.refunded` event.
