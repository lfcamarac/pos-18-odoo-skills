# Odoo 18 POS — Quick Reference Cheat Sheets

> Fast lookup reference for common POS development tasks.

---

## 1. Module Bootstrap

### Minimal __manifest__.py

```python
{
    'name': 'My POS Extension',
    'version': '18.0.1.0.0',
    'category': 'Sales/Point of Sale',
    'summary': 'Brief description',
    'depends': ['point_of_sale'],
    'data': [
        # Backend views (loaded normally)
        'views/pos_config_views.xml',
    ],
    'assets': {
        'point_of_sale._assets_pos': [
            # Frontend assets (loaded in POS SPA)
            'my_module/static/src/**/*.js',
            'my_module/static/src/**/*.xml',
        ],
    },
    'license': 'LGPL-3',
}
```

### Directory Structure

```
my_module/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── pos_config.py
├── static/src/
│   ├── overrides/pos_store.js
│   └── xml/templates.xml
├── views/
│   └── pos_config_views.xml
└── security/
    └── ir.model.access.csv
```

---

## 2. Backend Quick Reference

### Extend pos.config

```python
class PosConfig(models.Model):
    _inherit = 'pos.config'

    my_field = fields.Boolean('My Feature')
```

### Extend pos.order

```python
class PosOrder(models.Model):
    _inherit = 'pos.order'

    my_field = fields.Char('Custom')

    @api.model
    def _order_fields(self, ui_order):
        result = super()._order_fields(ui_order)
        result['my_field'] = ui_order.get('my_field')
        return result
```

### Add to Frontend Data Loading

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['my.model', 'pos.load.mixin']

    def _load_pos_data_domain(self):
        return []  # All records, or add filter

    def _load_pos_data_fields(self):
        return ['id', 'name', 'value']  # Fields to send to frontend
```

### Send BUS Notification

```python
self.env['bus.bus']._sendone(
    channel=(self._cr.dbname, 'pos.session', session_id),
    message_type="notification",
    payload={'type': 'my_event', 'data': {...}},
)
```

---

## 3. Frontend Quick Reference

### Patch PosStore

```javascript
/** @odoo-module */
import { PosStore } from "@point_of_sale/app/store/pos_store";
import { patch } from "@web/core/utils/patch";

patch(PosStore.prototype, {
    setup() {
        super.setup(...arguments);
        this.myState = false;
    },
    myMethod() { /* ... */ },
});
```

### Patch a Model

```javascript
/** @odoo-module */
import { PosOrder } from "@point_of_sale/app/models/pos_order";
import { patch } from "@web/core/utils/patch";

patch(PosOrder.prototype, {
    get myGetter() { return /* ... */; },
    myMethod() { /* ... */ },
});
```

### Register a Screen

```javascript
/** @odoo-module */
import { Component } from "@odoo/owl";
import { usePos } from "@point_of_sale/app/store/pos_hook";
import { registry } from "@web/core/registry";

class MyScreen extends Component {
    static template = "my_module.MyScreen";
    static storeOnOrder = false;
    setup() { this.pos = usePos(); }
}

registry.category("pos_screens").add("MyScreen", MyScreen);
```

### Navigate Between Screens

```javascript
// Go to screen
this.pos.showScreen("MyScreen", { data: value });

// Go back
this.pos.showScreen("ProductScreen");
```

### Use Services

```javascript
setup() {
    this.pos = usePos();
    this.orm = useService("orm");
    this.notification = useService("notification");
    this.dialog = useService("dialog");
    this.bus = useService("bus_service");
}
```

### Receive BUS Notification

```javascript
import { getOnNotified } from "@point_of_sale/utils";

setup() {
    getOnNotified(this.bus, "my_event").add(({ payload }) => {
        // Handle notification
    });
}
```

### Extend XML Template

```xml
<t t-name="my_module.Ext"
   t-inherit="point_of_sale.ProductCard"
   t-inherit-mode="extension"
   owl="1">
    <xpath expr="//div[hasclass('product-name')]" position="after">
        <span>My addition</span>
    </xpath>
</t>
```

---

## 4. Common Helpers

### Frontend Utilities

```javascript
import { uuidv4 } from "@point_of_sale/utils";
const id = uuidv4();  // Generate UUID

// In templates:
<t t-esc="env.utils.formatCurrency(1234.56)"/>     // "$1,234.56"
<t t-esc="env.utils.formatProductQty(2.5)"/>        // "2.50"
<t t-if="env.utils.floatIsZero(0.0)"/>              // true
```

### Currency Formatting

```javascript
// In component
const currency = this.pos.currency;
const formatted = this.env.utils.formatCurrency(amount);
```

### Hooks Quick Reference

| Hook | Import | Purpose |
|------|--------|---------|
| `usePos()` | `@point_of_sale/app/store/pos_hook` | Reactive store |
| `useService()` | `@web/core/utils/hooks` | Any service |
| `useErrorHandlers()` | `@point_of_sale/app/utils/hooks` | Error handling |
| `useAsyncLockedMethod()` | `@point_of_sale/app/utils/hooks` | Debounce async |
| `useTrackedAsync()` | `@point_of_sale/app/utils/hooks` | Track async state |
| `useAutoFocusToLast()` | `@point_of_sale/app/utils/hooks` | Auto-focus |
| `useTime()` | `@point_of_sale/app/utils/time_hook` | Clock |
| `useBarcodeReader()` | `@point_of_sale/app/utils/hooks` | Barcode actions |

---

## 5. Model Quick Reference

### pos.config Key Fields

| Field | Type | Purpose |
|-------|------|---------|
| `name` | Char | Terminal name |
| `payment_method_ids` | Many2many | Available payments |
| `pricelist_id` | Many2one | Default pricelist |
| `journal_id` | Many2one | Sale journal |
| `picking_type_id` | Many2one | Stock operation type |
| `is_restaurant` | Boolean | Restaurant mode |
| `module_pos_hr` | Boolean | Employee login |
| `cash_rounding` | Boolean | Cash rounding |
| `current_session_id` | Many2one | Active session |

### pos.session States

```
opening_control → opened → closing_control → closed
```

### pos.order States

```
draft → paid → done / cancel / invoiced
```

### pos.order Key Fields

| Field | Type | Purpose |
|-------|------|---------|
| `session_id` | Many2one | POS session |
| `partner_id` | Many2one | Customer |
| `lines` | One2many | Order lines |
| `payment_ids` | One2many | Payments |
| `amount_total` | Monetary | Total amount |
| `state` | Selection | Order state |
| `account_move` | Many2one | Journal entry |
| `picking_id` | Many2one | Delivery |

---

## 6. Payment Terminal Quick Reference

### Backend Registration

```python
class PosPaymentMethod(models.Model):
    _inherit = 'pos.payment.method'
    
    use_payment_terminal = fields.Selection(
        selection_add=[('my_terminal', 'My Terminal')],
    )
