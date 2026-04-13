---
name: odoo-18-pos
description: Complete Odoo 18 Enterprise POS (Point of Sale) development skill. Covers 94+ modules, OWL/JS frontend, Python backend, extension patterns, localization, IoT integration, and best practices.
---

# Odoo 18 POS (Point of Sale) — Complete Development Skill

> Master reference for developing, extending, and maintaining Odoo 18 Enterprise POS modules. Covers 94+ modules, OWL/JS frontend, Python backend, extension patterns, localization, IoT integration, and enterprise-exclusive features.

## Trigger Phrases

Use this skill when:
- User asks to develop, customize, or extend the Odoo 18 POS module
- User mentions "point of sale", "pos", "cash register", "ticket screen"
- User needs to add payment methods, loyalty programs, restaurant features
- User wants to create POS extensions, overrides, or custom modules
- User asks about POS architecture, OWL components, or data flow
- User needs localization (fiscal compliance) for POS in any country
- User works with IoT Box, payment terminals, printers, or hardware in POS
- User mentions self-order kiosks, preparation displays, or customer displays
- User needs to integrate POS with delivery platforms (Uber Eats, Swiggy, etc.)
- User asks about offline mode, order synchronization, or BUS communication
- User wants to understand POS asset bundles, module dependencies, or installation

---

## Quick Start

### Minimum POS Module Structure
```
my_pos_extension/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── pos_config.py          # Backend model extensions
├── static/src/
│   ├── overrides/              # JS overrides
│   │   └── pos_store.js
│   └── xml/                    # XML templates
│       └── my_component.xml
└── views/
    └── pos_config_views.xml    # Backend view extensions
```

### Minimal __manifest__.py
```python
{
    'name': 'My POS Extension',
    'version': '18.0.1.0.0',
    'category': 'Sales/Point of Sale',
    'depends': ['point_of_sale'],
    'assets': {
        'point_of_sale._assets_pos': [
            'my_pos_extension/static/src/overrides/**/*.js',
            'my_pos_extension/static/src/xml/**/*.xml',
        ],
    },
    'license': 'LGPL-3',
}
```

---

## Core Architecture Overview

### System Topology

```
┌─────────────────────────────────────────────────────┐
│                  Odoo Backend (Python)                │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ pos.config│  │pos.session│  │   pos.order      │   │
│  │ (settings)│  │ (state)  │  │   pos.order.line │   │
│  │          │  │          │  │   pos.payment    │   │
│  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │
│       │              │                 │              │
│  ┌────┴──────────────┴─────────────────┴──────────┐  │
│  │          pos.load.mixin (data loader)          │  │
│  │          pos.bus.mixin  (notifications)        │  │
│  └────────────────────┬───────────────────────────┘  │
│                       │ RPC                          │
└───────────────────────┼──────────────────────────────┘
                        │
              ┌─────────┴─────────┐
              │   POS SPA (OWL)    │
              │  ┌──────────────┐  │
              │  │   PosStore   │  │
              │  │  (Reactive)  │  │
              │  └──────┬───────┘  │
              │         │          │
              │  ┌──────┴───────┐  │
              │  │  Models      │  │
              │  │  (Base)      │  │
              │  └──────┬───────┘  │
              │         │          │
              │  ┌──────┴───────┐  │
              │  │  Screens     │  │
              │  │  Components  │  │
              │  │  Services    │  │
              │  └──────────────┘  │
              └────────────────────┘
                        │
              ┌─────────┴─────────┐
              │   IndexedDB       │
              │  (offline store)  │
              └───────────────────┘
```

### The Three Pillars

1. **`pos.config`** — Settings & behavior configuration (one per POS terminal)
2. **`pos.session`** — Runtime session state (opening, open, closing, closed)
3. **`pos.order`** — Sales transactions (draft → paid → done/invoiced)

---

## Asset Bundle System

