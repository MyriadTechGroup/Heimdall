# HEIMDALL — Development Backlog

## Companion Document to HEIMDALL Requirements V3

Prepared for Myriad Technology Group | 2026-05-19

| Document Field | Value |
| --- | --- |
| Document Type | Development Backlog (companion to V3 Requirements) |
| Scope | MVP (Phase 1) plus selected Phase 2 items where they affect MVP design |
| Format | GitHub-ready epics and issues with acceptance criteria, labels, priority, estimates, and dependencies |
| Target Repositories | `myriad/heimdall-core`, `myriad/heimdall-admin`, `myriad/heimdall-login`, `myriad/heimdall-sdk`, `myriad/heimdall-infra` |
| Estimate Unit | Story points (Fibonacci: 1, 2, 3, 5, 8, 13, 21) |
| Tentative MVP Timeline | 5–7 months for a 4-person backend team plus 1 frontend engineer plus part-time DevOps. |

# 1. Repository Layout

| Repository | Contents |
| --- | --- |
| `myriad/heimdall-core` | HEIMDALL Core TypeScript service: Fastify app, OIDC issuer, Drizzle schema, migrations, services, pg-boss jobs, OpenAPI spec. |
| `myriad/heimdall-admin` | Next.js Admin UI for Myriad and tenant administrators. |
| `myriad/heimdall-login` | Next.js Hosted Login Pages (white-label themed). |
| `myriad/heimdall-sdk` | `@heimdall/client` TypeScript SDK for consuming applications. |
| `myriad/heimdall-infra` | Infrastructure-as-code: Render service definitions, Supabase Postgres config, Cloudflare config, future Terraform for AWS. |

# 2. Labels

| Label | Meaning |
| --- | --- |
| `phase:mvp` | Required for MVP cutover. |
| `phase:2` | Phase 2; may have MVP design hooks. |
| `phase:3` | Phase 3; out of MVP scope. |
| `priority:p0` | Critical path. |
| `priority:p1` | High priority, scheduled in MVP. |
| `priority:p2` | MVP-deferrable. |
| `area:identity` | Identity graph, credentials, personas. |
| `area:auth` | Authentication and federation. |
| `area:mfa` | Multi-factor authentication. |
| `area:oidc` | OIDC issuer implementation. |
| `area:tenant` | Tenant and configuration registry. |
| `area:minors` | Minors, guardians, age policy. |
| `area:secrets` | Secrets and encryption. |
| `area:audit` | Audit ledgers and PII change capture. |
| `area:disclosures` | Disclosures and communications registry. |
| `area:anonymization` | Anonymization workflow. |
| `area:legal-hold` | Legal hold service. |
| `area:support` | Support and impersonation. |
| `area:authz` | Authorization service. |
| `area:platform` | Cross-cutting platform: HTTP, DB, jobs, logging, errors. |
| `area:edge` | Cloudflare integration. |
| `area:sdk` | Frontend SDK. |
| `area:admin-ui` | Admin UI. |
| `area:login-ui` | Hosted Login Pages UI. |
| `area:integration:pedl` | PEDL integration. |
| `area:integration:quest` | Quest integration. |
| `area:integration:myriad-dashboard` | Myriad Dashboard integration. |
| `area:operations` | Deployment, monitoring, runbooks. |
| `area:security-audit` | External security audit prep and remediation. |

# 3. Epic Index

| Epic ID | Title | Phase | Repository |
| --- | --- | --- | --- |
| E-01 | Repository Setup and Monorepo Foundation | MVP | `heimdall-core` |
| E-02 | Database Schema, Drizzle, and Migrations | MVP | `heimdall-core` |
| E-03 | Cloudflare Edge and DNS Setup | MVP | `heimdall-infra` |
| E-04 | HEIMDALL Core Service Scaffold | MVP | `heimdall-core` |
| E-05 | Secrets Module and Envelope Encryption | MVP | `heimdall-core` |
| E-06 | Parent and Credential Identity Services | MVP | `heimdall-core` |
| E-07 | OIDC Issuer Integration | MVP | `heimdall-core` |
| E-08 | Password Authentication Flow | MVP | `heimdall-core` |
| E-09 | Multi-Factor Authentication | MVP | `heimdall-core` |
| E-10 | SMS Authentication via Twilio Verify | MVP | `heimdall-core` |
| E-11 | Magic Link Email Authentication | MVP | `heimdall-core` |
| E-12 | Social Federation: Google, Facebook, Apple | MVP | `heimdall-core` |
| E-13 | Persona Service and Switching | MVP | `heimdall-core` |
| E-14 | Tenant Service and Configuration Registry | MVP | `heimdall-core` |
| E-15 | Authorization Service | MVP | `heimdall-core` |
| E-16 | Disclosures and Communications Registry | MVP | `heimdall-core` |
| E-17 | Minor and Guardian Service | MVP | `heimdall-core` |
| E-18 | Guest Identity Service | MVP | `heimdall-core` |
| E-19 | Audit Ledger and PII Change Capture | MVP | `heimdall-core` |
| E-20 | Anonymization Workflow | MVP | `heimdall-core` |
| E-21 | Legal Hold Service | MVP | `heimdall-core` |
| E-22 | Support and Impersonation Framework | MVP | `heimdall-core` |
| E-23 | Verification Registry and Stripe Identity Integration | MVP | `heimdall-core` |
| E-24 | Webhook Outbox and Delivery | MVP | `heimdall-core` |
| E-25 | HEIMDALL Hosted Login Pages | MVP | `heimdall-login` |
| E-26 | HEIMDALL Admin UI | MVP | `heimdall-admin` |
| E-27 | `@heimdall/client` SDK | MVP | `heimdall-sdk` |
| E-28 | PEDL Integration | MVP | external |
| E-29 | Quest Integration | MVP | external |
| E-30 | Myriad Dashboard Integration | MVP | external |
| E-31 | Observability and Runbooks | MVP | `heimdall-infra` |
| E-32 | External Security Audit and Remediation | MVP | all |
| E-33 | AWS Migration Readiness Validation | Phase 2 | `heimdall-infra` |
| E-34 | Offline Sync Intake | Phase 2 | `heimdall-core` |
| E-35 | Secret Rotation Policies and Workflows | Phase 2 | `heimdall-core` |
| E-36 | Spanish Locale Content and i18n | Phase 2 | all |