```

### Frontend Registration

```javascript
import { PaymentInterface } from "@point_of_sale/app/payment/payment_interface";
import { registry } from "@web/core/registry";

class PaymentMyTerminal extends PaymentInterface {
    async sendPaymentRequest(amount) { /* ... */ }
    async sendPaymentCancel() { /* ... */ }
}

registry.category("pos_payment_methods").add("my_terminal", PaymentMyTerminal);
```

### Payment Status Values

```
pending → waiting → done     (success)
pending → waiting → retry    (retry needed)
pending → waiting → reversed (cancelled)
```

---

## 7. Asset Bundle Quick Reference

### Bundle Names

| Bundle | Purpose |
|--------|---------|
| `point_of_sale._assets_pos` | Main POS SPA — use this 95% of time |
| `point_of_sale.customer_display_assets` | Customer-facing screen |
| `pos_self_order.assets` | Self-order kiosk |
| `pos_preparation_display.assets` | Kitchen display |

### Asset Types

```python
'assets': {
    'point_of_sale._assets_pos': [
        'my_module/static/src/**/*.js',    # JavaScript
        'my_module/static/src/**/*.xml',   # XML templates
        'my_module/static/src/**/*.scss',  # Styles
    ],
}
```

---

## 8. Common Commands

### Install/Update Module

```bash
# Upgrade module via command line
./odoo-bin -d dbname -u my_pos_extension --stop-after-init

# Install new module
./odoo-bin -d dbname -i my_pos_extension --stop-after-init
```

### Debug POS

```
# Open POS in debug mode
https://your-odoo.com/pos/ui?debug=1

# Access browser console
F12 → Console → Look for errors
```

### Check Loaded Data

```javascript
// In browser console (POS must be open)
// Access store via OWL dev tools
// Or via POS internals:
const pos = owl.__owl__.app.rootComponent.app.__proto__.pos;
console.log(pos.config);
console.log(pos.session);
console.log(pos.orders.length);
```

---

## 9. Screen Quick Reference

### Built-in Screens

| Screen Name | Purpose |
|------------|---------|
| `ProductScreen` | Main POS UI |
| `PaymentScreen` | Payment processing |
| `ReceiptScreen` | Post-payment receipt |
| `TicketScreen` | Order history/refunds |
| `LoginScreen` | Employee PIN login |
| `PartnerList` | Customer selection |
| `ScaleScreen` | Weighing scale |
| `SaverScreen` | Save for later |

### Screen Navigation

```javascript
// Open screen with props
this.pos.showScreen("PaymentScreen", { orderUuid: uuid });

// Screen component accessing props
class PaymentScreen extends Component {
    static props = { orderUuid: String };
    setup() {
        const orderUuid = this.props.orderUuid;
    }
}
```

---

## 10. Registry Quick Reference

### Available Registries

```javascript
import { registry } from "@web/core/registry";

// Screens
registry.category("pos_screens").add("MyScreen", MyScreen);

// Services
registry.category("services").add("my_service", MyService);

// Models
registry.category("pos_available_models").add("my.model", MyModel);

// Payment methods
registry.category("pos_payment_methods").add("my_terminal", PaymentClass);

// Systray
registry.category("systray").add("my_item", { Component: MyComp });
```

---

## 11. Troubleshooting Quick Fix

| Problem | Fix |
|---------|-----|
| Module not loading | Check `__manifest__.py` syntax, restart Odoo |
| Asset not loading | Check bundle name is `point_of_sale._assets_pos` |
| Patch not applying | Check import path, verify `/** @odoo-module **/` comment |
| Template not rendering | Check XML is valid, verify template name matches |
| Field not in frontend | Add to `_load_pos_data_fields()` |
| BUS notification not received | Check channel name matches |
| Order not syncing | Check `pendingOrder` queues, IndexedDB |
| Reactivity broken | Use `usePos()` not direct store access |
| Service not found | Check service registration name |
| `/** @odoo-module **/` missing | Add to top of every JS file |

---

## 12. File Extension Quick Reference

| Extension | Purpose |
|-----------|---------|
| `.py` | Python backend code |
| `.js` | JavaScript/OWL frontend |
| `.xml` | QWeb templates (frontend or backend views) |
| `.csv` | Security rules, demo data |
| `.scss` | Styles |

### Required Comment

```javascript
// Every JS file MUST start with:
/** @odoo-module **/
```
