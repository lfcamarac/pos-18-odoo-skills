# Odoo 18 POS — Backend (Python) Patterns

> Complete reference for developing Python backend extensions for the POS module.

---

## 1. Model Architecture

### pos.config

One record per POS terminal/register. Controls all POS behavior.

```python
class PosConfig(models.Model):
    _name = 'pos.config'
    _description = 'Point of Sale Configuration'

    # Basic fields
    name = fields.Char('POS Name', required=True)
    company_id = fields.Many2one('res.company', required=True)

    # Financial
    payment_method_ids = fields.Many2many('pos.payment.method')
    pricelist_id = fields.Many2one('product.pricelist')
    journal_id = fields.Many2one('account.journal')
    fiscal_position_ids = fields.Many2many('account.fiscal.position')

    # Stock
    picking_type_id = fields.Many2one('stock.picking.type')

    # Hardware
    printer_ids = fields.Many2many('pos.printer')
    iface_print_via_proxy = fields.Boolean(compute='_compute_iface')
    iface_electronic_scale = fields.Boolean(compute='_compute_iface')

    # Feature flags
    is_restaurant = fields.Boolean('Is a Restaurant')
    module_pos_hr = fields.Boolean('Employee Login')
    module_pos_loyalty = fields.Boolean('Loyalty Program')

    # Cash control
    cash_control = fields.Boolean('Advanced Cash Control')
    cash_rounding = fields.Boolean('Cash Rounding')
    rounding_method = fields.Many2one('cash.rounding.method')

    # Session tracking
    current_session_id = fields.Many2one('pos.session')
    session_state = fields.Selection(related='current_session_id.state')
```

### pos.session

Runtime session managing a POS terminal's business day.

```python
class PosSession(models.Model):
    _name = 'pos.session'
    _description = 'Point of Sale Session'
    _order = 'id desc'

    STATES = [('opening_control', 'Opening Control'),
              ('opened', 'In Progress'),
              ('closing_control', 'Closing Control'),
              ('closed', 'Closed & Posted')]

    name = fields.Char(required=True, readonly=True)
    pos_config_id = fields.Many2one('pos.config', required=True)
    state = fields.Selection(STATES, default='opening_control')

    # Financial
    journal_id = fields.Many2one('account.journal')
    cash_register_balance_end_real = fields.Monetary()

    # Tracking
    sequence_number = fields.Integer('Order Sequence Number')
    login_number = fields.Integer('Login Sequence Number')

    # Orders
    order_ids = fields.One2many('pos.order', 'session_id')

    @api.model
    def load_data(self, session_id):
        """THE BIG RPC — bootstraps all frontend data.
        
        Called by PosData.loadInitialData() on POS startup.
        Returns all models with fields, relations, and records.
        """
        self.check_session_access()
        session = self.browse(session_id)
        
        models = self._load_pos_data()
        return {
            'data': models,
            'name': session.pos_config_id.name,
            'company_id': session.company_id.id,
            'server_date': fields.Date.context_today(self),
            'server_time': fields.Datetime.now(self).strftime('%H:%M:%S'),
            'currencies': self._load_currencies(),
            'users': self._load_users(),
        }

    def _load_pos_data(self):
        """Loads all models for the frontend.
        
        Each model must implement:
        - _load_pos_data_domain()
        - _load_pos_data_fields()
        - _load_pos_data_relations() (optional)
        """
        # Models are loaded in dependency order
        loaded_models = []
        for model in self._load_pos_data_models():
            data = self._load_model_data(model)
            loaded_models.append(data)
        return loaded_models
```

### pos.order

Sales transaction model. Receives data from frontend via `sync_from_ui()`.