# 4. Epic Details

## E-01: Repository Setup and Monorepo Foundation

**Phase:** MVP. **Priority:** P0. **Estimate:** 8.
**Labels:** `area:platform`.
**Dependencies:** None.

**Description:** Establish the `heimdall-core` repository as a Turborepo monorepo using pnpm workspaces. Set up CI/CD, linting, formatting, and conventions.

**Issues:**

- **E-01.1** Initialize Turborepo monorepo with pnpm. `apps/core`, `apps/admin`, `apps/login`, `packages/db`, `packages/sdk`, `packages/openapi`, `packages/scripts`. *(2)*
- **E-01.2** Configure TypeScript strict mode across all packages. Single root `tsconfig.base.json`. *(1)*
- **E-01.3** Configure ESLint plus Prettier plus commit hooks. *(1)*
- **E-01.4** GitHub Actions: `ci` workflow for lint/typecheck/test on every PR. Required check for merge. *(2)*
- **E-01.5** Containerfile (Dockerfile) for `apps/core`. Multi-stage build. Targets `linux/amd64` and `linux/arm64`. *(2)*
- **E-01.6** Docker Compose for local dev: Postgres 15 with `pgcrypto`, HEIMDALL Core in watch mode, mock Twilio and SendGrid via local HTTP stubs. *(2)*
- **E-01.7** `.env.example` documenting all required runtime environment variables. *(1)*

**Acceptance Criteria:**

- `pnpm install` at the root completes cleanly.
- `pnpm dev` brings up Compose stack and runs HEIMDALL Core in watch mode.
- `pnpm lint`, `pnpm typecheck`, and `pnpm test` all green.
- CI passes on a smoke PR.

## E-02: Database Schema, Drizzle, and Migrations

**Phase:** MVP. **Priority:** P0. **Estimate:** 21.
**Labels:** `area:platform`, `area:identity`, `area:audit`, `area:secrets`.
**Dependencies:** E-01.

**Description:** Implement the full Drizzle schema per the V3 Data Model Companion. Generate initial migration. Set up migration runner with advisory lock.

**Issues:**

- **E-02.1** Drizzle schema files per domain, organized in `packages/db/schema/{domain}.ts`. *(13)*
- **E-02.2** `pgcrypto` extension enablement migration. *(1)*
- **E-02.3** Hash-chain triggers for `audit.audit_events`, `audit.pii_change_events`, `audit.communication_log`. *(3)*
- **E-02.4** PII change capture triggers on `parent_identities`, `credential_identities`, `minor_profiles`, `guardian_relationships`, `app_personas`, `verification_records`. Triggers read `current_setting('heimdall.request_context')` for actor and request context. *(3)*
- **E-02.5** Monthly partition strategy for audit tables. pg-boss job to roll partitions forward weekly. *(3)*
- **E-02.6** Seed data migration: applications, default tenants, system roles, system permissions, age policy rules (US-COPPA), rate-limit policy defaults, communication templates (placeholder bodies), disclosure placeholders. *(3)*
- **E-02.7** Migration runner at HEIMDALL Core startup with advisory lock. Refuses to run if schema is ahead of code. *(2)*
- **E-02.8** REVOKE UPDATE/DELETE on audit tables from all roles except `compliance_role`. *(1)*
- **E-02.9** Backup export job: nightly logical backup of HEIMDALL DB to non-Supabase storage (Backblaze B2 or Cloudflare R2). Test restore quarterly. *(3)*

**Acceptance Criteria:**

- Every table in the V3 Data Model exists with documented columns and constraints.
- Hash-chain triggers verified by SQL test: forced row content change without re-computing `row_hash` is detectable.
- PII change triggers verified by SQL test: an UPDATE to a tracked column writes a `pii_change_events` row.
- Audit tables reject UPDATE and DELETE from `service_role`.

## E-03: Cloudflare Edge and DNS Setup

**Phase:** MVP. **Priority:** P0. **Estimate:** 5.
**Labels:** `area:edge`, `area:operations`.
**Dependencies:** None (parallel with E-01).

**Issues:**

- **E-03.1** Cloudflare account configured with Myriad domains. *(1)*
- **E-03.2** Cloudflare WAF Pro enabled. Managed rule sets enabled. Custom rules drafted for HEIMDALL endpoints. *(2)*
- **E-03.3** Cloudflare Rate Limiting configured for `/oidc/auth`, `/oidc/token`, `/v1/auth/*`. *(1)*
- **E-03.4** Cloudflare Turnstile site keys created for production and dev environments. *(1)*
- **E-03.5** Logpush configured: WAF events, security events, rate limit hits sent to Cloudflare R2. *(1)*

**Acceptance Criteria:**

- DNS resolves through Cloudflare for `auth.myriad.com` and `login.myriad.com`.
- WAF rules block a synthetic SQL injection request.
- Rate limit triggers on synthetic burst.

## E-04: HEIMDALL Core Service Scaffold

**Phase:** MVP. **Priority:** P0. **Estimate:** 13.
**Labels:** `area:platform`.
**Dependencies:** E-01, E-02.

**Issues:**

- **E-04.1** Fastify application with OpenAPI spec endpoint. *(2)*
- **E-04.2** Request context middleware: parses bearer token, sets `current_setting('heimdall.request_context')` on each transaction. *(2)*
- **E-04.3** OpenTelemetry SDK integration. Spans for HTTP, DB, Twilio, SendGrid, KMS, pg-boss. *(3)*
- **E-04.4** Structured logging (pino) with correlation ID propagation. *(1)*
- **E-04.5** Centralized error handler with the standard error envelope per API Spec §3. *(1)*
- **E-04.6** Fastify rate-limit plugin backed by `rate_limit_counters` table. *(2)*
- **E-04.7** Health (`/healthz`) and readiness (`/readyz`) endpoints. *(1)*
- **E-04.8** Graceful shutdown: drain HTTP, finish pg-boss jobs in flight, close DB pool. *(1)*

