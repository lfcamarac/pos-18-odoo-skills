# Odoo 18 POS — Complete Module Catalog

> Detailed documentation for all 94+ POS-related modules in Odoo 18 Enterprise.

---

## 1. point_of_sale (Core Application)

**Path:** `odoo/addons/point_of_sale/` (core) / `enterprise/point_of_sale/` (enterprise override)  
**Version:** 1.0.2 | **License:** LGPL-3  
**Dependencies:** stock_account, barcodes, web_editor, digest, phone_validation

### Purpose
The main Point of Sale application. Provides the complete POS UI built as a Single Page Application (SPA) using OWL.

### Backend Models

| Model | File | Key Responsibilities |
|-------|------|---------------------|
| `pos.config` | `models/pos_config.py` (1,153 lines) | POS terminal configuration, feature flags, payment methods, hardware settings |
| `pos.session` | `models/pos_session.py` (1,961 lines) | Session lifecycle (open/close), order sequences, cash control, `load_data()` RPC endpoint |
| `pos.order` | `models/pos_order.py` (1,805 lines) | Order CRUD, `sync_from_ui()` RPC, state machine (draft→paid→done), `_process_order()` |
| `pos.order.line` | `models/pos_order_line.py` | Line items, price computation, tax calculation, discount handling |
| `pos.payment` | `models/pos_payment.py` | Payment records, amount tracking |
| `pos.payment.method` | `models/pos_payment_method.py` (226 lines) | Payment method config, terminal selection, journal linking |
| `pos.printer` | `models/pos_printer.py` | Printer management (IoT, Epson, network) |
| `pos.bill` | `models/pos_bill.py` | Bank statement/bill denomination tracking |
| `pos.category` | `models/pos_category.py` | Product categorization for POS |
| `pos.pack.operation.lot` | `models/pos_pack_operation_lot.py` | Lot/serial number tracking |
| `pos.repair.list` | `models/pos_repair_list.py` | Repair order linking |
| `pos.cash.control` | `models/pos_cash-control.py` | Opening/closing cash control |
| `res.config.settings` | `models/res_config_settings.py` | POS settings in Odoo backend |

### Frontend Structure

```
static/src/
├── app/
│   ├── store/
│   │   ├── pos_store.js          (2,351 lines — central state manager)
│   │   ├── pos_hook.js           (usePos() hook)
│   │   ├── router.js             (URL routing override)
│   │   ├── models.js             (empty — handled by data_service)
│   │   └── devices_synchronisation.js  (multi-device sync)
│   ├── models/
│   │   ├── related_models.js     (1,200 lines — Base class, factory)
│   │   ├── data_service.js       (PosData — ORM, IndexedDB, sync)
│   │   ├── data_service_options.js
│   │   ├── pos_order.js          (1,147 lines)
│   │   ├── pos_order_line.js     (761 lines)
│   │   ├── pos_payment.js
│   │   ├── pos_category.js
│   │   ├── product_product.js
│   │   ├── res_partner.js
│   │   └── utils/
│   │       ├── compute_combo_items.js
│   │       ├── currency.js
│   │       ├── indexed_db.js
│   │       ├── order_change.js
│   │       ├── tax_utils.js
│   │       └── unique_by.js
│   ├── screens/
│   │   ├── product_screen/       (main POS UI)
│   │   ├── payment_screen/       (payment processing)
│   │   ├── receipt_screen/       (post-payment receipt)
│   │   ├── ticket_screen/        (order history, refunds)
│   │   ├── login_screen/         (employee PIN login)
│   │   ├── partner_list/         (customer selection)
│   │   ├── scale_screen/         (weighing scale)
│   │   ├── saver_screen/         (save for later)
│   │   └── action_screen/        (embedded Odoo actions)
│   ├── components/               (UI widgets)
│   ├── generic_components/       (reusable: numpad, product_card, etc.)
│   ├── services/                 (report, account_move)
│   └── hooks/                    (custom OWL hooks)
├── overrides/                    (core framework patches)
├── backend/                      (webclient views)
├── xml/                          (customer display templates)
└── utils.js                      (top-level utilities)
```

