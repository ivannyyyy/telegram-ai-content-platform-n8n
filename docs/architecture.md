# Architecture

This document describes the public, NDA-safe architecture of the Telegram AI Content Platform.

It intentionally focuses on system boundaries, state flow and responsibilities. Production endpoints, credentials, provider identifiers, bot identities, commercial pricing and partner coefficients are not part of the public repository.

## 1. Product boundary

The platform connects five functional areas:

1. **Telegram product layer** — user interaction, menus, profile, language selection, content scenarios and reference intake.
2. **Application state** — users, balances, referrals, localized interface content, payments and generation events.
3. **Payment processing** — invoice creation in the bot and asynchronous confirmation through a payment callback.
4. **Generation runtime** — generation requests are converted into queued jobs and processed by an external GPU / ComfyUI layer.
5. **Result delivery** — generation completion is reported asynchronously and the produced media is delivered back to the correct Telegram user.

## 2. Main product flow

```text
Telegram user
     │
     ▼
Telegram Bot Core
     │
     ├── resolve/create user
     ├── load language + menu
     ├── show profile/balance
     ├── handle referral context
     └── start content scenario
                │
                ▼
        collect reference/input
                │
                ▼
        validate available balance
                │
                ▼
        create generation event
                │
                ▼
          enqueue generation job
                │
                ▼
          Redis / job transport
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
                ├── resolve event/user
                ├── fetch media
                ├── send result
                └── mark notification state
```

The public export preserves this separation without exposing the original queue names, infrastructure addresses or storage configuration.

## 3. Telegram Bot Core

The main workflow acts as the product orchestration layer.

Its responsibilities include:

- Telegram message/callback normalization;
- user creation and state lookup;
- account/profile navigation;
- balance presentation;
- language selection;
- localized menu lookup;
- referral-link generation;
- referral/partner screens;
- content-generation scenario selection;
- reference-file intake;
- token/balance validation;
- payment initiation;
- generation-event creation;
- job handoff to the generation backend.

### Database-driven localization

The source implementation supports five configured languages.

Interface content is read from database-backed configuration rather than being entirely hard-coded into routing logic. This means additional language variants can be introduced by extending the content/configuration layer rather than duplicating the full Telegram workflow.

The public repository describes this pattern but does not publish production copy, media assets or original external links.

## 4. Account, balance and referral model

The product maintains a token-style user balance.

A typical top-up lifecycle is:

```text
user selects package
       │
       ▼
bot creates payment request
       │
       ▼
external payment system
       │
       ▼
payment callback
       │
       ▼
payment state = confirmed
       │
       ▼
Balance Processor
       │
       ├── credit user balance
       ├── mark settlement processed
       └── apply partner/referral reward if eligible
```

The public workflow keeps the accounting structure visible but replaces exact production pricing, rates, percentages and provider-specific values with neutral portfolio configuration.

## 5. Referral / partner flow

Users can have a generated referral link.

When another user joins through that link, the referring user can become eligible for a share of resulting revenue according to the application's commercial rules.

The portfolio version preserves the architecture:

```text
partner
   │
   ▼
referral link
   │
   ▼
new user registration
   │
   ▼
referral relationship
   │
   ▼
confirmed paid transaction
   │
   ▼
partner reward calculation
   │
   ▼
partner balance / statistics
```

Exact financial coefficients are intentionally omitted.

## 6. Payment boundary

Payment creation belongs to the Telegram product flow, while settlement belongs to a separate callback processor.

This is an important boundary:

- the bot requests a payment/invoice;
- the external provider handles settlement;
- a callback updates payment state;
- the balance processor credits internal product value only after a confirmed payment state.

The source implementation supports more than one payment method. The public repository does not expose production provider endpoints, credential identifiers or provider-specific request contracts.

## 7. Generation queue

The Telegram workflow does not directly run the GPU model.

Instead, it creates/persists a generation event and hands a job to a queue layer.

```text
Telegram request
      │
      ▼
generation event
      │
      ▼
Redis-backed job handoff
      │
      ▼
generation worker
      │
      ▼
ComfyUI / GPU pipeline
```

This decouples Telegram request handling from long-running model execution.

The public repository intentionally excludes production Redis addresses, authentication and queue/channel identifiers.

## 8. Generation completion

Model execution is asynchronous.

When the external generation layer completes a job, it calls the result-delivery workflow.

The delivery path resolves the internal generation event, retrieves the generated media from object storage and sends the result to the corresponding Telegram user.

```text
generation backend
       │
       ▼
completion webhook
       │
       ▼
generation event lookup
       │
       ▼
object/media storage
       │
       ▼
Telegram send
       │
       ▼
delivery state update
```

The workflow does not need the model to remain coupled to the user's original Telegram execution.

## 9. State responsibilities

The public architecture separates several categories of state:

- **user state** — identity, language, product navigation and balance;
- **payment state** — transaction lifecycle and settlement status;
- **referral state** — inviter relationship and partner earnings;
- **generation state** — content request/event lifecycle;
- **UI configuration** — localized menu/caption/button content.

Exact production table names and full database DDL are outside the scope of the public export.

## 10. Why three workflows

The public portfolio uses three major workflows because they correspond to three independent lifecycle boundaries:

### Telegram Bot Core

User-facing orchestration and job creation.

### Payment & Balance Processor

External financial confirmation and internal balance/referral accounting.

### Generation Result Delivery

External compute completion and user-facing delivery.

Keeping these concerns separate reduces coupling between:

- Telegram interaction;
- payment-provider availability;
- long-running GPU processing.

## 11. NDA-safe publication boundary

The repository is intended to demonstrate architecture and implementation patterns, not recreate the original production environment.

The following are outside the public boundary:

- real bot usernames;
- customer/support URLs;
- real administrator IDs;
- payment-provider URLs and credentials;
- exact package prices;
- exact referral percentages;
- exchange-rate/commission constants;
- Redis endpoints and queue names;
- object-storage bucket names;
- production database credentials;
- generated n8n webhook IDs;
- n8n instance identifiers;
- production media/assets and customer-specific text.

The public workflow exports are therefore suitable for architecture review but not one-click deployment.
