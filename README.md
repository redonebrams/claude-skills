# Odoo-Claude Skills Library

A curated collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for Odoo ERP development, configuration, and integration. These skills transform Claude into a specialized Odoo consultant, developer, and integration architect.

## What Are Claude Code Skills?

Skills are markdown-based instruction sets that give Claude deep domain expertise. When loaded, Claude can guide you through complex Odoo workflows, write standards-compliant code, and troubleshoot issues — all grounded in version-specific knowledge.

## Repository Structure

```
.
├── odoo18/
│   └── skills/
│       ├── odoo18-developer.md              # Core development & APIs
│       ├── odoo18-accounting-agent.md        # Accounting & Finance
│       ├── odoo18-manufacturing-agent.md     # MRP, Shop Floor, PLM, Quality
│       ├── odoo18-hr-payroll-agent.md        # HR, Payroll, Leaves, Recruitment
│       ├── odoo18-ecommerce-agent.md         # Website, eCommerce, Payments
│       ├── odoo18-ai-automation-agent.md     # AI features & Workflow Automation
│       ├── integrate-cloud-services-setup.md # Centralized cloud config module
│       ├── integrate-azure-key-vault.md      # Azure Key Vault integration
│       ├── integrate-azure-service-bus.md    # Azure Service Bus messaging
│       ├── integrate-google-cloud-storage.md # GCS file storage & CDN
│       ├── integrate-sage-x3.md             # Sage X3 bidirectional sync
│       └── sage-x3-code-review.md           # Code review for Sage connector
└── odoo19/                                   # (Planned)
```

## Skills Overview

### Domain Specialists

| Skill | Covers |
|-------|--------|
| **Accounting Agent** | Chart of accounts, taxes, bank sync, SEPA, e-invoicing (Factur-X, UBL, Peppol), multi-currency, fiscal year closing, AI invoice digitization |
| **Manufacturing Agent** | Bills of Materials, work centers, Shop Floor v2, routing, subcontracting, quality control points, PLM/ECOs, maintenance & OEE |
| **HR & Payroll Agent** | Employee records, salary rules engine, multi-country payroll localizations, leave accruals, 360-degree appraisals, recruitment pipeline, expense management |
| **eCommerce Agent** | AI website builder, product variants, pricelists, payment providers (Stripe/PayPal/Adyen), shipping, B2B portals, SEO, cookie consent v2 |
| **AI & Automation Agent** | LLM provider config (OpenAI/Claude/Mistral), CRM lead scoring, automated actions, cron jobs, chatbot builder, approval workflows, webhooks |

### Technical & Development

| Skill | Covers |
|-------|--------|
| **Odoo 18 Developer** | Module scaffold, ORM patterns, OWL 2 components, security (ACL/record rules), testing, CI/CD, Odoo 16/17 to 18 migration guide |

### Cloud Integrations

| Skill | Covers |
|-------|--------|
| **Cloud Services Setup** | Foundational `cloud_service_config` module — shared mixin, centralized settings UI for all cloud services |
| **Azure Key Vault** | Secure secret retrieval, DefaultAzureCredential, TTL caching, database gating |
| **Azure Service Bus** | Queue and topic messaging, batch sending, session-based ordering, async queue pattern |
| **Google Cloud Storage** | File upload/download/delete, CDN URLs, signed URLs, Odoo model lifecycle hooks |

### ERP Integration

| Skill | Covers |
|-------|--------|
| **Sage X3 Connector** | Bidirectional sync architecture, field mapping & transformation, workflow orchestration, conflict resolution, webhook support |
| **Sage X3 Code Review** | Blocking issue checklist, BaseSkill rules, queue/cron validation, Odoo 18 compliance checks |

## Usage

### With Claude Code CLI

Copy skills into your project's `.claude/skills/` directory:

```bash
# Copy all Odoo 18 skills
cp odoo18/skills/*.md /path/to/your-project/.claude/skills/
```

Or symlink for easy updates:

```bash
ln -s /path/to/Odoo-Claude/odoo18/skills/*.md /path/to/your-project/.claude/skills/
```

Skills are automatically available to Claude Code once placed in the `.claude/skills/` directory.

### Invoke a Skill

In Claude Code, skills trigger automatically based on context, or you can reference them directly:

```
> Help me configure multi-currency accounting
  (triggers odoo18-accounting-agent)

> Create a new Odoo module for inventory barcode scanning
  (triggers odoo18-developer)

> Set up Azure Service Bus messaging for order sync
  (triggers integrate-azure-service-bus)
```

## Contributing

To add a new skill:

1. Create a markdown file in the appropriate version directory (e.g., `odoo18/skills/`)
2. Include frontmatter with `name`, `description`, and `triggers`
3. Structure content with clear sections, code examples, and checklists
4. Follow existing skills as a template for format and depth

## License

MIT