### Asset Bundles

```
point_of_sale.assets_prod           ← Main production bundle
├── point_of_sale._assets_pos       ← Core POS assets
│   ├── point_of_sale.base_app      ← Bootstrap, bus, barcode
│   ├── OWL, luxon, zxing
│   └── All POS src files
└── point_of_sale/static/src/app/main.js  ← App bootstrap

point_of_sale.customer_display_assets  ← Customer display
point_of_sale.base_tests               ← Test utilities
point_of_sale.assets_debug             ← Dev tools
```

### Key RPC Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `pos.session.load_data(session_id)` | POST | Bootstrap all frontend data |
| `pos.order.sync_from_ui(session_id, orders)` | POST | Sync orders from frontend |
| `pos.session.update_control(session_id, action)` | POST | Session state transitions |

---

## 2. pos_restaurant

**Path:** `odoo/addons/pos_restaurant/`  
**Dependencies:** point_of_sale

### Purpose
Restaurant-specific extensions: floor plans, table management, kitchen printing, bill splitting.

### Backend Models

| Model | Key Fields |
|-------|-----------|
| `pos.config` (inherit) | `is_restaurant`, `floor_ids`, `printer_ids`, `order_printing`, `bill_printing`, `split_method` |
| `restaurant.floor` | `name`, `table_ids` (One2many) |
| `restaurant.table` | `name`, `seats`, `shape`, `position`, `floor_id`, `orders_count` |
| `pos.order` (inherit) | `table_id`, `customer_count`, `to_invoice_backend` |
| `pos.order.line` (inherit) | `note`, `mptr_count` (kitchen print counter), `full_refund_product_id` |
| `pos.printer` (inherit) | Kitchen printer config |

### Frontend
- Floor plan UI component in ProductScreen
- Table selection overlay
- Kitchen order printing via `sendOrderInPreparation()`
- Bill splitting UI in PaymentScreen

---

## 3. pos_hr

**Path:** `odoo/addons/pos_hr/`  
**Dependencies:** point_of_sale, hr  
**Auto-install:** Yes

### Purpose
Employee identification via barcode or PIN. Tracks which employee processed each order.

### Backend Models

| Model | Key Fields |
|-------|-----------|
| `pos.config` (inherit) | `module_pos_hr`, `employee_id` (default employee), `restrict_price_control` |
| `pos.order` (inherit) | `employee_id`, `userid` |
| `pos.session` (inherit) | `login_number` (sequential login count) |
| `hr.employee` (inherit) | `barcode` (for POS login) |

### Frontend
- LoginScreen replaces ProductScreen as first screen when cashier login required
- Employee barcode/PIN input
- Cashier name displayed in header

---

## 4. pos_loyalty

**Path:** `odoo/addons/pos_loyalty/`  
**Dependencies:** loyalty, point_of_sale  
**Auto-install:** Yes

### Purpose
Loyalty program integration: points earning, coupon redemption, gift cards.

### Backend Models

| Model | Key Fields |
|-------|-----------|
| `pos.config` (inherit) | `available_pricelist_ids`, `has_enabled_loytalty_programs` |
| `pos.order` (inherit) | `loyalty_points_generated` |
| `pos.order.line` (inherit) | `reward_id` (for reward lines) |
| `loyalty.program` (inherit) | POS-specific program config |
| `loyalty.card` (inherit) | POS point tracking |

### Frontend
- Loyalty program selection UI
- Gift card code input
- Points balance display
- Reward line creation

---

## 5. pos_self_order

**Path:** `odoo/addons/pos_self_order/`  
**Dependencies:** pos_restaurant, http_routing  
**Auto-install:** Yes (when pos_restaurant present)

### Purpose
Self-order kiosk. Customers browse menu on smartphone via QR code and place orders directly.

### Backend Models

