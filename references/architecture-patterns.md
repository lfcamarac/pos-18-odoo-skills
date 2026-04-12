# Odoo 18 POS — Architecture Patterns

> Deep reference for understanding how the Odoo 18 POS system is architected — from bootstrap to multi-device synchronization.

---

## 1. SPA Bootstrap Architecture

### Asset Bundle Hierarchy

```
point_of_sale.assets_prod                    ← Loaded at /pos/ui
│
├── point_of_sale._assets_pos               ← Main bundle (90% of code)
│   ├── point_of_sale.base_app              ← Foundation layer
│   │   ├── Bootstrap CSS helpers
│   │   ├── bus_service.js                  ← WebSocket connection
│   │   ├── bus_parameters_service.js
│   │   ├── multi_tab_service.js
│   │   └── barcodes/                       ← Barcode parsing
│   │       ├── barcode_service.js
│   │       └── parsers/
│   ├── @odoo/owl (OWL framework)
│   ├── luxon (date/time)
│   ├── zxing (QR/barcode reading)
│   ├── account static helpers              ← Tax computation
│   ├── mail/static/src/sound_effects/      ← Notification sounds
│   └── point_of_sale/static/src/**/*       ← All POS source code
│
└── point_of_sale/static/src/app/main.js    ← App bootstrap entry point
```

### Bootstrap Sequence

```
1. Browser → /pos/ui
2. Odoo renders QWeb template "point_of_sale.assets_prod"
3. All JS/CSS assets load
4. main.js executes:
   a. Creates Odoo environment (services, registries)
   b. Calls PosData.loadInitialData()
   c. RPC → pos.session.load_data(session_id)
   d. Server returns all models with fields, relations, records
   e. createRelatedModels() instantiates JS model classes
   f. Each model.loadData() populates records from server data
   g. PosStore initialized with all data
   h. First screen rendered (LoginScreen or ProductScreen)
5. POS is interactive
```

### main.js Pattern

```javascript
/** @odoo-module **/
import { startWhenReady } from "@point_of_sale/app/main/start_when_ready";

startWhenReady();
// Waits for DOM ready → creates env → mounts PosApp component
```

### Separate SPAs

The POS system contains **three separate SPAs**, each with its own bundle and controller:

| SPA | Route | Bundle | Purpose |
|-----|-------|--------|---------|
| Main POS | `/pos/ui` | `point_of_sale.assets_prod` | Cashier UI |
| Preparation Display | `/pos_preparation_display/web?display_id=X` | `pos_preparation_display.assets` | Kitchen display |
| Self-Order Kiosk | `/pos/self_order/<uuid>` | `pos_self_order.assets` | Customer-facing kiosk |
| Customer Display | `/pos/customer_display` | `point_of_sale.customer_display_assets` | Secondary screen |

Each SPA has its own:
- Asset bundle declaration in `__manifest__.py`
- Controller route
- QWeb template for HTML boot
- Separate OWL app instance

---

## 2. Data Flow Patterns

### 2.1 Initial Data Loading

```
Frontend                          Backend
────────                          ───────
PosData.loadInitialData()
    │
    ├── RPC: pos.session.load_data(session_id)
    │       │
    │       ├── _load_pos_data() iterates all models
    │       │   in load_data configuration:
    │       │
    │       │   for each model:
    │       │     1. _load_pos_data_domain() → search domain
    │       │     2. _load_pos_data_fields() → field list
    │       │     3. _load_pos_data_relations() → nested models
    │       │     4. search_read() → fetch records
    │       │     5. Serialize with related data
    │       │
    │       └── Returns: { models: [ { name, fields, records, relations } ] }
    │
    ├── createRelatedModels()
    │   └── For each model in registry.category("pos_available_models"):
    │       - Instantiate with server data
    │       - Set up reactive properties
    │
    └── loadIndexedDBData()
        └── Restore offline-persisted orders
```

### 2.2 Model Instantiation

```javascript
// createRelatedModels() — from related_models.js
function createRelatedModels(models, dynamicModels, baseData) {
    // Iterates registry.category("pos_available_models")
    // For each registered model:
    //   1. Creates model class instance
    //   2. Passes { models, records, model, dynamicModels, baseData }
    //   3. Calls model.setup(vals) for each record
    //   4. Sets up reactive relationships
    return models;
}
```

### 2.3 ORM Operation Flow

