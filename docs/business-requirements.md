# Business requirements

## Business objectives

- Replace disconnected retail systems with a coordinated e-commerce backend for customers and administrators.
- Support product discovery, cart maintenance, ordering, simulated payments and order tracking.
- Maintain consistent inventory under concurrent purchases, including competition for the final available item.
- Prevent duplicate orders or payments when requests are retried, and recover inventory after failed payments.
- Preserve successful payments when Order Service communication times out.
- Provide maintainable, secure and adaptable service integration with operational logging and diagnostics.

## Actors

- **Customer:** registers, authenticates, manages a profile, browses products, maintains a cart, places and tracks orders, and requests cancellation.
- **Administrator:** manages products, categories, inventory and transactional information, and uses operational diagnostics.
- The Order Service coordinates reservation and payment operations on behalf of customers. A customer-facing story does not grant the customer direct access to service-only write endpoints.

## Functional requirements

### Reading conventions

- Story scope comes from Reference A, reproduced under Source.
- Reference E is used, with user approval, for endpoint mapping and expected HTTP status codes. It does not independently add business features to this draft.
- Each acceptance criterion has a service-unique ID. Numbering continues across stories within that service.
- Unless stated otherwise, requests have valid input and the caller has the authorization required by the endpoint.
- HTTP codes describe the named endpoint, not the overall business outcome. In particular, HTTP 201 can represent an order created in a cancelled state.
- Unspecified or conflicting behavior is recorded under Open questions rather than inferred.

### Identity (IDN)

#### US-IDN-01

As a customer, I want to register an account so that I can use the e-commerce platform.

- **AC-IDN-01:** Given I do not have an account, When I submit valid registration details to `POST /api/users/register`, Then my customer account is created and the response is **HTTP 201**.
- **AC-IDN-02:** Given an account already uses my email address, When I register with that email address, Then registration is rejected with `EMAIL_ALREADY_EXISTS` and **HTTP 409**.

#### US-IDN-02

As a customer, I want to authenticate so that I can access my account.

- **AC-IDN-03:** Given I have a registered account and valid credentials, When I call `POST /api/auth/login`, Then I receive an access token and the response is **HTTP 200**.
- **AC-IDN-04:** Given my credentials are invalid, When I call `POST /api/auth/login`, Then authentication is rejected with `INVALID_CREDENTIALS` and **HTTP 401**.

#### US-IDN-03

As a customer, I want to view and update my profile so that my customer information remains current.

- **AC-IDN-05:** Given I am authenticated, When I call `GET /api/users/me`, Then my own profile is returned with **HTTP 200**.
- **AC-IDN-06:** Given I am authenticated and provide valid profile changes, When I call `PUT /api/users/me`, Then my profile is updated and the response is **HTTP 200**.
- **AC-IDN-07:** Given my profile changes fail validation, When I call `PUT /api/users/me`, Then the update is rejected with **HTTP 400**.

### Catalog (CAT)

#### US-CAT-01

As an administrator, I want to manage product categories so that the catalog is organized for browsing.

- **AC-CAT-01:** Given a category does not already exist, When I create it through `POST /api/categories`, Then the category is created with **HTTP 201**.
- **AC-CAT-02:** Given categories have been created, When I call `GET /api/categories`, Then the category list is returned with **HTTP 200**.
- **AC-CAT-03:** Given a category already exists, When I attempt to create the same category, Then the request is rejected with `CATEGORY_ALREADY_EXISTS` and **HTTP 409**.

#### US-CAT-02

As an administrator, I want to create and update products so that customers see a maintained product catalog.

- **AC-CAT-04:** Given the product details are valid and its SKU is not already used, When I call `POST /api/products`, Then the product is created with **HTTP 201**.
- **AC-CAT-05:** Given a product exists and my changes are valid, When I call `PUT /api/products/{id}`, Then the product is updated with **HTTP 200**.
- **AC-CAT-06:** Given the product does not exist, When I call `PUT /api/products/{id}`, Then the response is **HTTP 404**.

#### US-CAT-03

As a customer, I want to browse products and view their details so that I can decide what to purchase.

- **AC-CAT-07:** Given the catalog is available, When I call `GET /api/products`, Then the product page is returned with **HTTP 200**.
- **AC-CAT-08:** Given a product exists, When I call `GET /api/products/{id}`, Then its details are returned with **HTTP 200**.
- **AC-CAT-09:** Given a product does not exist, When I request its details, Then the response is `PRODUCT_NOT_FOUND` with **HTTP 404**.