| Model | Key Fields |
|-------|-----------|
| `pos.config` (inherit) | `self_order_mode`, `self_order_qr_code`, `self_order_role`, `table_name` |
| `pos.order` (inherit) | `self_order_partner_id` |
| `pos.session` (inherit) | Self-order data loading |

### Frontend
- Separate SPA with its own asset bundle (`pos_self_order.assets`)
- Mobile-responsive menu browsing
- Table QR code generation
- Order confirmation flow

### Routes
- `/pos/self_order/<order_uuid>` — Order page
- `/pos/self_order/iframe/<order_uuid>` — Embedded iframe

---

## 6. pos_online_payment

**Path:** `odoo/addons/pos_online_payment/`  
**Dependencies:** point_of_sale, account_payment  
**Auto-install:** Yes

### Purpose
Online payment support. Generates payment portal links for POS orders.

### Backend Models

| Model | Key Fields |
|-------|-----------|
| `pos.order` (inherit) | Online payment linking |
| `pos.payment` (inherit) | `online_account_payment_id` |
| `pos.payment.method` (inherit) | `is_online_payment` |
| `pos.session` (inherit) | Online payment session data |

### Frontend
- Payment link generation
- Payment status polling
- QR code display for mobile payment

---

## 7. Payment Terminal Modules

### pos_adyen
**Dependencies:** point_of_sale  
Payment terminal integration via Adyen. Supports card-present transactions, refunds, and cancellation. Uses terminal proxy pattern for async communication.

### pos_stripe  
**Dependencies:** point_of_sale, payment_stripe  
Stripe Terminal integration. Similar pattern to Adyen.

### pos_six
**Dependencies:** point_of_sale  
Six payment terminal (European). Supports card payments via IoT Box or direct.

### pos_paytm, pos_razorpay, pos_pine_labs
**Dependencies:** point_of_sale  
India-specific payment gateways. Terminal-based card payments.

### pos_mercado_pago
**Dependencies:** point_of_sale  
Mercado Pago Smart Point terminal (LATAM).

### pos_tyro
**Dependencies:** point_of_sale  
Tyro payment terminal (Australia).

### pos_viva_wallet
**Dependencies:** point_of_sale  
Viva Wallet terminal (Europe).

### Common Payment Terminal Pattern

All payment terminal modules follow the same architecture:

```python
# Backend
class PosPaymentMethod(models.Model):
    _inherit = 'pos.payment.method'
    use_payment_terminal = fields.Selection(
        selection_add=[('my_terminal', 'My Terminal')],
        ondelete={'my_terminal': 'cascade'}
    )
```

```javascript
// Frontend
import { PaymentInterface } from "@point_of_sale/app/payment/payment_interface";

class PaymentMyTerminal extends PaymentInterface {
    async sendPaymentRequest(amount) {
        // Send to terminal
        // Wait for response
        // Update payment status
    }

    async sendPaymentCancel() {
        // Cancel pending transaction
    }
}
```

---

## 8. pos_preparation_display (Enterprise)

**Path:** `enterprise/pos_preparation_display/`  
**Dependencies:** point_of_sale  
**Auto-install:** Yes

### Purpose
Kitchen/order preparation display. Separate SPA showing orders in workflow stages (To Prepare → Ready → Completed).

### Backend Models

| Model | Purpose |
|-------|---------|
| `pos_preparation_display.display` | Screen configuration with stages, categories, POS config linking |
| `pos_preparation_display.stage` | Workflow stages with colors and alert timers |
| `pos_preparation_display.order` | Shadow order for preparation (One2many to original pos.order) |
| `pos_preparation_display.order.stage` | M2M through table: order ↔ stage with done flag |
| `pos_preparation_display.orderline` | Shadow order lines with todo, cancelled, quantity tracking |
| `pos.order` (inherit) | Overrides `sync_from_ui` to trigger preparation display update |

### Architecture
- **Standalone SPA** served at `/pos_preparation_display/web?display_id=X`
- Own asset bundle: `pos_preparation_display.assets`
- Own controller: `PosPreparationDisplayController`
- BUS communication: `LOAD_ORDERS`, `CHANGE_ORDER_STAGE`, `CHANGE_ORDERLINE_STATUS`

