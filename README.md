# Crypto Payment Gateway

**Type:** Diploma project (engineering)  
**Author:** Stanislav Kustov  
**Topic:** External cryptocurrency payment gateway for e‑commerce

This project is a **crypto payment gateway** that lets online shops accept stablecoins
(USDT / USDC) across the different networks (primary **Arbitrum**), similar to how they accept card payments.

The goal is to show a realistic backend design:
- clear API for merchants,
- proper payment lifecycle,
- clean architecture with separate layers and modules,
- focus on reliability and safety.

> ⚠️ Status: the codebase is under active development.
> Future changes to current architecture are possible.

---

## High‑Level Overview

A merchant does **not** handle blockchain details directly. Instead:

1. The merchant backend calls the gateway’s API to **create an invoice** with a fixed
   price in **USD**.
2. The gateway calculates how much **USDT/USDC** is needed and allocates a **unique
   deposit address** for this invoice.
3. The buyer is redirected to a **payment page** with amount, address and QR code.
4. When the buyer sends funds, the gateway listens to **on‑chain events** (via Alchemy)
   and detects the payment.
5. After enough confirmations, the gateway marks the invoice as **confirmed** and sends
   a **signed callback** back to the merchant.

So the merchant sees this gateway like a normal card payment provider:
simple REST API and callbacks, while the project hides blockchain complexity.

---

## Architecture at a Glance

The backend is built as a **modular monolith** with three Spring Boot applications
inside one Gradle project:

- `apps/api` — public REST API and payment pages
- `apps/listener` — webhook listener for blockchain events
- `apps/worker` — background jobs (status updates, retries, callbacks)

Shared business logic is in separate modules under `modules/*`:

- `modules/domain` — core domain model: `Invoice`, `PaymentIntent`, `Merchant`,
  `Money`, `Token`, etc.
- `modules/application` — use cases: `CreateInvoice`, `CreatePaymentIntent`,
  `ConfirmPayment`, `ExpireInvoice`, …
- `modules/infrastructure` — adapters: PostgreSQL, Flyway migrations, JPA,
  Alchemy client, HTTP clients for callbacks.
- `modules/security` — API keys, HMAC signatures, idempotency support.
- `modules/pricing` — conversion from fiat amount (USD) to token amount
  (USDT/USDC), fee and rounding rules.
- `modules/shared` — common error types, utilities and shared helpers.

This structure follows ideas from **Robert C. Martin’s “Clean Architecture”**
and **Domain‑Driven Design** (Eric Evans):

- the **domain layer** does not depend on frameworks;
- **use cases** depend on the domain, but not on the database or HTTP;
- infrastructure modules are simple adapters around the domain and use cases.

It is not a full textbook DDD implementation, but it uses these principles to keep
business rules clear and testable.

---

## Three Applications in Runtime

### 1. API (`apps/api`)

Responsibilities:

- Public REST API for merchants:
  - `POST /api/invoices` – create invoice
  - `GET /api/invoices/{id}` – read invoice
  - `POST /api/invoices/{id}/intents` – create payment intent
  - `GET /api/invoices/{id}/status` – get live status
- Serves the buyer payment page (`/pay/{invoiceId}`) with QR code, address, and timer.
- Handles **merchant authentication** via API key.

### 2. Listener (`apps/listener`)

Responsibilities:

- Receives **webhooks** from Alchemy:
  - raw transactions,
  - ERC‑20 `Transfer` events for USDT/USDC on Arbitrum.
- Normalizes on‑chain data into database tables:
  - `transactions`, `token_transfers`, `intent_funds`, …
- Links incoming transfers to payment intents by:
  - deposit address,
  - token contract,
  - chain.
- Writes business events into an **outbox** table for safe processing.

### 3. Worker (`apps/worker`)

Responsibilities:

- Reads events from the **outbox** and executes them:
  - update invoice and payment intent states (`PENDING`, `PAID`, `CONFIRMED`,
    `UNDERPAID`, `OVERPAID`, `EXPIRED`);
  - trigger **callbacks** back to the merchant.