#### US-CAT-04

As a customer, I want to search and filter products so that I can find products relevant to my needs.

- **AC-CAT-10:** Given products matching a search term exist, When I call `GET /api/products/search` with that term, Then the matching product page is returned with **HTTP 200**.
- **AC-CAT-11:** Given I supply supported category, price or active-status filters, When I call `GET /api/products/search`, Then returned products satisfy the supplied filters and the response is **HTTP 200**.

### Inventory (INV)

#### US-INV-01

As a customer, I want to check stock availability so that I know whether a product is available to purchase.

- **AC-INV-01:** Given stock information exists for a product, When I call `GET /api/inventory/{productId}`, Then its available and reserved quantities are returned with **HTTP 200**.
- **AC-INV-02:** Given no stock information exists for a product, When I request its inventory, Then the response is `STOCK_NOT_FOUND` with **HTTP 404**.

#### US-INV-02

As a customer, I want checkout to reserve available inventory reliably so that competing purchases cannot oversell the same stock.

- **AC-INV-03:** Given sufficient stock exists, When the authorized Order Service calls `POST /api/inventory/reservations` for my order, Then a pending reservation is created with **HTTP 201**.
- **AC-INV-04:** Given one unit remains and two different orders simultaneously request that unit, When both reservation requests are processed, Then exactly one request creates a reservation with **HTTP 201**, the other receives `INSUFFICIENT_STOCK` with **HTTP 409**, and stock is not oversold.
- **AC-INV-05:** Given stock is insufficient for the requested quantity, When the Order Service requests a reservation, Then the request is rejected with `INSUFFICIENT_STOCK` and **HTTP 409**.

#### US-INV-03

As a customer, I want my inventory reservation released after payment failure so that the stock becomes available again.

- **AC-INV-06:** Given my pending reservation exists and payment has failed, When the Order Service calls `POST /api/inventory/reservations/{id}/release`, Then the reservation becomes released, its stock becomes available again, and the response is **HTTP 200**.
- **AC-INV-07:** Given that reservation has already been released, When the authorized service repeats the release request, Then the stock is not released a second time and the response is **HTTP 200**.

#### US-INV-04

As a customer, I want inventory updated after a successful transaction so that purchased stock cannot be sold again.

- **AC-INV-08:** Given my order has a pending reservation and payment succeeds, When the Order Service calls `POST /api/inventory/reservations/{id}/confirm`, Then the reservation is confirmed, the purchase is reflected in inventory, and the response is **HTTP 200**.
- **AC-INV-09:** Given that reservation is already confirmed, When its confirmation is retried, Then inventory is not consumed again and the response is **HTTP 200**.

#### US-INV-05

As an administrator, I want to adjust inventory so that recorded stock reflects administrative changes.

- **AC-INV-10:** Given a stock adjustment is valid, When I call `POST /api/inventory/{productId}/adjustments`, Then the updated stock is returned with **HTTP 200**.
- **AC-INV-11:** Given an adjustment would make stock negative, When I submit it, Then it is rejected with `NEGATIVE_STOCK` and **HTTP 409**.

### Cart and Pricing (CART)

#### US-CART-01

As a customer, I want to manage my shopping cart so that I can maintain the products and quantities I intend to buy.

- **AC-CART-01:** Given I am authenticated, When I call `GET /api/carts/current`, Then my active cart is returned with **HTTP 200**.
- **AC-CART-02:** Given a product exists and the quantity is valid, When I call `POST /api/carts/items`, Then the product is added to my cart and the response is **HTTP 201**.
- **AC-CART-03:** Given an item is in my cart and the new quantity is valid, When I call `PUT /api/carts/items/{itemId}`, Then the quantity is updated and the response is **HTTP 200**.
- **AC-CART-04:** Given an item is in my cart, When I call `DELETE /api/carts/items/{itemId}`, Then it is removed and the response is **HTTP 204**.

#### US-CART-02

As a customer, I want my cart prices calculated so that I know the cost before placing an order.

- **AC-CART-05:** Given my cart contains products, When I call `POST /api/carts/checkout-summary`, Then the pricing summary is returned with **HTTP 200**.
- **AC-CART-06:** Given my cart is empty, When I request its checkout summary, Then the request is rejected with `EMPTY_CART` and **HTTP 422**.

#### US-CART-03