### Key Methods
- `get_preparation_display_data()` — Bulk load for SPA boot
- `process_order(order_id, cancelled, note, note_history)` — Entry from POS sync
- `change_order_stage(stage_id, display_id)` — Move order between stages
- `_process_preparation_changes()` — Diff algorithm: compares POS vs preparation lines
- `reset()` — Archive all open orders

### Frontend
- `preparation_display_service.js` — Service registered in `registry.category("services")`
- Uses `getOnNotified()` for BUS subscriptions
- `usePreparationDisplay()` hook
- Sound notifications on stage changes

---

## 9. pos_iot (Enterprise)

**Path:** `enterprise/pos_iot/`  
**Dependencies:** point_of_sale, iot  
**Auto-install:** Yes

### Purpose
IoT Box integration for POS. Enables hardware devices: printers, scales, payment terminals, customer displays, fiscal data modules.

### Backend Models

| Model | Key Fields |
|-------|-----------|
| `pos.config` (inherit) | `iface_printer_id`, `iface_display_id`, `iface_scanner_ids`, `iface_scale_id` (all M2one/M2many to iot.device) |
| `pos.payment.method` (inherit) | `iot_device_id`, `payment_terminal_ids`, terminal selection: `ingenico`, `worldline` |
| `iot.device` (inherit) | Inherits `pos.load.mixin`. Fields: `iot_ip`, `iot_id`, `identifier`, `type`, `manual_measurement` |

### Frontend
- `iot_printer.js` — extends `BasePrinter`, `sendPrintingJob()`, `openCashbox()`
- `payment.js` — `PaymentIngenico`, `PaymentWorldline` classes
- Device controller pattern: uniform `device.action({action: "...", ...})`
- WebSocket-based device communication

---

## 10. pos_enterprise (Enterprise)

**Path:** `enterprise/pos_enterprise/`  
**Dependencies:** web_enterprise, point_of_sale  
**Auto-install:** Yes

### Purpose
Thin bridge module. Adds enterprise-only features: better IoT Box config views, Studio integration, UrbanPiper feature flags.

### Content
- 2 small Python files (models for pos.config fields)
- No JavaScript, no templates beyond settings views
- Adds `module_pos_iot` and `module_pos_urban_piper` boolean fields

---

## 11. pos_mobile (Enterprise)

**Path:** `enterprise/pos_mobile/`  
**Dependencies:** pos_enterprise, web_mobile

### Purpose
Integrates Odoo Mobile App framework into POS. Pure asset bundling — no Python code.

### Asset Pattern
```python
'assets': {
    'point_of_sale._assets_pos': [
        'web_mobile/static/src/**/*',
        'pos_mobile/static/src/**/*',
    ],
}
```

---

## 12. pos_blackbox_be (Enterprise)

**Path:** `enterprise/pos_blackbox_be/`  
**Dependencies:** pos_iot, l10n_be, web_enterprise, pos_hr, pos_restaurant, pos_discount

### Purpose
Belgian Registered Cash Register. Certified cash system per FPS Finance. Uses Fiscal Data Module (FDM) via IoT Box.

### Backend Models

| Model | Key Features |
|-------|-------------|
| `pos.config` (inherit) | `iface_fiscal_data_module` (M2one to iot.device, type=fiscal_data_module), `certified_blackbox_identifier`. 15+ validation methods |
| `pos.session` (inherit) | `cash_box_opening_number`, `pro_forma_sales_number/amount`, `pro_forma_refund_number/amount`, `correction_number/amount`, `users_clocked_ids`. `_get_user_report_data()` groups by INSZ |
| `pos.order` (inherit) | `blackbox_date`, `blackbox_time`, `blackbox_ticket_counters`, `blackbox_unique_fdm_production_number`, `blackbox_signature`, `blackbox_tax_category_a/b/c/d` (21%/12%/6%/0%), `plu_hash`, `pos_version`. Write whitelist |
| `pos.order.line` (inherit) | `vat_letter` (A=21%, B=12%, C=6%, D=0%). Protected from modification |
| `pos_blackbox_be.log` | Audit log: user, action, date, model_name, record_name |

