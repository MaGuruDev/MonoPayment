# MaGuru Monobank Payment — User Guide

**Version:** 1.1.2  
**Compatible with:** Adobe Commerce / Magento Open Source 2.4.4 – 2.4.x  
**PHP:** 8.1 or higher  
**Requires:** MaGuru Monobank Core  
**License:** Paid

---

## Overview

MaGuru Monobank Payment integrates Monobank Acquiring as a full-featured payment method for your Adobe Commerce store. It supports standard card payments, authorization holds, saved cards, Apple Pay / Google Pay via MonoPay Button, QR terminal payments, split payments for marketplaces, automatic refunds, fiscal receipt sending, and more.

### Feature Summary

| Feature | Description |
|---|---|
| **Standard Card Payment** | Redirect or iFrame checkout with the Monobank payment form |
| **Hold / Capture** | Authorize-only payments: hold funds, capture on shipment |
| **Saved Cards** | Customers save cards for one-click future purchases |
| **MonoPay Button** | Apple Pay / Google Pay quick-pay button at checkout |
| **QR Terminal Payments** | Customers scan a Monobank QR code to pay |
| **Split Payments** | Route part of a payment to a submerchant account |
| **Automatic Refunds** | Refund via Monobank when a Magento credit memo is created |
| **Send Receipt** | Email the Monobank payment receipt to the customer |
| **Currency Rate Provider** | Fetches UAH exchange rates from Monobank every 4 hours |
| **Bank Statement Sync** | Syncs acquiring statement daily; admin grid shows profit |
| **Reversal Notification** | Customer email when Monobank reverses a payment |

---

## Requirements

| Requirement | Version / Note |
|---|---|
| Adobe Commerce / Magento Open Source | 2.4.4 or higher |
| PHP | 8.1 or higher |
| MaGuru Monobank Core | Must be installed and configured first |
| Monobank Acquiring account | Active account with payment API enabled |

---

## Installation

```bash
composer require maguru/module-mono-payment
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento cache:flush
```

Verify:

```bash
bin/magento module:status MaGuru_MonoPayment
```

### Message Queue Setup (required)

MonoPayment processes several operations asynchronously — Hold auto-capture on shipment, invoice cancel/remove, and receipt emails. These go through Magento's message queue and **do nothing until a consumer is running**; Magento does not start consumers by itself.

Add the following to `app/etc/env.php`:

```php
'cron_consumers_runner' => [
    'cron_run' => true,
    'max_messages' => 200,
    'consumers' => [
        'mono_payment.invoice.cancel',
        'mono_payment.invoice.finalize',
        'mono_payment.invoice.remove',
        'mono_payment.invoice.receipt',
    ]
],
```

Magento's own cron will then run these consumers automatically — no extra process needed. If this is skipped, Hold orders will never auto-capture on shipment and receipt emails will never send, with no error shown anywhere.

---

## Configuration

Navigate to **Stores → Configuration → Sales → Payment Methods → Monobank Payment**.

> **API credentials live elsewhere.** The Monobank **API Token**, the **Validate Token** button, the **Payment Webhook URL** note, and the **API Base URL (Dev Override)** field are configured under **Stores → Configuration → MaGuru → Monobank Integration → Acquiring API** — not on this payment-method page. See the *Monobank Core* user guide for details on that section.

---

### Section 1: General

| Setting | Description | Default |
|---|---|---|
| **Enabled** | Activates the payment method at checkout | No |
| **Title** | Name shown to customers at checkout (e.g., "Pay with Monobank") | Monobank (Visa/Mastercard, Apple Pay, Google Pay) |
| **Checkout Button Label** | Text shown on the payment submit button in checkout | Pay via Monobank |
| **Payment Type** | `debit` — charge immediately; `hold` — authorize only, capture on shipment | debit |
| **Checkout Display Mode** | `redirect` — customer leaves the store to pay; `iframe` — payment form embedded on the page | redirect |
| **Invoice Validity (seconds)** | How long the Monobank invoice stays open before expiring | 3600 (1 hour) |
| **New Order Status** | Order status assigned to newly placed orders paid via Monobank | Pending |
| **Debug Logging** | Write detailed payment request/response data to the log | No |

---

### Section 2: Saved Cards

