# Odoo 18 POS — Best Practices & Code Quality

> Architecture decisions, code quality guidelines, and common pitfalls for POS development.

---

## 1. Module Design Principles

### DO: Create Modular Extensions

```
# GOOD: One feature = one module
my_pos_loyalty_extended/     # Just loyalty extensions
my_pos_payment_custom/       # Just payment extensions
my_pos_receipt_printer/      # Just printing extensions

# BAD: Everything in one module
my_pos_everything/           # 50+ models, 100+ files
```

### Dependency Guidelines

```python
# GOOD: Minimal dependencies
'depends': ['point_of_sale', 'pos_hr'],  # Only what you need

# BAD: Unnecessary dependencies
'depends': ['point_of_sale', 'sale_management', 'stock', 'account', 'hr', '...'],
```

### Module Naming

```
# GOOD: Clear, descriptive names
pos_my_custom_payment/
pos_receipt_custom_branding/
pos_product_barcode_quick/

# BAD: Vague names
pos_custom/
my_pos/
pos_v2/
```

---

## 2. Backend Best Practices

### Field Definition

```python
# GOOD: Complete field definitions with help text
my_custom_field = fields.Float(
    string='Custom Surcharge',
    help='Additional surcharge applied to order lines',
    digits='Product Price',  # Use standard digit precision
    default=0.0,
)

# BAD: Incomplete definitions
my_field = fields.Float()
```

### Method Design

```python
# GOOD: Single responsibility, clear names
def _compute_order_totals(self):
    """Compute total amount from order lines."""
    for order in self:
        order.amount_total = sum(line.price_subtotal for line in order.lines)

def _apply_discount(self, percent):
    """Apply percentage discount to all order lines."""
    for line in self.lines:
        line.discount = percent

# BAD: God methods
def process_order(self):
    """Does everything: validates, computes, prints, emails..."""
    # 200 lines of mixed logic
```

### Error Handling

```python
# GOOD: Specific exceptions with clear messages
from odoo.exceptions import UserError, ValidationError

def _validate_payment(self):
    if self.amount_paid < self.amount_total:
        raise UserError(_('Payment amount (%s) is less than order total (%s)') % (
            self.amount_paid, self.amount_total
        ))

# BAD: Generic errors
def _validate_payment(self):
    assert self.amount_paid >= self.amount_total, "Error"
```

### Performance

```python
# GOOD: Batch operations
def _update_all_lines(self, vals):
    self.lines.write(vals)  # Single SQL UPDATE

# BAD: N+1 queries
def _update_all_lines(self, vals):
    for line in self.lines:
        line.write(vals)  # N UPDATE queries

# GOOD: Use search_read
def _get_order_data(self):
    return self.env['pos.order'].search_read(
        [('session_id', '=', self.id)],
        ['name', 'amount_total']
    )

# BAD: Load full records
def _get_order_data(self):
    orders = self.env['pos.order'].search([('session_id', '=', self.id)])
    return [{'name': o.name, 'amount': o.amount_total} for o in orders]
```

---

## 3. Frontend Best Practices

### Component Design

```javascript
// GOOD: Single responsibility, small components
class ProductPriceBadge extends Component {
    static template = "my_module.ProductPriceBadge";
    static props = {
        product: Object,
        className: { type: String, optional: true },
    };

    get formattedPrice() {
        return this.env.utils.formatCurrency(this.props.product.lst_price);
    }
}

// BAD: Giant components doing everything
class ProductScreen extends Component {
    // 2000 lines: rendering, API calls, validation, printing...
}
```

### State Management

```javascript
// GOOD: Use reactive store for UI state
setup() {
    this.pos = usePos();  // Reactive
    this.localState = useState({ expanded: false });
}

// BAD: Non-reactive state that doesn't trigger re-render
setup() {
    this.myState = { count: 0 };  // Changes won't trigger re-render!
}

// GOOD: Use markRaw for large non-UI data
setup() {
    this.unwatched = markRaw({
        cachedProducts: hugeProductList,  // Don't track this
    });
}
```

### Service Usage

```javascript
// GOOD: Destructure services
setup() {
    const { orm, notification, dialog } = this.env.services;
    this.orm = orm;
    this.notification = notification;
}

// BAD: Accessing services through env every time
async handleClick() {
    await this.env.services.orm.call(...);
    this.env.services.notification.add(...);
    this.env.services.dialog.add(...);
}
```

### Error Handling

