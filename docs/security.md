# Security & NDA Notes

This repository contains a deliberately sanitized portfolio export of a commercial-style AI content platform.

The goal is to demonstrate architecture and workflow engineering without exposing production access, customer-specific configuration or confidential commercial parameters.

## 1. Public-export boundary

The public workflow set was reviewed to remove or generalize:

- n8n credential bindings and credential names;
- generated webhook IDs;
- n8n workflow/version/instance metadata;
- production bot usernames and support identities;
- hard-coded administrator chat IDs;
- production payment endpoints and provider configuration;
- production database credential references;
- Redis connection details and production queue identifiers;
- object-storage bucket identifiers;
- production pricing;
- referral percentages, exchange rates and commission coefficients;
- customer-specific legal/support URLs;
- production schema names.

Neutral values such as `portfolio_demo`, `YOUR_*` placeholders, environment variables and example URLs are used where architecture still needs to be understandable.

## 2. Repository scan

The published workflow files were checked for known production identifiers and for n8n export fields that commonly reveal deployment metadata.

At final review, the three public workflows contained no known production identifiers from the source exports and no persisted:

```text
credentials
webhookId
versionId
instanceId
```

The remaining URLs in the Bot Core are placeholders or neutral example destinations.

This review reduces disclosure risk, but it is not a legal determination that every architectural detail is publishable under a specific NDA.

## 3. Secrets

Do not commit:

- bot tokens;
- database passwords/connection strings;
- Redis passwords;
- S3 access keys;
- payment-provider secrets;
- API keys;
- private webhook signing secrets;
- production chat/user identifiers.

Keep secrets in the n8n credential store, deployment secret manager or environment configuration appropriate to your infrastructure.

## 4. Webhook security

The portfolio workflows illustrate callback boundaries but do not publish a production authentication contract.

For a real deployment, payment and generation callbacks should be authenticated according to the capabilities of the upstream system, for example through signed payloads, shared-secret headers, provider-native signature verification or an authenticated gateway.

Do not trust a callback solely because it reaches the expected URL.

For payment callbacks in particular, validate:

- transaction identity;
- expected status transition;
- amount/currency where applicable;
- replay/idempotency behavior;
- whether the transaction has already been credited.

## 5. Payment idempotency

The published architecture separates:

```text
status_bank
```

from:

```text
status_balance
```

so external confirmation and internal crediting are represented separately.

A production implementation should enforce idempotent settlement at the database/application layer so repeated callbacks or worker retries cannot credit the same payment multiple times.

The exact production constraint/transaction strategy is outside the public repository.

## 6. Referral accounting

Partner/reward coefficients are represented by environment variables:

```text
PARTNER_FX_RATE
PARTNER_NET_FACTOR
PARTNER_SHARE
```

Do not expose real commercial coefficients in public workflow exports.

A production implementation should also protect reward updates from duplicate settlement and define auditable transaction history appropriate to the business model.

## 7. Database access

Use least-privilege database credentials.

The public workflows show application-level tables but do not require the n8n database user to have broad cluster-wide privileges.

For a real deployment, separate permissions where appropriate and avoid using administrative PostgreSQL credentials in application workflows.

## 8. Redis and generation workers

Redis is part of the asynchronous generation boundary.

Production deployments should not expose Redis directly to the public internet. Use authenticated/private network access and restrict the worker to the minimum queue operations it requires.

Treat generation-job payloads as potentially sensitive if they contain user-provided media or prompts.

## 9. Object storage

The public exports replace production bucket identifiers with placeholders.

For real media storage:

- keep buckets private unless public access is explicitly required;
- prefer scoped credentials;
- use time-limited access URLs where appropriate;
- define retention/deletion rules for user media;
- do not log secret storage URLs or access tokens into public workflow exports.

## 10. Telegram / user identifiers

Telegram IDs are application identifiers and should be treated as user data.

The portfolio version removes hard-coded production administrator IDs and does not include production execution data.

Avoid publishing pinned execution payloads or test data that contains real user IDs, usernames, media URLs or transaction data.

## 11. n8n export hygiene

Before publishing any future workflow export, check at minimum for:

```text
credentials
webhookId
versionId
instanceId
pinData
hard-coded chat IDs
production URLs
bot usernames
bucket names
Redis endpoints
database/schema names
pricing and partner coefficients
sample execution payloads
```

Also inspect Code/Set nodes, HTTP headers and expressions: secrets can be embedded there even when the n8n `credentials` property has been removed.

## 12. NDA note

Technical sanitization and legal permission are different questions.

This repository intentionally removes production-identifying and commercially sensitive values, but whether an anonymized architecture may be disclosed depends on the applicable agreement and organization policy.

The public repository should therefore be treated as an architecture-focused portfolio representation, not as a copy of the original production environment.