| Setting | Description | Default |
|---|---|---|
| **Save Cards (Tokenization)** | Allows logged-in customers to save their card during checkout. Requires Wallet/Tokenization to be activated by Monobank support | No |
| **Enable MonoPay Button (JS Widget)** | Shows Apple Pay / Google Pay button at checkout | No |
| **Generate & Register Key** | Button that generates a new ECDSA P-256 key pair, registers the public key with Monobank, and fills the Key ID and Private Key fields automatically | — |
| **MonoPay Button Key ID** | ECDSA public key ID from Monobank (required for MonoPay Button) | — |
| **MonoPay Button Private Key (PEM)** | Your ECDSA P-256 private key (stored encrypted; required for MonoPay Button) | — |

> **MonoPay Button setup:** Use the **Generate & Register Key** button above to generate and register the ECDSA key pair automatically. Alternatively, if you already have a private key configured, run the CLI command `bin/magento mono:monopay-button:setup` to derive the public key and complete registration with Monobank.

---

### Section 3: QR Terminal

| Setting | Description | Default |
|---|---|---|
| **Enable QR Payment** | Activates QR terminal payment option at checkout | No |
| **QR Terminal ID** | The terminal ID from your Monobank QR terminal setup | — |
| **QR Amount Type** | How the terminal handles the payment amount. **Merchant sets amount** (`amountType: merchant`) — order amount is sent per invoice; works with standard API checkout. **Client/Fixed** (`amountType: client` or `fix`) — customer enters amount on the terminal or it is fixed; no amount is sent in the API call. Check your terminal's `amountType` via `GET /api/merchant/qr/list`. | Merchant sets amount |

> **Important:** `client` and `fix` terminal types do not support creating invoices with a `qrId`. Use `merchant` type terminals for full API checkout integration.

---

### Section 4: Split Payments

| Setting | Description | Default |
|---|---|---|
| **Default Split Receiver ID** | The submerchant / receiving account ID from Monobank. Required if your account has split receivers configured — used as a fallback when per-product routing is disabled or a product has no receiver set | — |
| **Enable Per-Product Split Routing** | When enabled, each product can use its own receiver ID (set via the "Mono Split" product attribute); the Default Split Receiver ID above is used as a fallback | No |
| **Agent Fee Percent** | Optional commission percentage retained by the marketplace agent (e.g. 2.5). Leave empty to disable | — |

---

### Section 5: Receipt

| Setting | Description | Default |
|---|---|---|
| **Send Receipt Email** | Email the Monobank fiscal receipt link to the customer after successful payment. Does not work with a test token | No |

---

### Section 6: Advanced