The POS uses a hierarchical asset bundle system. **Every POS extension MUST register assets in `point_of_sale._assets_pos`**.

### Bundle Hierarchy

```
point_of_sale.assets_prod                    ← Loaded at /pos/ui
└── point_of_sale._assets_pos                ← Main POS bundle
    ├── point_of_sale.base_app               ← Core dependencies
    │   ├── Bootstrap helpers & variables
    │   ├── bus_service, multi_tab_service
    │   └── barcode services & parsers
    ├── OWL library, luxon, zxing
    ├── account helpers, mail.sound_effects
    └── point_of_sale/static/src/**/*        ← All core POS files

point_of_sale.customer_display_assets        ← Customer-facing screen
└── generic_components, customer_display/**/*

pos_self_order.assets                        ← Self-order kiosk
pos_preparation_display.assets               ← Kitchen display
```

### Asset Registration Pattern

```python
# __manifest__.py
{
    'assets': {
        'point_of_sale._assets_pos': [
            # JS/OWL overrides
            'my_module/static/src/overrides/**/*.js',
            'my_module/static/src/app/**/*.js',
            # XML templates
            'my_module/static/src/xml/**/*.xml',
            # SCSS styles
            'my_module/static/src/scss/**/*.scss',
        ],
        # For standalone apps (preparation display, self-order)
        'my_app.assets': [
            'my_app/static/src/app/**/*.js',
            'my_app/static/src/xml/**/*.xml',
        ],
    },
}
```

---

## Module Catalog (94+ Modules)

### Tier 1: Core POS (always required)

| Module | Path | Purpose |
|--------|------|---------|
| `point_of_sale` | core | Main POS application — UI, orders, payments, products |
| `pos_restaurant` | core | Floor plans, table management, kitchen printing, bill splitting |
| `pos_hr` | core | Employee login (barcode/PIN), cashier tracking |
| `pos_loyalty` | core | Loyalty programs, gift cards, coupon redemption |
| `pos_sale` | core | Link POS with Sales module (sales teams, quotations) |
| `pos_self_order` | core | Self-order kiosk with QR code menu browsing |
| `pos_online_payment` | core | Online payment links, payment portals for POS orders |

### Tier 2: Payment Integrations

| Module | Protocol | Region |
|--------|----------|--------|
| `pos_adyen` | Adyen Terminal | Global |
| `pos_stripe` | Stripe Terminal | Global |
| `pos_six` | Six Terminal | Europe |
| `pos_paytm` | PayTM | India |
| `pos_razorpay` | Razorpay | India |
| `pos_pine_labs` | Pine Labs | India |
| `pos_mercado_pago` | Mercado Pago | LATAM |
| `pos_tyro` | Tyro | Australia |
| `pos_viva_wallet` | Viva Wallet | Europe |
| `pos_epson_printer` | Epson ePOS | Global (direct, no IoT) |

### Tier 3: Enterprise-Exclusive

| Module | Purpose |
|--------|---------|
| `pos_enterprise` | Bridge module — enterprise views, IoT Box config, studio |
| `pos_mobile` | Mobile app integration (web_mobile shell) |
| `pos_preparation_display` | Kitchen/order preparation display (standalone SPA) |
| `pos_iot` | IoT Box integration — printers, scales, payment terminals, displays |
| `pos_iot_six` | Six payment terminal via IoT |
| `pos_barcodelookup` | Product creation via barcode camera scan |
| `pos_blackbox_be` | Belgian certified cash register (fiscal compliance) |
| `pos_l10n_se` | Swedish registered cash register |
| `pos_pricer` | Electronic shelf label integration |
| `pos_avatax` | Avalara tax calculation |
| `pos_account_reports` | Tax reporting bridge to account_reports |
| `pos_restaurant_appointment` | Table reservations via Appointment app |
| `whatsapp_pos` | WhatsApp receipt/invoice sending |

### Tier 4: UrbanPiper Suite (Food Delivery)