```python
class PosOrder(models.Model):
    _name = 'pos.order'
    _description = 'Point of Sale Orders'
    _inherit = ['pos.order', 'pos.bus.mixin', 'pos.load.mixin']

    # State machine
    STATES = [('draft', 'New'),
              ('cancel', 'Cancelled'),
              ('paid', 'Paid'),
              ('done', 'Posted'),
              ('invoiced', 'Invoiced')]

    state = fields.Selection(STATES, default='draft')
    session_id = fields.Many2one('pos.session')
    partner_id = fields.Many2one('res.partner')
    user_id = fields.Many2one('res.users')

    # Financial
    amount_total = fields.Monetary(compute='_compute_amounts')
    amount_paid = fields.Float(compute='_compute_amounts')
    amount_return = fields.Float('Amount Returned')
    amount_tax = fields.Float(compute='_compute_amounts')

    # Lines and payments
    lines = fields.One2many('pos.order.line', 'order_id')
    payment_ids = fields.One2many('pos.payment', 'pos_order_id')

    # Accounting
    account_move = fields.Many2one('account.move')
    picking_id = fields.Many2one('stock.picking')

    @api.model
    def sync_from_ui(self, session_id, order_data):
        """Receive order data from frontend and process it.
        
        This is THE main RPC endpoint for POS order synchronization.
        Called by PosStore.syncAllOrders() and pushSingleOrder().
        
        Args:
            session_id: Current POS session ID
            order_data: Dict with order data from frontend:
                - id: order ID (None for new orders)
                - lines: list of line dicts
                - payments: list of payment dicts
                - partner_id: customer ID
                - pricelist_id: pricelist ID
                - etc.
        
        Returns:
            Dict with created/updated order IDs
        """
        session = self.env['pos.session'].browse(session_id)
        order_id = self._process_order(order_data, existing_order=self.browse(order_data.get('id')))
        return {'id': order_id}

    def _process_order(self, order_data, existing_order=None):
        """Create or update an order from frontend data.
        
        Handles:
        1. Order creation/update
        2. Line processing (prices, taxes)
        3. Payment processing
        4. Stock move creation
        5. Accounting entries
        """
        # Determine if creating or updating
        if existing_order and existing_order.state == 'draft':
            order = existing_order
            order._handle_existing_order_data(order_data)
        else:
            order = self._create_pos_order(order_data)

        # Process lines
        order._process_order_lines(order_data.get('lines', []))

        # Process payments
        order._process_payments(order_data.get('payment_ids', []))

        # Handle order state transition
        if order._is_paid():
            order.state = 'paid'
            order._process_paid_order()

        return order.id
```

### pos.order.line

```python
class PosOrderLine(models.Model):
    _name = 'pos.order.line'
    _description = 'Point of Sale Order Lines'
    _inherit = ['pos.order.line', 'pos.bus.mixin', 'pos.load.mixin']

    order_id = fields.Many2one('pos.order')
    product_id = fields.Many2one('product.product')
    quantity = fields.Float(default=1)
    price_unit = fields.Float('Unit Price')
    discount = fields.Float('Discount (%)')
    tax_ids = fields.Many2many('account.tax')
    note = fields.Text('Internal Note')

    # Computed prices
    price_subtotal = fields.Float(compute='_compute_all')
    price_subtotal_incl = fields.Float(compute='_compute_all')
    price_total = fields.Float(compute='_compute_all')
```

### pos.payment

```python
class PosPayment(models.Model):
    _name = 'pos.payment'
    _description = 'Point of Sale Payments'
    _inherit = ['pos.payment', 'pos.bus.mixin', 'pos.load.mixin']

    pos_order_id = fields.Many2one('pos.order')
    payment_method_id = fields.Many2one('pos.payment.method')
    amount = fields.Float()
    payment_date = fields.Datetime()
    payment_status = fields.Selection([
        ('pending', 'Pending'),
        ('waiting', 'Waiting'),
        ('done', 'Done'),
        ('retry', 'Retry'),
        ('reversed', 'Reversed'),
    ])
```

### pos.payment.method

```python
class PosPaymentMethod(models.Model):
    _name = 'pos.payment.method'
    _description = 'Point of Sale Payment Methods'

    name = fields.Char(required=True)
    journal_id = fields.Many2one('account.journal')
    is_cash_count = fields.Boolean()
    split_transactions = fields.Boolean()

    # Terminal payment
    use_payment_terminal = fields.Selection([
        ('adyen', 'Adyen'),
        ('stripe', 'Stripe'),
        ('six', 'Six'),
        ('worldline', 'Worldline'),
        ('ingenico', 'Ingenico'),
        ('paytm', 'Paytm'),
        ('razorpay', 'Razorpay'),
    ])
```

---

## 2. Mixin System

### pos.bus.mixin

Enables WebSocket notifications to other POS devices.

```python
class PosBusMixin(models.AbstractModel):
    _name = 'pos.bus.mixin'
    _description = 'POS Bus Mixin'

    def _notify_pos_sessions(self):
        """Send BUS notification to all POS sessions for this record.
        
        Triggers automatic data refresh on all connected POS terminals.
        """
        sessions = self._get_pos_sessions_to_notify()
        for session in sessions:
            self.env['bus.bus']._sendone(
                channel=(self._cr.dbname, 'pos.session', session.id),
                message_type="notification",
                payload={
                    'model': self._name,
                    'id': self.id,
                    'command': 'UPDATE',
                }
            )

    @api.model_create_multi
    def create(self, vals_list):
        records = super().create(vals_list)
        records._notify_pos_sessions()
        return records

    def write(self, vals):
        result = super().write(vals)
        self._notify_pos_sessions()
        return result

    def unlink(self):
        self._notify_pos_sessions()
        return super().unlink()
```

### pos.load.mixin

Controls what data is sent to the frontend during session bootstrap.

