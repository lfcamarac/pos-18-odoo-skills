# Odoo 18 POS — Frontend (OWL/JavaScript) Patterns

> Complete reference for developing OWL components, extending models, creating screens, and customizing the POS UI.

---

## 1. OWL Component Structure

### Standard Component

```javascript
/** @odoo-module **/
import { Component, useState, onMounted, onWillStart } from "@odoo/owl";
import { usePos } from "@point_of_sale/app/store/pos_hook";
import { useService } from "@web/core/utils/hooks";

export class MyComponent extends Component {
    static template = "my_module.MyComponent";
    static components = { SubComponent };
    static props = {
        product: { type: Object },
        onClick: { type: Function, optional: true },
        className: { type: String, optional: true },
    };
    static defaultProps = {
        className: "",
    };

    setup() {
        this.pos = usePos();
        this.orm = useService("orm");
        this.notification = useService("notification");
        this.dialog = useService("dialog");

        this.state = useState({
            loading: false,
            expanded: false,
        });

        onWillStart(async () => {
            // Fetch initial data
        });

        onMounted(() => {
            // DOM is ready
        });
    }

    async handleClick() {
        this.state.loading = true;
        try {
            await this.pos.addLineToCurrentOrder({ product_id: this.props.product });
            this.props.onClick?.();
        } catch (error) {
            this.notification.add("Error adding product", { type: "danger" });
        } finally {
            this.state.loading = false;
        }
    }
}
```

### XML Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="my_module.MyComponent" owl="1">
        <div class="my-component" t-att-class="props.className">
            <h3 t-esc="props.product.display_name"/>
            <span t-esc="env.utils.formatCurrency(props.product.lst_price)"/>
            <button t-on-click="handleClick" t-att-disabled="state.loading">
                <t t-if="state.loading">Loading...</t>
                <t t-else="">Add to Cart</t>
            </button>
        </div>
    </t>
</templates>
```

---

## 2. State Management

### Local Reactive State

```javascript
setup() {
    // useState creates reactive object — changes trigger re-render
    this.state = useState({
        count: 0,
        items: [],
        filter: "all",
    });

    // Mutating state triggers re-render
    this.state.count++;
    this.state.items.push(newItem);
}
```

### Global Reactive Store

```javascript
setup() {
    this.pos = usePos();  // reactive proxy to PosStore

    // Accessing reactive properties triggers re-render
    this.pos.selectedOrder;       // current order
    this.pos.selectedPartner;     // current customer
    this.pos.numpadMode;          // "quantity" | "discount" | "price"
    this.pos.productListView;     // "grid" | "list"
}
```

### Non-Reactive Data

```javascript
setup() {
    // markRaw prevents reactivity — useful for large data caches
    this.unwatched = markRaw({
        productCache: new Map(),
        debouncedSearch: debounce(this.search, 300),
    });
}
```

---

## 3. Props Patterns

### Props Definition

```javascript
static props = {
    // Required prop
    order: Object,

    // Optional with default
    showTax: { type: Boolean, optional: true, default: false },

    // Enum-like validation
    size: { type: String, optional: true, validate: v => ['sm','md','lg'].includes(v) },

    // Function callback
    onClose: { type: Function, optional: true },

    // Slot support
    default: { type: Object, optional: true }, // t-slot="default"
};
```

### Props Drilling Alternative (Context via Store)

```javascript
// Instead of drilling props through 5 levels:
// Component A → B → C → D → E (orderUuid prop)

// Store it in PosStore:
this.pos.currentContext = { orderUuid, customerId, tableId };

// Access anywhere:
const ctx = this.pos.currentContext;
```

---

## 4. Hooks Reference

### usePos()

```javascript
import { usePos } from "@point_of_sale/app/store/pos_hook";

setup() {
    this.pos = usePos();
    // Returns reactive proxy to PosStore
    // Any access triggers tracking for re-render
}
```

### useService()

```javascript
import { useService } from "@web/core/utils/hooks";

setup() {
    this.orm = useService("orm");
    this.dialog = useService("dialog");
    this.notification = useService("notification");
    this.bus = useService("bus_service");
    this.hardware_proxy = useService("hardware_proxy");
    this.barcode = useService("barcode_reader");
}
```

### useErrorHandlers()

```javascript
import { useErrorHandlers } from "@point_of_sale/app/utils/hooks";