| Module | Purpose |
|--------|---------|
| `pos_urban_piper` | Core delivery API (Swiggy, Zomato, Uber Eats) |
| `pos_urban_piper_enhancements` | Scheduling, future delivery notifications |
| `pos_restaurant_urban_piper` | Restaurant UI for delivery orders |
| `pos_urban_piper_swiggy` | Swiggy platform integration |
| `pos_urban_piper_ubereats` | Uber Eats platform integration |
| `pos_urban_piper_zomato` | Zomato platform integration |

### Tier 5: Feature Add-ons

| Module | Depends On | Purpose |
|--------|-----------|---------|
| `pos_discount` | point_of_sale | Percentage discount button |
| `pos_event` | point_of_sale, event | Sell event tickets in POS |
| `pos_event_sale` | pos_event, pos_sale | Link events + sales |
| `pos_mrp` | point_of_sale, mrp | Manufacturing integration |
| `pos_sms` | point_of_sale, sms | SMS order confirmations |
| `pos_settle_due` | point_of_sale, followup | Settle partner debts |
| `pos_sale_loyalty` | pos_sale, pos_loyalty | Loyalty in sales-linked POS |
| `pos_sale_margin` | pos_sale, sale_margin | Margin tracking |
| `pos_sale_stock_renting` | pos_sale, sale_stock_renting | Rental products |
| `pos_sale_subscription` | pos_sale, sale_subscription | Subscriptions via POS |
| `pos_hr_restaurant` | pos_hr, pos_restaurant | Employee + restaurant combo |
| `pos_restaurant_loyalty` | pos_restaurant, pos_loyalty | Loyalty fix for restaurant |
| `pos_restaurant_adyen` | pos_adyen, pos_restaurant | Adyen tipping |
| `pos_restaurant_stripe` | pos_stripe, pos_restaurant | Stripe tipping |
| `pos_self_order_adyen` | pos_adyen, pos_self_order | Adyen in kiosk |
| `pos_self_order_stripe` | pos_stripe, pos_self_order | Stripe in kiosk |
| `pos_self_order_razorpay` | pos_razorpay, pos_self_order | Razorpay in kiosk |
| `pos_self_order_iot` | pos_iot, pos_self_order | IoT in kiosk |
| `pos_self_order_epson_printer` | pos_epson, pos_self_order | Epson in kiosk |
| `pos_self_order_sale` | pos_sale, pos_self_order | Sales in kiosk |
| `pos_online_payment_self_order` | pos_online_payment, pos_self_order | Online pay in kiosk |
| `pos_account_tax_python` | account_tax_python, pos | Custom tax formulas |

### Tier 6: Localization Modules (20+)

| Module | Country | Purpose |
|--------|---------|---------|
| `l10n_ar_pos` | Argentina | Argentine POS with AFIP |
| `l10n_be_pos_restaurant` | Belgium | Belgian restaurant POS |
| `l10n_be_pos_sale` | Belgium | Belgian POS + Sales link |
| `l10n_ch_pos` | Switzerland | Swiss POS |
| `l10n_co_pos` | Colombia | Colombian POS |
| `l10n_es_pos` | Spain | Spanish POS |
| `l10n_es_edi_tbai_pos` | Spain | TicketBAI (Basque Country) |
| `l10n_es_edi_verifactu_pos` | Spain | Veri*Factu |
| `l10n_fr_pos_cert` | France | Anti-fraud cash register (CGI 286) |
| `l10n_de_pos_cert` | Germany | TSS via Fiskaly cloud |
| `l10n_de_pos_res_cert` | Germany | TSS for restaurant |
| `l10n_mx_edi_pos` | Mexico | CFDI/SAT invoicing |
| `l10n_br_edi_pos` | Brazil | NFC-e via Avatax |
| `l10n_sa_edi_pos` | Saudi Arabia | ZATCA e-invoicing |
| `l10n_in_pos` | India | GST POS |
| `l10n_in_reports_gstr_pos` | India | GSTR-1 reporting |
| `l10n_ke_edi_oscu_pos` | Kenya | Kenya EDI (OSCU) |
| `l10n_it_pos` | Italy | Fiscal printer integration |
| `l10n_pe_pos` / `l10n_pe_edi_pos` | Peru | Peruvian SUNAT EDI |
| `l10n_ec_edi_pos` | Ecuador | Ecuadorian EDI |
| `l10n_cl_edi_pos` | Chile | Chilean EDI |
| `l10n_pl_reports_pos_jpk` | Poland | JPK_VAT reporting |
| `l10n_mt_pos` | Malta | Malta EXO compliance |
| `l10n_gcc_pos` | GCC | Gulf POS |
| `l10n_id_pos` | Indonesia | Indonesian POS |
| `l10n_my_edi_pos` | Malaysia | MyInvois e-invoicing |
| `l10n_uy_pos` | Uruguay | Uruguayan POS |
| `l10n_account_withholding_tax_pos` | Multi | Withholding tax on POS |
| `l10n_test_pos_qr_payment` | Test | QR payment tests |