### Frontend
- 675-line `pos_store.js` patch
- User clocking: `clock()`, `getUserSessionStatus()`, `setUserSessionStatus()`
- Blackbox push pipeline: `pushOrderToBlackbox()` → `pushToBlackbox()` → hardware proxy
- Receipt types: NS (Normal Sale), PS (Pro Forma), NR (Refund), PR (Pro Forma Refund)
- Correction flow with pro-forma pairs

### Compliance
- Cash rounding enforced at 0.05 HALF-UP
- Write protection on registered orders
- IP address tracking for certified devices
- Full audit trail

---

## 13. UrbanPiper Suite (Enterprise)

### pos_urban_piper (Core)
**Dependencies:** pos_preparation_display

Delivery platform integration. Supports Swiggy, Zomato, Uber Eats.

**Backend Models:**
- `pos.config` — `urbanpiper_store_identifier`, `urbanpiper_pricelist_id`, `urbanpiper_fiscal_position_id`, `urbanpiper_payment_methods_ids`, `urbanpiper_last_sync_date`
- `pos.order` — `delivery_status` (placed/acknowledged/food_ready/dispatched/completed/cancelled), `delivery_provider_id`, `delivery_identifier`
- `pos.delivery.provider` — Platform info
- `pos_urban_piper_request` — `UrbanPiperClient` HTTP wrapper

**Controller:** `/urbanpiper/webhook/<event_type>`
- Events: `order_placed`, `order_status_update`, `rider_status_update`
- Schema validation via `data-validator.py`
- JSON auth via `X-Urbanpiper-Uuid` header

### pos_urban_piper_enhancements
**Dependencies:** pos_urban_piper  
**Auto-install:** Yes

Adds scheduling and future delivery notifications:
- `pos.store.timing` — Weekly schedule model
- `pos.order` — `delivery_datetime`, `is_notified`
- Cron: `notify_future_deliveries()` sends `FUTURE_ORDER_NOTIFICATION` BUS event

### pos_restaurant_urban_piper
**Dependencies:** pos_restaurant, pos_urban_piper  
**Auto-install:** Yes

Restaurant UI for delivery orders. Pure JS overrides, no Python models.

### pos_urban_piper_swiggy / _ubereats / _zomato
**Dependencies:** pos_urban_piper  
**Auto-install:** Yes

Platform-specific integrations. Each extends the core with platform-specific mapping.

---

## 14. pos_avatax (Enterprise)

**Path:** `enterprise/pos_avatax/`  
**Dependencies:** point_of_sale, account_avatax  
**Auto-install:** Yes

### Purpose
Avalara tax calculation for POS.

### Backend
`pos.order` inherits `account.external.tax.mixin`:
- `_get_date_for_external_taxes()` → `date_order`
- `_get_lines_eligible_for_external_taxes()` → `self.lines`
- `_get_line_data_for_external_taxes()` → line dicts with product, qty, price, discount
- `_set_external_taxes()` → sets `tax_ids`, recalculates prices
- `get_order_tax_details()` — RPC endpoint called from frontend

### Frontend
No frontend JS — tax calculation via RPC before order completion.

---

## 15. pos_pricer (Enterprise)

**Path:** `enterprise/pos_pricer/`  
**Dependencies:** product, point_of_sale

### Purpose
Electronic shelf label (ESL) integration. Display and update product prices on Pricer digital tags.

### Backend Models

| Model | Key Fields |
|-------|-----------|
| `pricer.store` | `pricer_store_identifier`, `pricer_tenant_name`, `pricer_login`, `pricer_password`. Computed API URLs: `auth_url`, `create_or_update_products_url`, `link_tags_url` |
| `pricer.tag` | `name` (17-char barcode: letter + 16 digits), `product_id` |
| `product.product` (inherit) | `pricer_store_id`, `pricer_tag_ids` |

