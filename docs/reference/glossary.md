# Product Glossary

> **Canonical domain terminology.** All spec artifacts (FS, TS, WP) MUST use these terms exactly as defined here.
> If a term is missing, add it and open a PR for review before using it in a spec.

---

## Orders

| Term | Definition |
|---|---|
| **Order** | A confirmed intent to purchase one or more Products, created when a Customer or Guest completes checkout. Has a lifecycle: `pending → confirmed → processing → shipped → delivered → cancelled`. |
| **Order Line** | A single Product and quantity within an Order. |
| **Order Total** | The sum of all Order Line subtotals plus shipping cost plus tax, minus any applied discounts. |
| **Guest Order** | An Order placed without a registered Customer account. Associated with a Guest Session. |
| **Order Reference** | A human-readable unique identifier surfaced to the customer (e.g. `ORD-20240415-A3K9`). Distinct from the internal order ID. |
| **Checkout** | The multi-step process in which a shopper provides shipping address, selects a shipping method, enters payment details, and submits an Order. |
| **Guest Checkout** | Checkout completed without signing in or creating an account. |
| **Cart** | A temporary, pre-checkout collection of Products a shopper intends to purchase. Carts are not Orders. |
| **Basket** | Synonym for Cart. Avoid; prefer **Cart**. |

---

## Products & Inventory

| Term | Definition |
|---|---|
| **Product** | An item available for purchase. Has a name, description, price, and at least one SKU. |
| **SKU** | Stock-Keeping Unit. A unique identifier for a specific variant of a Product (e.g. size, colour). |
| **Stock Level** | The number of units of a SKU available for sale at a given moment. |
| **Reservation** | A temporary hold placed on Stock Level units when a shopper adds an item to their Cart. Released if the cart expires without checkout. Converted to a deduction on Order confirmation. |
| **Catalogue** | The full set of Products available in the store. |
| **Out of Stock** | A SKU with a Stock Level of zero. Cannot be added to a Cart. |
| **Backorder** | An Order placed for a SKU that is currently Out of Stock, to be fulfilled when stock arrives. (Feature flag controlled.) |

---

## Customers & Accounts

| Term | Definition |
|---|---|
| **Customer** | A registered user with an account. Has a persistent profile, address book, and order history. |
| **Guest** | A shopper who has not signed in and has no account. Identified by a short-lived Guest Session. |
| **Guest Session** | A server-side session created at the start of Guest Checkout. Stores cart contents and checkout progress. Expires after 24 hours of inactivity. |
| **Account** | The persistent record tied to a Customer. Holds authentication credentials, saved addresses, and payment methods. |
| **Address Book** | The collection of saved shipping addresses on a Customer Account. |

---

## Payments

| Term | Definition |
|---|---|
| **Payment Method** | The mechanism used to pay for an Order (e.g. credit card, digital wallet). |
| **Payment Intent** | A record of a requested payment transaction sent to the payment gateway. Has a status: `created → authorised → captured → failed → cancelled`. |
| **Authorisation** | The step where the payment gateway confirms the payment method is valid and funds are available, without yet capturing money. |
| **Capture** | The step where authorised funds are transferred. Triggered on Order confirmation. |
| **Refund** | A reversal of a Capture. Triggered when an Order or Order Line is cancelled after payment. |

---

## Shipping & Fulfilment

| Term | Definition |
|---|---|
| **Shipping Method** | A carrier + service level option presented at checkout (e.g. "Standard – 3–5 days", "Express – next day"). |
| **Shipping Rate** | The cost of a Shipping Method for a given Order. |
| **Fulfilment** | The physical picking, packing, and dispatching of items in an Order. |
| **Tracking Number** | A carrier-issued reference allowing the Customer to track their shipment. |

---

## System / Technical

| Term | Definition |
|---|---|
| **Idempotency Key** | A client-generated token sent with mutating API requests to ensure safe retries. The server returns the same response for duplicate requests with the same key. |
| **Webhook** | An HTTP callback sent by a service to notify another service of an event (e.g. `order.confirmed`, `payment.captured`). |
| **Event** | An immutable record of something that happened in the system. The basis for inter-service communication via message queue. |
| **Saga** | A distributed transaction pattern used to coordinate state changes across multiple microservices (e.g. reserving stock and capturing payment during checkout). |