setup() {
    useErrorHandlers();
    // Injects component._handlePushOrderError()
    // Catches order sync errors and shows user-friendly messages
}
```

### useAsyncLockedMethod()

```javascript
import { useAsyncLockedMethod } from "@point_of_sale/app/utils/hooks";

setup() {
    // Prevents double-click / double-submit
    this.validateOrder = useAsyncLockedMethod(this.validateOrder.bind(this));
}

async validateOrder() {
    // This will only run once at a time, even if called rapidly
    await this.pos.syncAllOrders();
}
```

### useTrackedAsync()

```javascript
import { useTrackedAsync } from "@point_of_sale/app/utils/hooks";

setup() {
    // Tracks async operation status: idle → loading → success/error
    this.printReceipt = useTrackedAsync(async () => {
        return await this.pos.printReceipt();
    });
}

// In template:
// <t t-if="printReceipt.status === 'loading'">Printing...</t>
// <t t-if="printReceipt.status === 'success'">Printed!</t>
// <t t-if="printReceipt.status === 'error'">Failed</t>
```

### useAutoFocusToLast()

```javascript
import { useAutoFocusToLast } from "@point_of_sale/app/utils/hooks";

setup() {
    useAutoFocusToLast();
    // Auto-focuses last <input> with t-ref="root" on mount
}
```

### useBarcodeReader()

```javascript
import { useBarcodeReader } from "@point_of_sale/app/utils/hooks";

setup() {
    useBarcodeReader({
        product: (barcode) => this.handleProductBarcode(barcode),
        client: (barcode) => this.handleClientBarcode(barcode),
        cashier: (barcode) => this.handleCashierBarcode(barcode),
        discount: (barcode) => this.handleDiscountBarcode(barcode),
        giftcard: (barcode) => this.handleGiftCardBarcode(barcode),
    });
}
```

### useTime()

```javascript
import { useTime } from "@point_of_sale/app/utils/time_hook";

setup() {
    this.time = useTime();
    // this.time.hours, this.time.day, this.time.date — updated every 500ms
}
```

---

## 5. Screen Development

### Creating a New Screen

```javascript
/** @odoo-module **/
import { Component, useState } from "@odoo/owl";
import { usePos } from "@point_of_sale/app/store/pos_hook";
import { registry } from "@web/core/registry";

class MyCustomScreen extends Component {
    static template = "my_module.MyCustomScreen";
    static storeOnOrder = false;  // Don't persist state on order
    static props = {
        close: { type: Function, optional: true },
        initialData: { type: Object, optional: true },
    };

    setup() {
        this.pos = usePos();
        this.state = useState({
            searchQuery: "",
            selectedItems: [],
        });
    }

    onConfirm() {
        // Navigate back
        this.pos.showScreen("ProductScreen");
    }

    onCancel() {
        this.pos.showScreen("ProductScreen");
    }
}

registry.category("pos_screens").add("MyCustomScreen", MyCustomScreen);
```

### Template

```xml
<t t-name="my_module.MyCustomScreen" owl="1">
    <div class="screen my-custom-screen">
        <!-- Header -->
        <div class="screen-header">
            <button class="btn btn-secondary" t-on-click="onCancel">
                <i class="fa fa-arrow-left"/> Back
            </button>
            <h2>My Custom Screen</h2>
        </div>

        <!-- Body -->
        <div class="screen-body">
            <input type="text" t-model="state.searchQuery" placeholder="Search..."/>
            <!-- Screen content -->
        </div>

        <!-- Footer -->
        <div class="screen-footer">
            <button class="btn btn-primary" t-on-click="onConfirm">
                Confirm <i class="fa fa-check"/>
            </button>
        </div>
    </div>
</t>
```

### Navigation to Custom Screen

```javascript
// From any component:
this.pos.showScreen("MyCustomScreen", {
    initialData: { source: "ProductScreen" },
    close: () => this.pos.showScreen("ProductScreen"),
});
```

---

## 6. Model Extension

### Patching PosOrder

```javascript
/** @odoo-module **/
import { PosOrder } from "@point_of_sale/app/models/pos_order";
import { patch } from "@web/core/utils/patch";