**Acceptance Criteria:**

- Smoke test confirms HTTP 200 from `/healthz`.
- Correlation ID is in every log line.
- Telemetry visible in chosen backend (Grafana Cloud or Datadog).

## E-05: Secrets Module and Envelope Encryption

**Phase:** MVP. **Priority:** P0. **Estimate:** 13.
**Labels:** `area:secrets`, `area:platform`.
**Dependencies:** E-02, E-04.

**Issues:**

- **E-05.1** `SecretsService` class with envelope encryption methods (`encrypt`, `decrypt`, `rotate`, `revoke`). *(3)*
- **E-05.2** Master key loading from `MASTER_KEY` env var at startup. Master key never written to log. *(1)*
- **E-05.3** DEK generation, wrapping, storage in `data_encryption_keys`. *(2)*
- **E-05.4** HMAC helpers for searchable PII (email, phone, social subject) with key versioning. *(2)*
- **E-05.5** PII encryption helpers used by Parent and Credential Identity services. *(2)*
- **E-05.6** In-memory TTL cache (60–300s) for resolved secret values, keyed by `secret_item_id` and version. Cache invalidates on rotation. *(2)*
- **E-05.7** Secret read endpoint (`POST /v1/secrets/{id}/access`) with audit emission. *(2)*
- **E-05.8** Master key rotation script in `packages/scripts/` (RB-01). *(3)*
- **E-05.9** DEK re-wrap pg-boss job. Batches of 500, `FOR UPDATE SKIP LOCKED`. *(3)*

**Acceptance Criteria:**

- Encrypt-then-decrypt round trip works for sample PII values.
- HMAC of `"test@example.com"` is stable across runs and unique across HMAC key versions.
- Master key rotation completes against a seeded test database without data loss.
- Secret access emits `secret.accessed` audit event with requester.

## E-06: Parent and Credential Identity Services

**Phase:** MVP. **Priority:** P0. **Estimate:** 21.
**Labels:** `area:identity`.
**Dependencies:** E-02, E-04, E-05.

**Issues:**

- **E-06.1** `ParentIdentityService` with CRUD: create, update display name and locale, deactivate, anonymize. *(3)*
- **E-06.2** `CredentialIdentityService`: create email/phone/social/webauthn/password credentials, verify, deactivate, set primary. *(5)*
- **E-06.3** HMAC-based lookup for email and phone. E.164 normalization via `libphonenumber-js`. *(2)*
- **E-06.4** Identity linking: explicit linking flow with verification step, 7-day reversal window, audit. *(5)*
- **E-06.5** Admin merge: 30-day reversal window. *(3)*
- **E-06.6** Identity API endpoints per API Spec §6. *(3)*

**Acceptance Criteria:**

- Create a Parent Identity with email credential; round-trip read.
- Add a second credential of a different type to the same parent.
- Link two parents by user action; unlink within window; cannot unlink after window expires.
- All operations write the appropriate `audit_events` and `pii_change_events`.

## E-07: OIDC Issuer Integration

**Phase:** MVP. **Priority:** P0. **Estimate:** 21.
**Labels:** `area:oidc`, `area:auth`.
**Dependencies:** E-06.

**Issues:**

- **E-07.1** Install and configure `oidc-provider`. *(2)*
- **E-07.2** Postgres adapter for `oidc-provider` against `heimdall.oidc_payloads`. *(3)*
- **E-07.3** Client registry backed by `heimdall.idp_clients`. *(3)*
- **E-07.4** Account resolver against `parent_identities` and `credential_identities`. *(2)*
- **E-07.5** Interaction URL routing to Hosted Login Pages. *(2)*
- **E-07.6** Custom claims emission (parent_id, persona_id, tenant_id, mfa_level, mfa_methods, auth_time). *(2)*
- **E-07.7** PKCE required for all clients. *(1)*
- **E-07.8** Refresh token rotation enabled. *(1)*
- **E-07.9** Token Exchange (RFC 8693) grant type for impersonation with `act` claim. *(3)*
- **E-07.10** JWKS rotation: signing key rotation procedure with 48-hour overlap. *(2)*

**Acceptance Criteria:**

- Authorization code flow completes end-to-end via the Hosted Login Pages.
- ID token validates against discovery document and JWKS.
- Token exchange with `heimdall_impersonation_approval_id` returns a JWT carrying `act` claim.

## E-08: Password Authentication Flow

**Phase:** MVP. **Priority:** P0. **Estimate:** 8.
**Labels:** `area:auth`.
**Dependencies:** E-06, E-07.

**Issues:**

- **E-08.1** Argon2id password hashing via `@node-rs/argon2` with configured cost parameters. *(2)*
- **E-08.2** Login endpoint `POST /v1/auth/login/password`. Returns continuation token, prompts MFA if enrolled. *(3)*
- **E-08.3** Password change `POST /v1/auth/password/change` with MFA step-up. *(2)*
- **E-08.4** Password reset start and complete endpoints. No enumeration. *(3)*
- **E-08.5** Password policy enforcement: minimum length 12, common-password reject list, breached-password reject (HIBP K-Anonymity API). *(3)*
- **E-08.6** Account lockout: 5 failed attempts in 15 minutes triggers 15-minute lockout. Notification to user. *(2)*

**Acceptance Criteria:**

- Login with correct password succeeds.
- Login with incorrect password returns generic error; lockout after 5.
- Password reset never reveals whether email exists.
- Common-password reject prevents `password123`.

## E-09: Multi-Factor Authentication

**Phase:** MVP. **Priority:** P0. **Estimate:** 21.
**Labels:** `area:mfa`.
**Dependencies:** E-06, E-08.

**Issues:**

