# Telegram AI Content Platform — n8n

An NDA-safe portfolio reconstruction of a multilingual Telegram product for AI content generation.

The original system combines a user-facing bot, account and balance management, referral mechanics, payment processing, a PostgreSQL-backed application state, Redis-based generation jobs, an external GPU/ComfyUI generation layer, and asynchronous delivery of generated media back to the user.

> This repository contains sanitized public exports only. Production credentials, endpoints, bot identities, administrator IDs, commercial coefficients, provider-specific details and deployment metadata are intentionally excluded.

## Status

**Public portfolio release in progress.**

The project is being published in layers so the architecture can be reviewed independently from the workflow implementation.

## Product capabilities

The product supports:

- scenario-driven AI content generation;
- user-provided reference media;
- a Telegram account/profile experience;
- token balance display and top-up;
- multiple payment methods;
- referral links and partner rewards;
- an administrator/partner view;
- database-driven multilingual UI;
- five configured languages in the source implementation;
- PostgreSQL-backed users, state, payments and generation events;
- Redis-backed generation job handoff;
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

The sanitized portfolio version is split into three workflows:

| Workflow | Responsibility |
|---|---|
| `01_telegram_bot_core.json` | Main Telegram product logic: routing, profile, localized menus, generation scenarios, reference intake, balance UX, referral flows, payment initiation and generation-job creation. |
| `02_payment_balance_processor.json` | Processes confirmed payment callbacks, credits the user's token balance and applies referral rewards when applicable. |
| `03_generation_result_delivery.json` | Handles generation-complete callbacks, resolves the generation event, fetches the resulting media and delivers it to the user. |

## Architecture goals

The public repository focuses on the engineering patterns rather than the original commercial implementation:

- keep UI content and language variants data-driven;
- separate user interaction from payment settlement;
- separate payment settlement from generation execution;
- persist user and generation state in PostgreSQL;
- decouple the Telegram layer from GPU generation through a queue;
- deliver generated content asynchronously;
- make referral accounting explicit rather than embedding it in payment-provider logic;
- keep deterministic business state outside the generative model.

## Repository structure

```text
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

The workflow files and remaining documentation are being added incrementally.

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
- original workflow IDs, version IDs and instance metadata.

Where implementation values are still needed to understand the architecture, neutral placeholders or portfolio-only names are used instead.

## Technologies

- n8n
- Telegram
- PostgreSQL
- Redis
- object storage
- payment webhooks
- referral/accounting logic
- external GPU generation
- ComfyUI
- JSON / HTTP / webhooks

## Documentation

- [Architecture](docs/architecture.md)

Additional data-model, setup and security documentation will be added as the public release is completed.