As a customer, I want an applicable discount included in my cart pricing so that my purchase total reflects the discount.

- **AC-CART-07:** Given a discount exists and applies to my cart, When I call `POST /api/carts/discount`, Then the discount is applied and the response is **HTTP 200**.
- **AC-CART-08:** Given the requested discount does not exist, When I attempt to apply it, Then the response is `DISCOUNT_NOT_FOUND` with **HTTP 404**.
- **AC-CART-09:** Given the discount exists but does not apply to my cart, When I attempt to apply it, Then the response is `DISCOUNT_NOT_APPLICABLE` with **HTTP 422**.
- **AC-CART-10:** Given a discount has been applied successfully, When I request `POST /api/carts/checkout-summary`, Then the returned summary includes the discount and resulting total with **HTTP 200**.

### Order (ORD)

#### US-ORD-01

As a customer, I want to place an order from my cart so that I can purchase my selected products.

- **AC-ORD-01:** Given my cart is nonempty, stock can be reserved and simulated payment succeeds, When I call `POST /api/orders` with a new idempotency key, Then an order is created in `CONFIRMED` state with **HTTP 201**.
- **AC-ORD-02:** Given my cart is empty, When I attempt to place an order, Then the request is rejected with `EMPTY_CART` and **HTTP 422**.
- **AC-ORD-03:** Given stock cannot satisfy my cart, When I attempt to place an order, Then the request is rejected with `INSUFFICIENT_STOCK` and **HTTP 409**.

#### US-ORD-02

As a customer, I want to track my orders and their state changes so that I know the progress of my purchases.

- **AC-ORD-04:** Given I own an order, When I call `GET /api/orders/{id}`, Then its current status and status history are returned with **HTTP 200**.
- **AC-ORD-05:** Given I request my order list, When I call `GET /api/orders/customer/{customerId}` for my customer ID, Then my orders are returned with **HTTP 200**.
- **AC-ORD-06:** Given the requested order does not exist, When I call `GET /api/orders/{id}`, Then the response is **HTTP 404**.

#### US-ORD-03

As a customer, I want a retried checkout request to return my original order so that a timeout does not create a duplicate purchase.

- **AC-ORD-07:** Given an order was created for my request and idempotency key, When I retry the same request with that key, Then the original order is returned with **HTTP 200** and no additional order or payment is created.
- **AC-ORD-08:** Given an idempotency key has already been used, When I reuse it for a conflicting order request, Then the request is rejected with `IDEMPOTENCY_CONFLICT` and **HTTP 422**.

#### US-ORD-04

As a customer, I want a failed payment compensated so that my order is not confirmed and reserved stock is not lost.

- **AC-ORD-09:** Given inventory was reserved and payment is declined, When order creation completes and reservation release succeeds, Then the created order is `CANCELLED`, includes a failure reason, and is returned with **HTTP 201**.
- **AC-ORD-10:** Given payment is declined but compensation cannot complete, When the created order is returned, Then its state is `COMPENSATION_PENDING`, it includes a failure reason, and the response is **HTTP 201**, not a confirmed order.

#### US-ORD-05

As a customer, I want to cancel an eligible order so that I can stop the purchase and receive the applicable simulated refund and stock restoration.

- **AC-ORD-11:** Given my confirmed order is eligible for cancellation, When I call `POST /api/orders/{id}/cancel`, Then the order becomes `CANCELLED`, a simulated refund is performed, stock is restored, and the response is **HTTP 200**.
- **AC-ORD-12:** Given my order is in a state that cannot be cancelled, When I request cancellation, Then the response is `INVALID_ORDER_STATE` with **HTTP 409**.

#### US-ORD-06

As an administrator, I want to inspect and recover interrupted order processing so that transactional records remain consistent after failures.

- **AC-ORD-13:** Given an order exists, When I call `GET /api/orders/{id}` as an administrator, Then the order and its status history are returned with **HTTP 200**.
- **AC-ORD-14:** Given a payment succeeded after the Order Service timed out and the order is pending recovery, When I call `POST /api/orders/{id}/recover`, Then the pending step is rerun using the original payment without a second charge, and the response is **HTTP 200**.
- **AC-ORD-15:** Given an order has pending compensation and its dependencies are now available, When I call `POST /api/orders/{id}/recover`, Then compensation completes without confirming the failed purchase and the response is **HTTP 200**.

### Payment (PAY)

#### US-PAY-01

As a customer, I want checkout to process a simulated payment so that I can complete a purchase without a real payment provider.