- **E-09.1** TOTP via `otplib`. Enrollment generates `otpauth://` URI plus QR code data URL. Verify, store secret encrypted. *(3)*
- **E-09.2** WebAuthn via `@simplewebauthn/server`. Registration challenge and verification, authentication challenge and verification. *(5)*
- **E-09.3** SMS OTP as second factor (delegates to Twilio Verify in E-10). *(2)*
- **E-09.4** Email OTP as second factor (delegates to SendGrid in E-11). *(2)*
- **E-09.5** Backup codes: 10 single-use codes, hashed with Argon2id. Returned once on enrollment. *(2)*
- **E-09.6** MFA challenge dispatch and verify endpoints. *(2)*
- **E-09.7** Enrollment management UI endpoints: list methods, name a method, revoke a method. *(2)*
- **E-09.8** Step-up gate middleware: `requireMfaStepUp({ freshnessSeconds })`. *(2)*
- **E-09.9** Trusted device 2FA registration (Phase 2 marker; MVP stores intent but does not skip MFA). *(1)*

**Acceptance Criteria:**

- Enroll TOTP, log out, log in, prompted for TOTP, succeed.
- Enroll WebAuthn passkey, log in via WebAuthn.
- Use a backup code; second use of the same code fails.
- Step-up gate rejects requests when MFA freshness expired.

## E-10: SMS Authentication via Twilio Verify

**Phase:** MVP. **Priority:** P0. **Estimate:** 8.
**Labels:** `area:auth`, `area:mfa`.
**Dependencies:** E-05, E-06, E-09.

**Issues:**

- **E-10.1** `SmsProvider` interface with Twilio Verify implementation. Credentials in `secrets`. *(3)*
- **E-10.2** SMS passwordless login flow: start (sends OTP), verify (returns continuation token). *(2)*
- **E-10.3** SMS as second factor (registered MFA method). *(2)*
- **E-10.4** Rate-limit enforcement per `rate_limit_policies`: per-phone, per-IP, per-account, per-country. *(3)*
- **E-10.5** Daily SMS cost-cap tracking: global, per-country. Alerts at 50, 75, 90, 100%. Auto-halt at 100%. *(3)*
- **E-10.6** Twilio webhook receiver for delivery status; updates `communication_log`. *(2)*
- **E-10.7** Voice OTP fallback marker (Phase 2). *(1)*

**Acceptance Criteria:**

- Send OTP to a Twilio test number; verify successfully.
- 6th request within 1 hour for the same phone is rejected with 429.
- Synthetic spike: cost-cap alert fires at the configured thresholds.

## E-11: Magic Link Email Authentication

**Phase:** MVP. **Priority:** P0. **Estimate:** 5.
**Labels:** `area:auth`.
**Dependencies:** E-05, E-06.

**Issues:**

- **E-11.1** `EmailProvider` interface with SendGrid implementation. *(2)*
- **E-11.2** Magic link start: generate single-use token, 15-minute expiry, store hash. *(2)*
- **E-11.3** Magic link consume: validate token, mark consumed, advance auth flow. *(2)*
- **E-11.4** Per-tenant From address per `tenant_branding.email_from_address` or default `noreply@myriad.com`. *(1)*
- **E-11.5** SendGrid webhook receiver: delivery, open, click, bounce, failure events written to `communication_log`. *(2)*

**Acceptance Criteria:**

- Request magic link; email received in test inbox.
- Consume link; logged in successfully.
- Replay of same link fails with 410.

## E-12: Social Federation: Google, Facebook, Apple

**Phase:** MVP. **Priority:** P0. **Estimate:** 13.
**Labels:** `area:auth`.
**Dependencies:** E-06, E-07.

**Issues:**

- **E-12.1** Google OIDC federation via `openid-client`. *(3)*
- **E-12.2** Facebook OAuth2 federation (Graph API user-info). *(3)*
- **E-12.3** Apple Sign-In integration. Web flow (form_post response_mode) and native iOS flow. *(5)*
- **E-12.4** Per-tenant provider enablement via `social_providers_policy`. *(2)*
- **E-12.5** Per-tenant client credentials (some tenants run their own OAuth apps) via `social_provider_credentials_policy`. *(2)*
- **E-12.6** White-label cross-signup detection: when an existing email at another tenant is detected, offer the user explicit opt-in to link or create separate account. *(3)*
- **E-12.7** Federated identity link storage in `federated_identity_links`. *(1)*

**Acceptance Criteria:**

- Sign in with Google in a test tenant; account created with `federated_identity_links` row.
- Sign in with Apple in iOS test environment.
- Cross-signup flow shows the explicit linking offer when conditions match.

## E-13: Persona Service and Switching

**Phase:** MVP. **Priority:** P1. **Estimate:** 8.
**Labels:** `area:identity`.
**Dependencies:** E-06.

**Issues:**

- **E-13.1** `PersonaService` with create, deactivate, list. *(3)*
- **E-13.2** Active persona on session record; switch endpoint refreshes JWT. *(3)*
- **E-13.3** Persona switch audit and `persona_switch_events` write. *(1)*
- **E-13.4** Persona-scoped role assignment endpoints. *(2)*

**Acceptance Criteria:**

- A parent has two personas (PEDL rider and PEDL driver); switching changes the active `persona_id` in the JWT.

## E-14: Tenant Service and Configuration Registry

**Phase:** MVP. **Priority:** P0. **Estimate:** 13.
**Labels:** `area:tenant`.
**Dependencies:** E-02, E-04.

**Issues:**

- **E-14.1** Tenant CRUD endpoints. *(3)*
- **E-14.2** Tenant branding endpoints. *(2)*
- **E-14.3** Tenant domain verification: DNS TXT challenge, DKIM/SPF check. *(3)*
- **E-14.4** Tenant policy versioning per category (10 categories from Requirements §10). *(5)*
- **E-14.5** Policy diff endpoint. *(2)*
- **E-14.6** Per-tenant overrides applied at runtime: `auth_methods_policy`, `mfa_policy`, etc. evaluated in auth and access decisions. *(3)*

