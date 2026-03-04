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

### 6.1 Advanced architecture: entry points and scenarios

- **Auto detalization after trip**:
  `Order#mark_got_back` -> `Order#create_receipts`
  -> `Receipt.process_purchase_advanced_receipts(order, receipt_price, 'detailed')`
  -> `Receipts::Builder#create`
  -> `LineItemsV2::Advanced::Builder#build_detailed_line_items`.
- **Detailed refund after detalization**:
  `Payment#refund_detailed!` -> `Payment#mark_refunded!(..., params)` -> `Payment#create_advanced_receipts`
  -> `Receipt.process_refund_advanced_receipts(payment, amount, 'detailed', params)`
  -> `Receipts::Builder#create`
  -> `LineItemsV2::Advanced::Builder#build_refund_line_items`.
- **Manual fiscalization**:
  `Admin::ManualFiscalizationController#fiscalize_receipts`
  -> `LineItemsV2::Advanced::Builder.create_manual`.

### 6.2 Licence line item (`LineItemV2::LICENCE_NAME`)

- Licence line item can be built only in detailed flows:
  - auto capture: `captured_licence_line_item`
  - detailed refund: `refunded_licence_line_item`
  - manual detailed: `manual_licence_line_item`
- `LineItemsV2::Advanced::Creator#create_line_item!` skips creation when `price.zero?`.
- For `subagent_wl` partner, licence is always `0` in calculator and therefore is not created.
- For certificate orders, detailed flow builds only certificate/prepayment item (no licence).

### 6.3 Detailed pricing rules (licence, tour, extras)

- Main calculator: `LineItemsV2::Advanced::PriceCalculator`.
- Detailed purchase amount source: `Order#receipt_price` (`captured_total + discount_price`).
- Core formula:
  `licence_price = amount - order_price - order_extras_price`, then apply remaining discount and clamp to `>= 0`.
- `order_price` is based on operator payments for `service_type: 'order'` and already detailed order part.
- `order_extras_price` is calculated only for `order.order_extras.active.non_split.without_priority_service`.
- Domain rule:
  non-split extras are separate detailed line items; other extras are treated as part of the tour price
  (they are included in operator payments and must not be subtracted again as separate extras).
- Discount distribution in calculator keeps each detailed tour/extra line item at least `1` (`MIN_LINE_ITEM_PRICE`),
  so with large discounts licence may become `0`.

### 6.4 Refund assumptions and fragile points

- Before first detalization, expected invariant:
  `payment.calculated_refunded_amount == order.line_items_v2.where(payment_type: LineItemV2::REFUND_TYPES).sum(&:price)`.
- In detailed refund, params keys are expected to be always present:
  `refund_tour_amount`, `refund_licence_amount`, `refund_extras` (`0` values are valid).
- `LineItemsV2::Advanced::Builder#refund?` currently uses an aggregate comparison
  (`existing refund line items` vs `payments.calculated_refunded_amount`) and is known fragile.
  Treat this as legacy behaviour and prefer explicit mode selection in future refactors.