### Integration
- REST API with JWT Bearer authentication
- Three endpoints: auth, product CRUD, label linking
- Cron-based sync with manual override

---

## 16. pos_barcodelookup (Enterprise)

**Path:** `enterprise/pos_barcodelookup/`  
**Dependencies:** point_of_sale, product_barcodelookup

### Purpose
Product creation via barcode camera scan using Barcode Lookup API.

### Content
- Pure JavaScript override module
- Zero Python code
- Injects barcode lookup widgets into POS via `_assets_pos`

---

## 17. pos_account_reports (Enterprise)

**Path:** `enterprise/pos_account_reports/`  
**Dependencies:** point_of_sale, account_reports  
**Auto-install:** Yes

### Purpose
Bridge module for tax reporting. Zero code — creates dependency chain so POS tax data flows into account_reports.

---

## 18. pos_restaurant_appointment (Enterprise)

**Path:** `enterprise/pos_restaurant_appointment/`  
**Dependencies:** appointment, pos_restaurant

### Purpose
Table reservation management via Appointment app.

### Backend Models

| Model | Key Fields |
|-------|-----------|
| `restaurant.table` (inherit) | `appointment_resource_id` (M2one). Auto-creates appointment resources |
| `calendar.event` (inherit) | Inherits `pos.load.mixin`. `_send_table_notifications()` sends `TABLE_BOOKING` BUS event |
| `appointment.resource` (inherit) | POS table linkage |
| `pos.config` (inherit) | `appointment_type_id` |

### Features
- Each table = `appointment.resource` with capacity = seats
- Bookings loaded into POS via `pos.load.mixin`
- Floor plan shows booked/available tables
- Gantt view embedded for booking management

---

## 19. whatsapp_pos (Enterprise)

**Path:** `enterprise/whatsapp_pos/`  
**Dependencies:** point_of_sale, whatsapp  
**Auto-install:** Yes

### Purpose
Send POS receipts/invoices via WhatsApp.

### Backend Models

| Model | Key Methods |
|-------|-----------|
| `pos.order` (inherit) | `action_sent_receipt_on_whatsapp(phone, ticket_image)`, `action_send_whatsapp()` |
| `pos.config` (inherit) | `whatsapp_enabled`, `receipt_template_id`, `invoice_template_id` |

### Flow
1. Receipt generated as JPEG attachment
2. WhatsApp composer dialog opened
3. Uses approved marketing templates
4. Invoice sending queued via cron (`force_send_by_cron=True`)

---

## 20. Enterprise-Exclusive Localization

### pos_l10n_se (Sweden)
**Dependencies:** pos_restaurant, pos_iot, l10n_se  
Swedish registered cash register. Similar pattern to Belgian blackbox but for Swedish Skatteverket requirements.

### l10n_de_pos_cert (Germany)
**Dependencies:** l10n_de, point_of_sale, iap  
TSS (Technische Sicherheitseinrichtung) via Fiskaly cloud IAP service. Cloud-based TSS vs. Belgian hardware FDM.

**Key Models:**
- `pos.order` — Fiskaly transaction UUID, number, time start/end, certificate serial, signature value
- `l10n_de_pos_dsfinvk_export` — DSFinV-K export for tax audit format
- Refunds blocked from backend — must use POS cashier interface

### l10n_de_pos_res_cert (Germany Restaurant)
**Dependencies:** pos_restaurant, l10n_de_pos_cert  
TSS for restaurant POS.

---

## 21. Core Localization Modules

### l10n_mx_edi_pos (Mexico)
**Dependencies:** l10n_mx_edi, point_of_sale

CFDI/SAT electronic invoicing for POS. 833-line model file.

**Key Features:**
- Individual CFDI per order
- Global invoices (factura global) — aggregate multiple orders
- SAT status polling
- Wizard: `l10n_mx_edi_global_invoice_create`
- Refund chain resolution

### l10n_br_edi_pos (Brazil)
**Dependencies:** l10n_br_edi, point_of_sale  
**Auto-install:** Yes