**Acceptance Criteria:**

- Create a tenant, configure branding, verify a domain.
- Author a new MFA policy version, activate it; new logins to that tenant honor the policy.

## E-15: Authorization Service

**Phase:** MVP. **Priority:** P0. **Estimate:** 13.
**Labels:** `area:authz`.
**Dependencies:** E-02, E-13, E-14.

**Issues:**

- **E-15.1** `AuthorizationService.check(actor, action, resource, context)` returning `{ allowed, reasons }`. *(3)*
- **E-15.2** Tier 1 fast path: indexed SQL lookup against `persona_roles` and `role_permissions`. *(2)*
- **E-15.3** Tier 2 predicates: TypeScript context-aware predicates. Built-in predicates for minor restrictions, legal hold, MFA freshness, geolocation, time-of-day. *(5)*
- **E-15.4** In-memory decision cache (30–60s TTL) keyed by `(parent_id, persona_id, app, action, context_hash)`. *(2)*
- **E-15.5** `POST /v1/authorize/check` endpoint for frontend gating. *(1)*
- **E-15.6** Audit event on every deny and every elevated allow. *(1)*

**Acceptance Criteria:**

- Round-trip authorization check with a known role returns expected allow/deny.
- Legal hold blocks all actions for a held identity.
- Minor restrictions block age-gated features.

## E-16: Disclosures and Communications Registry

**Phase:** MVP. **Priority:** P0. **Estimate:** 21.
**Labels:** `area:disclosures`.
**Dependencies:** E-02, E-04, E-14.

**Issues:**

- **E-16.1** Disclosure CRUD: disclosures, versions, publish, supersede. *(5)*
- **E-16.2** Disclosure acceptance recording with full request context. *(2)*
- **E-16.3** Required-re-acceptance computation: query active versions vs user's acceptances. *(2)*
- **E-16.4** Communication template CRUD. *(3)*
- **E-16.5** Communication dispatch service: renders template with locale, calls SmsProvider or EmailProvider, writes `communication_log`. *(5)*
- **E-16.6** Webhook ingestion: SendGrid and Twilio events update `communication_log`. *(3)*
- **E-16.7** Per-tenant From address resolution. *(1)*

**Acceptance Criteria:**

- Publish a TOS v1 disclosure; signup captures acceptance.
- Publish v2 with `requires_re_acceptance=true`; next login prompts user to re-accept.
- Send a templated communication; record reaches delivered status from SendGrid webhook.

## E-17: Minor and Guardian Service

**Phase:** MVP. **Priority:** P0. **Estimate:** 13.
**Labels:** `area:minors`.
**Dependencies:** E-06, E-14, E-15, E-16.

**Issues:**

- **E-17.1** `MinorProfileService` with create, age attestation, jurisdiction tagging. *(3)*
- **E-17.2** Age gate enforcement using `age_policy_rules` and `application_minor_features`. *(3)*
- **E-17.3** Guardian invite token generation, guardian acceptance, `guardian_relationships` row creation. *(3)*
- **E-17.4** Guardian consent grant (records `disclosure_acceptances` on behalf of minor). *(2)*
- **E-17.5** Age transition pg-boss job: detects DOB-based transitions, sets `age_transitioned_at`, marks guardian relationships `lapsed_age_transition`, prompts re-acceptance on next login. *(3)*
- **E-17.6** COPPA US-under-13 default seed in `age_policy_rules`. *(1)*

**Acceptance Criteria:**

- Register a US user attesting age 10. Account flagged minor, requires guardian.
- Guardian completes invite, consent granted.
- Age policy can be overridden per app via configuration.

## E-18: Guest Identity Service

**Phase:** MVP. **Priority:** P1. **Estimate:** 8.
**Labels:** `area:identity`.
**Dependencies:** E-06.

**Issues:**

- **E-18.1** Create device-bound guest identity per app. *(2)*
- **E-18.2** Guest persistence: configurable expiry per app. *(1)*
- **E-18.3** Claim/upgrade flow: anonymous guest becomes authenticated parent; carry-forward of preserved state. *(3)*
- **E-18.4** Per-app preserved data summary (what carries over from guest to authenticated). *(2)*

**Acceptance Criteria:**

- Create a guest, then sign up with email; guest claimed under the new parent.
- Guest data carries forward per app preservation policy.

## E-19: Audit Ledger and PII Change Capture

**Phase:** MVP. **Priority:** P0. **Estimate:** 13.
**Labels:** `area:audit`.
**Dependencies:** E-02, E-04.

**Issues:**

- **E-19.1** `AuditService.record(event)` API used by every other service. *(3)*
- **E-19.2** Hash-chain verification job (RB-14). *(3)*
- **E-19.3** Audit query endpoints (API Spec §17). *(3)*
- **E-19.4** PII change log query endpoints. *(2)*
- **E-19.5** Audit export pg-boss job: chunked JSONL or CSV export, signed download URL, 15-min expiry. *(3)*
- **E-19.6** Cold archive job: detach partitions older than 24 months, upload to Backblaze B2 / Cloudflare R2 with immutability. *(5)*

**Acceptance Criteria:**

- Every state-changing operation generates audit events.
- PII change triggers fire on every PII update.
- Hash-chain verification job reports clean baseline.

## E-20: Anonymization Workflow

**Phase:** MVP. **Priority:** P0. **Estimate:** 21.
**Labels:** `area:anonymization`.
**Dependencies:** E-06, E-15, E-16, E-19, E-21.

**Issues:**

- **E-20.1** `AnonymizationRequestService`: create, cancel, status. Three scopes (`full_account`, `per_app`, `per_persona`). *(5)*
- **E-20.2** 30-day notification scheduler in pg-boss: initial, day-15, final reminder. *(3)*
- **E-20.3** Anonymization execution job. Per-table effects per Requirements §16.3. *(8)*
- **E-20.4** Returning-user policy: fresh parent_id on signup with previously anonymized email/phone HMAC. *(2)*
- **E-20.5** Admin-initiated and legal-initiated request paths. *(2)*
- **E-20.6** Execute-now bypass (legal reference required). *(2)*
- **E-20.7** Manual restoration path with dual-control. *(3)*

