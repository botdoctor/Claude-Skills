# Checkout Sessions & Payment Intents

## Table of Contents
- [Checkout Sessions (Hosted)](#checkout-sessions-hosted)
- [Payment Intents (Custom UI)](#payment-intents-custom-ui)
- [One-Time Payments](#one-time-payments)
- [Subscription Checkout](#subscription-checkout)
- [Success and Cancel Handling](#success-and-cancel-handling)

---

## Checkout Sessions (Hosted)

Stripe-hosted payment page. Fastest integration, PCI-compliant by default.

### One-Time Payment Checkout

```python
# Python
import stripe

session = stripe.checkout.Session.create(
    customer=stripe_customer_id,  # Existing customer
    mode="payment",               # One-time payment
    line_items=[{
        "price_data": {
            "currency": "usd",
            "unit_amount": 2000,  # $20.00 in cents
            "product_data": {
                "name": "Premium Report",
                "description": "Detailed analysis report",
            },
        },
        "quantity": 1,
    }],
    success_url="https://yourapp.com/success?session_id={CHECKOUT_SESSION_ID}",
    cancel_url="https://yourapp.com/cancel",
    metadata={"order_id": order_id},  # Track in webhooks
)
# Redirect user to session.url
```

```javascript
// Node.js
const session = await stripe.checkout.sessions.create({
  customer: stripeCustomerId,
  mode: "payment",
  line_items: [{
    price_data: {
      currency: "usd",
      unit_amount: 2000,
      product_data: {
        name: "Premium Report",
      },
    },
    quantity: 1,
  }],
  success_url: "https://yourapp.com/success?session_id={CHECKOUT_SESSION_ID}",
  cancel_url: "https://yourapp.com/cancel",
  metadata: { order_id: orderId },
});
// Redirect user to session.url
```

### Using Pre-Created Prices

If Prices are created in Stripe Dashboard:

```python
session = stripe.checkout.Session.create(
    customer=stripe_customer_id,
    mode="payment",
    line_items=[{
        "price": "price_1ABC123...",  # Dashboard-created Price ID
        "quantity": 1,
    }],
    success_url="https://yourapp.com/success?session_id={CHECKOUT_SESSION_ID}",
    cancel_url="https://yourapp.com/cancel",
)
```

### Subscription Checkout

```python
session = stripe.checkout.Session.create(
    customer=stripe_customer_id,
    mode="subscription",  # Subscription mode
    line_items=[{
        "price": "price_pro_monthly",  # Recurring Price ID
        "quantity": 1,
    }],
    success_url="https://yourapp.com/success?session_id={CHECKOUT_SESSION_ID}",
    cancel_url="https://yourapp.com/pricing",
    subscription_data={
        "trial_period_days": 14,  # Optional trial
        "metadata": {"plan": "pro"},
    },
    allow_promotion_codes=True,  # Enable promo codes
)
```

### Checkout with Multiple Items

```python
session = stripe.checkout.Session.create(
    customer=stripe_customer_id,
    mode="subscription",
    line_items=[
        {"price": "price_base_plan", "quantity": 1},
        {"price": "price_addon_storage", "quantity": 3},  # 3 units of storage
    ],
    success_url="https://yourapp.com/success?session_id={CHECKOUT_SESSION_ID}",
    cancel_url="https://yourapp.com/pricing",
)
```

---

## Payment Intents (Custom UI)

For embedding payment forms directly in your UI using Stripe Elements.

### Server: Create PaymentIntent

```python
# Python
intent = stripe.PaymentIntent.create(
    amount=2000,        # $20.00 in cents
    currency="usd",
    customer=stripe_customer_id,
    metadata={"order_id": order_id},
    automatic_payment_methods={"enabled": True},
    idempotency_key=f"pi_{order_id}",
)
# Return intent.client_secret to frontend
```

```javascript
// Node.js
const intent = await stripe.paymentIntents.create({
  amount: 2000,
  currency: "usd",
  customer: stripeCustomerId,
  metadata: { order_id: orderId },
  automatic_payment_methods: { enabled: true },
}, {
  idempotencyKey: `pi_${orderId}`,
});
// Return intent.client_secret to frontend
```

### Client: Confirm Payment (React Example)

```javascript
import { loadStripe } from "@stripe/stripe-js";
import { Elements, PaymentElement, useStripe, useElements } from "@stripe/react-stripe-js";

const stripePromise = loadStripe("pk_test_...");

function CheckoutForm() {
  const stripe = useStripe();
  const elements = useElements();

  const handleSubmit = async (e) => {
    e.preventDefault();
    const { error } = await stripe.confirmPayment({
      elements,
      confirmParams: {
        return_url: "https://yourapp.com/success",
      },
    });
    if (error) {
      // Show error to user
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <PaymentElement />
      <button type="submit" disabled={!stripe}>Pay</button>
    </form>
  );
}

// Wrap with Elements provider
function CheckoutPage({ clientSecret }) {
  return (
    <Elements stripe={stripePromise} options={{ clientSecret }}>
      <CheckoutForm />
    </Elements>
  );
}
```

### Vanilla JS Client

```html
<div id="payment-element"></div>
<button id="submit">Pay</button>

<script src="https://js.stripe.com/v3/"></script>
<script>
  const stripe = Stripe("pk_test_...");
  const elements = stripe.elements({ clientSecret });
  const paymentElement = elements.create("payment");
  paymentElement.mount("#payment-element");

  document.getElementById("submit").addEventListener("click", async () => {
    const { error } = await stripe.confirmPayment({
      elements,
      confirmParams: { return_url: "https://yourapp.com/success" },
    });
    if (error) alert(error.message);
  });
</script>
```

---

## Success and Cancel Handling

### Verify Payment on Success Page

Never trust the redirect alone. Always verify server-side:

```python
# Success page endpoint
session = stripe.checkout.Session.retrieve(session_id)
if session.payment_status == "paid":
    # Fulfill order
    fulfill_order(session.metadata.get("order_id"))
```

### Best Practice: Use Webhooks for Fulfillment

Do not rely on the success redirect for critical fulfillment. Users can close the browser before redirect. Use `checkout.session.completed` webhook instead. See [webhooks.md](webhooks.md).

---

## Amounts

Stripe uses **smallest currency unit** (cents for USD):

| Display | API Amount | Currency |
|---------|-----------|----------|
| $10.00 | 1000 | usd |
| $99.99 | 9999 | usd |
| €15.50 | 1550 | eur |
| ¥1000 | 1000 | jpy (zero-decimal) |

Zero-decimal currencies (JPY, KRW, etc.) use the actual amount.
