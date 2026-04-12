# Odoo 18 POS — Extension & Override Patterns

> Practical, copy-paste-friendly patterns for extending the POS module safely.

---

## 1. Module Structure

### Minimal POS Extension Module

```
my_pos_extension/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── pos_config.py          # Backend extensions
├── static/src/
│   ├── overrides/
│   │   └── pos_store.js       # Frontend overrides
│   └── xml/
│       └── my_templates.xml   # XML template extensions
└── views/
    └── pos_config_views.xml   # Backend view extensions
```

### __manifest__.py Templates

```python
# Type 1: Extend existing POS UI
{
    'name': 'My POS Extension',
    'depends': ['point_of_sale'],
    'assets': {
        'point_of_sale._assets_pos': [
            'my_pos_extension/static/src/overrides/**/*.js',
            'my_pos_extension/static/src/xml/**/*.xml',
        ],
    },
}

# Type 2: Add new screen
{
    'name': 'My POS Screen',
    'depends': ['point_of_sale'],
    'assets': {
        'point_of_sale._assets_pos': [
            'my_pos_screen/static/src/screens/**/*.js',
            'my_pos_screen/static/src/xml/**/*.xml',
        ],
    },
}

# Type 3: Extend backend views
{
    'name': 'My POS Backend',
    'depends': ['point_of_sale'],
    'data': [
        'views/pos_config_views.xml',
        'views/pos_order_views.xml',
    ],
}

# Type 4: Full extension (backend + frontend)
{
    'name': 'My Full POS Extension',
    'depends': ['point_of_sale'],
    'data': [
        'security/ir.model.access.csv',
        'views/pos_config_views.xml',
    ],
    'assets': {
        'point_of_sale._assets_pos': [
            'my_full_extension/static/src/**/*.js',
            'my_full_extension/static/src/**/*.xml',
        ],
    },
}
```

---

## 2. Backend Extension Patterns

### Pattern: Add Field to pos.config

```python
# models/pos_config.py
from odoo import models, fields, api

class PosConfig(models.Model):
    _inherit = 'pos.config'

    my_custom_field = fields.Boolean('Enable My Feature')
    my_related_model = fields.Many2one('my.model', 'Related Model')
    my_payment_method_ids = fields.Many2many(
        'pos.payment.method',
        string='Custom Payment Methods',
    )

    def _get_pos_ui_pos_config(self):
        """Ensure custom fields are loaded in frontend."""
        data = super()._get_pos_ui_pos_config()
        # Add custom fields to the data sent to frontend
        data['my_custom_field'] = self.my_custom_field
        return data
```

### Pattern: Add Field to pos.order

```python
# models/pos_order.py
from odoo import models, fields, api

class PosOrder(models.Model):
    _inherit = 'pos.order'

    my_custom_field = fields.Char('Custom Field')
    my_line_ids = fields.One2many('my.order.line', 'order_id')

    @api.model
    def _order_fields(self, ui_order):
        """Extend order fields received from frontend."""
        result = super()._order_fields(ui_order)
        result['my_custom_field'] = ui_order.get('my_custom_field')
        return result

    def _process_order(self, order_data, existing_order=None):
        """Process custom data during order sync."""
        order_id = super()._process_order(order_data, existing_order)
        order = self.browse(order_id)
        
        # Process custom lines
        for line_data in order_data.get('my_line_ids', []):
            self.env['my.order.line'].create({
                'order_id': order_id,
                'name': line_data['name'],
                'value': line_data['value'],
            })
        
        return order_id
```

### Pattern: Extend pos.load.mixin

```python
class MyModel(models.Model):
    _name = 'my.model'
    _inherit = ['my.model', 'pos.load.mixin']

    name = fields.Char()
    pos_config_id = fields.Many2one('pos.config')

    def _load_pos_data_domain(self):
        return [('pos_config_id', '=', self._load_pos_data_config().id)]

    def _load_pos_data_fields(self):
        return ['id', 'name', 'pos_config_id']

    def _load_pos_data_relations(self):
        return []
```