> **Full module details**: See [references/module-catalog.md](references/module-catalog.md)

---

## Frontend Architecture (OWL/JavaScript)

### PosStore — Central State Manager

The entire POS UI revolves around `PosStore` (extends `Reactive`). It is registered as the `"pos"` service.

```javascript
// Access the store in any component
import { usePos } from "@point_of_sale/app/store/pos_hook";

export class MyComponent extends Component {
    setup() {
        this.pos = usePos();  // reactive — triggers re-render on changes
        this.pos.selectedOrderUuid;   // current order UUID
        this.pos.numpadMode;          // "quantity" | "discount" | "price"
        this.pos.productListView;     // "grid" | "list"
    }
}
```

### Key PosStore Properties

| Property | Type | Purpose |
|----------|------|---------|
| `selectedOrderUuid` | string\|null | Current order being edited |
| `selectedPartner` | ResPartner\|null | Selected customer |
| `selectedCategory` | PosCategory\|null | Active product category |
| `numpadMode` | string | "quantity", "discount", or "price" |
| `mainScreen` | {name, component} | Active screen |
| `pendingOrder` | {create, write, delete} | Offline sync queues |
| `syncingOrders` | Set | Orders in flight to server |
| `ticketScreenState` | object | Ticket screen pagination/filters |

### Key PosStore Methods

| Method | Purpose |
|--------|---------|
| `addLineToCurrentOrder(vals, opts)` | Add product to order (with merge logic) |
| `createNewOrder(data)` | Create new POS order with UUID |
| `syncAllOrders(options)` | Batch sync to server |
| `showScreen(name, props)` | Navigate to screen |
| `pay()` | Transition to PaymentScreen |
| `push_orders_with_closing_popup()` | Sync + close session |
| `selectPartner()` | Open customer selection |

### Frontend Model System

All POS data models extend `Base` from `related_models.js`:

```javascript
// Model registration pattern
import { Base } from "@point_of_sale/app/models/related_models";
import { registry } from "@web/core/registry";

class PosOrder extends Base {
    static pythonModel = "pos.order";

    setup(vals) {
        this.date_order = vals.date_order || new Date();
        this.state = vals.state || "draft";
        this.uuid = vals.uuid || uuidv4();
        this.uiState = { lineToRefund: null, displayed: true };
    }

    get taxTotals() { /* computed tax */ }
    get is_paid() { return this.state === "paid"; }

    add_paymentline(pm) { /* create payment line */ }
    set_pricelist(pl) { /* recompute all lines */ }
}

registry.category("pos_available_models").add("pos.order", PosOrder);
```

### Model Lifecycle