```javascript
// GOOD: Try/catch with user feedback
async handleSync() {
    try {
        await this.pos.syncAllOrders();
        this.notification.add('Orders synced', { type: 'success' });
    } catch (error) {
        console.error('Sync failed:', error);
        this.notification.add(`Sync failed: ${error.message}`, { type: 'danger' });
    }
}

// BAD: Silent failures
async handleSync() {
    await this.pos.syncAllOrders();  // If this fails, user sees nothing
}
```

---

## 4. Template Best Practices

### XML Organization

```xml
<!-- GOOD: Clean, indented, one expression per line -->
<t t-name="my_module.MyComponent" owl="1">
    <div class="my-component">
        <h3 t-esc="props.title"/>
        <t t-if="props.items.length">
            <ul>
                <t t-foreach="props.items" t-as="item" t-key="item.id">
                    <li t-esc="item.name"/>
                </t>
            </ul>
        </t>
        <t t-else="">
            <p>No items available</p>
        </t>
    </div>
</t>

<!-- BAD: Messy, hard to read -->
<t t-name="my_module.MyComponent" owl="1"><div class="my-component"><h3 t-esc="props.title"/><t t-if="props.items.length"><ul><t t-foreach="props.items" t-as="item" t-key="item.id"><li t-esc="item.name"/></t></ul></t></div></t>
```

### Conditional Rendering

```xml
<!-- GOOD: Clear conditions with t-if/t-elif/t-else -->
<t t-if="order.is_paid">
    <span class="badge bg-success">Paid</span>
</t>
<t t-elif="order.is_draft">
    <span class="badge bg-warning">Draft</span>
</t>
<t t-else="">
    <span class="badge bg-secondary">Unknown</span>
</t>

<!-- BAD: Nested t-if with negation -->
<t t-if="!order.is_paid">
    <t t-if="order.is_draft">
        Draft
    </t>
    <t t-else="">
        Unknown
    </t>
</t>
```

### Event Handlers

```xml
<!-- GOOD: Named methods -->
<button t-on-click="handleConfirm">Confirm</button>

<!-- GOOD: Inline arrow for simple props passing -->
<button t-on-click="() => this.pos.showScreen('Next')">Next</button>

<!-- BAD: Complex inline logic -->
<button t-on-click="() => { state.loading = true; doSomething().then(() => { state.loading = false; }) }">
    Do It
</button>
```

---

## 5. Extension Best Practices

### Patching

```javascript
// GOOD: Minimal patches, call super
patch(PosOrder.prototype, {
    get customTotal() {
        return super.customTotal + this.surcharge;
    },
});

// BAD: Replacing entire methods without calling super
patch(PosOrder.prototype, {
    addLine(vals) {
        // Completely ignores the original logic!
        return { id: 123 };
    },
});
```

### Template Extension

```xml
<!-- GOOD: Extension mode preserves original -->
<t t-inherit="point_of_sale.ProductCard" t-inherit-mode="extension">

<!-- BAD: Replacement mode destroys original -->
<t t-inherit="point_of_sale.ProductCard" t-inherit-mode="replacement">
```

### Asset Registration

```python
# GOOD: Specific file patterns
'assets': {
    'point_of_sale._assets_pos': [
        'my_module/static/src/overrides/pos_store.js',
        'my_module/static/src/xml/product_card.xml',
    ],
}

# BAD: Too broad (loads unnecessary files)
'assets': {
    'point_of_sale._assets_pos': [
        'my_module/static/**/*',  # Includes tests, docs, etc.
    ],
}
```

---

## 6. Offline Architecture Best Practices

### Offline-First Design

```javascript
// GOOD: Assume offline, sync when possible
async addOrder(orderData) {
    // 1. Add to local state immediately
    const order = this.models['pos.order'].create(orderData);
    this.pendingOrder.create.add(order.id);

    // 2. Try to sync if online
    if (this.isOnline) {
        try {
            await this.syncOrder(order);
        } catch {
            // Order stays in pending queue
        }
    }
}
```

### Conflict Resolution

```javascript
// GOOD: Last-write-wins with user notification
async handleConflict(order) {
    const serverData = await this.fetchFromServer(order.id);
    if (serverData.write_date > order.write_date) {
        this.notification.add(
            `Order ${order.name} was modified on another terminal. Updating...`,
            { type: 'warning' }
        );
        order.update(serverData);
    }
}
```

---

## 7. Testing Best Practices

### Test Structure