---

## 3. Frontend Model Override Patterns

### Pattern: Patch PosOrder

```javascript
/** @odoo-module */
import { PosOrder } from "@point_of_sale/app/models/pos_order";
import { patch } from "@web/core/utils/patch";

patch(PosOrder.prototype, {
    // New getter
    get myCustomTotal() {
        return this.lines.reduce((sum, line) => {
            return sum + (line.price * line.quantity * (1 - (line.discount || 0) / 100));
        }, 0);
    },

    // New method
    applyCustomDiscount(percent) {
        for (const line of this.lines) {
            line.update({ discount: percent });
        }
        this.updateLastOrderChange();
    },

    // Override existing method
    add_paymentline(payment_method) {
        // Custom logic before
        console.log("Adding payment method:", payment_method.name);
        const result = super.add_paymentline(payment_method);
        // Custom logic after
        return result;
    },
});
```

### Pattern: Patch PosStore

```javascript
/** @odoo-module */
import { PosStore } from "@point_of_sale/app/store/pos_store";
import { patch } from "@web/core/utils/patch";

patch(PosStore.prototype, {
    setup() {
        super.setup(...arguments);

        // Add custom reactive state
        this.myFeatureEnabled = this.config.my_custom_field || false;
        this.customData = null;
    },

    // Override method
    addLineToCurrentOrder(vals, opts) {
        // Custom validation
        if (!this._validateProductAddition(vals)) {
            return null;
        }
        return super.addLineToCurrentOrder(vals, opts);
    },

    // New method
    _validateProductAddition(vals) {
        // Custom validation logic
        return true;
    },

    // Custom action
    openMyScreen() {
        this.showScreen("MyCustomScreen", {
            orderUuid: this.selectedOrderUuid,
        });
    },
});
```

### DO and DON'T

| DO | DON'T |
|---|---|
| Use `patch()` to extend existing classes | Modify core POS files directly |
| Call `super.setup(...arguments)` in setup | Forget to call super in overridden methods |
| Use `usePos()` to access store | Access `this.pos` without the hook |
| Store custom state in `this.pos.myState` | Modify existing POS state directly |
| Use `updateLastOrderChange()` after modifications | Modify order data without notifying |

---

## 4. Template Extension Patterns

### Pattern: Add Element to Existing Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">

    <!-- Add price badge to product card -->
    <t t-name="my_module.ProductCardPrice"
       t-inherit="point_of_sale.ProductCard"
       t-inherit-mode="extension"
       owl="1">
        <xpath expr="//div[hasclass('product-name')]" position="after">
            <span class="product-price-badge">
                <t t-esc="env.utils.formatCurrency(props.product.lst_price)"/>
            </span>
        </xpath>
    </t>

</templates>
```

### Position Strategies

```xml
<!-- Add AFTER an element -->
<xpath expr="//div[hasclass('product-name')]" position="after">
    <span>After</span>
</xpath>

<!-- Add BEFORE an element -->
<xpath expr="//div[hasclass('product-name')]" position="before">
    <span>Before</span>
</xpath>

<!-- Add INSIDE an element -->
<xpath expr="//div[hasclass('order-widget')]" position="inside">
    <span>Inside at end</span>
</xpath>

<!-- Replace an element -->
<xpath expr="//div[hasclass('product-image')]" position="replace">
    <img t-att-src="props.product.image_url"/>
</xpath>

<!-- Add inside at specific position -->
<xpath expr="//div[hasclass('control-buttons')]/button[last()]" position="after">
    <button>My Button</button>
</xpath>
```

### XPath Targeting Tips

```xml
<!-- Target by class (recommended — most resilient to updates) -->
<xpath expr="//div[hasclass('product-name')]" position="after">

<!-- Target by attribute -->
<xpath expr="//input[@t-ref='product-search']" position="after">

<!-- Target by position in list -->
<xpath expr="//div[hasclass('buttons')]/button[1]" position="before">