```
Backend data → PosData.loadInitialData()
             → createRelatedModels()        ← instantiates all registered models
             → model.loadData()             → populates records
             → model instances usable in UI

UI changes → model.update(vals)            → marks dirty
           → serialize()                   → generates commands
           → syncAllOrders()               → RPC to backend
           → backend processes → BUS notification → other devices update
```

### Screen System

Screens are OWL components registered in `pos_screens`:

```javascript
import { registry } from "@web/core/registry";

class MyScreen extends Component {
    static template = "my_module.MyScreen";
    static storeOnOrder = false;  // persist state on order?

    setup() {
        this.pos = usePos();
    }

    validateOrder() {
        // validation logic
        this.pos.showScreen("NextScreen", { orderUuid: this.pos.selectedOrderUuid });
    }
}

registry.category("pos_screens").add("MyScreen", MyScreen);
```

### Built-in Screens

| Screen | Path | Purpose |
|--------|------|---------|
| ProductScreen | `screens/product_screen/` | Main POS UI — products, cart, actions |
| PaymentScreen | `screens/payment_screen/` | Payment processing |
| ReceiptScreen | `screens/receipt_screen/` | Post-payment receipt display |
| TicketScreen | `screens/ticket_screen/` | Order history, refunds, reprints |
| LoginScreen | `screens/login_screen/` | Employee PIN login (pos_hr) |
| PartnerList | `screens/partner_list/` | Customer selection |
| ScaleScreen | `screens/scale_screen/` | Weighing scale UI |
| SaverScreen | `screens/saver_screen/` | Save order for later |

### Services Architecture

Key services used by POS:

| Service | Key | Purpose |
|---------|-----|---------|
| PosDataService | `pos_data` | ORM wrapper, IndexedDB, sync |
| BusService | `bus_service` | WebSocket notifications |
| BarcodeReader | `barcode_reader` | Barcode scanning |
| HardwareProxy | `hardware_proxy` | IoT Box communication |
| DialogService | `dialog` | Modal dialogs |
| NotificationService | `notification` | Toast notifications |
| AlertService | `alert` | In-screen alerts |
| PrinterService | `printer` | Receipt printing |
| ActionService | `action` | Action execution |

### Essential Hooks

```javascript
import { usePos } from "@point_of_sale/app/store/pos_hook";      // reactive store
import { useService } from "@web/core/utils/hooks";              // any service
import { useErrorHandlers } from "@point_of_sale/app/utils/hooks";  // error handling
import { useAsyncLockedMethod } from "@point_of_sale/app/utils/hooks";  // debounce
import { useTrackedAsync } from "@point_of_sale/app/utils/hooks";    // async status
import { useAutoFocusToLast } from "@point_of_sale/app/utils/hooks";   // auto-focus
import { useTime } from "@point_of_sale/app/utils/time_hook";          // clock
```

---

## Backend Architecture (Python)

### Core Models

#### pos.config
Settings for each POS terminal. One record = one terminal/register.

Key fields:
- `payment_method_ids` — Available payment methods
- `pricelist_id` — Default pricelist
- `picking_type_id` — Stock operation type for deliveries
- `journal_id` — Default sale journal
- `printer_ids` — Kitchen printers (restaurant mode)
- `iface_*` — Hardware interface flags
- `cash_rounding` / `rounding_method` — Cash rounding rules
- `module_pos_*` — Feature flags for optional modules

#### pos.session
Runtime session. Manages order sequences, cash control, session lifecycle.

States: `opening_control` → `opened` → `closing_control` → `closed`

Key method: `load_data()` — RPC endpoint that bootstr ALL frontend data

#### pos.order
Sales transaction. Receives data from frontend via `sync_from_ui()`.

States: `draft` → `paid` → `done` / `cancel` / `invoiced`

#### pos.order.line
Individual line items within an order. Computed prices, taxes, discounts.

#### pos.payment
Payment records linked to orders. Cash, card, voucher, etc.

#### pos.payment.method
Payment method configuration. Links to journals, payment terminals.