**Acceptance Criteria:**

- User requests full account anonymization; receives 4 expected emails over 30 days; execution tombstones the parent and credentials.
- Per-app scope leaves other personas untouched.
- Returning user gets a fresh parent_id; no auto-link.

## E-21: Legal Hold Service

**Phase:** MVP. **Priority:** P0. **Estimate:** 8.
**Labels:** `area:legal-hold`.
**Dependencies:** E-06, E-15, E-19.

**Issues:**

- **E-21.1** Legal hold placement and release endpoints. *(2)*
- **E-21.2** Active hold gate in login flow: blocks logins, returns generic message. *(2)*
- **E-21.3** Active hold gate in authorization service: blocks credential changes, merges, anonymization execution. *(2)*
- **E-21.4** Dual-control override request and co-sign endpoints. 60-minute window. *(3)*
- **E-21.5** `legal_hold_blocked_actions` capture on every blocked attempt. *(1)*

**Acceptance Criteria:**

- Place a hold; affected user cannot log in; sees generic message.
- Override request without co-sign expires after 60 minutes.
- Co-sign by a different super admin executes the override.

## E-22: Support and Impersonation Framework

**Phase:** MVP. **Priority:** P0. **Estimate:** 13.
**Labels:** `area:support`.
**Dependencies:** E-07, E-13, E-15, E-19.

**Issues:**

- **E-22.1** Impersonation approval request and approve/deny endpoints. *(3)*
- **E-22.2** Slack and email notification to available approvers on pending request. *(2)*
- **E-22.3** Approval window 30 min; token issuance via OIDC token exchange. *(2)*
- **E-22.4** 15-min token, 60-min ceiling. Renew endpoint. *(2)*
- **E-22.5** Support session event capture: every action records `support_session_events`. *(2)*
- **E-22.6** Exit session endpoint: immediate revocation. *(1)*
- **E-22.7** Approval audit report endpoint (RB-06 support). *(2)*

**Acceptance Criteria:**

- L1 requests impersonation; L2 receives notification.
- L1 cannot start session without L2 approval.
- Started session token carries `act` claim.
- Exit immediately invalidates the token; subsequent calls 401.

## E-23: Verification Registry and Stripe Identity Integration

**Phase:** MVP. **Priority:** P1. **Estimate:** 8.
**Labels:** `area:identity`.
**Dependencies:** E-02, E-05, E-06.

**Issues:**

- **E-23.1** `IdvProvider` interface; Stripe Identity implementation. *(3)*
- **E-23.2** Verification session creation; provider redirect URL. *(2)*
- **E-23.3** Webhook ingestion: Stripe Identity verification results into `verification_records`. *(2)*
- **E-23.4** Verification record retrieval API. *(1)*
- **E-23.5** Re-verification trigger on document expiry. *(2)*

**Acceptance Criteria:**

- Initiate verification; complete in Stripe Identity sandbox; result stored as claims-only.
- No raw images persisted in HEIMDALL.

## E-24: Webhook Outbox and Delivery

**Phase:** MVP. **Priority:** P1. **Estimate:** 8.
**Labels:** `area:platform`.
**Dependencies:** E-02, E-04, E-05.

**Issues:**

- **E-24.1** Webhook endpoint registration with signing secret one-time return. *(2)*
- **E-24.2** Event outbox table; pg-boss delivery job with exponential backoff. *(3)*
- **E-24.3** HMAC signature header. Verification reference example in docs. *(2)*
- **E-24.4** Delivery history endpoint. Manual replay. *(2)*

**Acceptance Criteria:**

- Subscribed endpoint receives signed event on `identity.created`.
- Failed delivery retries on schedule; manual replay works.

## E-25: HEIMDALL Hosted Login Pages

**Phase:** MVP. **Priority:** P0. **Estimate:** 21.
**Labels:** `area:login-ui`.
**Dependencies:** E-07, E-08, E-09, E-10, E-11, E-12, E-14, E-16.

**Issues:**

- **E-25.1** Next.js scaffold for `apps/login`. Tenant-aware theming. *(3)*
- **E-25.2** Identification step (email or phone). *(2)*
- **E-25.3** Password step. *(2)*
- **E-25.4** MFA challenge UI (TOTP, SMS, email_otp, WebAuthn, backup code). *(3)*
- **E-25.5** Social provider buttons (Google, Facebook, Apple) with per-tenant enablement. *(3)*
- **E-25.6** Magic link landing page. *(1)*
- **E-25.7** Registration UI with disclosure acceptance blocking. *(3)*
- **E-25.8** Disclosure re-acceptance modal. *(2)*
- **E-25.9** Locale switching scaffolding (EN only at MVP). *(1)*
- **E-25.10** Tenant theming: logo, colors, custom CSS load per `tenant_branding`. *(2)*
- **E-25.11** Turnstile CAPTCHA on registration and high-risk login. *(1)*

**Acceptance Criteria:**

- Complete a full login flow in the white-label hosted UI.
- Tenant theming applies per query string `tenant_slug` or per host.
- Disclosure re-acceptance flow works end-to-end.

## E-26: HEIMDALL Admin UI

**Phase:** MVP. **Priority:** P0. **Estimate:** 21.
**Labels:** `area:admin-ui`.
**Dependencies:** All HEIMDALL Core service endpoints.

**Issues:**

- **E-26.1** Next.js scaffold for `apps/admin`. *(2)*
- **E-26.2** OIDC client config in Admin UI; uses HEIMDALL Core as IdP (dogfood). *(2)*
- **E-26.3** Identity browser: search users, view details (with PII view audit). *(3)*
- **E-26.4** Credential management UI: revoke, set primary. *(2)*
- **E-26.5** Tenant management UI: create, branding, domains, policies with version diff and activation. *(5)*
- **E-26.6** Role and permission editor. *(3)*
- **E-26.7** Disclosure authoring and publication UI with diff view. *(3)*
- **E-26.8** Anonymization request management. *(3)*
- **E-26.9** Legal hold management UI with dual-control. *(3)*
- **E-26.10** Impersonation approval queue: pending list, approve, deny, audit. *(3)*
- **E-26.11** Audit query UI with filter and export. *(3)*
- **E-26.12** Secrets management UI (metadata view, rotation, access). *(3)*