<!-- Target last element -->
<xpath expr="//div[hasclass('items')]/div[last()]" position="after">

<!-- Target by text content -->
<xpath expr="//span[text()='Total']" position="after">

<!-- DON'T: Fragile selectors -->
<xpath expr="/div/div[3]/span" position="after">  <!-- Breaks on core update -->
```

### DO and DON'T

| DO | DON'T |
|---|---|
| Use `t-inherit-mode="extension"` | Use `t-inherit-mode="replacement"` (destroys original) |
| Use `hasclass()` in XPath | Use numeric indices like `div[3]` |
| Keep extensions minimal | Rewrite entire templates |
| Test after Odoo updates | Assume template structure is stable |
| Use `env.utils.formatCurrency()` | Format numbers manually |

---

## 5. Screen Addition

### Pattern: Complete Custom Screen

```javascript
/** @odoo-module */
import { Component, useState, onWillStart } from "@odoo/owl";
import { usePos } from "@point_of_sale/app/store/pos_hook";
import { useService } from "@web/core/utils/hooks";
import { registry } from "@web/core/registry";

class MyCustomScreen extends Component {
    static template = "my_module.MyCustomScreen";
    static components = { };
    static storeOnOrder = false;
    static props = {
        close: { type: Function, optional: true },
        orderUuid: { type: String, optional: true },
    };

    setup() {
        this.pos = usePos();
        this.orm = useService("orm");
        this.notification = useService("notification");

        this.state = useState({
            loading: false,
            records: [],
            searchQuery: "",
            selectedIds: [],
        });

        onWillStart(async () => {
            await this._loadRecords();
        });
    }

    async _loadRecords() {
        this.state.loading = true;
        try {
            this.state.records = await this.orm.searchRead(
                "my.model", [], ["id", "name", "value"]
            );
        } finally {
            this.state.loading = false;
        }
    }

    onConfirm() {
        this.pos.showScreen("ProductScreen");
    }
}

registry.category("pos_screens").add("MyCustomScreen", MyCustomScreen);
```

### Screen Template

```xml
<t t-name="my_module.MyCustomScreen" owl="1">
    <div class="screen my-custom-screen d-flex flex-column h-100">
        <!-- Header -->
        <div class="screen-header d-flex align-items-center p-2 border-bottom">
            <button class="btn btn-secondary btn-sm me-2"
                    t-on-click="() => this.pos.showScreen('ProductScreen')">
                <i class="fa fa-arrow-left"/> Back
            </button>
            <h2 class="m-0">My Custom Screen</h2>
        </div>

        <!-- Body -->
        <div class="screen-body flex-grow-1 overflow-auto p-3">
            <t t-if="state.loading">
                <div class="text-center">Loading...</div>
            </t>
            <t t-else="">
                <div class="list-group">
                    <t t-foreach="state.records" t-as="record" t-key="record.id">
                        <div class="list-group-item">
                            <t t-esc="record.name"/>
                        </div>
                    </t>
                </div>
            </t>
        </div>

        <!-- Footer -->
        <div class="screen-footer p-2 border-top">
            <button class="btn btn-primary w-100" t-on-click="onConfirm">
                Done <i class="fa fa-check"/>
            </button>
        </div>
    </div>
</t>
```

---

## 6. Payment Method Addition

### Backend

```python
# models/pos_payment_method.py
class PosPaymentMethod(models.Model):
    _inherit = 'pos.payment.method'

    use_payment_terminal = fields.Selection(
        selection_add=[('my_terminal', 'My Terminal')],
        ondelete={'my_terminal': 'set default'}
    )
```

### Frontend

```javascript
/** @odoo-module */
import { PaymentInterface } from "@point_of_sale/app/payment/payment_interface";
import { patch } from "@web/core/utils/patch";

