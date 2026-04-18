# User Personas

> These personas represent the real people who interact with our product.
> All Feature Specs MUST reference at least one persona to ground acceptance criteria in user needs.

---

## 1. Grace — The Casual Shopper

**Role:** Storefront end-user (unauthenticated)

**Background:** Grace shops online a few times a month. She values speed and simplicity above all else. She rarely creates accounts on new sites because she doesn't want yet another password to manage. When she finds what she wants, she wants to check out as quickly as possible.

**Goals:**
- Find and buy a product in as few steps as possible.
- Not be forced to create an account.
- Get a confirmation that her order was placed successfully.

**Frustrations:**
- Being forced to register before checkout.
- Confusing or multi-page checkout flows.
- Unclear error messages when her payment fails.

**Primary entry points:** Storefront catalogue, product pages, Cart, Guest Checkout.

---

## 2. Marcus — The Returning Customer

**Role:** Storefront end-user (authenticated)

**Background:** Marcus shops regularly and has created an account to save time. He expects the site to remember his address and payment methods. He checks his order history and tracks deliveries through his account dashboard.

**Goals:**
- Check out quickly using saved address and payment method.
- Track the status of current and past orders.
- Get notified when an order ships.

**Frustrations:**
- Being asked to re-enter details he has already saved.
- Order status that doesn't update in real time.
- Notification emails that arrive late or contain wrong information.

**Primary entry points:** Sign-in, Checkout (authenticated), Order History, Account Settings.

---

## 3. Priya — The Operations Manager

**Role:** Internal admin user

**Background:** Priya works in the operations team and uses the admin portal all day. She monitors incoming orders, resolves fulfilment issues, manages stock levels, and handles customer complaints involving refunds or cancellations.

**Goals:**
- See a real-time view of all new and in-progress orders.
- Quickly find an order by reference number or customer email.
- Issue refunds or cancel orders without needing engineering help.
- Get alerted when an order gets stuck or a payment fails.

**Frustrations:**
- Having to switch between multiple tools to complete a single task.
- Bulk actions that time out or give no feedback.
- Poor search — having to scroll through a long list to find one order.

**Primary entry points:** Admin Order List, Order Detail, Inventory Management, Refunds.

---

## 4. Dev — The Platform Engineer

**Role:** Internal technical user

**Background:** Dev builds and maintains the services that power the platform. He works with APIs, deploys services, reviews contracts, and debugs production incidents. He interacts with this spec hub to understand what is being built and why.

**Goals:**
- Understand the full scope of a feature from a single document (Work Package).
- Know exactly which API endpoints and data schemas a feature touches.
- Be able to implement a WP without needing to ask clarifying questions.

**Frustrations:**
- Specs that are vague or contradict the API contract.
- Work packages that depend on reading other work packages.
- Acceptance criteria that cannot be turned into automated tests.

**Primary entry points:** Work Packages, OpenAPI contracts, data schemas, ADRs.