- Tracks **confirmations** (number of blocks) for each transaction.
- Handles **retries** with backoff when callbacks fail (for example, merchant
  endpoint is down).

---

## Payment Lifecycle

### 1. Invoice Creation

- Merchant server calls `POST /api/invoices` with:
  - amount in **USD** (e.g. `10.00`),
  - preferred tokens and networks (e.g. `USDT/Arbitrum`, `USDC/Arbitrum`),
  - merchant order id,
  - callback URL.
- Pricing module calculates how many USDT/USDC are needed, using atomic units
  (6 decimal places for ERC‑20 stablecoins).
- Address module allocates a **unique EVM address** for this invoice
  (rule: *1 invoice = 1 address*).
- An `invoice` row is stored in PostgreSQL with:
  - `status = PENDING`,
  - fixed price, chosen token options,
  - expiry time (e.g. 15 minutes).

### 2. Payment Intent (buyer chooses how to pay)

- Buyer visits `/pay/{invoiceId}`.
- Gateway shows allowed **payment options** for this invoice:
  - for example `USDT · Arbitrum` and `USDC · Arbitrum`.
- When buyer chooses an option, the gateway creates a **payment intent**:
  - selected token and chain,
  - **deposit address**,
  - exact amount in atomic units,
  - status `AWAITING_FUNDS`.

### 3. On‑Chain Funds and Confirmation

- Listener gets webhooks from Alchemy with transactions and ERC‑20 logs.
- It normalizes data and links transfers to the payment intent by deposit
  address and token contract.
- Rules for `PAID`:
  - sum of received amount ≥ target amount,
  - within invoice expiry time.
- Rules for `CONFIRMED`:
  - transaction has at least **N confirmations** (N is configurable; small value
    for Arbitrum).
- Worker updates invoice and intent statuses and publishes events for callbacks.

### 4. Callback to Merchant

- When an invoice becomes `CONFIRMED`, worker sends
  a **signed HTTP POST** to the merchant callback URL with:
  - invoice id, merchant order id, final status,
  - paid amount and token,
  - transaction hash.
- Callback is signed with **HMAC‑SHA256** (header like `X-Signature`), so the
  merchant can verify that the message is from the gateway.
- If merchant is temporarily unavailable, worker retries with exponential
  backoff until a limit is reached.

---

## Reliability and Safety

The project uses several patterns to make payment handling safe:

- **Idempotency keys**  
  - `Idempotency-Key` header for `POST /api/invoices` – safe retries without
    creating duplicate invoices.  
  - Idempotency for webhooks – repeated events from Alchemy do not break state.

- **Outbox pattern**  
  All important side effects (status changes, callbacks) are written to an
  `outbox` table in the same transaction as state changes.  
  The worker processes outbox rows later. This protects the system from
  partial failures and keeps state consistent.

- **HMAC signatures**  
  Every outbound callback to a merchant is signed. Merchants can trust that
  the message was not changed on the way.

- **Audit and logs**  
  The system stores who changed what and when (API or admin), and keeps logs of
  webhooks and callbacks for debugging.

---

## Technology Stack

Backend:

- **Language:** Java 21  
- **Framework:** Spring Boot 3.x  
- **Build:** Gradle with **Kotlin DSL**  
- **Database:** PostgreSQL  
- **Migrations:** Flyway  
- **Persistence:** Spring Data JPA  
- **Testing (planned):** JUnit 5, Testcontainers for integration tests

Blockchain & crypto:

- **Network:** Arbitrum (Ethereum Layer 2) (The project is flexible for more blockchain networks such as SOL, BRC20, etc.)
- **Tokens:** USDT, USDC
- **Provider:** Alchemy (webhooks and RPC)

Architecture & design:

- Modular monolith with separate apps and modules  
- Clean Architecture style layering (inspired by Robert C. Martin), etc.
- Domain‑driven design ideas for the core model and use cases  
- Outbox pattern for reliable messaging

---

## Project Status and Roadmap
The project is under active development.
The source code and tests will be published once the core service is stable.