```python
class PosLoadMixin(models.AbstractModel):
    _name = 'pos.load.mixin'
    _description = 'POS Load Mixin'

    def _load_pos_data_domain(self):
        """Return search domain for records to load.
        
        Override to filter which records are sent to frontend.
        """
        return []

    def _load_pos_data_fields(self):
        """Return list of field names to send to frontend.
        
        Override to include/exclude fields.
        """
        return []

    def _load_pos_data_relations(self):
        """Return list of relation fields to include related models.
        
        Override to trigger loading of related models.
        """
        return []

    def _load_pos_data(self, session):
        """Main loading method. Returns dict with fields, records, relations."""
        domain = self._load_pos_data_domain()
        fields = self._load_pos_data_fields()
        records = self.search_read(domain, fields)

        return {
            'model': self._name,
            'fields': fields,
            'records': records,
            'relations': self._load_pos_data_relations(),
        }
```

---

## 3. Session Lifecycle

### State Machine

```
opening_control ──→ opened ──→ closing_control ──→ closed
     ↑                                                │
     └──────────── (re-open, if not posted) ──────────┘
```

### Opening a Session

```python
def _validate_session_start(self):
    """Validate before opening session."""
    self.ensure_one()
    
    # Check session isn't already open
    if self.state != 'opening_control':
        raise UserError(_('Session already opened'))
    
    # Check opening control
    if self.pos_config_id.cash_control:
        # Verify opening cash balance
        self._check_opening_balance()
    
    # Set state
    self.state = 'opened'
    self.sequence_number = 1
    self.login_number += 1

def action_pos_session_open(self):
    """Open the session — called from UI."""
    self._validate_session_start()
    self.state = 'opened'
```

### During Session

```python
def _process_order(self, order_data):
    """Process an order during the open session."""
    self.ensure_one()
    assert self.state == 'opened', "Session must be open"
    
    # Create/update order
    order = self.env['pos.order'].sudo()._process_order(order_data)
    
    # Increment sequence
    self.sequence_number += 1
    
    return order
```

### Closing a Session

```python
def _validate_session_close(self):
    """Validate before closing."""
    self.ensure_one()
    
    # Check cash count if enabled
    if self.pos_config_id.cash_control:
        self._check_closing_balance()

def action_pos_session_closing_control(self):
    """Move to closing control."""
    self.state = 'closing_control'

def action_pos_session_close(self):
    """Close and post the session.
    
    Creates accounting entries, validates cash, posts session.
    """
    self._validate_session_close()
    self.state = 'closed'
    
    # Create account moves for all orders
    self._create_account_moves()
    
    # Post session
    self._post_session()
```

---

## 4. Tax Handling

### Tax Computation

```python
class PosOrderLine(models.Model):
    _inherit = 'pos.order.line'

    @api.depends('quantity', 'price_unit', 'discount', 'tax_ids')
    def _compute_all(self):
        for line in self:
            # Price without tax
            line.price_subtotal = line._get_price_with_discount(
                tax_included=False
            )
            
            # Price with tax
            line.price_subtotal_incl = line._get_price_with_discount(
                tax_included=True
            )
            
            # Tax amount
            line.price_total = line.price_subtotal_incl - line.price_subtotal

    def _get_price_with_discount(self, tax_included=False):
        """Compute line price with discount and optionally with tax."""
        price = self.price_unit * self.quantity
        discount_amount = price * (self.discount / 100)
        price_after_discount = price - discount_amount
        
        if tax_included:
            taxes = self.tax_ids.compute_all(
                price_after_discount,
                currency=self.order_id.pricelist_id.currency_id,
                quantity=1,
                product=self.product_id,
                partner=self.order_id.partner_id,
            )
            return taxes['total_included']
        
        return price_after_discount
```

### Fiscal Position

```python
class PosOrder(models.Model):
    _inherit = 'pos.order'

    fiscal_position_id = fields.Many2one('account.fiscal.position')

    def _apply_fiscal_position(self):
        """Apply fiscal position to order lines.
        
        Fiscal positions can:
        - Map taxes (tax A → tax B)
        - Map products (product A → product B)
        - Map accounts
        """
        if not self.fiscal_position_id:
            return
        
        for line in self.lines:
            # Map taxes
            new_taxes = self.fiscal_position_id.map_tax(line.tax_ids)
            line.tax_ids = new_taxes
```

---

## 5. Extension Patterns

### Extending pos.config

```python
class PosConfig(models.Model):
    _inherit = 'pos.config'

    my_custom_setting = fields.Boolean('My Custom Feature')
    my_related_field = fields.Many2one('my.model', 'Related Config')

    @api.constrains('my_custom_setting')
    def _check_my_custom_setting(self):
        for config in self:
            if config.my_custom_setting and not config.some_required_field:
                raise ValidationError(_('Some required field is missing'))
```

### Extending pos.order