### Mixin System

Two critical mixins used by ALL POS models:

#### pos.bus.mixin
Enables WebSocket notifications to other POS devices when records change.
- Notifies on create/write/unlink
- Other devices fetch updated data automatically

#### pos.load.mixin
Defines what data is sent to the frontend during session bootstrap.
- `_load_pos_data_domain(self)` — Filter records to load
- `_load_pos_data_fields(self)` — List of fields to send
- `_load_pos_data_relations(self)` — Related models to include

### Order Sync Flow

```
Frontend: pos.order.updateLastOrderChange()
        → serialize order data
        → this.orm.call("pos.order", "sync_from_ui", [session_id, order_data])

Backend:  PosOrder.sync_from_ui()
        → PosSession._process_order()
        → create/update pos.order, pos.order.line, pos.payment
        → stock moves, accounting entries
        → return serialized data

Frontend: PosData.execute() receives response
        → models.loadData() hydrates reactive store
        → UI auto-updates via OWL reactivity
```

---

## Extension & Override Patterns

### Pattern 1: Backend Model Inheritance

```python
# models/pos_config.py
from odoo import models, fields

class PosConfig(models.Model):
    _inherit = 'pos.config'

    my_custom_field = fields.Char("Custom Setting")
    my_custom_feature = fields.Boolean("Enable Feature")
```

### Pattern 2: Frontend Model Extension

```javascript
/** @odoo-module **/
import { PosOrder } from "@point_of_sale/app/models/pos_order";
import { patch } from "@web/core/utils/patch";

patch(PosOrder.prototype, {
    get myCustomComputed() {
        return /* something based on this.lines, this.partner, etc. */;
    },

    myCustomMethod() {
        // new method on all PosOrder instances
    },
});
```

### Pattern 3: PosStore Patch

```javascript
/** @odoo-module **/
import { PosStore } from "@point_of_sale/app/store/pos_store";
import { patch } from "@web/core/utils/patch";

patch(PosStore.prototype, {
    setup() {
        super.setup(...arguments);
        // Add custom reactive state
        this.myCustomState = false;
    },

    // Override existing method
    addLineToCurrentOrder(vals, opts) {
        // Custom logic before
        return super.addLineToCurrentOrder(vals, opts);
        // Custom logic after
    },
});
```

### Pattern 4: XML Template Extension

```xml
<t t-name="my_module.ProductCardExtension"
   t-inherit="point_of_sale.ProductCard"
   t-inherit-mode="extension"
   owl="1">
    <xpath expr="//div[hasclass('product-name')]" position="after">
        <span class="my-badge">
            <t t-esc="env.utils.formatCurrency(props.product.lst_price)"/>
        </span>
    </xpath>
</t>
```

### Pattern 5: Screen Registration

```javascript
/** @odoo-module **/
import { Component } from "@odoo/owl";
import { registry } from "@web/core/registry";

class MyCustomScreen extends Component {
    static template = "my_module.MyCustomScreen";
    static storeOnOrder = false;

    setup() {
        this.pos = usePos();
    }
}

registry.category("pos_screens").add("MyCustomScreen", MyCustomScreen);
```

### Pattern 6: Service Injection

```javascript
/** @odoo-module **/
import { registry } from "@web/core/registry";
import { useService } from "@web/core/utils/hooks";

const MyService = {
    dependencies: ["orm", "notification"],

    start(env, { orm, notification }) {
        return {
            async doSomething() {
                // service logic
            },
        };
    },
};

registry.category("services").add("my_pos_service", MyService);
```

### Pattern 7: BUS Notification Handling

```javascript
/** @odoo-module **/
import { PosStore } from "@point_of_sale/app/store/pos_store";
import { patch } from "@web/core/utils/patch";
import { getOnNotified } from "@point_of_sale/utils";

patch(PosStore.prototype, {
    setup() {
        super.setup(...arguments);
        const { bus_service } = this;

        getOnNotified(bus_service, "my_custom_channel").add(
            ({ payload }) => {
                // Handle notification
                this.myCustomState = payload.value;
            }
        );
    },
});
```