class PaymentMyTerminal extends PaymentInterface {
    async sendPaymentRequest(amount) {
        const payment = this.payment;

        try {
            // Call your payment terminal API
            const response = await this.env.services.orm.call(
                "my.payment.gateway",
                "process_payment",
                [amount, { currency: this.pos.currency.name }]
            );

            if (response.success) {
                payment.set_status("done");
                payment.set_transaction_id(response.transaction_id);
            } else {
                payment.set_status("retry");
            }
        } catch (error) {
            this.env.services.notification.add(
                "Payment failed: " + error.message,
                { type: "danger" }
            );
            payment.set_status("retry");
        }
    }

    async sendPaymentCancel() {
        await this.env.services.orm.call(
            "my.payment.gateway",
            "cancel_payment",
            []
        );
        this.payment.set_status("reversed");
    }
}

// Register for the terminal type
import { registry } from "@web/core/registry";
registry.category("pos_payment_methods").add("my_terminal", PaymentMyTerminal);
```

---

## 7. Notification System

### Backend: Send BUS Notification

```python
# Send to specific POS session
session = self.env['pos.session'].browse(session_id)
self.env['bus.bus']._sendone(
    channel=(self._cr.dbname, 'pos.session', session.id),
    message_type="notification",
    payload={
        'type': 'my_custom_notification',
        'data': {
            'message': 'Order processed',
            'order_id': order.id,
        },
    }
)

# Send to all POS sessions
self.env['bus.bus'].sendmany(
    [(self._cr.dbname, 'pos.session', sid) for sid in session_ids],
    message_type="notification",
    payload={...}
)
```

### Frontend: Receive BUS Notification

```javascript
/** @odoo-module */
import { PosStore } from "@point_of_sale/app/store/pos_store";
import { patch } from "@web/core/utils/patch";
import { getOnNotified } from "@point_of_sale/utils";

patch(PosStore.prototype, {
    setup() {
        super.setup(...arguments);

        const { bus_service } = this.env.services;

        getOnNotified(bus_service, "my_custom_notification").add(
            ({ payload }) => {
                const data = payload.data;
                this.notification.add(data.message, { type: "success" });
            }
        );
    },
});
```

---

## 8. Multi-module Integration

### Pattern: Bridge Module

```python
# __manifest__.py
{
    'name': 'POS Bridge: Module A + Module B',
    'depends': ['pos_module_a', 'pos_module_b'],
    'auto_install': True,  # Auto-installs when both deps are present
    'data': [
        'views/bridge_views.xml',
    ],
}
```

### Pattern: Feature Flag

```python
class PosConfig(models.Model):
    _inherit = 'pos.config'

    module_my_feature = fields.Boolean('Enable My Feature')

    @api.constrains('module_my_feature')
    def _check_my_feature(self):
        for config in self:
            if config.module_my_feature:
                # Check dependencies
                if not config.other_required_field:
                    raise ValidationError(_('Other field required'))
```

---

## 9. Testing Extensions

### Python Test

```python
from odoo.tests import tagged
from odoo.tests.common import TransactionCase

@tagged('post_install', '-at_install')
class TestMyPosExtension(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.pos_config = cls.env.ref('point_of_sale.main_pos_config')
        cls.pos_config.open()
        cls.session = cls.pos_config.current_session_id

    def test_custom_field_on_order(self):
        order = self.env['pos.order'].create({
            'session_id': self.session.id,
            'my_custom_field': 'test_value',
        })
        self.assertEqual(order.my_custom_field, 'test_value')

    def test_sync_from_ui_with_custom_data(self):
        order_data = {
            'my_custom_field': 'test',
            'lines': [],
            'payments': [],
        }
        result = self.env['pos.order'].sync_from_ui(
            self.session.id, order_data
        )
        order = self.env['pos.order'].browse(result['id'])
        self.assertEqual(order.my_custom_field, 'test')
```

### JavaScript Test

```javascript
import { createPosUnitEnvironment } from "@point_of_sale/../tests/unit/utils";

QUnit.module("My POS Extension", (hooks) => {
    QUnit.test("custom feature enabled", async (assert) => {
        const { pos } = await createPosUnitEnvironment();
        assert.true(pos.myFeatureEnabled, "Feature should be enabled");
    });
});
```
