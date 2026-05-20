# HEIMDALL

## Final Requirements and Design Document — Version 3

*TypeScript-Native Federated Identity, Trust, Access, and Governance Platform*

Prepared for Myriad Technology Group | 2026-05-19

| Document Field | Value |
| --- | --- |
| System Name | HEIMDALL |
| Document Version | V3 (supersedes V2 baseline of 2026-05-19) |
| Purpose | Federated identity, trust, access, secrets, consent, and governance platform for Myriad-owned applications and services. |
| Architecture Posture | Holistic internal platform owned end-to-end by Myriad. Single-language (TypeScript), single-runtime (Node.js LTS), minimum external dependencies. |
| Initial Deployment Model | Vercel (frontends) + Render (HEIMDALL Core container) + Supabase Postgres (database hosting, Postgres-only). Cloudflare in front of all traffic. |
| Future Deployment Model | AWS-ready migration path using RDS/Aurora PostgreSQL, ECS Fargate, KMS, Secrets Manager, EventBridge/SQS, CloudWatch, and S3 Object Lock. |
| Primary Users | Myriad admins, client/tenant operators, client employees and contractors, field users, end customers, minors and guardians, service accounts, and application services. |
| Status | Final design baseline for implementation planning. |
| Companion Documents | Data Model and Schema; API Specification; Security and Operations Runbooks; Development Backlog. |

**North Star Rule:** HEIMDALL authenticates. HEIMDALL decides. HEIMDALL protects. HEIMDALL remembers. Applications consume.

# 1. Executive Summary

HEIMDALL is the Myriad-wide internal platform for federated identity, user and persona relationships, tenant access, secrets governance, consent and disclosure tracking, and application trust. It is not intended to be sold as a standalone product in its first iteration. It supports Myriad-owned products including PEDL, Quest, the Myriad core dashboard, client-facing dashboards, mobile field apps, mobile web apps, and customer-facing native iOS and Android apps.

This Version 3 baseline reflects a deliberate architectural shift toward minimum viable external dependencies. Where Version 1 anticipated Keycloak as an authentication broker, an OPA policy engine, an OpenBao secrets layer, and a NATS event bus, this baseline collapses those external services into a single TypeScript service: HEIMDALL Core. The motivation is operational simplicity, language uniformity with downstream Next.js applications, ownership of every domain by Myriad engineers, and minimum security audit surface area. Where well-maintained TypeScript libraries can deliver the underlying capability, HEIMDALL Core integrates them directly rather than running them as separate services.

The recommended final architecture uses a custom OpenID Connect issuer built into HEIMDALL Core using the `oidc-provider` library (Filip Skokan, OpenID Foundation certified), PostgreSQL as the system of record (initially Supabase-hosted, AWS-ready), envelope encryption via `pgcrypto` for PII and secrets at rest, and an in-process authorization service for access decisions. SMS authentication uses Twilio Verify in Phase 1, with AWS SNS planned for the AWS migration. Cloudflare provides the edge layer for DNS, WAF, rate limiting, and CAPTCHA via Turnstile.

| Decision Area | Final Position |
| --- | --- |
| Product Scope | Internal Myriad platform service for Myriad-owned apps and services. Not a standalone commercial product initially. |
| Identity Source of Truth | HEIMDALL Identity Graph in PostgreSQL is authoritative. No second store of identity exists. |
| Authentication Broker | Custom OIDC issuer in HEIMDALL Core using `oidc-provider`. No Keycloak. Ory Kratos and Hydra remain documented fallback architectures. |
| Authorization Engine | In-process `AuthorizationService` in HEIMDALL Core. No OPA. Table-driven roles and permissions. |
| Secrets at Rest | `pgcrypto` envelope encryption in PostgreSQL plus a custom `SecretsService` module. No OpenBao for MVP. OpenBao remains a documented fallback. |
| Event Distribution | PostgreSQL `LISTEN/NOTIFY` plus outbox pattern plus Server-Sent Events from HEIMDALL Core. No NATS until Phase 3. |
| Primary Data Store | PostgreSQL, initially hosted on Supabase using Postgres-standard SQL only. No Supabase-proprietary features. AWS RDS/Aurora ready. |
| Frontend Hosting | Vercel (Next.js apps including HEIMDALL Admin UI, hosted login pages, and product frontends). |
| Backend Hosting | Render (HEIMDALL Core container) in Phase 1. AWS ECS Fargate in Phase 2. |
| Edge | Cloudflare DNS, Cloudflare WAF Pro, Cloudflare Turnstile, Cloudflare Rate Limiting. |
| Language and Runtime | TypeScript on Node.js LTS. Single language across backend and frontends. |
| Child and Minor Support | Immediate requirement. COPPA-anchored for MVP with jurisdiction-flexible schema for AB 2273, GDPR-K, and other regimes. |
| Compliance Posture | GDPR-style anonymization (not deletion) for data subject rights. PCI scope minimization by design. CJIS path preserved but deferred. |
| Native Mobile | Apple Sign-In included at MVP because PEDL and Quest both ship native iOS apps. |

# 2. Goals and Non-Goals

## 2.1 Goals

Provide a single centralized identity platform across all Myriad-owned apps and services, with HEIMDALL Core as the only authority for authentication, identity, authorization, secrets metadata, and audit.

Support Parent UUIDs, application and persona UUIDs, guest identities, minor and guardian identities, and intentional unlinked accounts.

Support email and password, multiple emails, phone credentials, multiple phone numbers, Google social login, Facebook social login, Apple Sign-In for native iOS, SMS one-time-password authentication (as second factor and as primary credential), TOTP authenticator-app two-factor authentication, WebAuthn passkeys, and magic-link authentication.

Support tenant-configurable authentication policies, including which credential types are permitted, which social providers are enabled, what two-factor methods are required for which roles, and what age gates and disclosure requirements apply.

Support white-label login experiences where the user may not initially know the account is related to other Myriad apps, with an explicit opt-in cross-tenant linking flow when an existing account is detected by email.

Support tenant and client isolation, app-specific roles, table-driven permissions, verification records, and account linking and unlinking by explicit user, support, or admin action.

Maintain a complete versioned disclosure and consent registry that tracks every Terms of Service, Privacy Policy, COPPA notice, SMS opt-in, and marketing consent ever accepted by every user, with full effective-date and re-acceptance tracking.

Maintain a complete communication log of every notification, reminder, disclosure update, and system email sent to every user, with delivery, open, click, and bounce tracking.

Maintain a complete PII change log capturing every modification to core-level PII fields, including the actor, IP address, device fingerprint, device type, locale, geographic origin, and the change itself, fully audit-immutable and trigger-enforced.

Maintain a secrets and credentials registry for application API keys, database credentials, OAuth client secrets, webhook signing keys, and third-party provider credentials, using envelope encryption with a key recovery plan suitable for executive escrow.

Support a complete data subject rights workflow for GDPR and CCPA, treating anonymization as the default response and explicitly never deleting records that would compromise audit integrity, with a 30-day notification window and a documented manual restoration path at Myriad discretion.

Support legal hold workflows that lock accounts, block anonymization, and require dual-control administrative override.

Support a complete support and impersonation framework with least-privilege role tiers, async approval workflows for elevated actions, RFC 8693 token-exchange impersonation claims, fixed banner display in all consuming applications, and full per-action audit.

Remain AWS-ready without becoming AWS-first.

## 2.2 Non-Goals

HEIMDALL will not be offered as a standalone commercial identity product at launch.

HEIMDALL will not store plaintext secrets in the database under any circumstance.

HEIMDALL will not store payment card data, payment processor refresh tokens, or any data that would bring it into PCI DSS cardholder data environment scope. HEIMDALL will store payment provider customer reference identifiers only.

HEIMDALL will not store raw identity verification images, raw government-issued ID photos, or biometric source data. Identity verification providers retain those artifacts; HEIMDALL stores only verification results and provider reference identifiers.

HEIMDALL will not force automatic account linking across apps, tenants, devices, or emails. Linking is always an explicit user action, support action with audit, or admin action with reason code.

HEIMDALL will not replace application-specific business profiles or become a CRM.

Frontend applications will not receive direct access to secrets, signing keys, encryption keys, or third-party provider credentials. All such access flows through HEIMDALL Core via the authenticated session.

AWS-native services are not required for the first implementation, but the system must be designed for future AWS migration with no business logic rewrites.

# 3. Reference Architecture