```
UI Action (click product)
    │
    ▼
model.update({ quantity: 2 })     ← PosOrderline.update()
    │
    ├── Updates reactive properties
    ├── Calls setDirty()          ← marks record for sync
    ├── Generates command: { type: 'UPDATE', data: { id, fields } }
    └── Triggers re-render (OWL reactivity)

Later: syncAllOrders()
    │
    ├── Collects all pending commands
    │   - pendingOrder.create → new orders
    │   - pendingOrder.write → modified orders
    │   - pendingOrder.delete → deleted orders
    │
    ├── RPC: pos.order.sync_from_ui(session_id, serialized_orders)
    │
    ├── Server processes:
    │   - Creates/updates pos.order records
    │   - Creates pos.order.line, pos.payment records
    │   - Generates stock moves
    │   - Triggers accounting entries
    │   - Returns serialized result
    │
    ├── PosData.execute() receives response
    │   ├── models.loadData() hydrates store
    │   ├── missingRecursive() fetches related records
    │   └── Clears pendingOrder queues
    │
    └── UI auto-updates via OWL reactivity
```

### 2.4 WebSocket Multi-Device Sync

```
Device A                          Server                          Device B
───────                           ───────                         ────────
                                  pos.bus.mixin.notify_pos_sessions()
                                      │
                                      ├── Writes to pos.bus table
                                      │
                                      └── WebSocket broadcast
                                              │
                                              ├── DevicesSynchronisation receives
                                              │
                                              ├── Reads notification:
                                              │   { model, id, command }
                                              │
                                              ├── RPC: readDataFromServer(model, ids)
                                              │
                                              ├── models.loadData() updates store
                                              │
                                              └── UI re-renders automatically
```

### 2.5 BUS Notification Pattern (Backend → Frontend)

```python
# Backend: send notification
class PosOrder(models.Model):
    _inherit = ['pos.order', 'pos.bus.mixin']

    def write(self, vals):
        res = super().write(vals)
        self._notify_pos_sessions()  # sends BUS notification
        return res
```

```javascript
// Frontend: receive notification
import { getOnNotified } from "@point_of_sale/utils";

getOnNotified(this.env.services.bus_service, "pos.order").add(
    ({ payload }) => {
        // payload: { model, id, command, data }
        if (payload.command === "UPDATE") {
            this.models['pos.order'].loadData([payload.id]);
        }
    }
);
```

---

## 3. Offline Architecture

### IndexedDB Storage

```
IndexedDB Database: "odoo-pos"
├── Store: "pos.order"
│   └── Pending orders (draft, not yet synced)
├── Store: "pos.order.line"
│   └── Lines belonging to pending orders
├── Store: "pos.payment"
│   └── Payment records for pending orders
├── Store: "stock.lot"
│   └── Cached lot/serial numbers
└── Store: "product.product"
    └── Cached product data (for offline search)
```

### Offline Queue

```javascript
// In PosStore
this.pendingOrder = {
    create: new Set(),   // Orders created offline
    write: new Set(),    // Orders modified offline
    delete: new Set(),   // Orders deleted offline
};
```

### Sync on Reconnect

```javascript
// DevicesSynchronisation effect
effect(() => {
    if (this.isOnline && this.hasPendingChanges) {
        this.syncDataWithIndexedDB();
    }
});

// Sync flow:
// 1. Read pending records from IndexedDB
// 2. Batch into RPC calls
// 3. Execute sync_from_ui
// 4. On success: delete from IndexedDB
// 5. On failure: keep in queue, retry later
```

### Mutex for Order Push

```javascript
// In PosStore
this.pushOrderMutex = new Mutex();

// Ensures only one order sync happens at a time
async pushSingleOrder(order) {
    return await this.pushOrderMutex.runExclusive(async () => {
        // Only one order syncs at a time
        return await this.orm.call("pos.order", "sync_from_ui", [...]);
    });
}
```

### Design Rules

1. **Never block UI** — all server calls are async
2. **Queue all mutations** — create/write/delete go to pendingOrder
3. **Sync opportunistically** — sync when online, queue when offline
4. **Handle conflicts** — same order on multiple devices
5. **Graceful degradation** — core POS works without network

---

## 4. State Management

### PosStore as Reactive Center

