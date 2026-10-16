# Data Model

This document describes the **observed public data model** used by the sanitized Telegram AI Content Platform workflows.

It is derived from the published n8n exports and is intentionally limited to fields and relationships visible there. It is **not** the original production DDL: primary/foreign-key constraints, indexes, database functions, triggers, retention rules and infrastructure-specific configuration are not asserted unless the workflows make them explicit.

The public workflows use the neutral schema:

```text
portfolio_demo
```

## 1. Entity overview

```text
                         ┌─────────────────┐
                         │      users      │
                         │ user / language │
                         │ token balance   │
                         └───────┬─────────┘
                                 │ user_id
                  ┌──────────────┼───────────────┐
                  │              │               │
                  ▼              ▼               ▼
          ┌────────────┐   ┌────────────┐   ┌────────────┐
          │ user_state │   │  payments  │   │  partners  │
          │ UI/runtime │   │ settlement │   │ referrals  │
          └─────┬──────┘   └────────────┘   └────────────┘
                │
                │ event_id / prompt_id
                ▼
          ┌────────────┐
          │   events   │
          │ generation │
          │ lifecycle  │
          └─────┬──────┘
                │ prompt_id
                ▼
          ┌────────────┐
          │ scenarios  │
          │ prompt data│
          └────────────┘

          ┌──────────────────┐
          │  localized_menu  │
          │ captions/buttons │
          │ per language     │
          └──────────────────┘
```

The workflows treat these entities as separate concerns:

- `users` — account identity, language and product balance;
- `user_state` — conversational/UI state for the Telegram workflow;
- `payments` — external transaction and internal settlement state;
- `partners` — referral link and partner earnings/statistics;
- `events` — asynchronous content-generation event state;
- `scenarios` — scenario/prompt configuration;
- `localized_menu` — database-driven UI text and media references.

## 2. `portfolio_demo.users`

Observed fields:

| Field | Observed type | Role |
|---|---|---|
| `user_id` | number | Telegram/user identity used to resolve the account |
| `created_date` | dateTime | account creation timestamp |
| `name` | string | username/account label |
| `first_name` | string | display name used by the Telegram UI |
| `balance_token` | number | internal token balance |
| `reference` | string | referral/inviter reference captured at registration |
| `language` | string | selected UI language |

The Bot Core reads the user by `user_id`, creates new users and updates account-level fields.

The source workflow contains five configured language codes:

```text
ru
en
es
uz
ar
```

The important architectural point is that language preference is stored on the user record while the actual UI copy is read from the separate localization entity.

## 3. `portfolio_demo.user_state`

Observed fields:

| Field | Observed type | Role |
|---|---|---|
| `user_id` | number | owner of the current Telegram state |
| `message_menu_id` | number | message used for menu update/delete operations |
| `url_photo` | string | reference media associated with the current generation flow |
| `prompt_id` | number | selected content scenario/prompt |
| `event_id` | number | active/related generation event |
| `updated_at` | dateTime | state update timestamp |
| `bot_blocked` | boolean | workflow state flag |
| `rezerv_bot_blocked` | boolean | secondary/reserve state flag retained by the public export |

This entity is different from `users`.

`users` represents the persistent account. `user_state` stores transient or workflow-oriented state needed to continue Telegram interactions and content-generation steps.

A simplified relation is:

```text
users.user_id
     │
     └────► user_state.user_id
```

The workflows also pass `prompt_id` and `event_id` through this state layer so the Telegram UI can reconnect later steps with the selected scenario and generation job.

## 4. `portfolio_demo.payments`

Observed fields:

| Field | Observed type | Role |
|---|---|---|
| `transaction_id` | string | external transaction identifier |
| `telegram_user_id` | number | user receiving the purchased product balance |
| `tokens` | number | internal tokens associated with the payment |
| `amount` | number | monetary amount |
| `currency` | string | payment currency |
| `payment_method` | string | selected payment method |
| `provider_payment_method` | string | provider-side payment method representation |
| `payment_url` | string | generated payment/invoice URL |
| `status_bank` | string | external/payment-side status |
| `status_balance` | string | internal balance-settlement status |
| `created_at` | dateTime | payment creation timestamp |

The public workflows deliberately separate two statuses:

```text
external payment state  → status_bank
internal credit state   → status_balance
```

This prevents a confirmed provider notification from being treated as equivalent to “internal balance definitely processed.”

The Balance Processor selects transactions where the external state is confirmed and the internal settlement is still pending, credits the user's tokens, then marks the settlement as done.

Conceptually:

```text
payments.telegram_user_id
          │
          ▼
      users.user_id
          │
          ▼
 users.balance_token
```

Exact package prices and provider-specific commercial configuration are intentionally absent from the public repository.

## 5. `portfolio_demo.partners`

Observed fields:

| Field | Observed type | Role |
|---|---|---|
| `user_id` | number | partner/referrer identity |
| `ref_url` | string | generated referral link |
| `invited_count` | number | referral statistics |
| `earned_tokens` | number | token-denominated partner metric |
| `earned_usdt` | number | monetary/reward balance metric |
| `drop_usdt` | number | payout/withdrawal-related metric visible in the workflow |

The Bot Core can create/read partner records and show partner statistics.

A new account can retain an inviter reference in:

```text
users.reference
```