```
Cloudflare Edge (DNS, WAF, Rate Limiting, Turnstile)
  ↓
Myriad / Client / Customer Applications (Next.js on Vercel + Native iOS + Native Android)
  - Myriad Core Dashboard
  - HEIMDALL Admin UI
  - HEIMDALL Hosted Login Pages (white-label themed)
  - PEDL Rider / Driver / Admin (web + native iOS + native Android)
  - Quest Attendee / Organizer / Admin (web + native iOS + native Android)
  - Client Field Apps
  - White-label Client Apps

  ↓ OIDC / OAuth2 / Application API Calls (Bearer JWT)

HEIMDALL Core (Single TypeScript Service on Render → AWS ECS Fargate)
  - OIDC Issuer (oidc-provider library)
  - Federation Module (Google, Facebook, Apple, future SAML/Okta via openid-client and @node-saml/node-saml)
  - Authentication Flows (password, SMS OTP, TOTP, WebAuthn, magic-link, recovery)
  - Identity Graph Service (Parent UUID, persona, tenant, linking)
  - Tenant Configuration Registry (versioned policy documents)
  - Authorization Service (in-process, table-driven roles and permissions)
  - Disclosures and Communications Registry (versioned disclosures, acceptances, communication log)
  - Anonymization and Data Subject Rights Service
  - Legal Hold Service
  - Verification Registry (IDV provider abstraction, claims-only storage)
  - Secrets Service (pgcrypto envelope encryption, in-memory TTL cache)
  - Support and Impersonation Service (RFC 8693 act claim, async approval)
  - Guest Identity Service
  - Minor and Guardian Service
  - Offline Sync Intake (Phase 2)
  - Audit Ledger (audit_events, pii_change_events, communication_log, hash-chained, trigger-protected)
  - Webhook Outbox and Delivery
  - pg-boss Job Runner (anonymization execution, communication delivery, rotation jobs)

  ↓ Postgres Wire Protocol

PostgreSQL (Supabase Phase 1 → AWS RDS/Aurora Phase 2)
  - HEIMDALL schema (identity, tenants, personas, audit, disclosures, communications)
  - Encrypted PII via pgcrypto envelope encryption
  - Encrypted secrets table
  - pg-boss schema (background jobs)

External Services
  - Twilio Verify (SMS OTP delivery and verification) → AWS SNS Phase 2
  - SendGrid (transactional email) → AWS SES Phase 2
  - Cloudflare Turnstile (CAPTCHA)
  - Stripe Identity or equivalent (IDV) — provider-abstracted
  - Stripe, PayPal, etc. (payment integration — refs only, no PCI data)
```

| Component | Primary Responsibility | Must Not Do |
| --- | --- | --- |
| Cloudflare Edge | DNS, WAF, bot mitigation, rate limiting, geographic IP signals, Turnstile CAPTCHA challenge. | Inspect or store user PII. Cache authenticated responses. |
| HEIMDALL Core | Authenticate users, federate identity, issue and revoke tokens, manage all identity graph and tenant state, decide authorization, store secrets metadata, log audit, deliver communications, execute anonymization jobs. | Trust client-supplied claims without verification. Bypass audit logging. Store raw payment data or raw IDV images. |
| Frontend Applications | Render UI, hold short-lived access tokens, consume OIDC discovery and userinfo endpoints, call HEIMDALL Core for identity decisions, render support impersonation banner when `act` claim is present. | Invent independent identity sources of truth except offline provisional records pending sync. Hold long-lived credentials or third-party secrets. |
| PostgreSQL | Persist HEIMDALL system-of-record data, audit ledger, events, identity relationships, policies, encrypted secrets, encrypted PII, and consent records. | Receive plaintext PII writes. Receive plaintext secret writes. Be exposed directly to frontends. |
| Twilio Verify | Deliver SMS OTP with carrier fraud and SIM-swap signals; verify codes; provide voice-OTP fallback. | Be addressed directly from frontends. Hold any identity correlation HEIMDALL needs. |
| SendGrid | Deliver transactional and disclosure emails. Provide delivery, open, click, and bounce webhooks. | Be addressed directly from frontends. Receive secret material. |
| Identity Verification Providers | Hold raw ID images, perform document and biometric verification, return result claims and provider reference IDs. | Store HEIMDALL Parent UUIDs as primary keys (only correlation IDs that HEIMDALL controls). |

# 4. Deployment Model

## 4.1 Initial Deployment (Phase 1)

| Layer | Initial Platform | Notes |
| --- | --- | --- |
| Edge / DNS / WAF | Cloudflare | All Myriad domains. Cloudflare Pro plan for WAF and managed rule sets. Turnstile for CAPTCHA on authentication endpoints. Rate limiting rules deployed at the edge ahead of HEIMDALL Core's Fastify rate limiter. |
| Frontend Apps | Vercel | Next.js applications including HEIMDALL Admin UI, hosted login pages (white-label themed), Myriad Core Dashboard, PEDL frontends, Quest frontends. Vercel environment variables hold only public configuration and the HEIMDALL Core API URL. No server secrets in Vercel. |
| Native Mobile Apps | Apple App Store / Google Play | PEDL and Quest native iOS and Android apps. Use platform OAuth flows (ASWebAuthenticationSession on iOS, Custom Tabs on Android) against HEIMDALL Core OIDC issuer. PKCE required; no client secrets in app bundles. |
| HEIMDALL Core | Render (container) | Single OCI container running TypeScript Fastify service. Auto-scaled. Private networking from Vercel via Cloudflare-fronted public endpoint with mTLS optional for internal-only routes. |
| Database | Supabase PostgreSQL | Standard PostgreSQL only. No Supabase Auth, Supabase Realtime, Supabase Edge Functions, or Supabase Vault in critical path. Connection via standard `pg` connection string and pgbouncer. Logical backups exported nightly to non-Supabase storage. |
| Background Jobs | pg-boss in HEIMDALL Core | No Redis dependency. Jobs include anonymization execution, communication delivery, secret rotation, webhook redelivery, key rotation, and audit archival. |
| Events | Postgres LISTEN/NOTIFY + outbox tables + SSE from HEIMDALL Core | No external message broker in Phase 1. |
| Object Storage (Audit Archive) | Backblaze B2 or Cloudflare R2 with bucket-level immutability | Phase 2 deliverable: nightly export of audit ledger partitions to immutable object storage. |
| SMS | Twilio Verify | Verification, voice OTP fallback, carrier risk signals. |
| Email | SendGrid | Transactional and disclosure email with delivery webhooks. |
| CAPTCHA | Cloudflare Turnstile | Privacy-respecting, free. |
| IDV | Stripe Identity (provider-abstracted) | One provider in MVP; abstraction allows multi-provider in Phase 2. |
| Observability | OpenTelemetry SDK → Grafana Cloud or Datadog | Single backend selection in Phase 1; switching trivial. |

## 4.2 AWS-Ready Migration Path (Phase 2)

| Current Capability | AWS-Ready Target | Design Requirement |
| --- | --- | --- |
| Supabase PostgreSQL | Amazon RDS PostgreSQL or Aurora PostgreSQL | Portable SQL only; no Supabase-only dependency in core identity logic. |
| HEIMDALL Core on Render | ECS Fargate behind ALB | OCI-standard container; environment configuration externalized to AWS Parameter Store and Secrets Manager. |
| pgcrypto envelope encryption | pgcrypto + AWS KMS root key | KMS holds the master key; HEIMDALL Core requests data-key decryption per process startup; in-memory only. |
| `SecretsService` (pgcrypto-backed) | `SecretsService` (AWS Secrets Manager-backed) | Same interface; implementation swapped. Rotation hooks already aligned with Secrets Manager. |
| Postgres LISTEN/NOTIFY + outbox | EventBridge + SQS + outbox | Same outbox table; delivery worker switches from in-process to EventBridge publisher. |
| pg-boss jobs | pg-boss continues OR migration to SQS + Lambda workers | pg-boss adequate at expected scale; SQS optional for spiky workloads. |
| Backblaze B2 immutable archive | S3 Object Lock (Compliance mode) | Same nightly archive job, different destination. |
| Twilio Verify | AWS SNS or AWS End User Messaging | Provider abstraction in HEIMDALL Core. |
| SendGrid | AWS SES | Provider abstraction in HEIMDALL Core. |
| Render container deploy | ECS Fargate deploy | Same Dockerfile, different deploy pipeline. |
| Cloudflare Edge | Cloudflare retained (preferred) or CloudFront + AWS WAF | Cloudflare is the default; AWS-native alternatives documented for residency-required tenants. |
| Audit Logs | Continue Postgres hot ledger + add CloudWatch mirror for SIEM ingestion | Append-only Postgres remains source of truth. |

# 5. Identity Requirements

| ID | Category | Requirement | Phase | Priority |
| --- | --- | --- | --- | --- |
| ID-001 | Parent Identity | HEIMDALL shall create and maintain Parent UUIDs as the central identity anchor across all Myriad applications. | MVP | Must |
| ID-002 | Persona Identity | HEIMDALL shall create application and persona UUIDs per app and per user type, linked to a Parent UUID. | MVP | Must |
| ID-003 | Unlinked Identities | Users may intentionally maintain multiple unlinked Parent UUIDs. | MVP | Must |
| ID-004 | Credential Flexibility | HEIMDALL shall support multiple emails, multiple phone numbers, multiple social identities (Google, Facebook, Apple), and future enterprise identities, all linked to a single Parent UUID. | MVP | Must |
| ID-005 | Explicit Linking | Account linking and unlinking shall occur only through explicit user, support (with audit), or admin action with reason code. | MVP | Must |
| ID-006 | No Auto Linking | HEIMDALL shall not automatically suggest or force linking based on email, device, payment, name, or phone similarity. The only exception is the explicit white-label cross-signup opt-in described in ID-009. | MVP | Must |
| ID-007 | Guest Identity | HEIMDALL shall support device-bound anonymous Guest UUIDs with configurable persistence per app and a documented upgrade and claim flow. | MVP | Must |
| ID-008 | Offline Provisional Identity | Applications may create provisional offline records that sync back to HEIMDALL later with Last-Write-Wins resolution and conflict logging. | Phase 2 | Should |
| ID-009 | White-Label Cross-Signup | When a user signs up at a white-label tenant with an email already known at a different tenant, HEIMDALL shall offer the user an explicit, non-coercive option to sign in with the existing account or create a separate account. They are never auto-linked. | MVP | Must |
| ID-010 | Source of Truth | HEIMDALL Identity Graph shall be the single authoritative identity source for all Myriad-owned apps. | MVP | Must |
| ID-011 | Identity Merge Reversal | User-initiated merges shall be reversible for 7 days. Admin-initiated and guardian-initiated merges shall be reversible for 30 days. After the window expires, merges become permanent with full audit retention. | MVP | Must |
| ID-012 | Partial Anonymization | Users may request anonymization of a specific app or persona without affecting their Parent UUID or other apps. | MVP | Must |

