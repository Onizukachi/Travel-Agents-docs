# Orders, Payments, and Receipts flow

This section describes the end-to-end flow of order creation, payment processing, and receipt generation.

## 1. Order prebuild (draft state)

During order formation (traveler details input, coupon application, price recalculation), the frontend sends incremental updates to:

- `Papi::V3::OrdersController#prebuild`

Characteristics:
- The order is not persisted yet.
- The endpoint is used for validation, pricing, and preview purposes.
- Multiple requests may be sent as the user modifies order data.
- No payment objects are created at this stage.

---

## 2. Order creation

When the user selects a payment method and confirms the purchase, the frontend sends a request to:

- `Papi::V3::OrdersController#create`

Characteristics:
- The order is persisted in the database.
- The order transitions from a draft/prebuild state to a created state.
- Further payment actions depend on the selected payment method.

---

## 3. Payment initiation

Available payment methods are determined by:

- `Order#payment_methods`

Based on the selected payment method, the frontend performs one of the following actions:

- **Card payment**:
  - Sends a request to:
    - `Papi::V3::PaymentsController#pay_card`

- **Non-card / redirect-based payment methods**:
  - Sends a request to:
    - `Papi::V3::PaymentsController#payment_params`
  - The response typically contains a payment URL to which the user must be redirected.

---

## 4. Payment creation and processors

Payments are created via:

- `PaymentBuilder`

Responsibilities of `PaymentBuilder`:
- Creates a `Payment` record.
- Assigns a `processor` to the payment.

The `processor`:
- Is a string representing a Ruby constant name.
- Points to a class responsible for interacting with a specific payment gateway API.
- All payment processors are located in:
  - `app/apis/payment_processor/`

Each processor encapsulates:
- API request building
- Authentication
- Gateway-specific logic

---

## 5. Asynchronous payment callbacks

Most payment gateways operate asynchronously.

After payment actions (authorization, unfreeze, capture, refund), external providers send callbacks (webhooks) to the system.

Callback handling:
- Implemented in classes with the `_callback_processor` suffix.
- Located in `app/apis/`
- Example:
  - `app/apis/alfa_pay_callback_processor.rb`

Responsibilities of callback processors:
- Validate incoming notifications.
- Update payment and order states.
- Trigger post-payment logic when applicable.

---

## 6. Receipts and line items generation

After successful payments or refunds:

- Receipts are generated.
- Line items are created and attached to receipts.

Domain models:
- `Receipt` — represents a fiscal receipt.
- `LineItemV2` — represents individual product or service items within a receipt.

Receipt generation logic:
- Implemented in:
  - `Receipts::Builder`

Responsibilities of `Receipts::Builder`:
- Create receipts for payments and refunds.
- Generate corresponding `LineItemV2` records.
- Ensure consistency between payments, receipts, and line items.