**Acceptance Criteria:**

- Admin can complete every workflow described in RB-04, RB-05, RB-06, RB-15 via the UI.
- All PII views generate `pii_change_events` rows.

## E-27: `@heimdall/client` SDK

**Phase:** MVP. **Priority:** P0. **Estimate:** 13.
**Labels:** `area:sdk`.
**Dependencies:** E-07.

**Issues:**

- **E-27.1** TypeScript SDK package with build to ESM + CJS. *(2)*
- **E-27.2** `HeimdallProvider` React context. *(2)*
- **E-27.3** `useIdentity`, `usePersona`, `useTenant`, `useAuthorize` hooks. *(3)*
- **E-27.4** `ImpersonationBanner` React component. Detects `act` claim, renders red fixed banner. *(2)*
- **E-27.5** `ConsentReAcceptanceModal` component. *(2)*
- **E-27.6** `heimdallClient.fetch` authenticated wrapper with 401 step-up, 429 retry, idempotency-key, correlation-id. *(3)*
- **E-27.7** `heimdallClient.logout()`. *(1)*
- **E-27.8** Integration tests against a running HEIMDALL Core. *(2)*
- **E-27.9** CI check to verify Banner + Modal are mounted in `apps/admin` and `apps/login`. *(2)*

**Acceptance Criteria:**

- SDK published to private npm registry, versioned.
- Banner renders when `act` claim present; cannot be dismissed.
- Modal blocks app interaction when re-acceptance required.

## E-28: PEDL Integration

**Phase:** MVP. **Priority:** P0. **Estimate:** 13.
**Labels:** `area:integration:pedl`.
**Dependencies:** E-27.

**Issues:**

- **E-28.1** Configure PEDL OIDC client in HEIMDALL. *(1)*
- **E-28.2** Integrate `@heimdall/client` in PEDL web frontend. *(3)*
- **E-28.3** Integrate `@heimdall/client` in PEDL iOS app (native flow via ASWebAuthenticationSession). *(3)*
- **E-28.4** Integrate `@heimdall/client` in PEDL Android app (Custom Tabs). *(3)*
- **E-28.5** Mount `ImpersonationBanner` and `ConsentReAcceptanceModal` in PEDL web. *(1)*
- **E-28.6** Migrate PEDL existing users (if any in legacy auth) into HEIMDALL parent identities. *(5)*
- **E-28.7** PEDL persona switching UI (rider/driver). *(3)*

**Acceptance Criteria:**

- New PEDL user signs up via HEIMDALL hosted login.
- Existing PEDL user logs in.
- Impersonation banner renders correctly when L2 support is viewing-as.

## E-29: Quest Integration

**Phase:** MVP. **Priority:** P0. **Estimate:** 13.
**Labels:** `area:integration:quest`.
**Dependencies:** E-27.

Same structure as E-28 for Quest (attendee, organizer, admin personas). White-label tenants get tenant-themed login.

## E-30: Myriad Dashboard Integration

**Phase:** MVP. **Priority:** P0. **Estimate:** 8.
**Labels:** `area:integration:myriad-dashboard`.
**Dependencies:** E-27.

Same structure for Myriad Core Dashboard. Internal-only realm; hardware MFA required for admin roles.

## E-31: Observability and Runbooks

**Phase:** MVP. **Priority:** P0. **Estimate:** 8.
**Labels:** `area:operations`.
**Dependencies:** E-04.

**Issues:**

- **E-31.1** Grafana Cloud or Datadog setup; standard dashboards. *(3)*
- **E-31.2** Alert rules: SEV-1, SEV-2, SEV-3 thresholds. *(3)*
- **E-31.3** PagerDuty integration. *(2)*
- **E-31.4** Runbook documents copied into ops wiki and linked from each alert. *(2)*
- **E-31.5** Drill schedule established (RB-23). *(1)*

**Acceptance Criteria:**

- Synthetic SEV-1 page reaches on-call.
- Dashboards show baseline metrics.

## E-32: External Security Audit and Remediation

**Phase:** MVP. **Priority:** P0. **Estimate:** 21.
**Labels:** `area:security-audit`.
**Dependencies:** All MVP scope complete.

**Issues:**

- **E-32.1** Select external auditor specializing in OIDC, OAuth, session management, MFA. Budget allocated. *(2)*
- **E-32.2** Pre-audit hardening checklist completed internally. *(3)*
- **E-32.3** Audit kickoff: scope, timeline, access provisioning. *(2)*
- **E-32.4** Findings triage. *(3)*
- **E-32.5** Remediation P0 and P1 findings before production launch. *(13)*
- **E-32.6** Penetration test re-run after remediations. *(3)*
- **E-32.7** Final report archived in compliance store. *(1)*

**Acceptance Criteria:**

- Final report shows no open P0 or P1 findings before production launch.

## E-33: AWS Migration Readiness Validation (Phase 2)

**Phase:** Phase 2. **Priority:** P1. **Estimate:** 21.
**Labels:** `area:operations`.

Validate every Phase 2 design assumption with a staging-environment dry run of RB-12. Documented in detail in the Runbooks companion.

## E-34: Offline Sync Intake (Phase 2)

**Phase:** Phase 2. **Priority:** P2. **Estimate:** 13.
**Labels:** `area:identity`.

Build the offline event intake, idempotency by `client_event_id`, Last-Write-Wins conflict resolution with `offline_conflict_records` capture.

## E-35: Secret Rotation Policies and Workflows (Phase 2)

**Phase:** Phase 2. **Priority:** P1. **Estimate:** 13.
**Labels:** `area:secrets`.