# 6. Authentication and Federation Requirements

HEIMDALL Core implements a complete OpenID Connect authorization server using the `oidc-provider` library, a maintained certified implementation by Filip Skokan. Upstream identity provider federation uses the `openid-client` library for OIDC providers, raw OAuth2 with Graph API integration for Facebook, and `@node-saml/node-saml` for SAML enterprise IdPs.

| ID | Requirement | Phase | Notes |
| --- | --- | --- | --- |
| AUTH-001 | Email and password authentication with Argon2id password hashing (`@node-rs/argon2`). | MVP | Primary login method. |
| AUTH-002 | Multiple email addresses per Parent UUID. Email values stored encrypted; HMAC for lookup. | MVP | Verification required before use as login identifier. |
| AUTH-003 | Phone credential support with E.164 normalization, encrypted storage, HMAC for lookup. | MVP | Can be login, MFA, or recovery factor. |
| AUTH-004 | Google OIDC federation. | MVP | Default-on per tenant policy. |
| AUTH-005 | Facebook OAuth2 federation. | MVP | Opt-in per tenant policy; default-off for COPPA-sensitive tenants. |
| AUTH-006 | Apple Sign-In for native iOS, web, and Android. | MVP | Required by App Store policy when any other social login is offered in iOS apps. |
| AUTH-007 | SMS passwordless authentication and SMS OTP as second factor via Twilio Verify. | MVP | Per-tenant policy controls whether SMS can be a primary credential. |
| AUTH-008 | Magic-link email authentication. | MVP | Time-limited single-use tokens (default 15 minutes). |
| AUTH-009 | App-isolated sessions by default. Cross-app SSO available within the Myriad shared tenant scope; cross-tenant SSO requires explicit linking. | MVP | Per-tenant policy. |
| AUTH-010 | In-app persona switching after authentication (e.g., PEDL rider/driver). | MVP | Persona switch generates an audit event. |
| AUTH-011 | WebAuthn passkey authentication (as primary or second factor) via `@simplewebauthn/server`. | MVP | Recommended for admin and high-privilege roles. |
| AUTH-012 | Future Auth0 federation as upstream OIDC IdP. | Phase 3 | Federated, not parallel. |
| AUTH-013 | Future Okta and generic enterprise SSO federation via OIDC and SAML. | Phase 3 | For B2B enterprise tenants. |
| AUTH-014 | Future Microsoft Entra ID federation. | Phase 3 | |
| AUTH-015 | Additional social and authentication providers via configuration-driven provider registry. | Phase 2+ | New providers ship as configuration plus a thin adapter, not architectural changes. |

# 7. Multi-Factor Authentication

| ID | Requirement | Phase | Notes |
| --- | --- | --- | --- |
| MFA-001 | TOTP (RFC 6238) via authenticator apps including Google Authenticator, Microsoft Authenticator, Authy, 1Password, Bitwarden, Duo, and FreeOTP. | MVP | `otpauth://` URI standard; QR code at enrollment. |
| MFA-002 | Backup recovery codes generated at MFA enrollment. Ten single-use codes, stored as bcrypt or Argon2id hashes. | MVP | |
| MFA-003 | SMS OTP as second factor via Twilio Verify. | MVP | Always disclosed as weaker than TOTP/WebAuthn due to SIM-swap risk. |
| MFA-004 | WebAuthn second factor (or passwordless primary). | MVP | |
| MFA-005 | Email OTP as second factor. | MVP | Fallback only; never sole reset path. |
| MFA-006 | Per-tenant MFA policy: optional, required, or required-for-roles-X-Y-Z. | MVP | Configured via Tenant Configuration Registry. |
| MFA-007 | Per-role MFA enforcement. Myriad internal admin roles and tenant admin roles always require MFA. | MVP | |
| MFA-008 | Step-up authentication: re-prompt for MFA before sensitive operations including secret access, identity merge, minor consent grant, support impersonation initiation, account deletion confirmation, and break-glass actions. | MVP | |
| MFA-009 | Trusted device "remember me" with configurable duration (default 30 days, tenant-overridable). Stored as `trusted_devices_2fa` records with device fingerprint and revocable individually. | Phase 2 | |
| MFA-010 | FIDO2 hardware security keys for Myriad internal admin roles. | Phase 2 | |
| MFA-011 | Multiple enrolled methods per user with a designated primary and fallbacks. | MVP | |
| MFA-012 | User self-service to view, name, and revoke enrolled MFA methods and trusted devices. | MVP | |

**SMS as a second factor is explicitly designated the weakest enrolled method.** Recovery paths must require a stronger factor when one is enrolled. SMS-only accounts are permitted but flagged in risk scoring.

# 8. Phone Number as Identity

| ID | Requirement | Phase | Priority |
| --- | --- | --- | --- |
| PHN-001 | Multiple phone numbers per Parent UUID with a designated primary. | MVP | Must |
| PHN-002 | Phone verification via SMS OTP (or voice OTP fallback) at credential add time. | MVP | Must |
| PHN-003 | Phone as sole login credential (passwordless SMS OTP flow), enabled per tenant policy. | MVP | Must |
| PHN-004 | Phone as second factor (SMS-based). | MVP | Must |
| PHN-005 | E.164 normalization; encrypted storage; HMAC of normalized form for dedup and lookup. | MVP | Must |
| PHN-006 | Voice OTP fallback for accessibility and SMS-blocked regions. | Phase 2 | Should |
| PHN-007 | SIM-swap risk telemetry from Twilio carrier signals integrated into risk-based authentication. | Phase 3 | Could |

# 9. Tenant, Organization, and White-Label Model

HEIMDALL implements multi-tenancy natively. Unlike the V1 design that proposed Keycloak Realms and Organizations, the V3 design treats tenancy as a first-class concept in the HEIMDALL data model.

## 9.1 Tenant Tiers

| Tenant Tier | Use When | Examples |
| --- | --- | --- |
| `myriad_internal` | For Myriad employees, support users, system administrators, and HEIMDALL administrators. | Myriad Core Dashboard, HEIMDALL Admin. |
| `myriad_products` | For shared Myriad consumer/customer apps where cross-product identity visibility may be supported and SSO across Myriad products is desired. | PEDL, Quest, shared Myriad apps. |
| `client_{slug}` | For client/operator tenants. Default operating model. Each client/operator is a tenant within HEIMDALL with its own configuration, branding, and policies. | Acme Tours (white-label Quest), Logistics Co (PEDL operator), educational program. |
| `isolated_client_{slug}` | When true legal, compliance, residency, or visibility isolation is required. Implemented as a dedicated database schema with no cross-tenant queries permitted. | Future CJIS-bound tenant, HIPAA-bound tenant. |

A tenant decision rule operates: default to a tenant within the shared `myriad_products` namespace. Use an isolated tenant only when contractual, legal, residency, or compliance separation requires it. Adding an isolated tenant is a configuration change, not a deploy event.

## 9.2 White-Label Requirements

Support tenant-specific login themes, domains, From-address email senders, SMS sender IDs, callback URLs, and brand metadata. Tenant theming applies to the HEIMDALL Hosted Login Pages (Next.js app), to the HEIMDALL Admin UI for tenant admin roles, and to all transactional communications. Tenants may carry their own DNS-validated sender domains with DKIM and SPF configured.

Support tenants that are not aware of, nor visibly connected to, other Myriad-backed apps. Cross-tenant existence is never disclosed to end users without explicit opt-in via ID-009 cross-signup flow.

Support per-tenant security policies including MFA, allowed login providers, session duration, device rules, age gates, disclosure requirements, and rate-limit overrides, all defined in the Tenant Configuration Registry (see Section 10).

# 10. Tenant Configuration Registry

The Tenant Configuration Registry is a first-class HEIMDALL domain. All tenant-customizable behavior is expressed as versioned policy documents stored in PostgreSQL, edited through the HEIMDALL Admin UI, and evaluated by HEIMDALL Core at runtime.

| Policy Category | Controls |
| --- | --- |
| `auth_methods_policy` | Which credential types are permitted (password, social-google, social-facebook, social-apple, sms-passwordless, magic-link, webauthn-passwordless). Adding a new provider in Phase 2+ is a configuration change. |
| `mfa_policy` | MFA required, optional, or required for specific roles. Which MFA methods are allowed. Step-up triggers. |
| `social_providers_policy` | Which social providers are enabled. Default opt-in/opt-out per provider. COPPA exclusions. |
| `age_gate_policy` | Minimum age, jurisdiction matrix, guardian consent requirements, age-gated features. |
| `disclosure_policy` | Which disclosure categories are required at signup. Re-acceptance rules on version change. |
| `rate_limit_policy` | Overrides on global default rate limits and SMS caps for this tenant. |
| `communication_policy` | From address, sender domain, DKIM and SPF status, channel preferences, locale defaults. |
| `session_policy` | Session length, idle timeout, MFA step-up triggers, trusted-device duration. |
| `data_residency_policy` | Phase 3: residency requirements for CJIS, EU, and other jurisdictions. |
| `social_provider_credentials_policy` | Which client IDs and credentials are used for Google, Facebook, and Apple OAuth per tenant (some tenants run their own OAuth apps). |

