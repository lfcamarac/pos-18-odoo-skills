# Odoo 18 POS — Localization Guide

> Fiscal compliance patterns for POS across different countries and regions.

---

## 1. Localization Architecture Overview

### Types of Fiscal Compliance

| Type | Countries | Description |
|------|-----------|-------------|
| **Hardware Blackbox** | Belgium (pos_blackbox_be) | Physical TSS via IoT Box, immutable audit log |
| **Cloud TSS** | Germany (l10n_de_pos_cert) | Cloud-based technical security (Fiskaly) |
| **EDI Invoice** | Mexico, Brazil, Peru, Ecuador, Chile, Saudi Arabia | Government e-invoicing API |
| **Anti-fraud Software** | France, Spain | Certified software, signature chains |
| **Tax Reporting** | India, Poland, Malaysia | Tax return data extraction |
| **Fiscal Printer** | Italy, some LATAM | Direct printer integration |

### Common Localization Pattern

All localization modules follow this structure:

```python
class PosOrder(models.Model):
    _inherit = 'pos.order'

    # 1. Fiscal tracking fields
    localization_state = fields.Char("Fiscal State")
    localization_uuid = fields.Char("Government UUID")
    qr_code = fields.Char("Verification QR")

    # 2. Override sync_from_ui for fiscal processing
    @api.model
    def sync_from_ui(self, session_id, order_data):
        result = super().sync_from_ui(session_id, order_data)
        self._process_fiscal_compliance()
        return result

    # 3. Restrict modifications on fiscal orders
    def write(self, vals):
        if self._is_fiscal_order():
            protected_fields = ['lines', 'amount_total', 'partner_id']
            for field in protected_fields:
                if field in vals:
                    raise UserError(_('Cannot modify fiscal order'))
        return super().write(vals)
```

---

## 2. Latin America — EDI Localization

### Mexico (l10n_mx_edi_pos)

**Purpose:** CFDI electronic invoicing with SAT.

**Key Backend Models:**

```python
class PosOrder(models.Model):
    _inherit = 'pos.order'

    # CFDI State
    l10n_mx_edi_cfdi_state = fields.Selection([
        ('sent', 'Sent'),
        ('global_sent', 'Global Invoice Sent'),
        ('global_cancel', 'Global Cancelled'),
    ])
    l10n_mx_edi_cfdi_uuid = fields.Char("CFDI UUID")
    l10n_mx_edi_cfdi_sat_state = fields.Char("SAT State")
    l10n_mx_edi_usage = fields.Char("CFDI Usage (G01, G03, etc.)")
    l10n_mx_edi_cfdi_to_public = fields.Boolean("Invoice to Public")

    # Document linking
    l10n_mx_edi_document_ids = fields.Many2many('l10n_mx_edi.document')

    # Global invoice
    def _l10n_mx_edi_global_invoice_try_send(self):
        """Create global invoice from multiple orders."""
        orders = self.filtered(
            lambda o: o.l10n_mx_edi_cfdi_state == 'draft'
        )
        # Create single CFDI for all orders
        # Link all orders to the CFDI
```

**Frontend Pattern:**
- POS displays SAT status of orders
- QR code for consumer verification
- Global invoice wizard in backend

### Brazil (l10n_br_edi_pos)

**Purpose:** NFC-e (electronic consumer invoice) via Avatax.

**Key Backend:**

```python
class PosOrder(models.Model):
    _inherit = ['pos.order', 'account.external.tax.mixin']

    # NFC-e fields
    l10n_br_edi_number = fields.Char("NFC-e Number")
    l10n_br_access_key = fields.Char("44-digit Access Key")
    l10n_br_edi_series = fields.Char("Document Series")
    l10n_br_edi_avatax_data = fields.Json("Avatax Data")

    # QR Code for consumer
    def _l10n_br_generate_nfce_qr_code(self):
        """Generate SHA1-based QR code for consumer lookup."""
        # Uses state-specific SEFAZ URL
        url = self._l10n_br_get_qr_url()
        return f"{url}#chave={self.l10n_br_access_key}"

    # Combined tax + EDI
    def _l10n_br_do_edi(self):
        """Process tax calculation and EDI submission."""
        # 1. Get tax calculation from Avatax
        tax_data = self.get_order_tax_details()
        # 2. Generate access key (MOD 11 check digit)
        self._l10n_br_generate_access_key()
        # 3. Submit to SEFAZ
        # 4. Store response
```