Scheduled rotation, approval workflow, expiration alerts, usage dashboards.

## E-36: Spanish Locale Content and i18n (Phase 2)

**Phase:** Phase 2. **Priority:** P1. **Estimate:** 13.
**Labels:** `area:platform`.

Add Spanish translations for all disclosures, communication templates, and Hosted Login Pages UI strings. Validate locale resolution from Accept-Language header and user preference.

# 5. MVP Critical Path

The dependency chain that determines MVP timeline:

```
E-01 → E-02 → E-04 → E-05 → E-06 → E-07
                                    ↓
                                    E-08, E-09, E-10, E-11, E-12 (parallel)
                                    ↓
                                    E-13, E-14, E-15
                                    ↓
                                    E-16, E-17, E-18
                                    ↓
                                    E-19 (parallel from E-02)
                                    ↓
                                    E-20, E-21, E-22, E-23, E-24
                                    ↓
                                    E-25 (login UI), E-26 (admin UI), E-27 (SDK)
                                    ↓
                                    E-28 (PEDL), E-29 (Quest), E-30 (Myriad) (parallel)
                                    ↓
                                    E-31 (ops), E-32 (audit + remediation)
                                    ↓
                                    PRODUCTION CUTOVER
```

Parallel tracks reduce wall-clock time: while backend builds the OIDC and auth services, frontend can build the Login UI and SDK against mock APIs informed by the OpenAPI spec.

# 6. Estimate Summary

| Phase | Total Story Points |
| --- | --- |
| E-01 to E-04 (Foundation) | 47 |
| E-05 to E-07 (Crypto + Identity + OIDC) | 55 |
| E-08 to E-12 (Auth flows + MFA + Federation) | 55 |
| E-13 to E-15 (Persona, Tenant, Authz) | 34 |
| E-16 to E-18 (Disclosures, Minors, Guests) | 42 |
| E-19 to E-24 (Audit, Anon, Legal, Support, Verify, Webhooks) | 71 |
| E-25 to E-27 (UIs and SDK) | 55 |
| E-28 to E-30 (App integrations) | 34 |
| E-31 to E-32 (Ops + Security Audit) | 29 |
| **MVP Total** | **422** |

At a sustainable team velocity of 60–80 points per two-week sprint (4 backend engineers + 1 frontend + part-time DevOps), MVP runs 11–14 sprints, i.e., 5.5–7 months wall-clock with no major surprises. External security audit and remediation (E-32) overlaps with PEDL/Quest integration polish in the final 4–6 weeks.

# 7. Acceptance Criteria for Production Readiness

The following gating checklist must be green before production cutover:

| Item | Owner | Done When |
| --- | --- | --- |
| All MVP epics complete | Engineering | Every epic's acceptance criteria pass. |
| External security audit | Security Lead + auditor | No open P0 or P1 findings. |
| Runbook drills | CTO | RB-01, RB-02, RB-03 rehearsed at least once in staging. |
| Master key escrow established | CEO | Five Shamir holders confirmed with shares distributed. |
| Backup and restore tested | Engineering | Nightly backup tested by restoring into a staging DB; smoke test passes. |
| Hash-chain integrity baseline | Security Auditor | RB-14 verification clean. |
| Cloudflare WAF rules tuned | Engineering | No false positives on legitimate traffic; synthetic attack blocked. |
| Provider failover tested | Engineering | RB-09, RB-10 tabletop completed. |
| Tenant onboarding runbook | Operations | Three tenants successfully onboarded in staging end-to-end. |
| User communication templates ready | Legal + Communications | Incident comm templates approved by Legal. |
| On-call rotation established | Engineering Lead | PagerDuty rotation populated; primary and secondary on every shift. |
| Documentation handoff | Engineering Lead | This document and the V3 Requirements, Data Model, API Spec, and Runbooks are reviewed and signed off. |

# 8. Out of Scope (Explicit)

The following are deliberately excluded from the MVP backlog and tracked separately:

| Excluded | Tracked As | Why |
| --- | --- | --- |
| Auth0 federation | Phase 3 backlog | No customer need yet. |
| Okta enterprise SSO | Phase 3 backlog | No enterprise tenant signed. |
| Microsoft Entra ID federation | Phase 3 backlog | No enterprise tenant signed. |
| SAML enterprise federation | Phase 3 backlog | No enterprise tenant signed. |
| CJIS-bound features | Phase 3 backlog | No signed law enforcement need. |
| HIPAA-bound features | Phase 3 backlog | No signed healthcare need. |
| Dynamic database credentials | Phase 3 backlog | Requires OpenBao or AWS Secrets Manager. |
| Risk-based authentication scoring | Phase 3 backlog | Requires telemetry baseline. |
| Service mesh / SPIFFE / SPIRE | Phase 3 backlog | Premature for current scale. |
| NATS event bus | Phase 3 backlog | Postgres LISTEN/NOTIFY plus outbox sufficient for current scale. |
| Mobile push notification platform | App-team backlogs | Each consuming app manages push directly. |

# 9. Risk Register Summary (Cross-Reference)

Risks listed in detail in V3 Requirements §25. The backlog handles them as follows:

| Risk | Mitigation in Backlog |
| --- | --- |
| Custom OIDC issuer security commitment | E-07 uses certified `oidc-provider`; E-32 external audit. |
| Supabase vendor concentration | E-02.9 nightly logical backups; design uses Postgres-standard SQL. |
| Master key loss | E-05.8 rotation script; runbook RB-02 with rehearsal cadence in E-31. |
| SMS pumping fraud | E-10.4–E-10.5 caps and alerts. |
| Identity merge mistakes | E-06.4–E-06.5 reversal windows. |
| Minor consent gaps | E-17 full minor service from MVP. |
| White-label identity confusion | E-12.6 explicit opt-in. |
| AWS migration friction | E-02, E-04 portability; E-33 staging validation in Phase 2. |

HEIMDALL — Development Backlog Companion | Myriad Technology Group