Each policy document is versioned with `effective_at` and `superseded_at` timestamps. Policy edits in the admin UI create a new version; activation requires confirmation. Diffs between versions are displayed for legal and compliance review. All policy changes generate audit events.

# 11. Child and Minor Account Requirements

Child and minor support is an immediate requirement. HEIMDALL must restrict roles, communications, data visibility, consent flows, and account actions for minors from day one.

The COPPA US-under-13 regime is the MVP starter. The schema and policy engine are designed to accommodate jurisdiction-flexible age policies including California AB 2273 (Age-Appropriate Design Code), Florida HB 3, Utah Social Media Regulation Act, GDPR-K (varying 13–16 by EU member state), and future state and international regimes without schema migration.

| ID | Requirement | Phase | Priority |
| --- | --- | --- | --- |
| MIN-001 | Minor profile flag and minor-specific account status, derived from declared age and jurisdiction policy. | MVP | Must |
| MIN-002 | Age attestation at minimum at signup. Per-app age gate configurable via `age_gate_policy`. | MVP | Must |
| MIN-003 | Encrypted date-of-birth field. Stored via envelope encryption; never exposed in admin UI by default. | MVP | Must |
| MIN-004 | Guardian relationship records linking guardian Parent UUID to minor Parent UUID. | MVP | Must |
| MIN-005 | Guardian consent records (versioned via Disclosures Registry) with grant and revocation tracking. | MVP | Must |
| MIN-006 | App-specific age policy rules with jurisdiction matrix (`age_policy_rules` table). | MVP | Must |
| MIN-007 | Restricted roles, permissions, and feature flags for minors via the `application_minor_features` table. | MVP | Must |
| MIN-008 | Age transition workflow: when a minor reaches the age threshold, an event fires, guardian relationships move to "lapsed" state, and the user is prompted to re-accept adult terms on next login. | Phase 2 | Should |
| MIN-009 | Per-app minor feature restrictions configurable by tenant (e.g., disable messaging, disable public profiles, disable purchase flows). | Phase 2 | Should |

# 12. Authorization and Access Control

HEIMDALL Core implements authorization as an in-process service. There is no separate policy engine (no OPA) in MVP. The decision model is tabular and predicate-based, fully expressible in TypeScript with index-backed SQL lookups.

## 12.1 Decision Model

```
Access Decision =
  Parent Identity
  + Active Persona
  + Application
  + Tenant / Organization
  + Roles assigned to (Parent or Persona) in this Tenant for this App
  + Permissions granted by those Roles
  + Context Object (event, ride, ticket, etc.)
  + Policy Constraints (minor restrictions, consent state, verification level, time of day, MFA freshness, geolocation, legal hold)
```

## 12.2 Two-Tier Resolution

**Tier 1 (Fast Path, SQL):** "Does this Parent/Persona have role X in tenant Y for app Z?" — answered by indexed query against `persona_roles` joined to `role_permissions`. Sub-millisecond.

**Tier 2 (Slow Path, TypeScript):** "Can this Parent perform action A on object B given current context?" — evaluated by typed predicates in TypeScript against a `DecisionContext` object. Returns allow/deny plus reasons. Decisions cached in memory with 30–60 second TTL keyed by `(parent_id, persona_id, app, action, context_hash)`. Deny decisions and elevated-permission allows generate audit events.

## 12.3 Table-Driven Permissions

All permissions are atomic strings stored in the `permissions` table. Roles aggregate permissions via `role_permissions`. Personas receive roles via `persona_roles`. All changes audited via `role_permission_changes`. Built-in roles seeded via migration; custom tenant roles supported from day one.

| Layer | Examples |
| --- | --- |
| Application | PEDL, Quest, Myriad Dashboard, Client Field App, HEIMDALL Admin. |
| Persona | Rider, driver, guide, attendee, organizer, dispatcher, support agent, admin. |
| Tenant/Organization | Client, operator, event organizer, fleet, venue, educational program. |
| Permission | `ride.accept`, `event.manage`, `ticket.scan`, `user.view`, `user.impersonate`, `secret.read`, `secret.rotate`, `tenant.config.update`, `disclosure.publish`, `audit.export`. |
| Context | Event ID, ride ID, tour ID, venue ID, fleet ID, ticket-type ID, dashboard module. |

# 13. Role Tiers

Starter role tiers are seeded in migration. Tenant admins may create custom roles within their tenant via the HEIMDALL Admin UI.

## 13.1 Myriad Internal Roles

| Role | Scope | Notes |
| --- | --- | --- |
| `myriad_super_admin` | All tenants, all configuration including HEIMDALL infrastructure. | Two to three people maximum. Hardware MFA required. |
| `myriad_admin` | Manage tenants, apps, and users across all tenants. No HEIMDALL infrastructure access. | Fewer than ten people. |
| `myriad_support_l2` | View all users, request impersonation, reset MFA, cannot change tenant configuration or delete records. | Approves L1 impersonation requests. |
| `myriad_support_l1` | View users in active support tickets, request impersonation (requires L2 approval), reset password (sends email link; never directly views passwords). | First-line support. |
| `myriad_billing_readonly` | Financial dashboards only. No PII access. | |
| `myriad_security_auditor` | Read all audit logs and PII change logs. No write access. | For compliance investigations. |
| `myriad_legal_officer` | Place and release legal holds. Approve dual-control overrides on locked accounts. | Sealed-record access in coordination with General Counsel. |

## 13.2 Tenant Roles

| Role | Scope |
| --- | --- |
| `tenant_owner` | Manage tenant configuration, billing reference, users within tenant. |
| `tenant_admin` | Manage users and roles within tenant. No billing access. |
| `tenant_support` | View tenant users, request impersonation of tenant users (requires `tenant_admin` approval), reset passwords. |
| `tenant_operator` | Application-specific operational role (PEDL dispatcher, Quest event manager, fleet manager). |

# 14. Secrets and Credentials Management

HEIMDALL includes a formal Secrets and Credentials Management domain. The MVP implementation is a custom `SecretsService` backed by `pgcrypto` envelope encryption in PostgreSQL. The interface is provider-agnostic; OpenBao and AWS Secrets Manager remain documented fallback implementations.

## 14.1 Three-Layer Environment Variable Strategy

| Layer | Contents | Storage |
| --- | --- | --- |
| Layer 1 — Runtime Environment | `DATABASE_URL`, `MASTER_KEY` (32-byte root for envelope encryption), `MASTER_KEY_VERSION`, `NODE_ENV`, `LOG_LEVEL`, observability endpoints. | Render environment (Phase 1, scoped to production deploy, platform audit log of access). AWS KMS-backed retrieval at startup (Phase 2). |
| Layer 2 — Encrypted Secrets Table | All other secrets: Twilio API keys, SendGrid keys, OIDC client secrets, OAuth provider credentials, webhook signing keys, IDV provider keys, payment processor publishable keys. | `secrets` table in PostgreSQL, encrypted with per-row data encryption keys (DEKs) wrapped by the master key. Accessed only via `SecretsService` from HEIMDALL Core. |
| Layer 3 — Frontend Environment | Public OIDC discovery URL, HEIMDALL Core API URL, payment processor publishable key (Stripe), Cloudflare Turnstile site key, hCaptcha site key (if used). | Vercel environment variables. No server secrets here under any circumstance. |

## 14.2 Secret Types and Handling

| Secret Type | Examples | Storage |
| --- | --- | --- |
| Application API keys | Mapbox, Stripe, Twilio, SendGrid, Eventbrite, analytics services. | Encrypted in `secrets` table. Metadata in `secret_items`. |
| Database secrets | Production database URL, read-replica URLs, migration credentials. | Master DB URL in runtime env (Layer 1). Per-service credentials encrypted (Layer 2). |
| OIDC client secrets | Internal OIDC clients (PEDL, Quest, Myriad Dashboard). | Encrypted in `secrets` table. Rotated by policy. |
| Webhook signing secrets | IDP webhooks, payment provider webhooks, app event webhooks. | Encrypted in `secrets` table. HMAC validation governed by HEIMDALL. |
| Mobile/app config secrets | Non-public app credentials and tenant-specific runtime config. | Never embedded in native or web client bundles. Delivered through HEIMDALL Core authenticated endpoints. |
| Service-to-service credentials | Internal API credentials, worker credentials, integration credentials. | Service-account scoped and audited. |

## 14.3 Secrets Requirements

| ID | Requirement | Phase | Priority |
| --- | --- | --- | --- |
| SEC-001 | All third-party secret values shall be stored encrypted using envelope encryption. | MVP | Must |
| SEC-002 | The `secrets` table shall be inaccessible to frontend code and accessible only to HEIMDALL Core via the service-role database connection. | MVP | Must |
| SEC-003 | Environment-specific secrets (dev, staging, prod) shall be isolated; production secrets shall never load in non-production environments. | MVP | Must |
| SEC-004 | Secrets shall be scoped per application, per tenant, per service, per provider, and per environment. | MVP | Must |
| SEC-005 | Every secret read, write, rotation, revocation, and failed access shall generate an audit event including the requesting principal. | MVP | Must |
| SEC-006 | Manual rotation and emergency revocation shall be supported via HEIMDALL Admin UI. | MVP | Must |
| SEC-007 | Scheduled rotation, expiration alerts, approval workflows, and usage dashboards. | Phase 2 | Should |
| SEC-008 | Dynamic database credentials. | Phase 3 (with OpenBao or Secrets Manager) | Could |
| SEC-009 | Master key recovery via Shamir's Secret Sharing escrow plan (see Appendix B). | MVP | Must |
| SEC-010 | Key versioning and zero-downtime rotation procedure. | MVP | Must |