```javascript
class PosStore extends Reactive {
    setup() {
        // Reactive state (triggers re-render on change)
        this.selectedOrderUuid = null;
        this.numpadMode = "quantity";
        this.productListView = "grid";

        // Non-reactive state (no re-render)
        this.unwatched = markRaw({
            printers: {},
            cachedData: {},
        });

        // Reactive Sets (tracked by OWL)
        this.syncingOrders = new Set();
        this.hiddenProductIds = new Set();
    }
}
```

### Computed Getters

```javascript
// Computed values — no storage, derived from other state
get selectedOrder() {
    return this.orders.find(o => o.uuid === this.selectedOrderUuid);
}

get session() {
    return this.data.models['pos.session'][0];
}

get config() {
    return this.data.models['pos.config'][0];
}

get currency() {
    return this.company.currency_id;
}

get productViewMode() {
    return `product-list ${this.productListView}`;
}
```

### UI State on Models

```javascript
// Each model instance has uiState for transient UI data
class PosOrder extends Base {
    setup(vals) {
        this.uiState = {
            lineToRefund: null,    // Line being refunded
            displayed: true,       // Visibility in ticket screen
            booked: false,         // Booking status
            screen_data: {},       // Screen-specific data
        };
    }
}
```

### Screen State Persistence

```javascript
class TicketScreen extends Component {
    static storeOnOrder = false;  // Don't persist on order

    setup() {
        // Reuse saved UI state when returning to screen
        const order = this.pos.selectedOrder;
        if (order?.uiState?.screen_data?.TicketScreen) {
            this.state = useState(order.uiState.screen_data.TicketScreen);
        }
    }

    saveState() {
        if (this.pos.selectedOrder) {
            this.pos.selectedOrder.uiState.screen_data.TicketScreen = {
                filter: this.state.filter,
                search: this.state.search,
                page: this.state.page,
            };
        }
    }
}
```

---

## 5. Screen Navigation

### Screen Registry

```javascript
registry.category("pos_screens").add("ProductScreen", ProductScreen);
registry.category("pos_screens").add("PaymentScreen", PaymentScreen);
registry.category("pos_screens").add("ReceiptScreen", ReceiptScreen);
registry.category("pos_screens").add("TicketScreen", TicketScreen);
// Custom screen
registry.category("pos_screens").add("MyScreen", MyCustomScreen);
```

### Navigation Flow

```javascript
// Navigate to screen
this.pos.showScreen("PaymentScreen", { orderUuid: this.pos.selectedOrderUuid });

// In PosStore:
showScreen(name, props) {
    const screen = registry.category("pos_screens").get(name);
    this.mainScreen = {
        name,
        component: screen,
        props: { key: name, ...props },
    };
}
```

### First Screen Determination

```javascript
get firstScreen() {
    // If pos_hr is installed and cashier login required
    if (this.config.cashier_login && !this.cashierLoggedIn) {
        return "LoginScreen";
    }
    return "ProductScreen";
}
```

### Screen Component Pattern

```javascript
class MyScreen extends Component {
    static template = "my_module.MyScreen";
    static storeOnOrder = false;
    static props = {
        orderUuid: { type: String, optional: true },
        // ... other props
    };

    setup() {
        this.pos = usePos();
        this.orm = useService("orm");
        // Load data on mount
        onWillStart(async () => {
            this.data = await this.orm.call("my.model", "get_data", []);
        });
    }

    onConfirm() {
        // Navigate to next screen
        this.pos.showScreen("NextScreen", { data: this.result });
    }
}
```

---

## 6. Component Architecture

### Generic Components

Located in `generic_components/`:
- **Numpad** — Reusable number pad with configurable buttons
- **ProductCard** — Product display card
- **OrderWidget** — Order summary sidebar
- **Orderline** — Individual line item display
- **CategorySelector** — Product category tabs
- **CenteredIcon** — Icon with centered text
- **ListContainer** — Virtual scrolling list
- **AccordionItem** — Collapsible section
- **Inputs** — Reusable input components

### Component Composition

```
ProductScreen
├── CategorySelector
├── ProductCard (repeated)
├── OrderWidget
│   └── Orderline (repeated)
├── ActionPad
│   ├── ControlButtons
│   ├── Numpad
│   └── PartnerButton
└── ProductInfoPopup (dialog)
```

### Slots Pattern

