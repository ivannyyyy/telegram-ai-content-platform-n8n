# Telegram AI Content Platform — n8n

An NDA-safe portfolio reconstruction of a multilingual Telegram product for AI content generation.

The system combines a user-facing Telegram bot, account and balance management, payment processing, referral mechanics, database-driven localization, persistent PostgreSQL state, Redis-backed generation jobs, an external GPU/ComfyUI generation layer and asynchronous delivery of generated media back to the user.

> This repository contains sanitized public exports only. Production credentials, endpoints, bot identities, administrator IDs, commercial coefficients, provider-specific details and deployment metadata are intentionally excluded.

## Status

**Public portfolio release complete.**

The repository contains **3 sanitized n8n workflows / 213 nodes** plus architecture, data-model, setup and security documentation.

## Product capabilities

- scenario-driven AI content generation;
- user-provided reference media;
- Telegram account/profile experience;
- token balance display and top-up;
- multiple payment methods;
- referral links and partner rewards;
- administrator/partner views;
- database-driven multilingual UI;
- five configured languages in the source implementation;
- PostgreSQL-backed users, workflow state, payments and generation events;
- Redis-backed generation-job handoff;
- external GPU / ComfyUI processing;
- asynchronous generation-result delivery back to Telegram.

## System overview

```text
                           Telegram User
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │   Telegram Bot Core    │
                    │ menus / profile / UX   │
                    │ scenarios / references │
                    └───────────┬────────────┘
                                │
               ┌────────────────┼─────────────────┐
               │                │                 │
               ▼                ▼                 ▼
         PostgreSQL       Payment layer      Partner logic
       users / state      invoice creation    referrals / rewards
       events / config          │
               │                ▼
               │          payment callback
               │                │
               │                ▼
               │        Balance Processor
               │
               ▼
          Redis job queue
               │
               ▼
      GPU generation backend
            / ComfyUI
               │
               ▼
        completion callback
               │
               ▼
     Generation Result Delivery
               │
        object/media storage
               │
               ▼
          Telegram User
```

## Public workflow set

| Workflow | Nodes | Responsibility |
|---|---:|---|
| `01_telegram_bot_core.json` | 188 | Main Telegram product logic: routing, profile, localized menus, scenarios, reference intake, balance UX, referral flows, payment initiation and generation-job creation. |
| `02_payment_balance_processor.json` | 19 | Processes confirmed payment callbacks, credits the user's internal balance and applies referral rewards when applicable. |
| `03_generation_result_delivery.json` | 6 | Handles generation-complete callbacks, resolves the generation event, fetches generated media and delivers it to the user. |

## Main engineering patterns

### 1. Database-driven multilingual UI

The source implementation uses five configured languages:

```text
ru · en · es · uz · ar
```

Language preference is stored with the user, while captions/buttons/media references are loaded from a separate localization entity. This keeps localization data-driven instead of duplicating the full Telegram workflow per language.

### 2. Payment state separated from internal settlement

```text
payment request
      ↓
external provider
      ↓
payment callback
      ↓
external status confirmed
      ↓
internal balance settlement
      ↓
user tokens + optional referral reward
```

The public model keeps external payment state and internal credit state separate so the two lifecycle steps are explicit.

### 3. Async generation boundary

```text
Telegram request
      ↓
generation event
      ↓
Redis job queue
      ↓
GPU / ComfyUI worker
      ↓
completion callback
      ↓
object storage
      ↓
Telegram delivery
```

The Telegram request path does not wait for long-running GPU generation. Generation is handed off through a queue and completed asynchronously.

### 4. Explicit application state

The public data model separates:

- user/account state;
- Telegram workflow state;
- payment state;
- referral/partner accounting;
- generation events;
- scenario configuration;
- localized UI configuration.

This keeps deterministic business state outside the generative model and makes lifecycle transitions inspectable.

## Architecture goals

- keep UI content and language variants data-driven;
- separate user interaction from payment settlement;
- separate payment settlement from generation execution;
- persist user and generation state in PostgreSQL;
- decouple the Telegram layer from GPU generation through Redis;
- deliver generated content asynchronously;
- make referral accounting explicit rather than embedding it inside payment-provider logic;
- keep deterministic business state outside the generative model.

## Repository structure

```text
README.md

workflows/
├── 01_telegram_bot_core.json
├── 02_payment_balance_processor.json
└── 03_generation_result_delivery.json

docs/
├── architecture.md
├── data-model.md
├── setup.md
└── security.md
```

## NDA and sanitization

The public version does **not** expose the original production configuration.

Removed or generalized before publication:

- n8n credential references and credential names;
- generated webhook IDs;
- production bot usernames and support links;
- hard-coded administrator chat IDs;
- production payment URLs and provider-specific configuration;
- production database credential references;
- Redis connection details and queue identifiers;
- object-storage bucket identifiers;
- exact commercial pricing;
- exact referral percentages and financial coefficients;
- original workflow IDs, version IDs and instance metadata;
- customer-specific legal/support URLs and production copy where appropriate.

Public workflow exports are committed with `active: false` and use neutral placeholders, environment variables and the `portfolio_demo` schema where production values were removed.

See [Security & NDA Notes](docs/security.md) for the publication boundary and review checklist.

## Technologies

- n8n
- Telegram
- PostgreSQL
- Redis
- S3-compatible object storage
- payment webhooks / HTTP integrations
- referral/accounting logic
- external GPU generation
- ComfyUI
- JSON / HTTP / webhooks

## Documentation

- [Architecture](docs/architecture.md) — system boundaries and end-to-end flow
- [Data Model](docs/data-model.md) — observed public entities, fields and logical relationships
- [Setup](docs/setup.md) — portfolio configuration, placeholders and environment variables
- [Security & NDA Notes](docs/security.md) — sanitization boundary and deployment/security considerations

## Scope

This repository is intended for portfolio and architecture review. It demonstrates the implementation patterns and system decomposition of the original product without recreating the confidential production environment or providing one-click deployment.