**Key Pattern — Access Key Generation:**
```python
def _l10n_br_generate_access_key(self):
    """44-digit key: state code + CNPJ + serial + check digit."""
    # MOD 11 algorithm
    key = self._build_base_key()
    check_digit = self._mod11(key)
    self.l10n_br_access_key = f"{key}{check_digit}"
```

### Peru, Ecuador, Chile, Colombia

Similar patterns — each country has:
- `l10n_{country}_edi_pos` module
- EDI state fields on pos.order
- Government API integration
- QR code for consumer verification

---

## 3. Europe — Fiscal Compliance

### Belgium (pos_blackbox_be)

**Purpose:** Certified cash register per FPS Finance. Uses Fiscal Data Module (FDM) via IoT Box.

**Architecture:**

```
POS Frontend ──→ Blackbox Push Pipeline ──→ FDM Hardware (IoT Box)
                                                    │
                                                    └── Fiscal Memory
                                                        (immutable)
```

**Key Backend:**

```python
class PosConfig(models.Model):
    _inherit = 'pos.config'

    iface_fiscal_data_module = fields.Many2one(
        'iot.device',
        domain="[('type', '=', 'fiscal_data_module')]",
    )
    certified_blackbox_identifier = fields.Char(
        compute='_compute_certified_id'
    )

    def _check_blackbox_compatible(self):
        """15+ validation checks before allowing POS open."""
        self._check_self_order()       # Self-order not allowed
        self._check_loyalty()          # Loyalty restricted
        self._check_insz_user()        # INSZ number required
        self._check_cash_rounding()    # Must round at 0.05 HALF-UP
        self._check_printer_connected()# Printer must be connected
```

**Frontend — Blackbox Push:**

```javascript
// In PosStore patch (675+ lines)
patch(PosStore.prototype, {
    async pushOrderToBlackbox(order) {
        // 1. Format order for FDM
        const data = this.createOrderDataForBlackbox(order);
        // 2. Push to fiscal module
        const result = await this.env.services.hardware_proxy.call(
            'fiscal_data_module',
            'push_data',
            [data]
        );
        // 3. Store signature
        order.blackbox_signature = result.signature;
    },

    // Receipt type management
    getReceiptType(order) {
        if (order.is_refund) return 'NR'; // Normal Refund
        if (order.is_pro_forma) return 'PS'; // Pro Forma Sale
        return 'NS'; // Normal Sale
    },

    // User clocking (work in/out orders)
    async clock() {
        const status = this.getUserSessionStatus();
        if (status === 'clocked_in') {
            // Create WORK OUT order
            await this.createWorkInOutOrder();
        } else {
            // Create WORK IN order
            await this.createWorkInOrder();
        }
    },
});
```

**Compliance Rules:**
- Cash rounding enforced at 0.05 HALF-UP
- Write protection on registered orders
- Every modification creates pro-forma pairs
- Full audit trail via `pos_blackbox_be.log`
- IP address tracking for certified devices

### Germany (l10n_de_pos_cert)

**Purpose:** TSS (Technische Sicherheitseinrichtung) via Fiskaly cloud.

**Difference from Belgium:**
- Cloud-based TSS (not local hardware)
- Uses IAP (In-App Purchase) service
- DSFinV-K export format for tax audits

```python
class PosOrder(models.Model):
    _inherit = 'pos.order'

    l10n_de_fiskaly_transaction_uuid = fields.Char()
    l10n_de_fiskaly_transaction_number = fields.Char()
    l10n_de_fiskaly_time_start = fields.Datetime()
    l10n_de_fiskaly_time_end = fields.Datetime()
    l10n_de_fiskaly_certificate_serial = fields.Char()
    l10n_de_fiskaly_signature_value = fields.Char()

    def _l10n_de_fiskaly_push_transaction(self):
        """Push transaction to Fiskaly cloud TSS."""
        # Uses IAP service
        # Returns signature value
```

### France (l10n_fr_pos_cert)

**Purpose:** Anti-fraud cash register per CGI 286 I-3 bis.

- Certified software (no hardware required)
- Inalterable data
- Secure signature chain
- Periodic compliance validation

### Spain — TicketBAI (l10n_es_edi_tbai_pos)

**Purpose:** Electronic invoicing for Basque Country.

- QR code on receipts
- Government API submission
- TBAI ticket generation

---

## 4. Middle East — E-Invoicing

### Saudi Arabia (l10n_sa_edi_pos)

**Purpose:** ZATCA e-invoicing (Phase 2).

**Key Pattern:** Thin localization — delegates to parent EDI module.

