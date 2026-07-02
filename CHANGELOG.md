# Changelog — MaGuru_MonoPayment

## 1.1.2 — 2026-07-02

### Fixed
- **Webhook status updates silently dropped on equal `modifiedDate`** — Monobank's `modifiedDate` has 1-second resolution, so a legitimate status transition (e.g. `processing` → `hold`) can share the exact timestamp with the previous webhook. The H2 stale-webhook guard used `<=` instead of `<`, discarding these updates; invoice status got stuck and admin actions gated on the current status (e.g. Capture) never appeared.

### Changed
- `Acquiring API` config group (API Token, Validate Token button, Payment Webhook URL note, API Base URL dev override) moved here from `MonoCore` — same admin section/path, same config values; only appears once `MonoPayment` is installed
- Added missing DocBlocks to `_construct()` overrides (Data/ResourceModel/Collection), `configure()` in Console Commands, `Cron::execute()`, and several private helper methods per project DocBlock conventions
- Documented required message queue consumer setup (`app/etc/env.php` → `cron_consumers_runner`) in README — without it, Hold auto-capture, invoice cancel/remove, and receipt emails silently never run

## 1.1.1 — 2026-07-01

### Fixed
- **Saved Cards page 404** — customer account nav link and `DeleteCard` redirects pointed to `mono/account/cards` instead of the module's actual frontName `mono-payment/account/cards`
- **Saved Cards layout never applied** — `mono_account_cards_index.xml` renamed to `mono_payment_account_cards.xml` to match the controller's full action name; the block was silently ignored before
- **Wallet `masked_pan` wiped on capture** — `OrderStatusHandler` no longer nulls out the invoice's `payment_method`/`masked_pan` when a webhook's `paymentInfo` omits those fields; Hold-capture success webhooks don't repeat card details, so the previously known value is now preserved
- **Saved card token never reached the gateway** — added `Observer/DataAssignObserver` on `payment_method_assign_data_monopay`; `mono_card_token` submitted from checkout was never copied into the payment's additional information, so "pay with saved card" silently fell back to the full redirect flow every time
- **`INVALID_MERCHANT_PAYM_INFO` on saved-card payments** — `WalletService::charge()` now dispatches `mono_payment_before_invoice_create` / `mono_payment_before_invoice_send`, the same events `InvoiceService::create()` uses, so MonoFiscal's basket building and split-receiver injection also apply to wallet payments (Monobank rejects an empty `basketOrder` when fiscalization or split payments are enabled)
- **Hold ignored for saved-card payments** — `WalletService::charge()` now sends `paymentType` from config to `/api/merchant/wallet/payment` (previously omitted, so Monobank defaulted to immediate `debit` regardless of the merchant's Hold setting) and stores the same value on the local invoice instead of a hardcoded `debit`

### Added
- `SaveWalletAfterSuccess` falls back to the invoice's `masked_pan` when the webhook payload doesn't include one
- `Test/Unit/Observer/DataAssignObserverTest` — 3 tests
- `WalletServiceTest::testChargeSendsPaymentTypeFromConfigInPayload` — Hold `paymentType` propagation
- `SaveWalletAfterSuccessTest::testFallsBackToInvoiceMaskedPanWhenPayloadHasNone`

## 1.1.0 — 2026-07-01

### Added
- `Gateway/Command/FinalizeCommand` — gateway `capture` command for Hold payment flow; calls `POST /api/merchant/invoice/finalize` with amount in kopiyky
- `checkout_button_title` config field — text on the checkout payment button is now configurable via admin (Stores → Config → Payment Methods → Monobank → Checkout Button Label); default: `Pay via Monobank`
- `Model/Config/Source/QrAmountType` — source model for QR amount type select (`merchant` / `client or fix`)
- `qr_amount_type` config field — controls whether order amount is sent to Monobank API for QR terminal invoices; `client`/`fix` terminals reject any amount field, the module strips it automatically
- `i18n/uk_UA.csv` — complete Ukrainian translations (180 strings) covering all admin config fields, invoice statuses, hold/capture flow, saved cards, QR, split, subscriptions, fiscal hooks
- `i18n/en_US.csv` — complete English translations (182 strings) matching uk_UA coverage
- `FinalizeCommandTest` — 4 unit tests: amount conversion to kopiyky, rounding, logging, skip on empty invoiceId
- `CreateInvoiceCommandTest` — full rewrite: 5 tests for redirect no-op and wallet charge flows

### Fixed
- **Double invoice/create bug** — `CreateInvoiceCommand` was executing during `placeOrder` (orderId = 0), then `Start.php` couldn't find the invoice and created a second one; redirect flow is now a no-op in `CreateInvoiceCommand`, invoice is created in `Start.php` after order is saved with a real ID
- **Premature Magento invoice** — `payment_action` changed from `authorize_capture` to `authorize`; Magento no longer auto-creates an invoice during `placeOrder` for a payment that has not been captured yet; invoice is created by `OrderStatusHandler` on webhook `success` only
- **Hold capture amount** — `Controller/Adminhtml/Order/Capture.php` now passes the invoice amount to `invoiceService->finalize()` (was passing no amount, causing "finalization amount exceeds hold amount" API error)

### Changed
- `CommandPool` — `capture` command now maps to `FinalizeCommand` instead of `CreateInvoiceCommand`
- `CreateInvoiceCommand` — redirect flow sets `isTransactionClosed = false` and returns; wallet flow unchanged (charges via `WalletService`)

## 1.0.5 — 2026-05-18

### Added
- `Controller/Adminhtml/Invoice/IndexTest` (2 tests) — ADMIN_RESOURCE constant + execute() returns page with title/menu
- `Controller/Adminhtml/Qr/IndexTest` (2 tests) — same, 'Monobank QR Terminals'
- `Controller/Adminhtml/Statement/IndexTest` (2 tests) — 'Monobank Statement'
- `Controller/Adminhtml/Submerchant/IndexTest` (2 tests) — 'Monobank Split Receivers'
- `Controller/Adminhtml/Subscription/IndexTest` (2 tests) — 'Monobank Subscriptions'
- Pattern: `getMockBuilder()->disableOriginalConstructor()` + `ReflectionProperty::setValue($pageFactory)` + mock chain `Page → PageConfig → Title`

## 1.0.4 — 2026-05-18

### Added
- `MonoPayButtonServiceInterface::importPublicKey(string $keyValue, string $keyName = ''): string` — register ECDSA public key with Monobank (`POST /api/merchant/monopay/pubkey-import`), returns keyId
- `MonoPayButtonServiceInterface::deletePublicKey(string $keyId): void` — delete a registered key (`POST /api/merchant/monopay/pubkey-delete`)
- `MonoPayButtonServiceInterface::listPublicKeys(): array` — list all registered keys (`GET /api/merchant/monopay/pubkey-list`)
- `Console/Command/MonoPayButtonSetupCommand` (`mono:monopay-button:setup`) — derives public key from configured PEM, imports to Monobank, prints keyId
- `MonoPayButtonServiceTest`: 3 new tests for import/delete/list
- `MonoPayButtonSetupCommandTest`: 4 tests (no key configured, invalid PEM, happy path + ApiException)

## 1.0.3 — 2026-05-18

### Added
- `Block/InfoTest` (3 tests): coverage for `getSpecificInformation()` — includes/omits `mono_invoice_id` based on value

## 1.0.2 — 2026-05-18

### Added
- `InvoiceServiceTest`: 2 new direct tests for `InvoiceService::finalize()` — with and without amount

## 1.0.1 — 2026-05-18

### Added
- `Block\Adminhtml\Order\InvoiceInfo::getStatementProfitAmount(string $invoiceId): ?int` — shows Monobank net payout (after fees) from statement in admin order view
- `invoice_info.phtml`: new **Net Amount** row (profitAmount from `mono_payment_statement`) between Captured Amount and Payment Type
- 3 new unit tests for `InvoiceInfoTest` (profit amount happy path, not found, null)

## 1.0.0 — 2026-05-17

### Added
- Payment method `monopay` via `Magento\Payment\Model\Method\Adapter`
- **Redirect flow** — `afterPlaceOrder` → `/mono/payment/start` → Monobank `pageUrl`
- **iFrame mode** — inline modal with `<iframe allow="payment *">` + `postMessage` return flow
- **Hold/Capture** — `paymentType: hold`; capture on shipment via `invoice/finalize`
- **Saved Cards (Wallet)** — tokenization on payment; customer account `/mono/account/cards`; one-click recharge via `wallet/payment`
- **MonoPay Button** — ECDSA P-256 signed initialization; JS SDK integration
- **QR Payments** — `qrId` injection; `QrService` (list/getDetails/resetAmount); admin page
- **Split / Marketplace** — `splitReceiverId` in `basketOrder` via `AddSplitReceiverToInvoice` observer; `SubmerchantService`
- **Subscriptions** — full CRUD via `SubscriptionService`; `mono_payment_subscription` table; admin grid; webhook handler at `POST /mono/payment/subscription`
- **Receipt Email** — `sendReceipt()` via Monobank API; `SendReceiptAfterSuccess` observer; config flag
- **Statement Reconciliation** — `StatementService::sync()`; admin grid with CSV export; CLI; daily cron
- **Currency Rates** — `Monobank extends AbstractImport`; every-4h cron
- **Webhook controller** `POST /mono/payment/webhook` — ECDSA-verified, always HTTP 200
- **Subscription webhook** `POST /mono/payment/subscription` — handles chargeUrl and statusUrl events
- **Return/Cancel controllers** — `GET /mono/payment/return`, `GET /mono/payment/cancel`
- **Observers** — ShipmentSaveAfter (finalize hold), CancelAfter (remove invoice), CreditMemoSaveAfter (refund), SaveWalletAfterSuccess
- **Events** — `mono_payment_invoice_success`, `mono_payment_before_invoice_create`, `mono_payment_before_invoice_cancel`, `mono_payment_before_invoice_send`
- **Status polling cron** `*/5 * * * *` — fallback for missed webhooks
- **Admin grids** — Invoices, Statement, QR Terminals, Split Receivers, Subscriptions
- **REST API** — getForOrder, getCheckoutUrl, syncStatus
- DB tables: `mono_payment_invoice`, `mono_payment_wallet`, `mono_payment_statement`, `mono_payment_subscription`
- Virtual logger → `var/log/mono_payment.log`
- **Admin order view block** `Block/Adminhtml/Order/InvoiceInfo` — shows invoice ID, status, amount, pageUrl inline in order view
- **WalletService** reads `tdsUrl` (3DS redirect) with fallback to `pageUrl`; Start controller handles empty `pageUrl` (wallet without 3DS): iFrame → `direct: true`, redirect → success page
- 229 unit tests, PHPStan Level 8
- `Gateway/Config/Config` — unit tests: getPaymentType default, getInvoiceValidity default, isIframeMode, getAgentFeePercent
- `Ui/Component/Listing/Column/AmountKopecks` — unit tests: format kopecks → UAH, null skip, no items