```python
class PosOrder(models.Model):
    _inherit = 'pos.order'

    my_custom_field = fields.Char('Custom Field')
    my_related_line = fields.One2many('my.custom.line', 'order_id')

    @api.model
    def sync_from_ui(self, session_id, order_data):
        # Call original method
        result = super().sync_from_ui(session_id, order_data)
        
        # Add custom processing
        order_id = result.get('id')
        order = self.browse(order_id)
        order._process_custom_data(order_data)
        
        return result

    def _process_custom_data(self, order_data):
        """Process custom data from frontend."""
        if 'my_custom_data' in order_data:
            self.my_custom_field = order_data['my_custom_data']
```

### Adding New Model Related to POS

```python
class MyPosFeature(models.Model):
    _name = 'my.pos.feature'
    _description = 'My POS Feature'
    _inherit = ['pos.bus.mixin', 'pos.load.mixin']

    name = fields.Char(required=True)
    pos_config_id = fields.Many2one('pos.config')
    value = fields.Float()

    def _load_pos_data_domain(self):
        return [('pos_config_id', 'in', self.env['pos.session'].browse(
            self.env.context.get('active_id')
        ).pos_config_id.ids)]

    def _load_pos_data_fields(self):
        return ['id', 'name', 'value', 'pos_config_id']
```

---

## 6. Payment Method Extension

### Adding a New Payment Terminal

```python
class PosPaymentMethod(models.Model):
    _inherit = 'pos.payment.method'

    use_payment_terminal = fields.Selection(
        selection_add=[('my_terminal', 'My Terminal')],
        ondelete={'my_terminal': 'set default'}
    )
    my_terminal_config = fields.Char('Terminal Configuration')

    @api.onchange('use_payment_terminal')
    def _onchange_use_payment_terminal(self):
        if self.use_payment_terminal == 'my_terminal':
            # Set default journal
            self.journal_id = self.env['account.journal'].search([
                ('type', '=', 'cash'),
                ('company_id', '=', self.company_id.id),
            ], limit=1)
```

---

## 7. Data Loading Pipeline

### How load_data() Works

```python
def _load_pos_data(self):
    """The master data loading method.
    
    Loads models in dependency order:
    1. res.company (company data)
    2. res.currency
    3. res.partner (customers)
    4. product.pricelist
    5. product.product
    6. pos.category
    7. pos.payment.method
    8. account.tax
    9. account.fiscal.position
    10. pos.config
    11. pos.order (pending orders)
    ... etc.
    
    Each model must implement pos.load.mixin methods.
    """
    result = []
    
    for model_name in self._load_pos_data_models():
        model = self.env[model_name]
        
        domain = model._load_pos_data_domain()
        fields_list = model._load_pos_data_fields()
        records = model.search_read(domain, fields_list)
        
        # Load relations
        relations = model._load_pos_data_relations()
        
        result.append({
            'name': model_name,
            'fields': fields_list,
            'records': records,
            'relations': relations,
        })
    
    return result
```

### Performance Considerations

```python
# GOOD: Use search_read with field list
def _load_pos_data(self):
    return self.search_read(
        self._load_pos_data_domain(),
        self._load_pos_data_fields()
    )

# BAD: Loading full records
def _load_pos_data(self):
    return [record.read()[0] for record in self.search([])]

# GOOD: Use read_group for aggregated data
def _load_pos_data(self):
    return self.read_group(
        domain=self._load_pos_data_domain(),
        fields=['product_id', 'quantity'],
        groupby=['product_id']
    )
```

---

## 8. Common Backend Patterns

### Computed Fields

```python
class PosConfig(models.Model):
    _inherit = 'pos.config'

    order_count = fields.Integer(compute='_compute_order_count')

    @api.depends('current_session_id.order_ids')
    def _compute_order_count(self):
        for config in self:
            config.order_count = len(
                config.current_session_id.order_ids
            )
```

### Related Fields

```python
class PosOrder(models.Model):
    _inherit = 'pos.order'

    company_id = fields.Many2one(related='session_id.company_id')
    currency_id = fields.Many2one(related='pricelist_id.currency_id')
```

### Constraints

```python
class PosOrder(models.Model):
    _inherit = 'pos.order'

    @api.constrains('state')
    def _check_order_state(self):
        for order in self:
            if order.state == 'paid' and not order.payment_ids:
                raise ValidationError(_('Paid order must have payments'))
```

### Wizards

```python
class PosSessionCloseWizard(models.TransientModel):
    _name = 'pos.session.close.wizard'

    pos_session_id = fields.Many2one('pos.session')
    closing_cash = fields.Float('Closing Cash Count')

    def action_close_session(self):
        self.ensure_one()
        session = self.pos_session_id
        # Validate and close
        session._validate_closing_balance(self.closing_cash)
        session.action_pos_session_close()
        return {'type': 'ir.actions.act_window_close'}
```