```python
# GOOD: Given-When-Then pattern
def test_discount_applied_correctly(self):
    # Given
    order = self._create_pos_order()
    line = self._create_order_line(order, quantity=10, price=100)

    # When
    order._apply_discount(10)

    # Then
    self.assertEqual(line.discount, 10)
    self.assertEqual(line.price_subtotal, 900)
```

### Test Data Setup

```python
# GOOD: Reusable setup methods
@classmethod
def setUpClass(cls):
    super().setUpClass()
    cls.pos_config = cls.env.ref('point_of_sale.main_pos_config')
    cls.pos_config.open()
    cls.session = cls.pos_config.current_session_id

def _create_pos_order(self, **kwargs):
    """Helper to create POS orders in tests."""
    vals = {'session_id': self.session.id}
    vals.update(kwargs)
    return self.env['pos.order'].create(vals)
```

---

## 8. Common Pitfalls & Solutions

| Pitfall | Cause | Solution |
|---------|-------|----------|
| Custom field not in POS frontend | Not included in `_load_pos_data_fields()` | Add field to the mixin method |
| Template changes not showing | Asset not registered in manifest | Verify `_assets_pos` bundle |
| Order sync fails silently | Error caught but not logged | Add try/catch with console.error |
| Reactivity not triggering | Using plain object instead of useState | Wrap with useState() |
| Override not applied | Module not loaded, wrong import path | Check manifest dependencies |
| N+1 queries in load_data | Loading records one by one | Use search_read with field list |
| Offline orders lost | Not persisted to IndexedDB | Use pendingOrder queues |
| BUS notifications not received | Wrong channel name | Verify channel matches backend |
| Payment terminal timeout | No timeout configured | Add timeout to terminal proxy call |
| Product search slow | No DB indexes | Add indexes on barcode, default_code, name |

---

## 9. Performance Checklist

### Backend

- [ ] Use `search_read()` instead of `search()` + `read()`
- [ ] Use `read_group()` for aggregated data
- [ ] Batch operations: `write(vals)` instead of loop of `write()`
- [ ] Proper DB indexes on frequently queried fields
- [ ] Avoid computing fields in loops
- [ ] Use `@api.depends` correctly for computed fields

### Frontend

- [ ] Lazy-load products (paginated, not all at once)
- [ ] Cache pricelist rules (`computeProductPricelistCache()`)
- [ ] Use `markRaw()` for large non-reactive objects
- [ ] Debounce search inputs
- [ ] Virtual scrolling for long lists
- [ ] Avoid unnecessary `useState()` calls

### Asset Loading

- [ ] Only include needed files in asset bundles
- [ ] Use specific file patterns, not wildcards
- [ ] Minimize CSS in POS bundle
- [ ] Remove debug files from production bundle

---

## 10. Security Best Practices

### Access Control

```python
# GOOD: Proper security rules
class PosOrder(models.Model):
    _inherit = 'pos.order'

    @api.model
    def _load_pos_data_domain(self):
        # Only load orders for current user's session
        return [('session_id.pos_config_id.users_id', 'in', [self.env.user.id])]
```

### Multi-Company

```python
# GOOD: Company-aware queries
def _get_orders(self):
    return self.env['pos.order'].search([
        ('company_id', 'in', self.env.companies.ids),
        ('session_id', '=', self.session_id.id),
    ])
```

### Data Sanitization

```javascript
// GOOD: Sanitize user input
handleSearch(query) {
    // OWL auto-escapes in templates, but sanitize for RPC calls
    const sanitized = query.replace(/[<>]/g, '');
    this.orm.searchRead('product.product', [['name', 'ilike', sanitized]]);
}
```

---

## 11. Upgrade-Safe Development

### Patterns to Avoid

```javascript
// BAD: Direct property access (breaks on core update)
const order = this.pos.orders[0];

// GOOD: Use getters/methods
const order = this.pos.selectedOrder;

// BAD: Template structure assumption
<xpath expr="/div/div[3]/span" position="after">

// GOOD: Class-based targeting
<xpath expr="//div[hasclass('product-name')]" position="after">
```

### Documentation

```python
# GOOD: Document extension points
class PosOrder(models.Model):
    _inherit = 'pos.order'

    def _process_order(self, order_data, existing_order=None):
        """Process order from frontend data.

        Extension point: Override this method to add custom order processing.
        Always call super() to maintain core functionality.

        Args:
            order_data: Dict from frontend
            existing_order: Existing order record or None

        Returns:
            int: Order ID
        """
        # ... implementation
```