NFC-e (electronic consumer invoice) via Avatax.

**Key Features:**
- 44-digit access key generation (MOD 11 check digit)
- QR code for consumer verification (state-specific SEFAZ URLs)
- Combined tax + EDI pipeline (`_l10n_br_do_edi()`)
- Refunds processed through account.move, not POS

### l10n_sa_edi_pos (Saudi Arabia)
**Dependencies:** l10n_sa_pos, l10n_sa_edi  
**License:** LGPL-3 (only non-enterprise license in this set)

ZATCA e-invoicing (simplified). Thinnest localization — delegates to `l10n_sa_edi`.

**Key Fields:**
- `l10n_sa_invoice_qr_code_str` (related from account.move)
- `l10n_sa_invoice_edi_state` (related from account.move.edi_state)

### l10n_fr_pos_cert (France)
**Dependencies:** l10n_fr_account, point_of_sale  
Anti-fraud cash register per CGI 286 I-3 bis. Certified software (no hardware).

### l10n_es_edi_tbai_pos (Spain — Basque Country)
**Dependencies:** l10n_es_edi_tbai, point_of_sale  
TicketBAI electronic invoicing.

### l10n_es_edi_verifactu_pos (Spain)
**Dependencies:** l10n_es_edi_verifactu, point_of_sale  
Veri*Factu electronic invoicing (core-only, not in enterprise).

### l10n_in_pos (India)
**Dependencies:** l10n_in, point_of_sale  
GST-compliant POS invoicing.

### l10n_in_reports_gstr_pos (India)
**Dependencies:** l10n_in_reports_gstr, point_of_sale  
**Auto-install:** Yes  
GSTR-1 return data extraction from POS orders.

### l10n_ke_edi_oscu_pos (Kenya)
**Dependencies:** l10n_ke_edi_oscu_stock, point_of_sale  
Kenya Revenue Authority OSCU integration.

### l10n_it_pos (Italy)
**Dependencies:** l10n_it, point_of_sale  
**Auto-install:** Yes  
Italian fiscal printer integration.

### l10n_pe_pos / l10n_pe_edi_pos (Peru)
**Dependencies:** l10n_pe / l10n_pe_edi, point_of_sale  
**Auto-install:** Yes (EDI variant)  
SUNAT electronic invoicing.

### l10n_ec_edi_pos (Ecuador)
**Dependencies:** l10n_ec_edi, point_of_sale  
**Auto-install:** Yes  
Ecuadorian SRI electronic invoicing.

### l10n_cl_edi_pos (Chile)
**Dependencies:** l10n_cl_edi, point_of_sale  
**Auto-install:** Yes  
Chilean SII electronic invoicing.

### l10n_pl_reports_pos_jpk (Poland)
**Dependencies:** l10n_pl_reports, point_of_sale  
**Auto-install:** Yes  
JPK_VAT reporting from POS data.

### l10n_mt_pos (Malta)
**Dependencies:** point_of_sale  
Malta EXO compliance.

### l10n_gcc_pos (GCC Countries)
**Dependencies:** point_of_sale, l10n_gcc_invoice  
Gulf POS localization.

### l10n_id_pos (Indonesia)
**Dependencies:** l10n_id, point_of_sale  
Indonesian POS.

### l10n_my_edi_pos (Malaysia)
**Dependencies:** l10n_my_edi_extended, point_of_sale  
MyInvois e-invoicing (core-only).

### l10n_uy_pos (Uruguay)
**Dependencies:** l10n_uy, point_of_sale  
Uruguayan POS (core-only).

---

## 22. Feature Add-on Modules

### pos_discount
Quick percentage-based discount button. Simple module — adds discount button to POS action pad.

### pos_event
Link POS and Events. Sell event tickets in POS.
**Dependencies:** point_of_sale, event_product

### pos_event_sale
Bridge between pos_event and pos_sale.
**Dependencies:** pos_event, pos_sale

