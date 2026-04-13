# Odoo 18 POS Development Skills

> Complete development skill package for Odoo 18 Enterprise POS (Point of Sale). Covers 94+ modules, OWL/JS frontend, Python backend, extension patterns, localization, IoT integration, and best practices.

[![Odoo 18.0](https://img.shields.io/badge/Odoo-18.0-blue.svg)](https://www.odoo.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Modules Documented](https://img.shields.io/badge/Modules-94%2B-orange.svg)](references/module-catalog.md)

---

## Overview

This is a comprehensive, AI-optimized skill package for developing Odoo 18 POS extensions. It was created by exhaustively analyzing the entire Odoo 18 Enterprise POS codebase (both core and enterprise modules) and complementing with community best practices.

### What's Included

| Document | Description |
|----------|-------------|
| [SKILL.md](SKILL.md) | **Main skill file** — Quick start, architecture overview, module catalog summary, extension patterns, troubleshooting |
| [Module Catalog](references/module-catalog.md) | Detailed documentation for all **94+ POS modules** with models, fields, dependencies |
| [Architecture Patterns](references/architecture-patterns.md) | SPA bootstrap, data flow, offline architecture, multi-device sync |
| [Frontend Patterns](references/frontend-patterns.md) | OWL components, hooks, screens, services, testing |
| [Backend Patterns](references/backend-patterns.md) | Python models, mixins, session lifecycle, order processing |
| [Extension Patterns](references/extension-patterns.md) | Safe override patterns, module structure, testing |
| [Localization Guide](references/localization-guide.md) | Fiscal compliance for 20+ countries |
| [Best Practices](references/best-practices.md) | Code quality, performance, security, upgrade-safety |
| [Quick Reference](references/quick-reference.md) | Cheat sheets for fast lookups |

### Stats

- **94+ modules** documented with full details
- **8 reference files** covering all aspects of POS development
- **15,000+ lines** of documentation
- **100+ code examples** ready for copy-paste

---

## Installation

### ⚡ One-Line Install (Recommended)

**For Claude Code / Gemini CLI / any AI agent using `npx skills`:**

```bash
npx skills add lfcamarac/pos-18-odoo-skills
```

This single command downloads the repo and installs the `SKILL.md` into your AI agent's skills directory automatically.

### Option A: Install via `npx skills add` (Claude Code, Gemini, Copilot)

```bash
# Installs to ~/.claude/skills/ (Claude Code) or equivalent
npx skills add lfcamarac/pos-18-odoo-skills

# Or with full URL
npx skills add https://github.com/lfcamarac/pos-18-odoo-skills
```

**Supported agents:** Claude Code, Gemini CLI (`gemini skills install`), GitHub Copilot CLI, and any agent compatible with the `npx skills` registry.

### Option B: Install via `skillfish`

```bash
# Install skillfish CLI (if not already installed)
npm install -g skillfish

# Install the skill globally
skillfish install lfcamarac/pos-18-odoo-skills
```

### Option C: Manual Git Install

```bash
# Clone the repository
git clone https://github.com/lfcamarac/pos-18-odoo-skills.git

# Copy to your AI agent's skills directory
mkdir -p ~/.claude/skills/pos-18-odoo-skills
cp -r pos-18-odoo-skills/SKILL.md pos-18-odoo-skills/references ~/.claude/skills/pos-18-odoo-skills/
```

### Option D: Manual Download (ZIP)

```bash
curl -L https://github.com/lfcamarac/pos-18-odoo-skills/archive/refs/heads/main.zip -o pos-skills.zip && \
  unzip pos-skills.zip -d /tmp && \
  mkdir -p ~/.claude/skills/pos-18-odoo-skills && \
  cp -r /tmp/pos-18-odoo-skills-main/{SKILL.md,references} ~/.claude/skills/pos-18-odoo-skills/ && \
  rm -rf /tmp/pos-18-odoo-skills-main pos-skills.zip
```

### Option E: Direct Reference (No Install)

Simply keep this repository accessible and reference `SKILL.md` when working with Odoo POS:

```
When working on Odoo 18 POS, reference: /path/to/pos-18-odoo-skills/SKILL.md
```

---

### Installation Matrix

| Method | Command | Best For |
|--------|---------|----------|
| **One-liner** | `npx skills add lfcamarac/pos-18-odoo-skills` | Quick install, any agent |
| **skillfish** | `npm i -g skillfish && skillfish install lfcamarac/pos-18-odoo-skills` | Teams, shared skills |
| **Git clone** | `git clone` + manual copy | Full repo access |
| **ZIP** | `curl` + `unzip` | No git/npm needed |

---

## Quick Start

### Minimum POS Module

```
my_pos_extension/
├── __init__.py
├── __manifest__.py
├── models/__init__.py, pos_config.py
├── static/src/overrides/pos_store.js
├── static/src/xml/templates.xml
└── views/pos_config_views.xml
```

### __manifest__.py

```python
{
    'name': 'My POS Extension',
    'depends': ['point_of_sale'],
    'assets': {
        'point_of_sale._assets_pos': [
            'my_pos_extension/static/src/**/*.js',
            'my_pos_extension/static/src/**/*.xml',
        ],
    },
}
```

### Frontend Patch

```javascript
/** @odoo-module **/
import { PosStore } from "@point_of_sale/app/store/pos_store";
import { patch } from "@web/core/utils/patch";

patch(PosStore.prototype, {
    setup() {
        super.setup(...arguments);
        this.myFeature = false;
    },
});
```

### Backend Extension

```python
from odoo import models, fields

class PosConfig(models.Model):
    _inherit = 'pos.config'
    my_feature = fields.Boolean('My Feature')
```

---

## Module Coverage

### Tier 1 — Core
- ✅ point_of_sale (main application)
- ✅ pos_restaurant (floor plans, kitchen printing)
- ✅ pos_hr (employee login)
- ✅ pos_loyalty (loyalty programs, gift cards)
- ✅ pos_self_order (self-order kiosk)
- ✅ pos_sale (sales integration)
- ✅ pos_online_payment (online payments)

### Tier 2 — Payments
- ✅ pos_adyen, pos_stripe, pos_six
- ✅ pos_paytm, pos_razorpay, pos_pine_labs
- ✅ pos_mercado_pago, pos_tyro, pos_viva_wallet
- ✅ pos_epson_printer

### Tier 3 — Enterprise
- ✅ pos_enterprise, pos_mobile
- ✅ pos_preparation_display (kitchen display)
- ✅ pos_iot (IoT Box integration)
- ✅ pos_blackbox_be (Belgian fiscal compliance)
- ✅ pos_avatax, pos_pricer, pos_barcodelookup
- ✅ pos_restaurant_appointment
- ✅ whatsapp_pos

### Tier 4 — UrbanPiper Suite
- ✅ pos_urban_piper, pos_urban_piper_enhancements
- ✅ pos_restaurant_urban_piper
- ✅ pos_urban_piper_swiggy, _ubereats, _zomato

### Tier 5 — Localization (20+ countries)
- ✅ Mexico (CFDI), Brazil (NFC-e), Saudi Arabia (ZATCA)
- ✅ Belgium (Blackbox), Germany (TSS/Fiskaly)
- ✅ France (anti-fraud), Spain (TicketBAI, Veri*Factu)
- ✅ India (GST), Peru, Ecuador, Chile, Colombia
- ✅ Italy, Kenya, Poland, Malaysia, Indonesia, and more

---

## Architecture Highlights

### Three-Pillar System
1. **`pos.config`** — Settings per terminal
2. **`pos.session`** — Runtime session (open → close)
3. **`pos.order`** — Sales transactions

### Data Flow
```
Frontend (OWL SPA) ←→ RPC ←→ Backend (Python models)
        ↓                              ↓
   IndexedDB                      PostgreSQL
   (offline)                   (permanent store)
```

### Asset Bundles
```
point_of_sale.assets_prod
└── point_of_sale._assets_pos  ← Register your extensions here
```

---

## When to Use This Skill

Use this when you need to:
- Create a new POS extension module
- Customize the POS UI (add buttons, screens, fields)
- Integrate payment terminals or hardware
- Implement fiscal compliance for any country
- Extend POS models or views in the backend
- Debug POS issues (sync, offline, payments)
- Understand how the POS SPA works internally

---

## License

MIT — Use freely in personal and commercial projects.

---

## Contributing

Contributions welcome! Please submit PRs for:
- Corrections to module documentation
- Additional code examples
- New patterns discovered in production
- Localization details for specific countries

---

## Acknowledgments

Based on analysis of:
- Odoo 18 Enterprise POS codebase (94+ modules)
- Odoo official documentation
- Community best practices from Cybrosys, BrainCuber, and other contributors
- Real-world production POS implementations