---

## IoT & Hardware Integration

### Architecture

```
POS Browser ──WebSocket──► IoT Box (Raspberry Pi)
                                │
                    ┌───────────┼───────────┐
                    │           │           │
                Printer     Scale      Payment Terminal
                (USB)      (Serial)      (USB/Network)
```

### Device Types

| Type | identifier | Use |
|------|-----------|-----|
| `printer` | posbox_printer | Receipt printing via IoT Box |
| `scale` | posbox_scale | Weight measurement |
| `payment_terminal` | posbox_payment_terminal | Card payments |
| `customer_display` | posbox_customer_display | Secondary screen |
| `fiscal_data_module` | posbox_fiscal_data_module | Certified cash register |
| `barcode_scanner` | posbox_barcode_scanner | Barcode scanning |

### IoT Device Access Pattern

```javascript
// In PosStore
const device = this.data.models['iot.device'].find(d => d.id === deviceId);

// Via hardware proxy
const hardwareProxy = useService('hardware_proxy');
await hardwareProxy.call('printer', 'print_receipt', { receipt: imageData });
```

### Payment Terminal Pattern

Payment terminals use an event-driven proxy pattern:

```javascript
// Terminal transaction lifecycle
// 1. POS sends payment request to terminal
// 2. Terminal processes card
// 3. Terminal pushes result back via listener
// 4. POS updates payment status

class PaymentTerminal extends PaymentInterface {
    async sendPaymentRequest(amount) {
        const result = await this.terminal_proxy.sendPay(amount);
        this.updatePaymentStatus(result);
    }
}
```

---

## Localization Patterns

### Fiscal Compliance Patterns

| Pattern | Countries | Description |
|---------|-----------|-------------|
| **Hardware Blackbox** | Belgium, Germany | Physical TSS device via IoT Box, immutable audit log |
| **Cloud TSS** | Germany (Fiskaly) | Cloud-based technical security system |
| **EDI Invoice** | Mexico, Brazil, Peru, Ecuador, Chile, Saudi Arabia | Government e-invoicing API |
| **Anti-fraud** | France, Spain | Certified software, no hardware required |
| **Reporting** | India, Poland, Malaysia | Tax return data extraction |

### Common Localization Extensions

```python
class PosOrder(models.Model):
    _inherit = 'pos.order'

    # Universal localization fields
    localization_country = fields.Char(related='company_id.country_id.code')

    # EDI-specific fields
    edi_state = fields.Char("EDI State")  # pending/sent/cancelled
    edi_uuid = fields.Char("EDI UUID")    # government tracking ID
    qr_code = fields.Char("QR Code URL")  # consumer verification

    @api.model
    def sync_from_ui(self, session_id, order_data):
        # Intercept standard sync
        res = super().sync_from_ui(session_id, order_data)
        # Add localization processing
        self._process_localization()
        return res
```

### Frontend Localization Pattern

```javascript
// Detect localization
const country = this.pos.company.country_id;
if (country.code === "MX") {
    // Mexican CFDI flow
} else if (country.code === "BE") {
    // Belgian blackbox flow
}
```

---

## Offline Architecture

### IndexedDB Storage

POS stores critical data locally for offline operation:

**Stored Models:**
- `pos.order` (pending orders)
- `pos.order.line`
- `pos.payment`
- `stock.lot` (lot/serial numbers)
- `product.product` (cached product data)

### Offline Sync Flow

```
Online → Orders sync immediately via RPC
Offline → Orders queued in IndexedDB (pendingOrder.create/write/delete)
Reconnect → DevicesSynchronisation detects connection
           → syncData() processes queued operations
           → BUS notifies other devices
```