## 14.4 Frontend Secret Access Pattern

Frontends (web and native) hold only short-lived OIDC access tokens. Any operation requiring a third-party credential is brokered server-side:

1. Frontend calls HEIMDALL Core API with `Authorization: Bearer <jwt>`.
2. HEIMDALL Core validates JWT, resolves persona and tenant, evaluates authorization.
3. If the operation requires a third-party credential, HEIMDALL Core fetches it from `secrets` via `SecretsService` (60–300 second in-memory cache).
4. HEIMDALL Core performs the third-party API call server-side.
5. For browser-required SDKs (Stripe Elements, Mapbox tiles), HEIMDALL Core returns only the publishable or scoped key. Secret-tier keys never leave HEIMDALL Core.
6. For client-direct workflows (large file uploads), HEIMDALL Core mints short-lived signed URLs or scoped tokens (default 5–15 minutes).

# 15. Disclosures and Communications Registry

The Disclosures and Communications Registry is a first-class HEIMDALL domain that powers signup consent, re-consent on version change, anonymization notifications, guardian consent, COPPA notices, SMS opt-in, marketing opt-in, and any other versioned communication or disclosure.

## 15.1 Disclosure Layers

Disclosures stack in three layers with inheritance:

1. **Global** — Myriad master Terms of Service, master Privacy Policy. Apply to all users of all Myriad apps.
2. **Per-App** — App-specific terms (PEDL Rider Terms, PEDL Driver Terms, Quest Organizer Terms). Stack on top of global.
3. **Per-Tenant** — Tenant-specific terms (Acme Tours Terms for the Acme Tours white-label Quest deployment). Stack on top of app and global.

## 15.2 Categories

`terms_of_service`, `privacy_policy`, `tenant_terms`, `coppa_consent`, `gdpr_data_processing`, `sms_consent`, `marketing_opt_in`, `app_specific_eula`, `community_guidelines`, `acceptable_use_policy`, `cookie_consent`.

## 15.3 Versioning and Acceptance

Every disclosure version has an `effective_at` and an optional `superseded_at`. Acceptance records reference the specific version accepted. When a new version is published with `requires_re_acceptance=true`, affected users receive a blocking re-consent prompt on next login. Diffs between versions are displayed in the HEIMDALL Admin UI for legal review before activation.

## 15.4 Communication Log

The `communication_log` table records every notification, email, SMS, in-app message, or system communication sent. Captured fields include parent ID, template version ID, channel, recipient address, send timestamp, delivery timestamp, open timestamp, click timestamp, bounce timestamp, failure reason, correlation ID linking to the triggering event (anonymization request, disclosure update, security alert), and locale. The communication log is append-only by trigger and serves as legal evidence of notice for compliance purposes (CAN-SPAM, TCPA, GDPR data subject rights, COPPA notice requirements).

## 15.5 Locale Support

Schema supports `locale` on every template version, disclosure version, and communication log entry from day one. MVP delivers English-only content; Spanish content expected shortly after MVP. Templates and disclosures may be authored in multiple locales with the active locale selected per user preference and per tenant default.

## 15.6 Delivery Channels

Disclosure acceptance at signup is blocking and required to complete signup. Re-acceptance on version change is shown as a blocking modal on next login. Informational and marketing communications are non-blocking and delivered via in-app inbox plus the user's preferred channel (email or SMS).

## 15.7 Sender Domains

Per-tenant From address with DKIM and SPF configured per tenant domain. If a tenant has not configured a sender domain, the default `noreply@myriad.com` is used. Tenant onboarding includes a DKIM/SPF setup wizard. SendGrid (Phase 1) and AWS SES (Phase 2) both support per-domain sender verification.

# 16. Anonymization and Data Subject Rights

HEIMDALL implements GDPR Article 17 (right to erasure), CCPA right to delete, and analogous regimes through anonymization rather than deletion. PII is anonymized; identity anchors and audit references are retained. Restoration is manual at Myriad discretion and is not a self-service action.

## 16.1 Anonymization Scopes

| Scope | Effect |
| --- | --- |
| `full_account` | Anonymize Parent UUID and all linked credentials, personas, verification records, and minor profiles. User loses access to all Myriad apps. |
| `per_app` | Anonymize a specific application's persona, verification records, and session/device records. Parent UUID remains active for other apps. |
| `per_persona` | Anonymize a specific persona within an app. Parent UUID and other personas remain active. |

## 16.2 30-Day Notification Window

Anonymization requests enter a 30-day window before execution:

| Day | Communication |
| --- | --- |
| Day 0 | Initial confirmation email and SMS (if phone enrolled). "Your anonymization request was received. It will be processed in 30 days. You may cancel any time before then." |
| Day 15 | Second reminder. "Your anonymization request will be processed in 15 days." |
| Day 28 | Final reminder. "Your anonymization request will be processed in 48 hours." |
| Day 30 | Execution. Anonymization runs as a pg-boss job. Confirmation communication sent to user (if any contact method remains valid). |

Each communication is tracked in `communication_log` with the request's correlation ID. The `anonymization_requests` table holds `requested_at`, `notification_1_sent_at`, `notification_2_sent_at`, `final_reminder_sent_at`, `executed_at`, `cancelled_at`, `cancelled_by`, `cancelled_reason`, and `scope`.

## 16.3 Anonymization Effects per Table

| Table | Effect on Anonymization |
| --- | --- |
| `parent_identities` | PII fields (display_name, etc.) replaced with `[ANONYMIZED-{date}]`. `anonymized_at` and `anonymized_reason` set. Row preserved as audit anchor. |
| `credential_identities` | Email, phone, social subject IDs replaced with tombstones. HMACs zeroed so future signups never auto-link. |
| `app_personas` | Persona-level PII (display names, profile photos) tombstoned. Persona record preserved for audit. |
| `minor_profiles` | DOB tombstoned. Guardian relationships preserved with audit reference. |
| `verification_records` | Provider verification ID preserved (a pointer the user can no longer retrieve); cached PII claims (name, DOB) tombstoned. |
| `session_records` | Deleted entirely. |
| `trusted_devices` and `trusted_devices_2fa` | Deleted entirely. |
| `audit_events` | Untouched. Already PII-free by design (references parent_id only). |
| `pii_change_events` | Untouched. Already PII-free by design (stores HMACs of changed values, not plaintext). |
| `communication_log` | Recipient address tombstoned after 90-day retention. Send timestamps and template references preserved. |

## 16.4 Returning Users

A user who returns and signs up with the same email or phone after anonymization receives a fresh Parent UUID with no linkage to the anonymized record. This is the privacy-protective default. Manual relinking is available at Myriad's discretion via the support workflow.

## 16.5 Initiators

| Initiator | Workflow |
| --- | --- |
| Self-service | User clicks "Delete my account" in account settings. Confirms via MFA step-up. Enters 30-day flow. |
| Support-assisted | User contacts support via email/phone. Support agent verifies identity, records the request in HEIMDALL Admin UI with required fields. Enters 30-day flow. |
| Legal or compliance | Formal DPA request, regulator order, court order. Recorded with reference number. May enter accelerated timeline at Legal Officer discretion. |

# 17. Legal Holds

The Legal Hold service prevents anonymization, identity changes, or account access for parent identities under active litigation, regulatory investigation, or other legal preservation requirements.

| Behavior | Implementation |
| --- | --- |
| Account access | All sessions revoked immediately upon hold placement. All login attempts rejected with a generic "account temporarily unavailable, contact support" message. The legal hold reason is never disclosed to the user. |
| Anonymization | Active anonymization requests are queued (not cancelled) and execute after the hold is released. |
| Identity merge | Blocked for the held identity. |
| Credential changes | Blocked. |
| Admin override | Requires `myriad_super_admin` role plus dual-control: a second admin must co-sign within 60 minutes. Every override generates a high-priority audit event with both actors and the override reason. |
| Placement | Requires `myriad_legal_officer` role. Records authority, reason code, effective dates, and reference to legal matter. |
| Release | Requires `myriad_legal_officer` role with reason. Queued anonymization requests, if any, execute on schedule from the release date plus the remaining window. |
| Audit | Every block, override request, override co-sign, and release event generates an audit event. |

# 18. Support and Impersonation

Support and impersonation tooling implements least-privilege access with full audit and explicit in-app indication to the impersonated user's UI surface.

## 18.1 Impersonation Token Mechanics

Impersonation uses RFC 8693 OAuth 2.0 Token Exchange. When an approved impersonation begins, HEIMDALL issues an access token with an `act` claim:

```json
{
  "sub": "<impersonated_parent_id>",
  "act": {
    "sub": "<support_agent_parent_id>",
    "reason": "ticket-12345",
    "approval_id": "<approval_record_id>"
  },
  "exp": "<15 minutes from now>",
  "scope": "..."
}
```

- Initial impersonation token lifetime: 15 minutes.
- Maximum total session duration: 60 minutes (renewable in 15-minute increments via re-approval prompt at the 15-minute boundary).
- Token revocable instantly via "Exit support session" action.

## 18.2 In-App Banner

All consuming applications (PEDL, Quest, Myriad Dashboard, every tenant frontend) implement a fixed-position banner from day one of HEIMDALL integration. The banner contract:

- Detection: presence of `act` claim in the OIDC ID token or access token.
- Display: red background, white text, fixed at top of viewport, undismissible.
- Content: "Support is viewing as {user.display_name}. All actions are audited. Exit support session →"
- Action: "Exit" link posts to HEIMDALL Core, revokes the impersonation token, and redirects the support agent to the support tools home.
- The banner is part of the frontend contract included in every PEDL and Quest release; no app may suppress it.

## 18.3 Async Approval Workflow

L1 impersonation requests require L2 approval. The flow:

1. L1 support agent in HEIMDALL Admin UI clicks "Request impersonation" for a user, with required fields: ticket reference, justification, requested duration (up to 60 minutes).
2. HEIMDALL Core creates an `impersonation_approvals` record in pending state.
3. Notification sent to all available L2 staff via Slack webhook and email. Notification includes user ID (not PII), ticket reference, justification, requested duration.
4. Any L2 staff can approve or deny in HEIMDALL Admin UI.
5. On approval, L1 receives notification. L1 has 30 minutes from approval to start the impersonation session, after which the approval expires.
6. On session start, the 15-minute token clock begins.
7. Every action during the session writes to `support_session_events` with both `support_parent_id` and `impersonated_parent_id`.

Tenant-scope impersonation (`tenant_support` impersonating a user within their own tenant) requires `tenant_admin` approval in the same async flow.

## 18.4 Least-Privilege Access

Support roles see only the information required for their function:

| Role | Sees | Does Not See |
| --- | --- | --- |
| `myriad_support_l1` | User display name and ID, ticket associations, MFA enrollment status, recent failed logins, last login timestamp. | Email plaintext (HMAC indicator only), phone plaintext, DOB, encrypted PII fields, audit logs, secret access logs. |
| `myriad_support_l2` | All of L1 plus: email, phone (with PII access audit event generated on view), full session history, MFA reset capability. | Encrypted PII outside the user record, secret values, infrastructure config. |
| `tenant_support` | All within their tenant only. Same field-level visibility as L2 scoped to their tenant. | Any data from other tenants. |

Every PII field view by a support role generates a `pii_change_events` row of type `view` (the same table that records changes).

# 19. PII Change Logging

The PII Change Logging domain captures every modification and view of core-level PII fields with rich device, location, and actor context. This is a separate ledger from `audit_events`, optimized for compliance queries such as "show me everything that ever touched this user's PII."

## 19.1 Tracked Fields

| Table | Fields Tracked |
| --- | --- |
| `parent_identities` | `display_name`, `preferred_locale`, `legal_name` (if collected). |
| `credential_identities` | `email`, `phone`, social subject IDs. |
| `minor_profiles` | `date_of_birth`, `guardian_id`. |
| `guardian_relationships` | All fields. |
| `app_personas` | App-specific PII fields (driver license number for PEDL drivers, etc.). |
| `verification_records` | Verification result claims. |

## 19.2 Captured Context per Event

| Field | Notes |
| --- | --- |
| `parent_id` | The identity affected. |
| `actor_parent_id` | Who made the change. |
| `actor_type` | `self`, `support_impersonation`, `admin`, `system`, `migration`. |
| `field_path` | Dotted path to the affected field. |
| `change_type` | `create`, `update`, `delete`, `anonymize`, `view`. |
| `previous_value_hash` | HMAC of prior value. Never plaintext. |
| `new_value_hash` | HMAC of new value. |
| `ip_address` | Source IP (IPv4 or IPv6). |
| `user_agent` | Full UA string. |
| `device_fingerprint` | Where available. |
| `device_type` | `web`, `ios_native`, `android_native`, `mobile_web`, `api`, `admin_console`. |
| `locale` | From Accept-Language. |
| `geo_country` | From Cloudflare IP geo headers. |
| `geo_region` | From Cloudflare IP geo headers. |
| `correlation_id` | Links to triggering session, ticket, or admin action. |
| `created_at` | Append-only, trigger-protected. |

## 19.3 Enforcement

PII change capture is enforced by Postgres triggers on the tracked tables. Triggers fire regardless of the connecting database role, including `service_role`. No application code can bypass the trigger. The `pii_change_events` table itself has no UPDATE or DELETE grants; only an explicit compliance role (used in disaster recovery and legally mandated redactions) may modify rows.

# 20. Identity Verification

Identity verification (IDV) is performed by external providers. HEIMDALL stores only verification results, provider reference identifiers, and minimal extracted claims. Raw ID images, biometric source data, and full extracted PII remain at the provider.

| ID | Requirement | Phase |
| --- | --- | --- |
| IDV-001 | Provider abstraction interface (`IdvProvider`) supporting Stripe Identity (MVP), with documented adapter pattern for Persona, Onfido, Plaid IDV, ID.me. | MVP |
| IDV-002 | Verification results stored as: provider name, provider reference ID, verification level, verification timestamp, expiration, and minimal claims (verified DOB if needed for age, verified legal name match, verified document type, verified expiry). | MVP |
| IDV-003 | Apps that require persistent ID documents (PEDL DOT compliance for drivers) handle their own retention and never store these in HEIMDALL. | MVP |
| IDV-004 | Re-verification triggered on document expiry, on policy change, or on risk signal. | Phase 2 |
| IDV-005 | Multi-provider routing for cost and geographic coverage optimization. | Phase 2 |

# 21. Payments and PCI Scope

HEIMDALL is designed to remain out of PCI DSS Cardholder Data Environment (CDE) scope. The platform stores payment provider customer reference identifiers only.

| ID | Requirement | Phase |
| --- | --- | --- |
| PAY-001 | No card data (PAN, CVV, expiration, magnetic stripe, EMV chip) shall transit or be stored by HEIMDALL. | MVP |
| PAY-002 | Apps integrate payment providers (Stripe, PayPal, others) directly using publishable keys delivered through HEIMDALL Core's secret broker. | MVP |
| PAY-003 | HEIMDALL stores `payment_provider_customer_refs`: `parent_id`, `app_code`, `provider` (stripe, paypal, etc.), `provider_customer_id`, `display_brand` (Visa, Mastercard, etc.), `display_last_four`, `is_default`, `created_at`. | MVP |
| PAY-004 | Provider tokens (e.g., Stripe `cus_xxxx`) are the only payment-method identifiers retained. | MVP |
| PAY-005 | Refresh tokens or long-lived provider credentials shall be stored in the encrypted `secrets` table only when required for server-side reconciliation. | MVP |
| PAY-006 | Later phases support per-app preferred payment integration (PEDL may use Stripe Connect; Quest may use Eventbrite payments) without changing HEIMDALL's PCI posture. | Phase 2 |

# 22. Event and Audit Model

HEIMDALL maintains three append-only ledgers, all hash-chained and trigger-protected.

## 22.1 audit_events

All identity, secrets, authorization, tenant, disclosure, anonymization, legal hold, support session, and security events.

| Event Category | Examples |
| --- | --- |
| Identity | `identity.created`, `identity.updated`, `identity.deactivated`, `credential.linked`, `credential.unlinked`, `identity.merged`, `merge.reversal_requested`, `merge.reversal_completed`, `merge.window_expired`. |
| Persona | `persona.created`, `persona.activated`, `persona.switched`, `persona.disabled`. |
| Tenant | `tenant.created`, `tenant.config.updated`, `tenant.policy.updated`, `tenant.theme.updated`. |
| Authentication | `auth.login.success`, `auth.login.failure`, `auth.password.changed`, `auth.method.enrolled`, `auth.method.revoked`, `mfa.challenged`, `mfa.success`, `mfa.failure`, `social.linked`, `social.unlinked`. |
| Authorization | `authz.denied`, `authz.elevated_allow`, `role.assigned`, `role.removed`, `permission.granted`, `permission.revoked`, `role.created`, `role.deleted`. |
| Minor / Guardian | `minor.created`, `guardian.linked`, `guardian.unlinked`, `consent.granted`, `consent.revoked`, `age.transitioned`. |
| Guest / Offline | `guest.created`, `guest.claimed`, `guest.upgraded`, `offline_event.received`, `offline_conflict.resolved`. |
| Verification | `verification.requested`, `verification.completed`, `verification.failed`, `verification.expired`, `verification.revoked`. |
| Secrets | `secret.created`, `secret.accessed`, `secret.rotated`, `secret.revoked`, `secret.access_denied`, `secret.lease.issued`, `secret.lease.expired`. |
| Disclosures | `disclosure.published`, `disclosure.accepted`, `disclosure.re_acceptance_required`, `consent.recorded`. |
| Communications | `communication.queued`, `communication.sent`, `communication.delivered`, `communication.bounced`, `communication.failed`. |
| Anonymization | `anonymization.requested`, `anonymization.notification_sent`, `anonymization.cancelled`, `anonymization.executed`. |
| Legal Hold | `hold.placed`, `hold.blocked_action`, `hold.override_requested`, `hold.override_approved`, `hold.released`. |
| Support | `support.impersonation_requested`, `support.impersonation_approved`, `support.impersonation_denied`, `support.session_started`, `support.session_action`, `support.session_ended`. |
| Security | `security.brute_force_lockout`, `security.suspicious_login`, `security.token_revoked`, `security.break_glass.requested`, `security.break_glass.approved`. |

## 22.2 pii_change_events

PII-specific ledger described in Section 19.

## 22.3 communication_log

Communication ledger described in Section 15.4.

## 22.4 Hash Chain