The payment processor uses that relationship to determine whether a confirmed paid transaction should create a partner reward.

Simplified flow:

```text
partners.user_id
       │
       │ referral link
       ▼
new user
       │
       └────► users.reference

confirmed payment
       │
       ▼
users.reference
       │
       ▼
partners.user_id
       │
       ▼
partner earnings update
```

The public version keeps the relationship and accounting flow visible but moves the exact rate/share coefficients to environment configuration.

## 6. `portfolio_demo.events`

Observed fields:

| Field | Observed type | Role |
|---|---|---|
| `id` | number | generation event identifier |
| `user_id` | number | user that owns the generation request |
| `date` | dateTime | event timestamp |
| `status` | string | generation/event status |
| `prompt_id` | number | selected scenario/prompt |
| `message_id` | number | Telegram message association |
| `url_photo` | string | input/reference media |
| `url_video` | string | generated video/media location |
| `retry_count` | number | retry state |
| `error_msg` | string | generation error information |
| `notified` | boolean | whether the generated result was delivered/notified |

The Bot Core creates a generation event before handing work to the generation queue.

The Result Delivery workflow receives an external completion callback containing an event identifier, resolves the matching `events` record, obtains the target `user_id`, fetches the generated media and finally updates:

```text
notified = true
```

This makes the event record the bridge between the interactive Telegram request and the asynchronous generation backend.

```text
users.user_id
    │
    ▼
events.user_id
    │
    ├────► generation queue / worker
    │
    ◄──── completion callback
    │
    ▼
Telegram delivery
    │
    ▼
events.notified = true
```

## 7. `portfolio_demo.scenarios`

The public Bot Core queries `scenarios` by an `id`-like scenario identifier and consumes at least:

```text
prompt_text
```

This entity represents generation scenario configuration.

The published workflow does not expose enough authoritative schema metadata to document the full table definition, so this repository intentionally does not invent additional columns.

Conceptually:

```text
user selects scenario
        │
        ▼
scenarios
        │
        ▼
prompt_text
        │
        ▼
generation request
```

## 8. `portfolio_demo.localized_menu`

The Bot Core queries a database-backed UI configuration entity named `localized_menu`.

Observed content includes menu/profile/language/balance/payment/rules/admin/wait/prompt captions and buttons, plus associated media/URL fields.

Examples of observed categories include:

- main menu captions/buttons;
- profile captions/buttons;
- language-selection captions/buttons;
- balance and payment captions/buttons;
- generation-scenario captions/buttons;
- post-generation controls;
- rules/help content;
- partner/admin portal content;
- wait/no-token messages.

The complete production content is not published because it may contain customer-specific copy and assets.

The architectural pattern is:

```text
users.language
      │
      ▼
localized_menu
      │
      ▼
caption / buttons / media references
      │
      ▼
Telegram UI
```

This allows the workflow to support additional UI languages through data/configuration rather than duplicating the full orchestration graph.

## 9. Cross-workflow ownership

The three published workflows use the data model differently.

| Entity | Telegram Bot Core | Payment & Balance Processor | Generation Result Delivery |
|---|---:|---:|---:|
| `users` | read/write | read/write balance | — |
| `user_state` | read/write | — | — |
| `payments` | create | update/select/settle | — |
| `partners` | create/read/update | read/update reward | — |
| `events` | create | — | read/update delivery |
| `scenarios` | read | — | — |
| `localized_menu` | read | — | — |

This separation mirrors the system boundaries:

```text
Bot Core
  → interaction and job creation

Payment Processor
  → financial confirmation and internal crediting

Result Delivery
  → asynchronous compute completion and user notification
```

## 10. Main relationships

The following relationships are **observed logically from workflow usage**. They are not claims about physical database foreign-key constraints.

```text
users.user_id
 ├──── user_state.user_id
 ├──── payments.telegram_user_id
 ├──── partners.user_id
 └──── events.user_id

users.reference
 └──── partners.user_id        (referral relationship)

user_state.prompt_id
 └──── scenarios.id            (selected scenario)

user_state.event_id
 └──── events.id               (active generation context)

events.prompt_id
 └──── scenarios.id            (scenario used for generation)
```

## 11. State transitions visible in the workflows

### Payment settlement

```text
payment created
      │
      ▼
external callback
      │
      ▼
status_bank updated
      │
      ▼
confirmed + internal pending
      │
      ▼
user token balance credited
      │
      ├──── optional partner reward
      │
      ▼
status_balance = done
```

### Generation lifecycle

```text
scenario selected
      │
reference/input collected
      │
generation event created
      │
job queued
      │
external GPU / ComfyUI processing
      │
completion callback
      │
event lookup
      │
media delivery
      │
notified = true
```

## 12. Public-model limitations

The public repository intentionally does not claim or publish:

- exact PostgreSQL DDL;
- primary-key/foreign-key constraint definitions;
- production indexes;
- database triggers/functions;
- production schema/table names outside the neutral public model;
- payment-provider secrets or request contracts;
- exact pricing and package economics;
- exact referral percentages or exchange/commission coefficients;
- production Redis/job identifiers;
- production object-storage configuration;
- customer-specific localized copy or media.

The purpose of this document is to explain the application state model required by the published workflow architecture without reconstructing confidential production infrastructure.