- **AC-PAY-01:** Given the payment simulation will succeed, When the authorized Order Service calls `POST /api/payments` for my order, Then a simulated payment is created with **HTTP 201** and no real payment provider is called.
- **AC-PAY-02:** Given the simulation declines payment, When the Order Service submits the payment, Then a declined transaction is recorded rather than a successful charge; the payment-creation response is **HTTP 201**.

#### US-PAY-02

As a customer, I want payment retries to reuse my original transaction so that an interrupted response cannot charge me twice.

- **AC-PAY-03:** Given a transaction exists for an idempotency key and request, When the authorized service repeats `POST /api/payments` with the same key and request, Then the original transaction is returned with **HTTP 200** and no second payment is created.
- **AC-PAY-04:** Given a transaction exists for an idempotency key, When that key is reused for a different payment request, Then the response is `IDEMPOTENCY_CONFLICT` with **HTTP 422**.

#### US-PAY-03

As an administrator, I want to inspect payment records after an interrupted checkout so that a successful payment is not lost or repeated.

- **AC-PAY-05:** Given a payment has completed, When I call `GET /api/payments/{id}`, Then its recorded outcome and attempts are returned with **HTTP 200**.
- **AC-PAY-06:** Given a simulated payment succeeds after the Order Service times out, When I call `GET /api/payments?orderId=` with that order ID after payment completion, Then the successful transaction is discoverable with **HTTP 200**, allowing reconciliation without another payment.

#### US-PAY-04

As a customer, I want an eligible cancelled order to receive a simulated refund so that its payment is reversed.

- **AC-PAY-07:** Given my payment succeeded and my order is being cancelled, When the authorized Order Service calls `POST /api/payments/{id}/refund`, Then a completed simulated refund is created with **HTTP 201**.
- **AC-PAY-08:** Given the payment did not succeed, When a refund is requested, Then the response is `INVALID_PAYMENT_STATE` with **HTTP 409**.

## Non-functional requirements

- Service integration must remain reliable, maintainable, secure and adaptable. Reference A supplies no measurable service-level thresholds.
- Concurrent reservations must not oversell inventory; Reference A specifies optimistic locking with a SQLAlchemy version column and an atomic conditional update.
- Retried order and payment requests must avoid duplicate transactions; timed-out communication must not lose a successful payment.
- Operational logging and diagnostics must support analysis of application health and failures. Required log content, diagnostic coverage and HTTP contracts remain open.
- The stated engineering stack is Python 3.12 with FastAPI/Uvicorn. No additional operational or security requirements from other references are introduced here.

## Out of scope

- Real payment-provider integration: payment processing and refunds are simulated.
- Features not stated in Reference A are not added by this draft. Unclear portions of broad terms such as "management" and "administrative operations" remain open rather than being assumed.
- Training and engineering outcomes are preserved in Source but are not converted into customer-facing product features.

## Open questions

1. **Category and product management boundaries:** Reference A says "management" without defining every operation. Reference E supplies category create/list and product create/read/update, but no category update/delete or product delete contract. Are any additional operations intended? No stories for these unspecified operations have been added.
2. **Administrative scope:** What customer-information, transactional-information and discount administration is required beyond the concrete product, inventory, order and payment operations above? Reference E lists user lookup and discount administration, but Reference A does not explicitly define their scope. The discount administration row also has no success/error HTTP codes.
3. **Profile validation:** What constraints apply to each editable profile field? Reference E specifies HTTP 400 for validation failure but does not define all field-level rules.
4. **Pricing and discount policy:** Which calculation, currency, rounding, eligibility and combination rules should be adopted? Reference A does not define them. Reference E contains policies, but approval here covers its HTTP contracts, not adoption of additional business rules.
5. **Order state model and cancellation:** What is the complete transition model, and which states permit cancellation? Reference E names `CONFIRMED`, `CANCELLED`, `PAYMENT_PENDING` and `COMPENSATION_PENDING` and an invalid-state response, but Reference A does not enumerate the full lifecycle.
6. **Payment-timeout response:** What HTTP response and order state must the initial checkout request return when payment succeeds after the Order Service times out? Reference E specifies reconciliation/recovery endpoints but does not explicitly bind this initial timeout outcome to a response code. No initial-timeout status code is guessed.
7. **Recovery ownership and scheduling:** Is recovery manually initiated, automatic, or both, and what retry limits or timing apply? Reference E supplies a recovery endpoint; neither the manual-only nor automatic-only policy is assumed.
8. **Cancellation partial failure:** What should the cancellation endpoint return if refund or restocking fails after another cancellation step has completed? The successful HTTP 200 contract does not define all partial-failure responses.
9. **Repeated refund response:** Reference E says a repeated refund returns the existing refund but does not give its HTTP code. Confirm the expected code before adding a retry-refund acceptance criterion.
10. **Logging and diagnostics:** What events, fields, retention and diagnostic endpoints are required across the six services? Reference E explicitly documents HTTP 200 for the Identity health endpoint, but Reference A does not define diagnostic HTTP contracts for every service.
11. **Reservation lifecycle:** What expiry policy and recovery behavior apply to abandoned reservations? Reference A requires reservations and compensation but gives no expiry policy. Expiry behavior has not been added from other references.
12. **Simulation controls:** Which payment simulation inputs and deterministic outcomes must be exposed to customers or administrators? Reference A requires simulation but does not define those controls.