Each ledger table has a `prev_hash` and `row_hash` column. A BEFORE INSERT trigger computes `prev_hash` from the last row's `row_hash` (per aggregate where appropriate) and computes `row_hash` from the row's content. This produces a tamper-evident chain verifiable in SQL. Cold archive exports include hash verification.

## 22.5 Partitioning and Retention

Ledger tables are partitioned by month. Live retention is 24 months in the hot Postgres store. Older partitions are archived to immutable object storage (Backblaze B2 or Cloudflare R2 with bucket immutability in Phase 1; S3 Object Lock in Compliance mode in Phase 2). Cold archives retain only references to parent_id and other non-PII metadata; raw PII never lands in cold archive.

# 23. Security Requirements

## 23.1 Edge Security (Cloudflare)

| Cloudflare Product | Function |
| --- | --- |
| Cloudflare DNS | Authoritative DNS for all Myriad domains. |
| Cloudflare WAF (Pro plan) | Managed rule sets, OWASP rules, custom rules for HEIMDALL endpoints. |
| Cloudflare Rate Limiting | First-line rate limiting at the edge before HEIMDALL Core's Fastify rate limiter. |
| Cloudflare Turnstile | CAPTCHA challenge on authentication endpoints. Privacy-respecting, free. |
| Cloudflare Access (Phase 2) | SSO gate in front of HEIMDALL Admin UI for additional internal protection. |
| Cloudflare Bot Management (Phase 2) | Activated if credential-stuffing patterns observed. |

## 23.2 Application-Level Security

Encrypt sensitive PII using envelope encryption (`pgcrypto` with per-row data keys wrapped by the master key). Store normalized HMACs for searchable PII (email, phone) — never plaintext search fields.

Require strong admin access control for identity merges, secret access, production credential views, support impersonation, and break-glass sessions.

Use least-privilege service accounts and scoped credentials for apps, workers, and integrations.

Do not expose production secret values to frontend code or browser environments under any circumstance.

Log and alert on failed secret access, unusual admin actions, repeated login failures, suspicious credential linking, high-value guest upgrade attempts, SIM-swap signals from Twilio, and credential-stuffing patterns from Cloudflare.

Support soft deletion, deactivation, legal hold flags, and the full anonymization workflow.

Plan for CJIS-oriented controls in Phase 3 only when a signed client need exists.

## 23.3 SMS Rate Limit and Cost Caps

All rate-limit policies are table-driven via `rate_limit_policies` and tenant-overridable.

| Cap | Default Limit | Rationale |
| --- | --- | --- |
| Per phone number, OTP send rate | 5 per hour, 10 per day | Prevent abuse. |
| Per phone number, send cooldown | 60 seconds between requests | Prevent flooding. |
| Per IP address | 10 OTPs per hour across all numbers | Prevent attack from one source. |
| Per account (parent_id) | 20 OTPs per day | Prevent compromised-account abuse. |
| Per country code (configurable per tenant) | $100/day cost cap default | SMS pumping protection. |
| Global daily SMS cost cap | $500/day default, alert at 50%, hard stop at 100% | Catastrophic abuse stop. |
| Carrier risk score (Twilio Verify) | Block "high-risk" numbers from voice-OTP-only countries unless explicitly allowed | Premium-rate fraud. |

Tenant overrides may relax or tighten these defaults via HEIMDALL Admin UI. Changes are versioned in `rate_limit_policies` and audited.

# 24. Phased Roadmap

| Phase | Scope |
| --- | --- |
| MVP | HEIMDALL Core scaffolded (Fastify + Drizzle + pg-boss); OIDC issuer (`oidc-provider`); Identity Graph (parent, persona, tenant, guest, minor/guardian); credential authentication (password, Google, Facebook, Apple, magic-link, SMS); MFA (TOTP, SMS, WebAuthn, backup codes, recovery); Authorization Service with starter roles; Secrets module with pgcrypto envelope encryption; audit ledger with hash chain and triggers; PII change logging; Tenant Configuration Registry; Disclosures & Communications Registry; Anonymization workflow (30-day flow, partial scopes); Legal Hold service; Support and Impersonation framework with async approval and banner contract; Cloudflare edge integration; PEDL pilot integration; Quest pilot integration; Myriad Dashboard pilot integration. |
| Phase 2 | Passwordless WebAuthn primary; Apple App Store and Google Play release polish; Spanish locale content; Offline sync intake; Secret rotation policies and approval workflows; expiration alerts; multi-provider IDV; trusted-device MFA management UI; FIDO2 hardware keys for admin; per-tenant DKIM/SPF wizard; AWS-ready posture validation; full S3 Object Lock cold archive on AWS; per-tenant Cloudflare Access for tenant admin portals; advanced policy engine review (only if needed). |
| Phase 3 | Okta and Microsoft Entra ID federation; Auth0 federation; SAML enterprise IdP federation; CJIS-oriented controls for opted-in PEDL law-enforcement integration; dynamic database credentials via OpenBao or AWS Secrets Manager rotation; risk-based authentication using device, geo, behavioral, and SIM-swap signals; AWS-native production deployment; SIEM integration (CloudWatch, Datadog, Splunk); NATS or EventBridge cross-service event bus; service mesh and workload identity (SPIFFE/SPIRE). |

# 25. Risks and Mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Custom OIDC issuer is a security commitment, not a feature. | Vulnerabilities in HEIMDALL Core's auth layer affect all Myriad apps. | Use `oidc-provider` (Filip Skokan, OpenID Foundation certified). External security audit before production launch ($25K–60K budget). Subscribe to OAuth and OIDC mailing lists. Track CVEs on all auth dependencies. Designated auth-domain engineer on the team. |
| Supabase vendor concentration. | Outages, deprecations, pricing changes affect us directly. | Postgres-standard SQL only. Nightly logical backups to non-Supabase storage. Documented 90-day migration runbook to AWS RDS/Aurora. |
| Overbuilding identity infrastructure before app traction. | Slower product delivery and higher ops burden. | Single-service HEIMDALL Core. Defer Phase 2 and Phase 3 capabilities until traction justifies them. |
| Master key loss. | Encrypted PII and secrets unrecoverable. | Shamir's Secret Sharing 3-of-5 escrow plan (Appendix B). Quarterly rehearsal of recovery procedure. |
| Master key compromise. | Full secret and PII exposure. | Key versioning and zero-downtime rotation. Documented breach response runbook (Appendix C). |
| SMS pumping fraud. | $50K+ overnight cost spikes. | Per-country cost caps. Daily global cost cap. Twilio carrier risk signals. Cloudflare rate limiting at the edge. |
| Identity merge mistakes. | Privacy and account integrity issues. | Explicit user action required. 7-day user reversal, 30-day admin reversal window. Reason codes audited. |
| Minor consent gaps. | Legal and trust exposure. | Guardian relationships and consent records in MVP. Jurisdiction-flexible schema for CA, FL, UT, GDPR-K, future. |
| White-label identity confusion. | User trust and privacy issues. | Explicit cross-signup opt-in flow with disclosed wording reviewed by legal. |
| AWS migration friction. | Costly refactor later. | Postgres portability, OCI containers, provider interfaces (`SecretsService`, `SmsProvider`, `EmailProvider`, `IdvProvider`), event abstraction (outbox table) all designed from day one. |
| Compliance creep (CJIS, HIPAA, FedRAMP). | Scope expansion. | Architectural posture preserves the path. Realm-isolated tenants and tenant-residency policy in place. Costs and timelines re-estimated when a signed client need arises. |
| Audit log immutability vs right-to-erasure conflict. | Legal contradiction. | Audit ledgers reference parent_id, never PII. PII stored only in identity tables and tombstoned on anonymization. Cold archive carries no PII. |
| Native mobile app store policy changes. | Sign-in features blocked. | Apple Sign-In included from MVP per App Store rule. Quarterly review of App Store and Play Store policy updates. |

# 26. Development Prompting Package

This section provides a sequence of prompts intended to drive implementation work efficiently. They are reproduced here so that they remain coupled to the requirements baseline.

### Prompt 1 — Architecture Decision Record

Create an Architecture Decision Record (ADR) for HEIMDALL V3 with the TypeScript single-service architecture, custom OIDC issuer using `oidc-provider`, pgcrypto envelope encryption, in-process authorization, Postgres LISTEN/NOTIFY plus outbox, no Keycloak, no OPA, no OpenBao for MVP, Vercel + Render + Supabase Postgres deployment, AWS-ready posture, and Cloudflare edge. Include context, decision, alternatives rejected (Keycloak, Ory Kratos+Hydra, OpenBao, OPA, NATS, polyglot Elixir), consequences, security boundaries, fallback paths, and migration triggers.

### Prompt 2 — Database Schema

Create a Drizzle ORM TypeScript schema for HEIMDALL Core covering: applications, application_environments, application_minor_features, tenants, tenant_domains, tenant_branding, tenant_policies (versioned), tenant_policy_versions, parent_identities, credential_identities, identity_link_edges, identity_link_edges_history, app_personas, persona_switch_events, tenant_memberships, roles, permissions, role_permissions, role_permission_changes, persona_roles, guardian_relationships, guardian_consents, minor_profiles, age_policy_rules, application_minor_features, guest_identities, guest_upgrade_events, offline_identity_events, offline_sync_batches, offline_conflict_records, device_installations, verification_records, verification_sources, verification_policy_rules, session_records, trusted_devices, trusted_devices_2fa, device_risk_events, secret_collections, secret_items, secret_versions, secret_access_policies, secret_access_grants, secret_rotation_policies, secret_rotation_events, service_accounts, service_account_credentials, service_account_grants, disclosures, disclosure_versions, disclosure_acceptances, communication_templates, communication_template_versions, communication_log, anonymization_requests, legal_holds, support_session_events, impersonation_approvals, audit_events, pii_change_events, deletion_requests, break_glass_requests, break_glass_sessions, idp_clients, api_clients, webhook_endpoints, webhook_deliveries, payment_provider_customer_refs, rate_limit_policies. Include hash-chain trigger DDL for the three immutable ledgers.