```xml
<!-- Parent template -->
<t t-name="my_module.CardWrapper">
    <div class="card">
        <t t-slot="header"/>
        <div class="card-body">
            <t t-slot="default"/>
        </div>
        <t t-slot="footer"/>
    </div>
</t>

<!-- Usage -->
<t t-call="my_module.CardWrapper">
    <t t-set-slot="header">
        <h3>Product Name</h3>
    </t>
    <span>Product details</span>
    <t t-set-slot="footer">
        <button>Add to Cart</button>
    </t>
</t>
```

---

## 7. Service Architecture

### Service Registration

```javascript
const MyService = {
    dependencies: ["orm", "notification", "bus_service"],

    start(env, { orm, notification, bus_service }) {
        const service = {
            async doSomething() {
                // Use dependencies
            },
        };
        return service;
    },
};

registry.category("services").add("my_service", MyService);
```

### Service Consumption

```javascript
class MyComponent extends Component {
    setup() {
        this.myService = useService("my_service");
        this.orm = useService("orm");
        this.bus = useService("bus_service");
    }
}
```

### Core POS Services

| Service | Key | Purpose |
|---------|-----|---------|
| PosDataService | `pos_data` | ORM wrapper, IndexedDB, data sync |
| NumberBufferService | `number_buffer` | Keyboard input buffer |
| ContextualUtilsService | `contextual_utils` | formatCurrency, formatProductQty |
| ReportService | `report` | PDF report download |
| AccountMoveService | `account_move` | Invoice download |

---

## 8. Hardware Integration Architecture

### IoT Box Topology

```
┌──────────────────────────────────────────────┐
│               Odoo Backend                    │
│  ┌──────────────────────────────────────────┐ │
│  │  iot.device (model)                      │ │
│  │  - identifier: unique device ID          │ │
│  │  - type: printer/scale/payment/display   │ │
│  │  - connection: local/network/usb         │ │
│  │  - iot_ip: IoT Box IP address            │ │
│  └──────────────────────────────────────────┘ │
└──────────────────┬───────────────────────────┘
                   │ HTTPS
                   ▼
┌──────────────────────────────────────────────┐
│              IoT Box (Raspberry Pi)           │
│  ┌─────────┐ ┌──────┐ ┌──────────┐          │
│  │ Printer  │ │ Scale │ │ Terminal │          │
│  │ (USB)    │ │ (USB) │ │ (USB)    │          │
│  └─────────┘ └──────┘ └──────────┘          │
└──────────────────────────────────────────────┘
```

### Device Proxy Pattern

```javascript
// Frontend accesses device through hardware proxy
const device = this.pos.data.models['iot.device'].find(d => d.id === deviceId);

// Send action to device
await device.action({
    action: "print_receipt",
    receipt: imageData,
});

// Device types and actions:
// printer   → print_receipt, print_bill
// scale     → weigh (returns weight)
// payment_terminal → send_pay, cancel, force_done
// customer_display → show_message, show_price, hide
// fiscal_data_module → push_data
// barcode_scanner → scan
```

### Payment Terminal Async Pattern

```
POS Component          PaymentInterface          Terminal Proxy        Physical Terminal
─────                  ────────────────          ──────────────        ───────────────────
sendPaymentRequest()
    │
    ├──► sendPay(amount)
    │       │
    │       ├──► Hardware call to IoT Box
    │       │       │
    │       │       └──► Terminal processes card
    │       │               │
    │       │               └──► Result pushed back
    │       │                   │
    │       └──► Update payment status
    │           │
    │           └──► POS UI updates
```

---

## 9. Multi-Device Architecture

### Session-per-Config Pattern

```
pos.config (Terminal A) ──→ pos.session (Open) ──→ pos.orders (Terminal A)
pos.config (Terminal B) ──→ pos.session (Open) ──→ pos.orders (Terminal B)
```

Each POS terminal (config) has its own session. Sessions are independent.

### Device Synchronization

```
Device A creates order ──→ sync_from_ui ──→ Server creates pos.order
                                              │
                                              ├── BUS notification
                                              │   │
                                              │   ├── Device B receives
                                              │   │   └── Fetches new order data
                                              │   │
                                              │   └── Device C receives
                                              │       └── Fetches new order data
```

### Conflict Detection

When multiple devices modify the same order:
1. Last write wins (standard Odoo write_version)
2. BUS notification triggers re-fetch
3. UI updates to reflect latest state
4. Mutex prevents concurrent sync of same order