| Setting | Description | Default |
|---|---|---|
| **Enable App Deep-Link (monobank://)** | Request an app deep-link along with the payment URL so mobile customers can open the Monobank app directly. Not supported for QR or verification payments | No |

---

## Admin Panels

### Monobank Merchant

**Monobank → Merchant**

Displays merchant account details fetched live from Monobank Acquiring (e.g. merchant name and MCC).

---

### Monobank Invoices

**Monobank → Invoices**

A grid showing all Monobank payment invoices created by your store.

| Column | Description |
|---|---|
| **Invoice ID** | Monobank's internal invoice identifier |
| **Order #** | Magento order increment ID |
| **Status** | Current payment status (see table below) |
| **Amount (kopecks)** | Invoice amount, displayed formatted in UAH |
| **Payment Type** | debit or hold |
| **Payment Method** | Card scheme / wallet used for the payment (when available) |
| **Masked PAN** | Last 4 digits of the card used (when available) |
| **Created At** | Invoice creation timestamp |
| **Updated At** | Timestamp of the last status update |

#### Invoice Statuses

| Status | Meaning |
|---|---|
| **Created** | Invoice created with Monobank, awaiting payment |
| **Processing** | Payment initiated, awaiting customer action or confirmation |
| **Success** | Payment completed successfully |
| **Hold** | Amount authorized and held, awaiting capture |
| **Reversed** | Payment was reversed or refunded by Monobank |
| **Failure** | Payment failed or was declined |
| **Expired** | Invoice expired without payment |

---

### Monobank Statements

**Monobank → Statements**

Displays the daily bank statement records synced from Monobank Acquiring. Each record shows transaction details and profit amount for accounting purposes.

---

### Monobank QR Terminals

**Monobank → QR Terminals**

Lists the configured QR payment terminals linked to your store.

---

### Monobank Split Receivers

**Monobank → Split Receivers**

Lists the submerchant accounts configured for split payment routing.

---

## Admin Order View

When an order was paid via Monobank, the order view page shows a **Monobank Payment** block with:

- Invoice ID, status, amount, payment type, masked PAN
- **Sync Status** button — fetches the latest status from Monobank immediately
- **Send Receipt** button — resends the Monobank payment receipt to the customer (visible for successful payments only)
- **Capture Payment** button — finalizes a hold payment (visible for Hold-type payments only)

---

## Customer Account: My Cards

When Saved Cards is enabled, logged-in customers see a **My Cards** section in their account. They can:

- View saved cards (displayed as masked PAN)
- Delete saved cards
- Select a saved card during checkout to pay without re-entering card details

---

## Hold / Capture Workflow

When **Payment Type** is set to `hold`:

1. At checkout, the amount is authorized (held) on the customer's card — no charge yet
2. The order enters **Payment Review** state; the Monobank invoice status shows **Hold**
3. No Magento invoice is created at this point
4. To capture (charge the held funds):
   - **Automatically on shipment** — when you create a shipment in Magento, the hold is captured automatically
   - **Manually** — use the **Capture Payment** button in the admin order view
5. After successful capture, Monobank confirms via webhook (`success`) and the Magento invoice is created automatically

> If you do not capture within Monobank's authorization window (typically 9 days for `hold` type), the hold is released and no charge occurs.

---

## Refunds

To refund a Monobank payment:

1. Open the order in the Magento admin
2. Create a **Credit Memo** as you normally would
3. The module automatically sends a refund request to Monobank for the credit memo amount

No manual action in Monobank is required — the refund is triggered by the Magento credit memo workflow.

---

## CLI Commands

| Command | Description |
|---|---|
| `bin/magento mono:payment:status-check` | Force a status check for all pending invoices |
| `bin/magento mono:payment:status-check --invoice-id=X` | Force status check for a specific invoice |
| `bin/magento mono:currency:update` | Force-refresh UAH exchange rates from Monobank |
| `bin/magento mono:statement:sync` | Sync the latest bank statement |
| `bin/magento mono:statement:sync --from=YYYY-MM-DD --to=YYYY-MM-DD` | Sync statement for a specific date range |

---

## Cron Jobs

| Job | Schedule | What It Does |
|---|---|---|
| Invoice status check | Every 5 minutes | Polls pending invoices for status updates |
| Currency rate update | Every 4 hours | Refreshes UAH exchange rates from Monobank public API |
| Statement sync | Daily at 8:00 AM | Syncs the daily acquiring statement |

---

## Frequently Asked Questions

**Q: How do I enable Hold / Capture payments?**

Set **Payment Type** to `hold` in the payment configuration. The payment will be held on the customer's card at checkout and automatically captured when you create a shipment. You can also capture manually from the order view using the **Capture Payment** button.

---

**Q: A customer wants a refund. How do I process it?**

Create a standard Magento Credit Memo for the order. The module automatically sends the refund to Monobank — no separate action in the Monobank panel is needed.

---

**Q: An invoice status is stuck at "Processing". How do I fix it?**

Use the **Sync Status** button in the admin order view to fetch the latest status from Monobank immediately. Alternatively, run `bin/magento mono:payment:status-check --invoice-id=X` from the command line.

---

**Q: How do I set up the MonoPay Button (Apple Pay / Google Pay)?**

1. In **Stores → Configuration → Sales → Payment Methods → Monobank Payment → Saved Cards**, enable **Enable MonoPay Button (JS Widget)**
2. Click **Generate & Register Key** to generate an ECDSA P-256 key pair and register it with Monobank automatically
3. Alternatively, if you already have a private key configured manually, run `bin/magento mono:monopay-button:setup` to derive the public key and register it with Monobank

---

**Q: What currencies are supported?**

Monobank Acquiring operates in UAH only. For stores with display currencies other than UAH, the Currency Rate Provider fetches live exchange rates from Monobank's public API every 4 hours so products can be displayed in other currencies, but the actual charge is always in UAH.

---

**Q: A customer's payment was reversed by Monobank. Will they be notified?**

Yes. When Monobank sends a reversal webhook, the module updates the invoice status to **Reversed** and sends an email notification to the customer.

---

## Support

Before contacting support, please gather:

1. The Monobank Invoice ID and Order # from the Invoices grid
2. The invoice status and any error information visible in the admin
3. Contents of `var/log/mono_payment.log` if Debug Logging is enabled
4. The approximate time the issue occurred

---

*MaGuru Monobank Payment — part of the MaGuru Monobank Integration Suite for Adobe Commerce*