### Offline Design Rules

1. **Never block the UI** waiting for server responses
2. **Queue all mutations** for later sync
3. **Use Mutex** for order push operations (`pushOrderMutex`)
4. **Handle conflicts** — same order modified on multiple devices
5. **Graceful degradation** — core POS works without network

---

## Testing Strategy

### Unit Tests (JavaScript)

```javascript
// static/tests/unit/my_test.test.js
import { createPosUnitEnvironment } from "@point_of_sale/../tests/unit/utils";

QUnit.module("My POS Extension", {});

QUnit.test("basic functionality", async (assert) => {
    const { pos, clickProduct } = await createPosUnitEnvironment();
    await clickProduct(productId);
    assert.strictEqual(pos.selectedOrder.orderlines.length, 1);
});
```

### Python Tests

```python
# tests/test_my_module.py
from odoo.tests import tagged
from odoo.addons.point_of_sale.tests.common import TestPosCommon

@tagged("post_install", "-at_install")
class TestMyModule(TestPosCommon):
    def test_my_feature(self):
        self.pos_config.my_custom_feature = True
        order = self.create_pos_order()
        # assertions
```

### Tour Tests

```javascript
// static/tests/tours/my_tour.js
import tourSteps from "@web_tour/tour/tour_steps";

tourSteps.push({
    trigger: '.product-name:contains("Product")',
    content: 'Click on a product',
});
```

---

## Performance Guidelines

### Frontend
1. **Lazy load products** — ProductScreen loads products in pages (20 at a time)
2. **Cache pricelist rules** — `computeProductPricelistCache()` in utils
3. **Avoid unnecessary reactivity** — use `markRaw()` for large objects that don't need tracking
4. **Debounce search** — Product search uses input debouncing
5. **Virtual scrolling** — Long lists use virtualization

### Backend
1. **Use `_load_pos_data_fields`** — Only send needed fields to frontend
2. **Batch operations** — `sync_from_ui` handles multiple orders at once
3. **Indexed lookups** — Ensure proper DB indexes on `pos.order` fields
4. **Avoid N+1 queries** — Use `read_group`, `search_read` with proper batching
5. **Async processing** — Use `bus` for notifications, not polling

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Custom field not available in POS frontend | Add model/fields to `_load_pos_data_fields()` or include in `load_data()` |
| Template changes not showing | Verify asset bundle registration in `__manifest__.py` under `_assets_pos` |
| Order sync fails silently | Check `pendingOrder` queues in PosStore, look at IndexedDB |
| Override not applied | Ensure `patch()` import is correct and module is loaded |
| Reactivity not triggering | Verify `usePos()` returns reactive proxy, not raw object |
| BUS notifications not received | Check channel name matches, bus_service is connected |
| Offline orders not syncing | Check `syncDataWithIndexedDB()` effect, look for mutex conflicts |
| Product search slow | Enable DB indexes on `default_code`, `name`, `barcode` |

---

## Reference Files

| Document | Path | Purpose |
|----------|------|---------|
| Module Catalog | [references/module-catalog.md](references/module-catalog.md) | Detailed docs for all 94+ modules |
| Architecture Patterns | [references/architecture-patterns.md](references/architecture-patterns.md) | Deep dive into data flow, SPA structure |
| Frontend Patterns | [references/frontend-patterns.md](references/frontend-patterns.md) | OWL/JS development guide |
| Backend Patterns | [references/backend-patterns.md](references/backend-patterns.md) | Python model development guide |
| Extension Patterns | [references/extension-patterns.md](references/extension-patterns.md) | How to extend POS safely |
| Localization Guide | [references/localization-guide.md](references/localization-guide.md) | Fiscal compliance patterns |
| Best Practices | [references/best-practices.md](references/best-practices.md) | Code quality, architecture decisions |
| Quick Reference | [references/quick-reference.md](references/quick-reference.md) | Cheat sheets and common patterns |
