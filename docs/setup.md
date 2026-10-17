# Setup

This repository is an **NDA-safe portfolio reconstruction**, not a one-click production deployment.

The published workflows preserve the orchestration patterns and system boundaries, but production credentials, endpoints, exact commercial values, infrastructure identifiers and the original database DDL are intentionally excluded.

## 1. Components required

To run an equivalent architecture, you would need:

- **n8n** — workflow orchestration;
- **Telegram Bot API** — user-facing bot;
- **PostgreSQL** — persistent application state;
- **Redis** — generation-job handoff;
- **S3-compatible object storage** — reference and generated media;
- **payment provider** — invoice creation and asynchronous payment callback;
- **external GPU generation worker / ComfyUI** — long-running content generation.

The public repository does not include infrastructure provisioning for these services.

## 2. Import order

Import the workflows in this order:

```text
01_telegram_bot_core.json
02_payment_balance_processor.json
03_generation_result_delivery.json
```

All public exports are committed with:

```text
active: false
```

Review and reconnect each integration before activating any workflow.

## 3. Reconnect n8n credentials

Credential bindings were deliberately removed from the public exports.

After import, configure appropriate credentials for the nodes used by your environment:

- Telegram;
- PostgreSQL;
- Redis;
- S3-compatible object storage;
- payment-provider HTTP authentication.

Do not place secrets directly into workflow JSON before publishing or sharing an export.

## 4. PostgreSQL

The portfolio workflows use the neutral schema name:

```text
portfolio_demo
```

Observed entities are documented in [data-model.md](data-model.md):

- `users`
- `user_state`
- `payments`
- `partners`
- `events`
- `scenarios`
- `localized_menu`

The original production DDL is not included. You must create an equivalent schema suitable for your own implementation before activating the workflows.

The data model document describes only fields and logical relationships visible in the sanitized workflows; it should not be treated as authoritative migration SQL.

## 5. Telegram Bot Core configuration

The Bot Core expects database, Telegram, Redis, object-storage and payment integrations to be available.

Public placeholders that must be replaced include:

```text
YOUR_BOT_USERNAME
YOUR_SUPPORT_USERNAME
YOUR_MEDIA_BUCKET
YOUR_FILE_KEY
YOUR_GENERATION_QUEUE
```

The public workflow also uses neutral example URLs for privacy/terms/payment-provider destinations. Replace them with values owned by your deployment.

### Environment variables

The sanitized Bot Core references these environment variables:

```text
ADMIN_CHAT_ID
APP_TIMEZONE
GENERATION_COST_TOKENS
WELCOME_TOKENS

PACKAGE_BASIC_TOKENS
PACKAGE_BASIC_PRICE

PACKAGE_STANDARD_TOKENS
PACKAGE_STANDARD_PRICE

PACKAGE_PLUS_TOKENS
PACKAGE_PLUS_PRICE

PACKAGE_PRO_TOKENS
PACKAGE_PRO_PRICE

PACKAGE_ADVANCED_TOKENS
PACKAGE_ADVANCED_PRICE

PACKAGE_MAX_TOKENS
PACKAGE_MAX_PRICE
```

These values are intentionally environment-specific. The repository does not contain the original production pricing.

## 6. Payment & Balance Processor configuration

The payment processor receives payment-status callbacks and then processes confirmed-but-unsettled transactions.

Public webhook path:

```text
payment-status-callback
```

The public workflow uses these environment variables for partner/reward calculation:

```text
ADMIN_CHAT_ID
PARTNER_FX_RATE
PARTNER_NET_FACTOR
PARTNER_SHARE
```

The original production coefficients are intentionally not included.

Before activation, confirm that your payment provider's callback payload is mapped to the fields expected by your implementation, especially the external transaction identifier and payment status.

## 7. Generation Result Delivery configuration

The delivery workflow receives the asynchronous generation-completion callback.

Public webhook path:

```text
generation-result-callback
```

Replace:

```text
YOUR_OUTPUT_BUCKET
```

with the object-storage bucket used by your deployment.

The published workflow expects the completion callback to provide an event identifier and a generated-media location that can be mapped back to the internal generation event.

## 8. Redis / generation worker

The Bot Core hands generation work to a Redis-backed queue rather than executing GPU generation inside the Telegram workflow.

Configure your own:

- Redis connection;
- queue/list/channel identifier;
- worker process;
- ComfyUI or equivalent GPU pipeline;
- callback destination back to the n8n result-delivery workflow.

No production Redis endpoint, queue name or worker infrastructure is included in this repository.

## 9. Localization

The source implementation uses five configured language codes:

```text
ru
en
es
uz
ar
```

The UI pattern is database-driven:

```text
users.language
      ↓
localized_menu
      ↓
captions / buttons / media references
      ↓
Telegram UI
```

To add another language, extend your own localization data and routing rules rather than duplicating the entire workflow.

Production copy and media assets are not included in the public repository.

## 10. Activation checklist

Before activating the workflows:

1. create an equivalent PostgreSQL schema;
2. reconnect all n8n credentials;
3. replace all `YOUR_*` placeholders and example URLs;
4. configure required environment variables;
5. configure Telegram webhook/trigger behavior;
6. configure payment creation and callback mapping;
7. configure Redis and the generation worker;
8. configure object storage;
9. test payment idempotency and balance settlement;
10. test generation-event correlation and result delivery;
11. test referral accounting using non-production values;
12. review the deployment against your own security and compliance requirements.

## 11. Scope

This setup guide explains how the published architecture fits together. It intentionally does not reproduce confidential production deployment details or guarantee that the portfolio exports can be activated without adaptation.