## Source

**Primary source:** Reference A, "Business scenario and outcomes", in [Happiest_Minds_ECommerce_Python_Project_Step_by_Step_Guide_v1.0.docx](../../../Downloads/Happiest_Minds_ECommerce_Python_Project_Step_by_Step_Guide_v1.0.docx).

**Supporting source authorized by the user:** Reference E, "API contracts and seed data", in the same document, used for endpoint mapping and HTTP status codes only. Seed data is not used to create requirements. The guide's Step 6 identifies this file and its seven sections as the destination.

### Reference A - Business scenario and outcomes

References A–U contain the specification, rules and lookup tables that the steps point to. The business scenario, data model, API contracts, rules and scenarios are identical to the Java original: only the engineering technology changes.

### Vision

The objective is to enable Happiest Minds backend developers to move beyond basic AI code generation and engineer complete software solutions using GitHub Copilot Agent, building the same enterprise e-commerce backend in Python 3.12 / FastAPI (Uvicorn). The programme shows how Context Engineering, Loop Engineering, Prompt Engineering, Token Optimisation and AI Production Engineering work together to improve the efficiency and quality of day-to-day development.

### Functional outcomes

The finished application must support all of the following. Each maps to a service and to an end-to-end scenario in Reference O.

- Customer registration and authentication.
- User profile management.
- Product catalog and category management.
- Product search and filtering.
- Stock availability and inventory reservations.
- Shopping cart management.
- Pricing and discount calculation.
- Order creation and order-state transitions.
- Simulated payment processing.
- Payment failure and compensation.
- Order cancellation and simulated refunds.
- Inventory updates following successful transactions.
- Administrative operations.
- Operational logging and diagnostics.

### Engineering outcomes

By the end of the programme every participant must be able to: interpret business requirements and identify ambiguities; translate requirements into technical specifications; design microservice boundaries; create Context Engineering artefacts; use Copilot Agent for multi-file implementation; review and validate generated code; establish controlled development-feedback loops; diagnose compilation and runtime failures; implement secure backend functionality; maintain consistency across repositories; introduce new requirements into existing services; execute automated validation through GitHub Actions; analyse application health and operational failures; and measure AI-assisted development effectiveness.

### Enterprise retail commerce platform

A retail organisation manages product availability, customer information and order processing through disconnected systems. As order volumes increase it experiences inventory inconsistencies, duplicate checkout requests, unsuccessful payments and difficulty coordinating changes across backend components. The business requires a modern microservices-based e-commerce backend that lets customers browse products, maintain carts, place orders, make simulated payments and track orders, while administrators manage products, inventory and transactional information. The architecture must provide reliable service integration while remaining maintainable, secure and adaptable.

### The central engineering challenge

A customer wants to purchase the final available laptop while another customer attempts to purchase the same product at the same moment. Only one customer may receive a valid reservation. If the chosen customer's payment fails, the inventory must become available again. If the payment succeeds but the Order Service times out, the system must avoid a second payment or a lost confirmed payment. This drives three design decisions you implement:

- Optimistic locking in the Inventory Service (a SQLAlchemy version_id column plus an atomic UPDATE ... WHERE conditional statement) so concurrent reservations cannot oversell.
- Compensation in the Order Service so a failed payment releases the reservation and the order never reaches CONFIRMED.
- Idempotency keys in the Order and Payment Services so a retried request after a timeout returns the original transaction instead of charging twice.
