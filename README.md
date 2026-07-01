# MaGuru MonoPayment — Monobank Acquiring Gateway for Magento 2

Full-featured Monobank Acquiring payment gateway. Supports redirect flow, iFrame embedded checkout, saved cards (wallet tokenization), MonoPay Button JS widget, QR payments, subscriptions, split payments, hold/capture, refund, fiscal integration hooks, and more.

**Requires:** `maguru/module-mono-core`

---

## Requirements

| Component | Version |
|-----------|---------|
| Magento Open Source / Adobe Commerce | 2.4.4+ |
| PHP | 8.1+ |
| MaGuru MonoCore | ^1.0 |

---

## Installation

```bash
composer require maguru/module-mono-core maguru/module-mono-payment
bin/magento module:enable MaGuru_MonoCore MaGuru_MonoPayment
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento cache:flush
```

---

## Configuration

**Stores → Configuration → Payment Methods → Monobank (MaGuru)**

| Field | Description |
|-------|-------------|
| Enabled | Enable/disable the payment method |
| Title | Display name shown on checkout |
| Payment Type | `debit` (immediate charge) or `hold` (authorize, capture on shipment) |
| Display Type | `redirect` (to Monobank page) or `iframe` (inline modal) |
| Invoice Validity | Seconds before invoice expires (default: 3600) |
| Save Card (Tokenization) | Allow customers to save cards for future purchases |
| Send Receipt Email | Send Monobank electronic receipt to customer after payment |
| QR Enabled | Inject `qrId` into invoice for QR terminal payments |
| QR Terminal ID | QR terminal ID from your Monobank merchant account |
| QR Amount Type | `merchant` — send order amount per invoice (use for `amountType: merchant` terminals); `client/fix` — do not send amount (for `amountType: client` or `fix` terminals) |
| Split Enabled | Inject `splitReceiverId` into basketOrder (marketplace split) |
| Split Receiver ID | Monobank split receiver ID |
| Agent Fee Percent | Optional agent commission percent |
| MonoPay Button | Enable MonoPay Button JS widget |
| MonoPay Button Key ID | Key ID from Monobank pubkey list |
| MonoPay Button Private Key | ECDSA P-256 private key PEM (encrypted) |
| Debug Logging | Write detailed logs to `var/log/mono_payment.log` |

---

## Features

### Checkout Flows
- **Redirect** — customer is redirected to Monobank `pageUrl`, then back to your store
- **iFrame** — Monobank payment page loads in an inline modal overlay; customer never leaves checkout
- **Saved Card** — return customers can pay with a previously saved card in one click
- **MonoPay Button** — Monobank's native JS payment button with ECDSA-signed initialization

### Payment Types
- **Debit** — immediate charge
- **Hold** — authorize on order placement, capture on shipment creation

### Saved Cards (Wallet)
- Cards are tokenized via Monobank after first payment
- Customer account page at `/mono/account/cards` — view and delete saved cards
- Merchant-initiated recurring charges via `WalletService::charge()`

### QR Payments
- Inject `qrId` into any invoice for QR terminal-based payments
- Admin page: **System → Monobank → QR Terminals** (live list from Monobank API)
- `QR Amount Type` config: `merchant` terminals require `amount` in the API call; `client`/`fix` terminals reject any amount field — the module strips it automatically (including `basketOrder.sum`/`total`)
- Only `merchant`-type QR terminals are compatible with the full API checkout flow

### Split / Marketplace
- Inject `splitReceiverId` into `basketOrder` per order item
- Admin page: **System → Monobank → Split Receivers** (live list from Monobank API)

### Subscriptions (Recurring)
- Full CRUD: create, status, list, edit (cancel), remove, payment history
- Local DB tracking in `mono_payment_subscription` table
- Admin grid: **System → Monobank → Subscriptions**
- Webhook handler `POST /mono/payment/subscription` for charge and status events

### Statement Reconciliation
- Daily sync via `GET /api/merchant/statement`
- Admin grid with CSV export: **System → Monobank → Statement**
- CLI: `bin/magento mono:statement:sync [--from=Y-m-d] [--to=Y-m-d]`
- Cron: daily at 08:00

### Currency Rates
- `GET /bank/currency` (no auth required) → `directory_currency_rate`
- Cron: every 4 hours

### Order Lifecycle
- `payment_action = authorize` — no Magento invoice is created during `placeOrder`; invoice is created only after webhook `success` (debit) or after Hold capture → webhook `success` (hold)
- Hold invoice → auto-capture on shipment (`ShipmentSaveAfter` → `finalize` queue)
- Hold manual capture → **Capture Payment** button in admin order view → `POST /api/merchant/invoice/finalize` → webhook `success` → Magento invoice created
- Cancel invoice on order cancel (`CancelAfter`)
- Refund via credit memo (`CreditMemoSaveAfter`)
- Status polling cron every 5 minutes (fallback if webhook fails)

### Fiscal Integration Hooks
- Dispatches `mono_payment_before_invoice_create` — allows MonoFiscal to inject `basketOrder`
- Dispatches `mono_payment_before_invoice_cancel` — allows MonoFiscal to inject fiscalization items
- Dispatches `mono_payment_invoice_success` — triggers fiscal check fetch and receipt email

---

## Webhook URLs

Configure these in your Monobank merchant cabinet:

| URL | Purpose |
|-----|---------|
| `POST https://yourstore.com/mono/payment/webhook` | Invoice status updates |
| `POST https://yourstore.com/mono/payment/subscription` | Subscription charge / status events |
| `GET  https://yourstore.com/mono/payment/return` | `successUrl` / `redirectUrl` after payment |
| `GET  https://yourstore.com/mono/payment/cancel` | `failUrl` on payment failure |

---

## REST API

```
GET  /V1/mono/payment/order/:orderId/invoice         [admin]
GET  /V1/mono/payment/order/:orderId/checkout-url    [anonymous]
POST /V1/mono/payment/invoice/:invoiceId/sync        [admin]
```

---

## Admin Menu

**System → Monobank:**
- Configuration
- Invoices
- Statement
- QR Terminals
- Split Receivers
- Subscriptions

---

## Tests

```bash
bin/clinotty php vendor/bin/phpunit -c app/code/MaGuru/MonoPayment/Test/Unit/phpunit.xml
```

420 unit tests · 630 assertions · PHPStan Level 8 ✅

---

## Support

- Email: maguru.sup@gmail.com
- Issues: via Magento Marketplace order page

---

## License

Proprietary — one-time license per domain