```python
class PosOrder(models.Model):
    _inherit = 'pos.order'

    # Minimal fields — mostly delegated
    l10n_sa_invoice_qr_code_str = fields.Char(
        related='account_move.l10n_sa_invoice_qr_code'
    )
    l10n_sa_invoice_edi_state = fields.Char(
        related='account_move.edi_state'
    )
```

**Frontend:** Display QR code on receipt for consumer verification.

---

## 5. Asia-Pacific

### India (l10n_in_pos / l10n_in_reports_gstr_pos)

**Purpose:** GST-compliant POS invoicing and GSTR-1 reporting.

```python
class PosOrder(models.Model):
    _inherit = 'pos.order'

    # GST fields
    l10n_in_gst_rate = fields.Float("GST Rate")
    l10n_in_gst_amount = fields.Float("GST Amount")
    l10n_in_place_of_supply = fields.Char("Place of Supply")
```

**GSTR-1 Reporting:**
```python
def _l10n_in_gstr1_pos_data(self):
    """Extract POS data for GSTR-1 return."""
    return {
        'b2c_small': self._get_b2c_small_invoices(),
        'b2c_large': self._get_b2c_large_invoices(),
        'credit_notes': self._get_credit_notes(),
    }
```

---

## 6. Common Localization Extension Pattern

### Adding Custom Localization

```python
# __manifest__.py
{
    'name': 'My Country POS Localization',
    'depends': ['point_of_sale', 'l10n_my_country'],
    'data': [
        'data/fiscal_data.xml',
        'views/pos_order_views.xml',
    ],
}

# models/pos_order.py
class PosOrder(models.Model):
    _inherit = 'pos.order'

    # Fiscal fields
    my_country_fiscal_state = fields.Char()
    my_country_fiscal_uuid = fields.Char()
    my_country_qr_code = fields.Char()

    @api.model
    def sync_from_ui(self, session_id, order_data):
        result = super().sync_from_ui(session_id, order_data)
        
        # Process fiscal compliance
        for order in self.browse([result['id']]):
            order._my_country_fiscal_process()
        
        return result

    def _my_country_fiscal_process(self):
        """Submit to government API, store response."""
        # 1. Build fiscal document
        # 2. Submit to government API
        # 3. Store response (UUID, QR code)
        # 4. Update fiscal state
        pass

    def write(self, vals):
        if self.my_country_fiscal_state == 'submitted':
            raise UserError(_('Cannot modify submitted fiscal order'))
        return super().write(vals)
```

### Frontend Localization Detection

```javascript
setup() {
    const country = this.pos.company.country_id;
    
    switch (country.code) {
        case 'MX':
            // Mexican CFDI flow
            this.showCfdiButton = true;
            break;
        case 'BE':
            // Belgian blackbox flow
            this.blackboxEnabled = true;
            break;
        case 'BR':
            // Brazilian NFC-e flow
            this.showQrCode = true;
            break;
        default:
            // Standard flow
            break;
    }
}
```

---

## 7. Localization Module Checklist

When creating a localization module, ensure:

### Backend
- [ ] Inherit `pos.order` with fiscal fields
- [ ] Override `sync_from_ui` for fiscal processing
- [ ] Protect fiscal orders from modification
- [ ] Implement government API integration
- [ ] Handle error cases (API down, rejected invoices)
- [ ] Implement refund/cancellation flow
- [ ] Add backend views for fiscal data viewing

### Frontend
- [ ] Detect country and enable appropriate features
- [ ] Display fiscal status on orders
- [ ] Show QR code if required
- [ ] Handle fiscal-specific UI flows
- [ ] Display error messages for fiscal failures

### Compliance
- [ ] Document all fiscal requirements
- [ ] Test edge cases (network down, API errors)
- [ ] Implement audit logging
- [ ] Handle offline fiscal scenarios
- [ ] Ensure data integrity (no partial submissions)

---

## 8. Multi-Country Considerations

```python
class PosConfig(models.Model):
    _inherit = 'pos.config'

    # Country detection
    fiscal_country = fields.Selection([
        ('MX', 'Mexico'),
        ('BR', 'Brazil'),
        ('BE', 'Belgium'),
        ('DE', 'Germany'),
        # ...
    ], related='company_id.country_id.code')

    # Multi-country feature flags
    enable_fiscal_module = fields.Boolean(
        compute='_compute_fiscal_module'
    )

    @api.depends('fiscal_country')
    def _compute_fiscal_module(self):
        for config in self:
            # Enable fiscal module based on country
            config.enable_fiscal_module = config.fiscal_country in (
                'MX', 'BR', 'BE', 'DE', 'FR', 'ES', 'SA'
            )
```