### Prompt 3 — OIDC Issuer Implementation

Create an `oidc-provider` configuration for HEIMDALL Core including: client registration model backed by `idp_clients`; adapter implementation against Postgres; account resolver against `parent_identities` and `credential_identities`; interactions URL pointing to the HEIMDALL Hosted Login Pages on Vercel; PKCE required for all public clients; refresh token rotation; offline_access scope; custom claims emission (parent_id, persona_id, tenant_id, active_persona, mfa_level, `act` for impersonation); revocation, introspection, userinfo endpoints; JWKS rotation. Include the upstream federation modules for Google, Facebook (raw OAuth2), Apple, and prepared SAML and OIDC enterprise federation adapters.

### Prompt 4 — Secrets and PII Encryption Design

Create the `SecretsService` module with: pgcrypto envelope encryption (per-row data keys wrapped by master key); master key versioning and rotation runbook; in-memory TTL cache (60–300 seconds); audit event emission on every read/write/rotation; PII encryption helpers for credential_identities and minor_profiles; HMAC helpers for searchable fields; documented adapter pattern for future OpenBao and AWS Secrets Manager implementations.

### Prompt 5 — API Specification

Create an OpenAPI 3.1 specification for HEIMDALL Core covering: OIDC standard endpoints (authorize, token, userinfo, revoke, introspect, end_session, jwks); registration and account recovery; MFA enrollment and challenge; credential management (email, phone, social, WebAuthn); identity linking and unlinking; persona switching; tenant configuration; disclosure publishing and acceptance; communication template management; anonymization request submission and management; legal hold placement and release; impersonation request and approval; admin operations; audit and PII change log query; webhook subscription. Document auth scope, request, response, errors, and MVP/Phase designation for each endpoint.

### Prompt 6 — MVP Backlog

Create a GitHub-ready MVP backlog for HEIMDALL with epics, issues, acceptance criteria, labels, priority, estimates, dependencies, and target repositories for Cloudflare setup, Supabase setup, schema, HEIMDALL Core scaffolding, OIDC issuer, authentication flows (password, social, SMS, magic-link, WebAuthn), MFA, identity graph services, tenant configuration, disclosures, communications, anonymization, legal holds, support and impersonation, audit and PII change logging, secrets module, authorization service, PEDL pilot, Quest pilot, Myriad Dashboard pilot.

### Prompt 7 — Operational Runbooks

Produce operational runbooks for: master key rotation (zero downtime); key recovery via Shamir's Secret Sharing reconstruction; breach response (sessions revocation, secret rotation, user notification, forensics); anonymization execution audit; legal hold placement and override dual-control; impersonation approval audit; SMS cost cap response; credential stuffing response; Twilio outage failover; SendGrid outage failover; Supabase outage failover; AWS migration cutover.

### Prompt 8 — Frontend Integration Contracts

Produce a TypeScript SDK package (`@heimdall/client`) with: OIDC client integration helper; React hooks for current identity, active persona, tenant context, MFA state; impersonation banner React component (single import, mounts at app root); session refresh; logout; consent re-acceptance modal; disclosure version checker; standard error handling. Document the integration contract for non-React frontends and native iOS/Android apps.

# 27. Recommended Build Order

1. Approve this Architecture Decision Record.
2. Create local Docker Compose for HEIMDALL Core dev (Postgres, HEIMDALL Core in watch mode, Twilio mock, SendGrid mock).
3. Scaffold monorepo (Turborepo + pnpm) with `apps/core`, `apps/admin`, `apps/login`, and `packages/*`.
4. Create HEIMDALL Drizzle schema and migration framework.
5. Implement `SecretsService` module with pgcrypto envelope encryption.
6. Scaffold HEIMDALL Core: Fastify, OpenAPI, logging, OpenTelemetry, health endpoints, request context, error handling.
7. Implement Parent Identity service.
8. Implement Credential Identity service with explicit linking model and PII encryption helpers.
9. Implement OIDC issuer (`oidc-provider`) integrated against Parent and Credential services.
10. Implement password authentication flow with Argon2id.
11. Implement TOTP MFA via `otplib` and WebAuthn via `@simplewebauthn/server`.
12. Implement SMS authentication flow via Twilio Verify (with provider abstraction).
13. Implement email magic-link flow and SendGrid integration.
14. Implement Google, Facebook, Apple social federation.
15. Implement Persona service and persona switching.
16. Implement Tenant service and Tenant Configuration Registry.
17. Implement Authorization Service with starter roles and table-driven permissions.
18. Implement Disclosures and Communications Registry.
19. Implement Minor and Guardian service with COPPA age-gate policy.
20. Implement Guest Identity service with device-bound persistence and upgrade flow.
21. Implement audit_events with hash chain and triggers.
22. Implement pii_change_events with triggers on PII tables.
23. Implement communication_log with delivery webhooks from SendGrid and Twilio.
24. Implement Anonymization workflow (30-day flow, partial scopes, pg-boss job).
25. Implement Legal Hold service with dual-control override.
26. Implement Support and Impersonation framework with async approval and `act` claim emission.
27. Implement HEIMDALL Admin UI (Next.js on Vercel).
28. Implement HEIMDALL Hosted Login Pages (Next.js on Vercel, white-label themed).
29. Integrate Cloudflare WAF, Turnstile, and rate limiting.
30. Integrate `@heimdall/client` TypeScript SDK in Myriad Dashboard.
31. Integrate `@heimdall/client` in PEDL frontend; coordinate banner rendering contract.
32. Integrate `@heimdall/client` in Quest frontend; coordinate banner rendering contract.
33. Conduct external security audit focused on OIDC, OAuth, session management, MFA, and impersonation.
34. Phase 2: offline sync, secret rotation, Spanish locale, FIDO2 admin, AWS migration validation.

# 28. Appendices

The detailed runbooks, data model schema, and API specification are maintained in companion documents:

- **Appendix A — Source Notes** (this document): see below.
- **Companion: Data Model and Schema** — full Drizzle schema, DDL, indexes, triggers, hash-chain implementation, partition strategy.
- **Companion: API Specification** — OpenAPI 3.1 spec covering all HEIMDALL Core endpoints.
- **Companion: Security and Operations Runbooks** — Key Recovery, Breach Response, Anonymization Execution, Impersonation Approval, SMS Cost-Cap Response, Provider Outage Failover, AWS Migration.
- **Companion: Development Backlog** — MVP epics, issues, acceptance criteria, dependencies, estimates.

# Appendix A — Source Notes

These references support platform capability assumptions used in this document. They should be reviewed again before final implementation and production deployment.

| Source | Relevant Point |
| --- | --- |
| `oidc-provider` (npm: panva/node-oidc-provider) | Certified OpenID Connect implementation in Node.js maintained by Filip Skokan, OpenID Foundation certified implementer. |
| `openid-client` (npm: panva/openid-client) | Companion library for upstream OIDC client federation. |
| `@node-saml/node-saml` | Maintained SAML 2.0 implementation in Node.js. |
| `@simplewebauthn/server` | TypeScript-native WebAuthn / FIDO2 server library. |
| `@node-rs/argon2` | Native-speed Argon2id password hashing. |
| `otplib` | RFC 6238 TOTP implementation; standard `otpauth://` URI. |
| `jose` | JOSE / JWT / JWE / JWK library; Filip Skokan. |
| Fastify | High-performance TypeScript HTTP framework with schema-first design. |
| Drizzle ORM and Drizzle Kit | Type-safe SQL with portable migration files. |
| pg-boss | Postgres-backed job queue with no Redis dependency. |
| Twilio Verify | SMS and voice OTP delivery with carrier fraud and SIM-swap signals. |
| Cloudflare WAF and Turnstile | Edge security, rate limiting, and privacy-respecting CAPTCHA. |
| Supabase PostgreSQL documentation | Postgres hosting; HEIMDALL uses Postgres-standard SQL only. |
| Vercel Functions and Hosting documentation | Frontend hosting platform. |
| Render container deployment documentation | Container hosting platform for HEIMDALL Core. |
| AWS RDS / Aurora / ECS Fargate / KMS / Secrets Manager / S3 Object Lock documentation | Phase 2 AWS migration targets. |
| Apple Sign-In Documentation and App Store Review Guidelines section 4.8 | Apple Sign-In is required when other social login is offered in iOS apps. |
| OAuth 2.0 Token Exchange (RFC 8693) | `act` claim semantics for support impersonation. |
| OIDC Core 1.0 and OAuth 2.0 Security Best Current Practice | Foundational standards for the custom OIDC issuer. |
| GDPR Article 17, CCPA / CPRA, COPPA, California AB 2273, Florida HB 3, Utah Social Media Regulation Act | Jurisdictional drivers for the anonymization, minor, and disclosures domains. |
| HEIMDALL Final Requirements and Design V1 (2026-05-19) | Predecessor baseline document, superseded by this V3. |

HEIMDALL Final Requirements and Design V3 | Myriad Technology Group