patch(PosOrder.prototype, {
    // New computed getter
    get myCustomTotal() {
        return this.lines.reduce((sum, line) => sum + line.price * line.quantity, 0);
    },

    // New method
    applyCustomDiscount(percent) {
        for (const line of this.lines) {
            line.update({ discount: percent });
        }
        this.updateLastOrderChange();
    },

    // Override existing method
    add_paymentline(pm) {
        // Custom logic before
        console.log("Adding payment:", pm.name);
        const result = super.add_paymentline(pm);
        // Custom logic after
        return result;
    },
});
```

### Patching PosOrderline

```javascript
/** @odoo-module **/
import { PosOrderline } from "@point_of_sale/app/models/pos_order_line";
import { patch } from "@web/core/utils/patch";

patch(PosOrderline.prototype, {
    get myCustomPrice() {
        // Custom price computation
        const base = this.price;
        const surcharge = this.product_id.my_surcharge || 0;
        return base + surcharge;
    },

    set_custom_quantity(qty) {
        this.update({ quantity: qty });
        this.order.updateLastOrderChange();
    },
});
```

### Registering a New Model

```javascript
/** @odoo-module **/
import { Base } from "@point_of_sale/app/models/related_models";
import { registry } from "@web/core/registry";

class MyCustomModel extends Base {
    static pythonModel = "my.custom.model";

    setup(vals) {
        this.name = vals.name;
        this.value = vals.value;
    }

    get displayName() {
        return `${this.name} (${this.value})`;
    }
}

registry.category("pos_available_models").add("my.custom.model", MyCustomModel);
```

---

## 7. Store Extension

### Patching PosStore

```javascript
/** @odoo-module **/
import { PosStore } from "@point_of_sale/app/store/pos_store";
import { patch } from "@web/core/utils/patch";

patch(PosStore.prototype, {
    setup() {
        super.setup(...arguments);

        // Add custom reactive state
        this.myFeatureEnabled = false;
        this.customData = null;

        // Add to serviceDependencies if needed
        // this.myCustomService = useService("my_service");
    },

    // Override existing method
    addLineToCurrentOrder(vals, opts) {
        // Custom validation
        if (vals.product_id && !this._canAddProduct(vals.product_id)) {
            return null;
        }
        return super.addLineToCurrentOrder(vals, opts);
    },

    // New method
    _canAddProduct(productId) {
        const product = this.models['product.product'].find(p => p.id === productId);
        return product && !product.is_archived;
    },

    // Custom action
    openMyCustomScreen() {
        this.showScreen("MyCustomScreen", {
            orderUuid: this.selectedOrderUuid,
        });
    },
});
```

---

## 8. Template Extension

### XML Template Extension (Recommended)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">

    <!-- Add price to product card -->
    <t t-name="my_module.ProductCardPrice"
       t-inherit="point_of_sale.ProductCard"
       t-inherit-mode="extension"
       owl="1">
        <xpath expr="//div[hasclass('product-name')]" position="after">
            <span class="product-price text-muted text-sm">
                <t t-esc="env.utils.formatCurrency(props.product.lst_price)"/>
            </span>
        </xpath>
    </t>

    <!-- Add badge to order line -->
    <t t-name="my_module.OrderLineBadge"
       t-inherit="point_of_sale.Orderline"
       t-inherit-mode="extension"
       owl="1">
        <xpath expr="//div[hasclass('orderline-name')]" position="after">
            <t t-if="props.line.product_id.is_new">
                <span class="badge bg-success">NEW</span>
            </t>
        </xpath>
    </t>

    <!-- Add button to control buttons -->
    <t t-name="my_module.CustomButton"
       t-inherit="point_of_sale.ControlButtons"
       t-inherit-mode="extension"
       owl="1">
        <xpath expr="//div[hasclass('control-buttons')]" position="inside">
            <button class="btn btn-secondary" t-on-click="() => this.props.pos.openMyCustomScreen()">
                My Feature
            </button>
        </xpath>
    </t>

</templates>
```

### env.utils Helpers

```xml
<!-- Format currency -->
<t t-esc="env.utils.formatCurrency(amount)"/>

<!-- Format product quantity -->
<t t-esc="env.utils.formatProductQty(qty)"/>

<!-- Check if float is zero -->
<t t-if="env.utils.floatIsZero(value)">Zero</t>
```

---

## 9. Service Creation

