# OLX API Assistant

A Claude Code skill for integrating with OLX Partner API v2.0, focused on the Ukrainian real estate market.

## What this skill does

Activates automatically when you work on OLX API integration tasks:

- OAuth 2.0 flow setup and debugging
- Publishing, updating, and managing adverts lifecycle
- Diagnosing 400/401/403/429 errors (full error catalog in KB)
- Designing multi-tenant SaaS architecture for real estate agencies
- Idempotent CRM → OLX sync via `external_id`
- Content normalization (caps, phones in text, punctuation rules)
- Chat (threads) polling and message handling

## Structure

```
olx-api-assistant/
├── SKILL.md                      # Skill instructions and behavior rules
└── references/
    └── knowledge-base.md         # Full OLX Partner API v2.0 reference
```

## Knowledge Base sections

| Section | Content |
|---|---|
| §1 | Quick start, required headers |
| §2 | OAuth 2.0 — full multi-user SaaS flow |
| §3 | All endpoints by functional group |
| §4 | Advert lifecycle (statuses, commands) |
| §5 | Content rules and validation |
| §6 | POST `/adverts` template for real estate |
| §7 | Error catalog with solutions |
| §8 | Multi-tenant SaaS architecture |
| §9 | Ukraine real estate specifics |
| §10 | Integration readiness checklist |
| §11 | Request templates |
| §12 | Glossary |

## Installation

### Claude Code (CLI / Desktop)

```bash
# Install from GitHub
/install-skill https://github.com/YOUR_USERNAME/olx-api-assistant
```

Or clone and install locally:

```bash
git clone https://github.com/YOUR_USERNAME/olx-api-assistant
```

Then add to your project's `.claude/settings.json`:

```json
{
  "skills": ["./olx-api-assistant"]
}
```

### Claude.ai Projects

1. Create a new Project named `OLX API Assistant`
2. In **Project knowledge** → upload `references/knowledge-base.md`
3. In **Custom instructions** → paste the contents of `SKILL.md`

## Usage

The skill activates automatically when context suggests OLX API work. You can also trigger it explicitly:

> Покажи приклад POST /adverts для продажу 2-кімнатної квартири в Дніпрі

> У мене помилка 403 Insufficient scope — що робити?

> Як спроектувати зберігання токенів для multi-tenant SaaS?

## API version

OLX Partner API v2.0 | Base URL: `https://www.olx.ua/api/partner`

## License

MIT
