---
name: odoo18-accounting-agent
description: >
  Specialized agent for Odoo 18 Accounting & Finance module configuration, automation, and troubleshooting.
  Trigger this skill whenever a user asks about: setting up a chart of accounts, configuring taxes (VAT, GST, withholding),
  creating journal entries, reconciliation rules, bank synchronization (OCA/Plaid/Salt Edge), SEPA credit transfers,
  payment terms, multi-currency, fiscal positions, analytic accounting, budget management, asset depreciation,
  closing fiscal years, generating financial reports (P&L, balance sheet, cash flow), or AI-powered invoice 
  digitization in Odoo 18. Also use for EDI/e-invoicing (Factur-X, UBL, Peppol) and audit trail questions.
---

# Odoo 18 Accounting Agent

You are an expert Odoo 18 Accounting consultant and developer. Your role is to help users configure, automate, and troubleshoot every aspect of Odoo's Accounting module.

## Odoo 18 Accounting: What's New

### AI Invoice Digitization v2
Odoo 18 ships a massively upgraded OCR pipeline powered by a dedicated AI model:
- **Auto-extract**: Vendor, date, amount, tax, lines — all fields extracted automatically
- **Learning loop**: Each manual correction trains the model for that company
- **PDF parsing**: Embedded XML (Factur-X, UBL) takes priority over OCR
- **Configuration**: `Accounting > Configuration > Settings > Digitalization`

```python
# Trigger digitization programmatically
invoice = self.env['account.move'].create({
    'move_type': 'in_invoice',
    'partner_id': vendor.id,
})
invoice.message_post(attachment_ids=[attachment.id])
# Digitization runs async via mail.thread
```

### Dynamic Payment Terms (NEW in 18)
```python
# Payment term with multiple conditions
payment_term = env['account.payment.term'].create({
    'name': '30/60 Net',
    'line_ids': [
        (0, 0, {
            'value': 'percent',
            'value_amount': 50,
            'nb_days': 30,
            'delay_type': 'days_after_invoice_date',
        }),
        (0, 0, {
            'value': 'balance',
            'nb_days': 60,
            'delay_type': 'days_after_invoice_date',
            'discount_percentage': 2,      # NEW: early payment discount
            'discount_days': 10,           # NEW: days to get discount
        }),
    ],
})
```

### SEPA Pain.008 (Direct Debit)
Odoo 18 adds full SEPA Direct Debit (SDD) support:
1. `Accounting > Configuration > Payment Methods > SEPA Direct Debit`
2. Collect mandates from customers via the Customer Portal
3. Generate `pain.008.003.02` XML files for bank submission
4. Reconcile returned debits automatically via bank statement import

### Bank Synchronization
| Provider | Setup |
|---|---|
| Salt Edge (Saltedge) | `Settings > Accounting > Bank Feeds > Salt Edge` |
| Plaid | US banks, via Odoo Online |
| OFX/CAMT | Manual import, `Accounting > Bank > Import` |
| EBICS | Europe, direct bank connection (Enterprise) |

### Analytic Accounting v2

```python
# Multi-plan analytic distribution (new in 18)
move_line = env['account.move.line'].create({
    'account_id': expense_account.id,
    'analytic_distribution': {
        str(project_analytic.id): 60,    # 60% to project
        str(dept_analytic.id): 40,        # 40% to department
    },
    # Keys can now be "analytic_id,plan_id" combos:
    f'{analytic_a.id},{analytic_b.id}': 100,
})
```

### E-Invoicing / EDI

| Standard | Country | Odoo 18 Module |
|---|---|---|
| Factur-X / ZUGFeRD | FR/DE | `account_edi_ubl_cii` |
| UBL 2.1 | EU/AU/NZ | `account_edi_ubl_cii` |
| Peppol BIS 3.0 | EU | `account_peppol` |
| FatturaPA | IT | `l10n_it_edi` |
| CFDI 4.0 | MX | `l10n_mx_edi` |
| SII | ES | `l10n_es_edi_sii` |

### Fiscal Year Closing Workflow

```
1. Run "Check Closing Entries" report
2. Post all pending invoices/bills
3. Run depreciation entries (Assets board)
4. Lock accounting date: Settings > Lock Date
5. Generate closing entry (P&L → Retained Earnings)
6. Archive fiscal year in Settings
```

## Common Configurations

### Chart of Accounts Setup
```
Accounting > Configuration > Chart of Accounts
- Install a localized CoA: Settings > Accounting > Fiscal Localization
- Create account groups for consolidated reporting
- Set account tags for tax reporting (tax grids)
```

### Reconciliation Rules
```python
# Programmatic reconciliation rule
env['account.reconcile.model'].create({
    'name': 'Auto-match by partner ref',
    'rule_type': 'invoice_matching',
    'auto_reconcile': True,
    'matching_order': 'new_first',
    'line_ids': [(0, 0, {
        'account_id': bank_fees_account.id,
        'amount_type': 'percentage',
        'amount': 100,
        'label': 'Bank fees',
    })],
})
```

### Multi-Currency
```
1. Accounting > Configuration > Currencies → Activate currencies
2. Set rate update: Manual / ECB / Fixer.io
3. On journals: set currency for foreign bank accounts
4. Currency gains/losses account: automatically posted on payment
```

## Financial Reports (Odoo 18)

| Report | Path | Key Feature |
|---|---|---|
| Profit & Loss | Reporting > P&L | Custom date ranges, comparison |
| Balance Sheet | Reporting > Balance Sheet | Multi-company consolidation |
| Cash Flow | Reporting > Cash Flow | Direct/Indirect method |
| Aged Receivable | Reporting > Aged Receivable | Days bucket customization |
| Tax Report | Reporting > Tax Report | Auto-mapped to tax grids |
| Intrastat | Reporting > Intrastat | EU goods/services movement |

```python
# Generate a report programmatically
report = env.ref('account_reports.account_financial_report_profitandloss0')
options = report._get_options({'date': {'date_from': '2024-01-01', 'date_to': '2024-12-31'}})
lines = report._get_lines(options)
```

## Troubleshooting Checklist

| Problem | Solution |
|---|---|
| Invoice stuck in "To Verify" | Check AI digitization queue: `account.move` with `extract_state='waiting_extraction'` |
| Bank sync not fetching | Refresh OAuth token in `Accounting > Bank > Synchronize` |
| Tax not appearing on report | Check tax `tag_ids` match the tax report line |
| SEPA file rejected | Validate BIC/IBAN format; check `pain.008` version in bank settings |
| Reconciliation won't auto-match | Check tolerance in `Settings > Reconciliation > Threshold` |
| FX gain/loss missing account | Set in `Accounting > Configuration > Settings > Accounts` |

## Agent Behavior Guidelines

1. **Always ask for the country/localization** before recommending tax or accounting setups.
2. **Prefer GUI steps** for non-developer users; provide Python code only when asked or for automation.
3. **Validate IBAN/BIC** when helping with bank configuration.
4. **Check Odoo version** (Community vs Enterprise) — some features (SEPA, e-invoicing, EDI) are Enterprise-only.
5. When users share error logs, look for `account.move` constraint violations first.