### pos_mrp
Link POS and Manufacturing (MRP).
**Dependencies:** point_of_sale, mrp

### pos_sms
SMS order confirmations.
**Dependencies:** point_of_sale, sms

### pos_settle_due
Settle partner's outstanding debts in POS UI.
**Dependencies:** point_of_sale, account_followup

### pos_sale_loyalty
Loyalty programs in sales-linked POS.
**Dependencies:** pos_sale, pos_loyalty

### pos_sale_margin
View margin of POS orders in Sales Margin report.
**Dependencies:** pos_sale, sale_margin

### pos_sale_stock_renting
Rental products in POS.
**Dependencies:** pos_sale, sale_stock_renting

### pos_sale_subscription
Subscriptions via POS.
**Dependencies:** pos_sale, sale_subscription

### pos_hr_restaurant
Adapts UI when both pos_hr and pos_restaurant are installed.
**Dependencies:** pos_hr, pos_restaurant

### pos_restaurant_loyalty
Fixes behavior when both pos_restaurant and pos_loyalty are installed.
**Dependencies:** pos_restaurant, pos_loyalty

### pos_restaurant_adyen / pos_restaurant_stripe
American-style tipping for Adyen/Stripe payment terminals.
**Dependencies:** pos_adyen/pos_stripe, pos_restaurant

### pos_self_order_*
Self-order kiosk variants with different payment methods.
**Dependencies:** pos_self_order + specific payment/feature module

### pos_account_tax_python
Allow custom tax formulas in POS.
**Dependencies:** account_tax_python, point_of_sale

### pos_epson_printer
Epson ePOS Printers without IoT Box (direct network printing).
**Dependencies:** point_of_sale

### spreadhseet_dashboard_pos_hr / _pos_restaurant
Spreadsheet-based dashboards for POS analytics.
**Dependencies:** spreadsheet_dashboard + pos_hr / pos_hr + pos_restaurant

---

## 23. Hardware Drivers (Not Installable)

### hw_escpos
**Dependencies:** pyusb, pyserial, qrcode  
ESC/POS printer and cash drawer driver. **Not installable** — used by IoT Box.

### hw_posbox_homepage
IoT Box homepage replacement. **Not installable** — used by IoT Box.

---

## Module Dependency Graph (Simplified)

```
point_of_sale
├── pos_restaurant
│   ├── pos_self_order
│   │   ├── pos_self_order_adyen
│   │   ├── pos_self_order_stripe
│   │   ├── pos_self_order_razorpay
│   │   ├── pos_self_order_iot
│   │   └── pos_self_order_sale
│   ├── pos_hr_restaurant
│   ├── pos_restaurant_loyalty
│   ├── pos_restaurant_adyen
│   ├── pos_restaurant_stripe
│   └── pos_restaurant_appointment
├── pos_hr
│   └── pos_hr_restaurant
├── pos_loyalty
│   ├── pos_sale_loyalty
│   └── pos_restaurant_loyalty
├── pos_sale
│   ├── pos_sale_loyalty
│   ├── pos_sale_margin
│   ├── pos_sale_stock_renting
│   └── pos_sale_subscription
├── pos_online_payment
│   └── pos_online_payment_self_order
├── pos_event
│   └── pos_event_sale
├── pos_discount
├── pos_mrp
├── pos_sms
├── pos_settle_due
├── pos_account_tax_python
├── pos_epson_printer
├── pos_adyen / pos_stripe / pos_six / etc.
└── localization modules (all countries)

enterprise/
├── pos_enterprise
│   └── pos_mobile
├── pos_preparation_display
│   ├── pos_hr_preparation_display
│   ├── pos_order_tracking_display
│   ├── pos_restaurant_preparation_display
│   └── pos_self_order_preparation_display
├── pos_iot
│   ├── pos_iot_six
│   └── pos_blackbox_be
├── pos_urban_piper suite
├── pos_avatax
├── pos_account_reports
├── pos_pricer
├── pos_barcodelookup
├── whatsapp_pos
└── enterprise localization modules
```