```javascript
/** @odoo-module **/
import { registry } from "@web/core/registry";

const MyPosService = {
    dependencies: ["orm", "notification", "bus_service"],

    start(env, { orm, notification, bus_service }) {
        return {
            async syncCustomData(records) {
                try {
                    const result = await orm.call("my.model", "sync_data", [records]);
                    notification.add("Data synced successfully", { type: "success" });
                    return result;
                } catch (error) {
                    notification.add("Sync failed: " + error.message, { type: "danger" });
                    throw error;
                }
            },

            setupNotifications() {
                // Listen for custom events
                bus_service.addEventListener("notification", ({ detail }) => {
                    if (detail[0].type === "my_custom_event") {
                        // Handle event
                    }
                });
            },
        };
    },
};

registry.category("services").add("my_pos_service", MyPosService);
```

### Consuming Service

```javascript
setup() {
    this.myService = useService("my_pos_service");

    // Use in component
    onMounted(() => {
        this.myService.setupNotifications();
    });
}
```

---

## 10. BUS Communication

### Frontend Subscription

```javascript
import { getOnNotified } from "@point_of_sale/utils";

setup() {
    const { bus_service } = this.env.services;

    // Subscribe to custom channel
    getOnNotified(bus_service, "my.custom.channel").add(
        ({ payload }) => {
            this.handleCustomNotification(payload);
        }
    );
}
```

### Backend Notification

```python
# Send BUS notification from Python
self.env['bus.bus']._sendone(
    channel=(self._cr.dbname, 'pos.session', session_id),
    message_type="notification",
    payload={
        'type': 'my_custom_event',
        'data': { 'order_id': order.id },
    },
)
```

---

## 11. Payment Integration

### Custom Payment Terminal

```javascript
/** @odoo-module **/
import { PaymentInterface } from "@point_of_sale/app/payment/payment_interface";
import { registry } from "@web/core/registry";

class PaymentMyTerminal extends PaymentInterface {
    async sendPaymentRequest(amount) {
        const payment = this.payment;

        try {
            // Send payment request to terminal via hardware proxy or API
            const result = await this.terminal_proxy.sendPay(amount);

            if (result.success) {
                payment.set_status("done");
                payment.set_transaction_id(result.transaction_id);
            } else {
                payment.set_status("retry");
            }
        } catch (error) {
            payment.set_status("retry");
        }
    }

    async sendPaymentCancel() {
        await this.terminal_proxy.cancel();
        this.payment.set_status("reversed");
    }
}

registry.category("pos_payment_methods").add("my_terminal", PaymentMyTerminal);
```

### Payment Status Lifecycle

```
pending → waiting → done      (success)
pending → waiting → retry     (needs retry)
pending → waiting → reversed  (cancelled)
```

---

## 12. Dialog Patterns

### Using Dialog Service

```javascript
setup() {
    this.dialog = useService("dialog");
}

async showDialog() {
    const result = await this.dialog.add(MyDialogComponent, {
        title: "My Dialog",
        body: "Dialog content",
    });

    if (result.confirmed) {
        // User confirmed
    }
}
```

### Making Action Awaitable

```javascript
import { makeActionAwaitable } from "@point_of_sale/app/store/make_awaitable_dialog";

// Turn a screen navigation into an awaitable action
const result = await makeActionAwaitable(this.pos, {
    type: "ir.actions.act_window",
    name: "My Action",
    res_model: "my.model",
    views: [[false, "form"]],
    target: "new",
});
```

---

## 13. Number Buffer Integration

```javascript
import { useService } from "@web/core/utils/hooks";

setup() {
    this.numberBuffer = useService("number_buffer");

    // Current buffer value
    const value = this.numberBuffer.get();

    // Mode: "quantity", "discount", "price"
    const mode = this.pos.numpadMode;

    // Apply buffer to order line
    this.numberBuffer.apply();
}
```

---

## 14. Testing Patterns

### Unit Test

```javascript
import { createPosUnitEnvironment } from "@point_of_sale/../tests/unit/utils";

QUnit.module("My POS Extension", (hooks) => {
    QUnit.test("add product to order", async (assert) => {
        const { pos, clickProduct } = await createPosUnitEnvironment();
        await clickProduct(productId);
        assert.strictEqual(pos.selectedOrder.orderlines.length, 1);
        assert.strictEqual(pos.selectedOrder.orderlines[0].quantity, 1);
    });

    QUnit.test("custom feature works", async (assert) => {
        const { pos } = await createPosUnitEnvironment();
        pos.myCustomMethod();
        assert.true(pos.myFeatureEnabled);
    });
});
```
