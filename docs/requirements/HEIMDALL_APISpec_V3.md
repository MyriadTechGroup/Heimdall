# HEIMDALL — API Specification

## Companion Document to HEIMDALL Requirements V3

Prepared for Myriad Technology Group | 2026-05-19

| Document Field | Value |
| --- | --- |
| Document Type | API Specification (companion to V3 Requirements) |
| API Style | REST + OpenID Connect 1.0 + OAuth 2.0 |
| Specification Format | OpenAPI 3.1 (this document presents the human-readable form; machine-readable YAML lives in `packages/openapi/heimdall.yaml`) |
| Transport | HTTPS only. TLS 1.2 minimum. TLS 1.3 preferred. |
| Default Host | `https://auth.myriad.com` (production); `https://auth.staging.myriad.com` (staging); `https://auth.dev.myriad.com` (dev). |
| Default Content Type | `application/json; charset=utf-8` for HEIMDALL APIs; `application/x-www-form-urlencoded` for OIDC token endpoint per RFC 6749. |
| Authentication | Bearer JWT (RFC 6750) for HEIMDALL admin and identity APIs. OAuth 2.0 client authentication (per RFC 6749 §2.3 and OIDC Core §9) for token-endpoint clients. |
| Versioning | URL path versioning (`/v1/...`). Backward-compatible additions do not bump version. Breaking changes increment the version. |

# 1. API Surface Overview

| API Group | Base Path | Purpose |
| --- | --- | --- |
| OIDC Discovery and Endpoints | `/.well-known/...`, `/oidc/...` | Standard OIDC and OAuth 2.0 endpoints handled by `oidc-provider`. |
| Authentication Flows | `/v1/auth/...` | Login, registration, MFA challenge and enrollment, magic link, recovery, social federation callbacks. |
| Identity | `/v1/identities/...` | Parent identity management, credential management, persona management, linking and unlinking. |
| Tenant Configuration | `/v1/tenants/...` | Tenant creation, branding, policy versioning, domain verification. |
| Minor and Guardian | `/v1/minors/...`, `/v1/guardians/...` | Minor profiles, guardian relationships, consent grants. |
| Verification | `/v1/verification/...` | Identity verification requests, results, provider abstraction. |
| Disclosures and Communications | `/v1/disclosures/...`, `/v1/communications/...` | Versioned disclosures, acceptances, communication templates and history. |
| Anonymization | `/v1/anonymization/...` | Request, cancel, and execute anonymization workflows. |
| Legal Holds | `/v1/legal-holds/...` | Place, release, override. |
| Support and Impersonation | `/v1/support/...` | Request impersonation, approve, exit, audit. |
| Authorization | `/v1/authorize/check`, `/v1/roles/...`, `/v1/permissions/...` | Decision endpoint and role/permission management. |
| Secrets | `/v1/secrets/...` | Secret metadata, access, rotation, revocation. |
| Service Accounts | `/v1/service-accounts/...` | Service account management, client credentials. |
| Audit | `/v1/audit/...` | Audit event and PII change event query. |
| Webhooks | `/v1/webhooks/...` | Webhook endpoint registration, delivery history, replay. |
| Admin Operations | `/v1/admin/...` | Cross-cutting admin endpoints (break-glass, system status, metrics). |
| Health and Readiness | `/healthz`, `/readyz` | Operational probes. |

# 2. Authentication and Authorization for API Calls

## 2.1 Bearer JWT (Standard Path)

Every HEIMDALL API endpoint outside `/oidc/`, `/.well-known/`, `/v1/auth/*`, `/healthz`, and `/readyz` requires `Authorization: Bearer <jwt>`. The JWT is issued by HEIMDALL Core's OIDC issuer and includes the claims:

| Claim | Source | Purpose |
| --- | --- | --- |
| `sub` | parent_id | Resource owner. |
| `iss` | HEIMDALL Core issuer URL | Token validation. |
| `aud` | Requesting client_id and any granted resource servers | Audience restriction. |
| `iat` and `exp` | Issued and expiration timestamps | Lifetime. |
| `scope` | Space-separated granted scopes | Authorization. |
| `azp` | Authorized presenter (client_id) | RFC 7519. |
| `client_id` | OIDC client | Logging. |
| `persona_id` | Active persona at session establishment | Authorization. |
| `tenant_id` | Tenant scope | Authorization. |
| `mfa_level` | `none`, `single_factor`, `multi_factor`, `strong_multi_factor` | Step-up checks. |
| `mfa_methods` | Methods completed in this session | Diagnostics. |
| `auth_time` | Original authentication timestamp | Step-up freshness. |
| `act` | RFC 8693 actor claim (impersonation only) | Support framework. |

## 2.2 Service Accounts (Client Credentials Grant)

Service accounts authenticate via OAuth 2.0 client credentials grant at `/oidc/token` with `grant_type=client_credentials`. They receive a JWT with `sub=<service_account_id>`, `client_id=<api_client_id>`, and a fixed permission set derived from `service_account_grants`.

## 2.3 Authorization Decisions

Every endpoint declares its required permissions. HEIMDALL Core's authorization middleware evaluates the requesting JWT's `sub`, `persona_id`, `tenant_id`, and `mfa_level` against `persona_roles`, `role_permissions`, and any required step-up conditions. Authorization is in-process (no OPA). Denied requests return `403 Forbidden` with an error code; the deny is audited.

## 2.4 Step-Up Triggers

Endpoints marked **MFA Step-Up** in their definitions require:

- The session's `mfa_level` is `multi_factor` or `strong_multi_factor`.
- The MFA was completed within the freshness window (default 5 minutes; configurable via `session_policy`).

If unsatisfied, the response is `401 Unauthorized` with `WWW-Authenticate: Bearer error="insufficient_mfa", mfa_required="<freshness>"`. The frontend prompts for re-MFA and retries.

# 3. Standard Errors

All errors return a standard envelope:

```json
{
  "error": {
    "code": "ERROR_CODE_SCREAMING_SNAKE",
    "message": "Human-readable summary (never leaks PII or internals).",
    "details": [
      { "field": "email", "issue": "INVALID_FORMAT" }
    ],
    "correlation_id": "01J5K8YGZ4XPN6PMRC2H8JZHGN",
    "documentation_url": "https://docs.heimdall.myriad.com/errors/ERROR_CODE"
  }
}
```

| HTTP Status | When |
| --- | --- |
| 400 | Validation failures. `error.code` describes which input. |
| 401 | Missing or invalid token. |
| 401 with `insufficient_mfa` | Step-up required. |
| 403 | Token valid but lacks permission, or operation blocked by policy. |
| 404 | Resource not found. PII-bearing endpoints prefer 404 over 403 when the requester would not normally know the resource exists. |
| 409 | Conflict (duplicate credential, version mismatch, idempotency replay). |
| 410 | Resource gone (anonymized). |
| 422 | Semantic validation failure (e.g., minor cannot self-grant consent). |
| 423 | Resource locked (legal hold). |
| 429 | Rate-limited. `Retry-After` header set. |
| 500 | Internal server error. `correlation_id` traceable in audit log. |
| 503 | Service degraded (provider outage). |

# 4. OIDC and OAuth 2.0 Endpoints

These endpoints are implemented by `oidc-provider`. HEIMDALL Core exposes them at the issuer base URL.

## 4.1 GET `/.well-known/openid-configuration`

Returns the OIDC discovery document including `authorization_endpoint`, `token_endpoint`, `userinfo_endpoint`, `jwks_uri`, `end_session_endpoint`, `revocation_endpoint`, `introspection_endpoint`, supported response types, supported scopes, supported grant types, supported claims, supported PKCE methods (S256 required), and `id_token_signing_alg_values_supported`.

## 4.2 GET `/.well-known/jwks.json`

Returns the JSON Web Key Set used to sign JWTs. Keys rotate per the documented schedule (default every 90 days; rotated keys remain in JWKS for 48 hours after retirement to satisfy in-flight tokens).

## 4.3 GET `/oidc/auth` (Authorization Endpoint)

Initiates the authorization code flow. Parameters per OIDC Core §3.1.2.1 (`response_type`, `client_id`, `redirect_uri`, `scope`, `state`, `nonce`, `code_challenge`, `code_challenge_method`, `prompt`, `max_age`, `acr_values`, `login_hint`, `ui_locales`).

HEIMDALL Core's authorization endpoint either:

1. Renders an interaction (login, consent, MFA, disclosure re-acceptance) by redirecting to the Hosted Login Pages at `https://login.myriad.com/interaction/{uid}`, or
2. Returns the authorization response (302 redirect to the registered `redirect_uri` with `code` and `state`).

PKCE (`code_challenge` with `code_challenge_method=S256`) is required for all clients regardless of type.

## 4.4 POST `/oidc/token` (Token Endpoint)

Per OIDC Core §3.1.3 and OAuth 2.0 Token Exchange (RFC 8693).

Supported grant types:

| Grant | Purpose |
| --- | --- |
| `authorization_code` | Standard interactive flow. |
| `refresh_token` | Refresh access tokens. Rotation enabled. |
| `client_credentials` | Service account tokens. |
| `urn:ietf:params:oauth:grant-type:token-exchange` | Support impersonation per RFC 8693. Issued only after an approved `impersonation_approvals` record. |

Token-exchange request example:

```http
POST /oidc/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <support-client-credentials>

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<support_agent_jwt>
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&actor_token=<support_agent_jwt>
&actor_token_type=urn:ietf:params:oauth:token-type:access_token
&audience=pedl-api
&heimdall_impersonation_approval_id=<approval_id>
&heimdall_target_subject=<impersonated_parent_id>
```

Response includes a token with the `act` claim populated.

## 4.5 GET `/oidc/userinfo`

Returns claims for the authenticated user per OIDC Core §5.3. Claims emitted depend on granted scopes (`profile`, `email`, `phone`, `address`) and per-client claim mappers.

## 4.6 POST `/oidc/token/introspection`

RFC 7662 token introspection. Confidential clients only.

## 4.7 POST `/oidc/token/revocation`

RFC 7009 token revocation.

## 4.8 GET `/oidc/session/end`

RP-Initiated Logout per OIDC RP-Initiated Logout 1.0. Revokes the OIDC session and redirects to `post_logout_redirect_uri` if registered.

## 4.9 GET `/oidc/interaction/{uid}`

Interaction details endpoint consumed by the Hosted Login Pages frontend. Returns the current interaction state (what step the user is on: identification, password, MFA, consent, disclosure re-acceptance).

## 4.10 POST `/oidc/interaction/{uid}/login`

Submits credentials to complete the login step of an interaction.

## 4.11 POST `/oidc/interaction/{uid}/confirm`

Submits consent or interaction confirmation.

## 4.12 POST `/oidc/interaction/{uid}/abort`

Aborts the interaction. Redirects back to the client with `error=access_denied`.

# 5. Authentication Flows (HEIMDALL-Specific)

These endpoints sit alongside the OIDC interactions and provide direct API access for native apps and integrations that prefer not to use the interactive web flow.

## 5.1 POST `/v1/auth/register`

**Auth:** None.
**Purpose:** Register a new Parent UUID and primary credential.
**Body:**
```json
{
  "tenant_slug": "myriad-products",
  "app_code": "pedl",
  "credential_type": "email",
  "email": "user@example.com",
  "password": "<plaintext, TLS only>",
  "display_name": "Alex Rider",
  "locale": "en-US",
  "age_attestation": { "is_at_least_18": true, "jurisdiction_hint": "US-NC" },
  "disclosure_acceptances": [
    { "disclosure_version_id": "...", "accepted": true },
    { "disclosure_version_id": "...", "accepted": true }
  ],
  "device_fingerprint": "abc123",
  "captcha_token": "<turnstile>"
}
```
**Responses:** `201 Created` with `parent_id`, `pending_verification` flag, verification instructions; `409` on duplicate credential.

## 5.2 POST `/v1/auth/login/password`

**Auth:** None.
**Body:** `{ tenant_slug, app_code, identifier, password, captcha_token, device_fingerprint }`
**Responses:** `200` with `oidc_login_continuation_token` plus `mfa_required` boolean; `401` on bad credentials; `429` on rate limit; `423` on locked account.

## 5.3 POST `/v1/auth/login/sms/start`

**Auth:** None.
**Body:** `{ tenant_slug, phone, captcha_token, device_fingerprint }`
**Responses:** `202 Accepted` with `challenge_id` and `expires_in`. SMS sent via Twilio Verify. Subject to SMS rate-limit and cost-cap policies.

## 5.4 POST `/v1/auth/login/sms/verify`

**Auth:** None.
**Body:** `{ challenge_id, code, device_fingerprint }`
**Responses:** `200` with `oidc_login_continuation_token`; `401` on bad code; `429` on too many attempts.

## 5.5 POST `/v1/auth/login/magic-link/start`

**Auth:** None.
**Body:** `{ tenant_slug, email, captcha_token, device_fingerprint }`
**Responses:** `202 Accepted`. Email sent via SendGrid. Always returns 202 regardless of email existence (no enumeration).

## 5.6 GET `/v1/auth/login/magic-link/consume`

**Auth:** None. Token in query string.
**Query:** `token=<single-use magic link token>`
**Responses:** `200` with `oidc_login_continuation_token`; `410` if token expired or already consumed.

## 5.7 GET `/v1/auth/login/social/{provider}/start`

**Auth:** None.
**Path:** `provider` ∈ `google`, `facebook`, `apple`.
**Query:** `tenant_slug`, `app_code`, `device_fingerprint`, `state`.
**Responses:** `302` to upstream IdP authorization endpoint.

## 5.8 GET `/v1/auth/login/social/{provider}/callback`

**Auth:** None.
**Purpose:** OAuth callback from upstream IdP.
**Responses:** `302` to next step (interaction completion, MFA challenge, or final redirect).

## 5.9 POST `/v1/auth/mfa/challenge`

**Auth:** Login continuation token (interim, not full JWT).
**Body:** `{ continuation_token, method_id, method_type }`
**Responses:** `202` with `challenge_id` and `expires_in`.

## 5.10 POST `/v1/auth/mfa/verify`

**Auth:** Login continuation token.
**Body:** `{ continuation_token, challenge_id, code }`  (for TOTP, SMS, email_otp, backup code)
or `{ continuation_token, challenge_id, webauthn_response }` (for WebAuthn).
**Responses:** `200` with `oidc_login_continuation_token` advanced to MFA-complete; `401` on bad code; `429` on too many attempts.

## 5.11 POST `/v1/auth/mfa/enroll/{method_type}/start`

**Auth:** Bearer JWT. **MFA Step-Up** required if adding to an account that already has MFA.
**Path:** `method_type` ∈ `totp`, `sms`, `webauthn`, `email_otp`.
**Body:** method-specific.
**Responses:** `200` with enrollment artifacts (TOTP `otpauth://` URI and QR code data URL; WebAuthn `PublicKeyCredentialCreationOptions`).

## 5.12 POST `/v1/auth/mfa/enroll/{method_type}/complete`

**Auth:** Bearer JWT.
**Body:** verification of the enrollment (TOTP first code; WebAuthn attestation).
**Responses:** `200` with enrolled `method_id` and (on first MFA enrollment) backup codes returned exactly once.

## 5.13 POST `/v1/auth/password/reset/start`

**Auth:** None.
**Body:** `{ tenant_slug, identifier, captcha_token }`
**Responses:** `202` always (no enumeration). Email or SMS sent with reset link.

## 5.14 POST `/v1/auth/password/reset/complete`

**Auth:** Reset token from email or SMS.
**Body:** `{ reset_token, new_password }`
**Responses:** `200`; `410` if expired; `422` on policy violation.

## 5.15 POST `/v1/auth/password/change`

**Auth:** Bearer JWT. **MFA Step-Up** required.
**Body:** `{ current_password, new_password }`
**Responses:** `200`; `401` on current_password mismatch; `422` on policy violation.

# 6. Identity Endpoints

## 6.1 GET `/v1/identities/me`

**Auth:** Bearer JWT.
**Permissions:** Self.
**Returns:** Parent identity record (decrypted display name, locale, timezone, status), enrolled credentials (display values masked), enrolled MFA methods (no secret material), enrolled personas, tenant memberships, current legal hold status, current anonymization request status.

## 6.2 GET `/v1/identities/{parent_id}`

**Auth:** Bearer JWT.
**Permissions:** `user.view` (Myriad admin) or `tenant.user.view` (tenant admin within tenant scope).
**Returns:** Same shape as `/v1/identities/me`. Generates a `pii_change_events` row of type `view` if the requester is a support role.

## 6.3 PATCH `/v1/identities/me`

**Auth:** Bearer JWT.
**Body:** `{ display_name?, locale?, timezone? }`
**Effect:** Updates parent identity. Triggers `pii_change_events` capture.

## 6.4 GET `/v1/identities/me/credentials`

**Auth:** Bearer JWT.
**Returns:** List of credentials (type, masked display, status, verified_at, is_primary).

## 6.5 POST `/v1/identities/me/credentials`

**Auth:** Bearer JWT. **MFA Step-Up** required.
**Body:** `{ credential_type, value, verification_method }`
**Responses:** `202 Accepted`. Verification email or SMS sent.

## 6.6 POST `/v1/identities/me/credentials/{credential_id}/verify`

**Auth:** Bearer JWT.
**Body:** `{ code }` (for SMS) or `{ token }` (for email link).
**Responses:** `200` on success; `410` on expired.

## 6.7 DELETE `/v1/identities/me/credentials/{credential_id}`

**Auth:** Bearer JWT. **MFA Step-Up** required.
**Effect:** Soft-revokes the credential. Blocked if it is the last active credential of any kind.

## 6.8 POST `/v1/identities/me/credentials/{credential_id}/primary`

**Auth:** Bearer JWT.
**Effect:** Sets the credential as primary for its type.

## 6.9 POST `/v1/identities/me/link/start`

**Auth:** Bearer JWT. **MFA Step-Up** required.
**Body:** `{ other_credential_type, other_value }`
**Effect:** Initiates explicit identity linking. Sends a verification code to the other credential.
**Responses:** `202` with `link_challenge_id`.

## 6.10 POST `/v1/identities/me/link/complete`

**Auth:** Bearer JWT.
**Body:** `{ link_challenge_id, code }`
**Effect:** Creates an `identity_link_edges` row merging the two Parent UUIDs. The 7-day reversal window starts.

## 6.11 POST `/v1/identities/me/unlink/{link_edge_id}`

**Auth:** Bearer JWT. **MFA Step-Up** required.
**Effect:** Within the reversal window, reverses the merge. After the window, returns `410 Gone`.

## 6.12 POST `/v1/identities/{parent_id}/admin-merge`

**Auth:** Bearer JWT.
**Permissions:** `user.merge` (Myriad admin).
**Body:** `{ surviving_parent_id, merged_parent_id, reason_code, reason_notes }`
**Effect:** Admin-initiated merge with 30-day reversal window. Logged with reason.

## 6.13 GET `/v1/identities/me/personas`

**Auth:** Bearer JWT.
**Returns:** List of personas across all apps the user is enrolled in.

## 6.14 POST `/v1/identities/me/personas/switch`

**Auth:** Bearer JWT.
**Body:** `{ persona_id }`
**Effect:** Switches active persona. Updates session record. Emits `persona.switched` audit event and `persona_switch_events` row. Returns a refreshed JWT with the new `persona_id` claim.

## 6.15 GET `/v1/identities/me/sessions`

**Auth:** Bearer JWT.
**Returns:** Active sessions across all apps with device, IP, geo (country and region), last_active_at.

## 6.16 DELETE `/v1/identities/me/sessions/{session_id}`

**Auth:** Bearer JWT.
**Effect:** Revokes a specific session.

## 6.17 POST `/v1/identities/me/sessions/revoke-all`

**Auth:** Bearer JWT. **MFA Step-Up** required.
**Effect:** Revokes all sessions except the current one.

# 7. Tenant Configuration Endpoints

## 7.1 POST `/v1/tenants`

**Auth:** Bearer JWT. **Permissions:** `tenant.create` (Myriad admin).
**Body:** `{ slug, display_name, tier, primary_app_codes[] }`
**Responses:** `201` with tenant record.

## 7.2 GET `/v1/tenants/{tenant_id}`

**Auth:** Bearer JWT. **Permissions:** `tenant.view` or membership in tenant.

## 7.3 PATCH `/v1/tenants/{tenant_id}/branding`

**Auth:** Bearer JWT. **Permissions:** `tenant.config.update`.
**Body:** Partial branding payload.

## 7.4 POST `/v1/tenants/{tenant_id}/domains`

**Auth:** Bearer JWT. **Permissions:** `tenant.config.update`.
**Body:** `{ domain }`
**Effect:** Creates a domain record in `pending` verification status. DNS TXT challenge returned.

## 7.5 POST `/v1/tenants/{tenant_id}/domains/{domain_id}/verify`

**Auth:** Bearer JWT. **Permissions:** `tenant.config.update`.
**Effect:** Re-checks DNS TXT record and DKIM/SPF status.

## 7.6 GET `/v1/tenants/{tenant_id}/policies`

**Auth:** Bearer JWT. **Permissions:** `tenant.view`.
**Returns:** All active policy versions with metadata.

## 7.7 POST `/v1/tenants/{tenant_id}/policies/{policy_category}/versions`

**Auth:** Bearer JWT. **Permissions:** `tenant.policy.update`. **MFA Step-Up** required.
**Body:** `{ body, change_summary, effective_at }`
**Effect:** Creates a new draft policy version. Activation is a separate call to preserve a review checkpoint.

## 7.8 POST `/v1/tenants/{tenant_id}/policies/{policy_category}/versions/{version_id}/activate`

**Auth:** Bearer JWT. **Permissions:** `tenant.policy.update`. **MFA Step-Up** required.
**Effect:** Activates the version. Previous active version's `superseded_at` set.

## 7.9 GET `/v1/tenants/{tenant_id}/policies/{policy_category}/versions/{version_id}/diff`

**Auth:** Bearer JWT. **Permissions:** `tenant.view`.
**Returns:** Diff against the prior active version.

# 8. Minor and Guardian Endpoints

## 8.1 POST `/v1/minors`

**Auth:** Bearer JWT.
**Permissions:** Self if establishing own minor profile during signup (when age attestation indicates minor); `minor.manage` for admin-created.
**Body:** `{ parent_id, declared_minor, age_at_signup, jurisdiction_code, attestation_method, dob? }`
**Responses:** `201`. If `requires_guardian_consent` per `age_policy_rules`, response includes `guardian_invite_token` to share.

## 8.2 GET `/v1/minors/{parent_id}`

**Auth:** Bearer JWT. **Permissions:** Guardian, self (if of-age and consenting to view own record), or admin.

## 8.3 POST `/v1/guardians/relationships`

**Auth:** Bearer JWT.
**Body:** `{ minor_parent_id, relationship_type, guardian_invite_token? }`
**Effect:** Creates a `guardian_relationships` row. The guardian must already have a verified Parent UUID.

## 8.4 POST `/v1/guardians/consent`

**Auth:** Bearer JWT (guardian).
**Body:** `{ minor_parent_id, disclosure_version_id, scope }`
**Effect:** Records guardian consent in `disclosure_acceptances` on behalf of the minor.

## 8.5 DELETE `/v1/guardians/consent/{acceptance_id}`

**Auth:** Bearer JWT (guardian).
**Effect:** Revokes consent. Triggers `consent.revoked` audit event and downstream restriction of the minor's account features.

## 8.6 POST `/v1/minors/{parent_id}/age-transition`

**Auth:** System or admin.
**Effect:** Internal endpoint invoked by the age-transition pg-boss job. Sets `age_transitioned_at` and marks all guardian relationships `lapsed_age_transition`.

# 9. Verification Endpoints

## 9.1 POST `/v1/verification/sessions`

**Auth:** Bearer JWT.
**Body:** `{ provider_code, verification_level, app_code?, callback_url }`
**Effect:** Creates a verification session via the configured provider. Returns the provider-specific session token / URL.

## 9.2 GET `/v1/verification/records`

**Auth:** Bearer JWT.
**Query:** `app_code?`, `level?`, `status?`
**Returns:** Verification records for the authenticated parent.

## 9.3 POST `/v1/verification/webhooks/{provider_code}`

**Auth:** Provider HMAC signature verified in `webhook_endpoints.signing_secret_ref`.
**Effect:** Ingest verification result from provider. Updates `verification_records`.

## 9.4 POST `/v1/verification/records/{id}/revoke`

**Auth:** Bearer JWT.
**Permissions:** `verification.revoke` (admin).
**Body:** `{ reason }`

# 10. Disclosures and Communications

## 10.1 GET `/v1/disclosures`

**Auth:** Bearer JWT or none (for unauthenticated signup flow).
**Query:** `tenant_slug?`, `app_code?`, `category?`, `locale?`
**Returns:** Active disclosure versions matching the scope, ordered by scope_layer (global, app, tenant).

## 10.2 POST `/v1/disclosures`

**Auth:** Bearer JWT. **Permissions:** `disclosure.create`.
**Body:** `{ category, scope_layer, app_code?, tenant_id?, display_name }`

## 10.3 POST `/v1/disclosures/{disclosure_id}/versions`

**Auth:** Bearer JWT. **Permissions:** `disclosure.publish`.
**Body:** `{ version, body_markdown, locale, effective_at, requires_re_acceptance, summary_of_changes }`

## 10.4 POST `/v1/disclosures/{disclosure_id}/versions/{version_id}/publish`

**Auth:** Bearer JWT. **Permissions:** `disclosure.publish`. **MFA Step-Up** required.
**Effect:** Sets `published_at`. Activates the version. Previous active version's `superseded_at` set. If `requires_re_acceptance=true`, every affected user is flagged for re-consent on next login.

## 10.5 GET `/v1/disclosures/{disclosure_id}/versions/{version_id}/diff`

**Auth:** Bearer JWT.
**Returns:** Diff against prior version.

## 10.6 POST `/v1/disclosures/acceptances`

**Auth:** Bearer JWT.
**Body:** `{ disclosure_version_ids[], accepted_via }`
**Effect:** Records acceptance(s) in `disclosure_acceptances` with full request context.

## 10.7 GET `/v1/disclosures/me/required-re-acceptance`

**Auth:** Bearer JWT.
**Returns:** List of versions the user must re-accept before continuing.

## 10.8 GET `/v1/communications/templates`

**Auth:** Bearer JWT. **Permissions:** `communication.view`.

## 10.9 POST `/v1/communications/templates/{template_id}/versions`

**Auth:** Bearer JWT. **Permissions:** `communication.author`.

## 10.10 GET `/v1/communications/me/log`

**Auth:** Bearer JWT.
**Query:** `since?`, `until?`, `channel?`
**Returns:** Communication log entries for the authenticated user (PII masked).

## 10.11 GET `/v1/communications/log`

**Auth:** Bearer JWT. **Permissions:** `communication.audit`.
**Query:** `parent_id?`, `since?`, `until?`, `channel?`, `status?`, `correlation_id?`

## 10.12 POST `/v1/communications/webhooks/sendgrid`, `/twilio`

**Auth:** Provider HMAC signature.
**Effect:** Ingest delivery, open, click, bounce, failure events into `communication_log`.

# 11. Anonymization Endpoints

## 11.1 POST `/v1/anonymization/requests`

**Auth:** Bearer JWT. **MFA Step-Up** required.
**Body:**
```json
{
  "scope": "full_account",
  "scope_target": null,
  "reason_notes": "optional"
}
```
or for per-app:
```json
{
  "scope": "per_app",
  "scope_target": { "app_code": "pedl" }
}
```
**Effect:** Creates `anonymization_requests` row. 30-day clock starts. Initial notification queued.
**Responses:** `201 Created` with `scheduled_execution_at`; `423 Locked` if a legal hold blocks (queued instead).

## 11.2 GET `/v1/anonymization/me/requests`

**Auth:** Bearer JWT.
**Returns:** All requests for the authenticated parent including notification history.

## 11.3 DELETE `/v1/anonymization/requests/{request_id}`

**Auth:** Bearer JWT.
**Effect:** Cancels the request if still in window.
**Responses:** `200` on cancel; `410` if already executed.

## 11.4 GET `/v1/anonymization/requests`

**Auth:** Bearer JWT. **Permissions:** `anonymization.audit`.
**Query:** `status?`, `since?`, `until?`, `tenant_id?`, `app_code?`

## 11.5 POST `/v1/anonymization/requests/admin`

**Auth:** Bearer JWT. **Permissions:** `anonymization.admin_initiate`. **MFA Step-Up** required.
**Body:** `{ parent_id, scope, scope_target?, initiator_type, legal_reference?, reason_notes }`
**Effect:** Support- or legally-initiated request on behalf of a user.

## 11.6 POST `/v1/anonymization/requests/{request_id}/execute-now`

**Auth:** Bearer JWT. **Permissions:** `anonymization.execute_now`. **MFA Step-Up** required.
**Body:** `{ reason }`
**Effect:** Bypass the 30-day window. Used only for legal compliance with formal reference. Audit-required.

## 11.7 POST `/v1/anonymization/admin/restore/{parent_id}`

**Auth:** Bearer JWT. **Permissions:** `anonymization.restore`. **MFA Step-Up** plus dual-control required.
**Body:** `{ reason, dual_control_approver_parent_id }`
**Effect:** Attempts manual restoration of an anonymized parent. Returns informational success or partial-failure detail. This is the only path; users cannot self-restore.

# 12. Legal Hold Endpoints

## 12.1 POST `/v1/legal-holds`

**Auth:** Bearer JWT. **Permissions:** `legal_hold.place` (`myriad_legal_officer`).
**Body:** `{ parent_id, authority, reference_id, reason_code, reason_notes, effective_from }`
**Effect:** Places legal hold. All sessions revoked. Pending anonymization requests queued.

## 12.2 GET `/v1/legal-holds`

**Auth:** Bearer JWT. **Permissions:** `legal_hold.view`.
**Query:** `parent_id?`, `active?`, `reference_id?`

## 12.3 POST `/v1/legal-holds/{hold_id}/release`

**Auth:** Bearer JWT. **Permissions:** `legal_hold.release` (`myriad_legal_officer`). **MFA Step-Up** required.
**Body:** `{ reason }`
**Effect:** Releases the hold. Queued anonymization requests resume.

## 12.4 POST `/v1/legal-holds/{hold_id}/override-requests`

**Auth:** Bearer JWT. **Permissions:** `legal_hold.override_request` (`myriad_super_admin`).
**Body:** `{ reason, requested_action }`
**Effect:** Creates a pending dual-control override request.

## 12.5 POST `/v1/legal-holds/override-requests/{override_id}/cosign`

**Auth:** Bearer JWT. **Permissions:** `legal_hold.override_cosign` (`myriad_super_admin`, must be different person).
**Effect:** Co-signs within the 60-minute window. Executes the requested action.

# 13. Support and Impersonation Endpoints

## 13.1 POST `/v1/support/impersonation/approvals`

**Auth:** Bearer JWT. **Permissions:** `support.impersonation_request`.
**Body:** `{ target_parent_id, ticket_reference, justification, requested_duration_minutes }`
**Effect:** Creates `impersonation_approvals` row in pending state. Notifies available approvers via Slack and email.

## 13.2 GET `/v1/support/impersonation/approvals/pending`

**Auth:** Bearer JWT. **Permissions:** `support.impersonation_approve`.
**Returns:** Pending approvals visible to the caller's approval scope.

## 13.3 POST `/v1/support/impersonation/approvals/{approval_id}/approve`

**Auth:** Bearer JWT. **Permissions:** `support.impersonation_approve`. **MFA Step-Up** required.
**Body:** `{ approved_duration_minutes?, notes? }`
**Effect:** Sets `approved_by_parent_id` and `approved_at`. Notifies requester. 30-minute window starts.

## 13.4 POST `/v1/support/impersonation/approvals/{approval_id}/deny`

**Auth:** Bearer JWT. **Permissions:** `support.impersonation_approve`.
**Body:** `{ denial_reason }`

## 13.5 POST `/v1/support/impersonation/sessions`

**Auth:** Bearer JWT (the approved support agent).
**Body:** `{ approval_id }`
**Effect:** Calls `/oidc/token` token-exchange grant under the hood. Returns an impersonation JWT with `act` claim. 15-minute initial token; 60-minute total ceiling.

## 13.6 POST `/v1/support/impersonation/sessions/renew`

**Auth:** Bearer JWT (impersonation token).
**Effect:** Renews the impersonation token for another 15 minutes up to the 60-minute ceiling. The impersonated user's UI may show a freshness prompt.

## 13.7 POST `/v1/support/impersonation/sessions/exit`

**Auth:** Bearer JWT (impersonation token).
**Effect:** Revokes the impersonation token immediately. Records `session_ended`.

## 13.8 GET `/v1/support/impersonation/sessions/{approval_id}/events`

**Auth:** Bearer JWT. **Permissions:** `support.impersonation_audit`.
**Returns:** `support_session_events` rows for the approved session.

# 14. Authorization Decision and Role Endpoints

## 14.1 POST `/v1/authorize/check`

**Auth:** Bearer JWT.
**Body:** `{ action, resource: { type, id, attributes? }, context? }`
**Returns:** `{ allowed: boolean, reasons: string[] }`. Useful for frontend gating where the application wants to ask before issuing the actual API call.

## 14.2 GET `/v1/roles`

**Auth:** Bearer JWT. **Permissions:** `role.view`.
**Query:** `tenant_id?`, `app_code?`, `scope?`

## 14.3 POST `/v1/roles`

**Auth:** Bearer JWT. **Permissions:** `role.create`.

## 14.4 POST `/v1/roles/{role_id}/permissions`

**Auth:** Bearer JWT. **Permissions:** `role.permission.manage`. **MFA Step-Up** required.
**Body:** `{ permission_ids[], action: "grant" | "revoke" }`

## 14.5 GET `/v1/permissions`

**Auth:** Bearer JWT. **Permissions:** `permission.view`.

## 14.6 POST `/v1/personas/{persona_id}/roles`

**Auth:** Bearer JWT. **Permissions:** `role.assign`.
**Body:** `{ role_id, expires_at? }`

## 14.7 DELETE `/v1/personas/{persona_id}/roles/{role_id}`

**Auth:** Bearer JWT. **Permissions:** `role.assign`.

# 15. Secrets Endpoints

## 15.1 GET `/v1/secrets`

**Auth:** Bearer JWT. **Permissions:** `secret.list`.
**Query:** `collection?`, `environment?`, `app_code?`, `tenant_id?`
**Returns:** Secret metadata only (collection, key, environment, scope, current_version, rotation policy, last access timestamp). Never values.

## 15.2 POST `/v1/secrets`

**Auth:** Bearer JWT. **Permissions:** `secret.create`. **MFA Step-Up** required.
**Body:** `{ collection_id, key, environment, app_code?, tenant_id?, value, description?, rotation_policy_id? }`
**Effect:** Encrypts the value via envelope encryption, creates `secret_items` and initial `secret_versions`.

## 15.3 GET `/v1/secrets/{secret_id}`

**Auth:** Bearer JWT. **Permissions:** `secret.view`.
**Returns:** Metadata only.

## 15.4 POST `/v1/secrets/{secret_id}/access`

**Auth:** Bearer JWT or service account JWT. **Permissions:** `secret.read` plus matching `secret_access_policies`.
**Effect:** Decrypts and returns the current version value. Generates `secret.accessed` audit event with requester, IP, and reason. Subject to in-memory TTL cache on the calling HEIMDALL Core instance.
**Responses:** Returns value once; never persisted to logs.

## 15.5 POST `/v1/secrets/{secret_id}/rotate`

**Auth:** Bearer JWT. **Permissions:** `secret.rotate`. **MFA Step-Up** required.
**Body:** `{ new_value, rotation_type, reason }`
**Effect:** Creates a new `secret_versions` row, sets it as current. Old version retained for the configured grace window.

## 15.6 POST `/v1/secrets/{secret_id}/revoke`

**Auth:** Bearer JWT. **Permissions:** `secret.revoke`. **MFA Step-Up** required.
**Body:** `{ reason }`
**Effect:** Marks the secret revoked. All future access denied.

## 15.7 POST `/v1/secrets/{secret_id}/grants`

**Auth:** Bearer JWT. **Permissions:** `secret.grant`. **MFA Step-Up** required.
**Body:** `{ parent_id, reason, expires_at }`
**Effect:** Creates `secret_access_grants` for time-bound break-glass access.

# 16. Service Account Endpoints

## 16.1 POST `/v1/service-accounts`

**Auth:** Bearer JWT. **Permissions:** `service_account.create`. **MFA Step-Up** required.

## 16.2 POST `/v1/service-accounts/{id}/credentials`

**Auth:** Bearer JWT.
**Body:** `{ credential_type }`
**Responses:** `201` with the credential value returned **once**. Never recoverable.

## 16.3 POST `/v1/service-accounts/{id}/grants`

**Auth:** Bearer JWT. **Permissions:** `service_account.grant`.
**Body:** `{ permission_id, scope_descriptor }`

# 17. Audit Endpoints

## 17.1 GET `/v1/audit/events`

**Auth:** Bearer JWT. **Permissions:** `audit.view`.
**Query:** `parent_id?`, `actor_parent_id?`, `event_category?`, `event_type?`, `since?`, `until?`, `correlation_id?`, `app_code?`, `tenant_id?`, `cursor?`, `limit?`
**Returns:** Paginated audit events. Cursor-based pagination.

## 17.2 GET `/v1/audit/events/{event_id}`

**Auth:** Bearer JWT. **Permissions:** `audit.view`.
**Returns:** Single event with `prev_hash` and `row_hash` for chain verification.

## 17.3 GET `/v1/audit/pii-changes`

**Auth:** Bearer JWT. **Permissions:** `audit.pii_view`.
**Query:** `parent_id?`, `actor_parent_id?`, `field_path?`, `change_type?`, `since?`, `until?`

## 17.4 GET `/v1/audit/me`

**Auth:** Bearer JWT.
**Returns:** Subject's own audit history (login events, persona switches, MFA enrollments, credential changes, consent grants). Used for GDPR right-to-know.

## 17.5 POST `/v1/audit/export`

**Auth:** Bearer JWT. **Permissions:** `audit.export`. **MFA Step-Up** required.
**Body:** `{ query, format: "jsonl" | "csv", correlation_id? }`
**Effect:** Queues an export job. Returns an export reference and an expected completion time.

## 17.6 GET `/v1/audit/exports/{export_id}`

**Auth:** Bearer JWT.
**Returns:** Export status and (when complete) signed download URL. Download URL expires in 15 minutes.

# 18. Webhook Endpoints

## 18.1 POST `/v1/webhooks/endpoints`

**Auth:** Bearer JWT. **Permissions:** `webhook.manage`.
**Body:** `{ display_name, url, event_types[], tenant_id?, app_code? }`
**Effect:** Creates an endpoint. Returns the signing secret one time.

## 18.2 GET `/v1/webhooks/endpoints`

**Auth:** Bearer JWT. **Permissions:** `webhook.view`.

## 18.3 POST `/v1/webhooks/endpoints/{id}/rotate-secret`

**Auth:** Bearer JWT. **Permissions:** `webhook.manage`. **MFA Step-Up** required.

## 18.4 GET `/v1/webhooks/deliveries`

**Auth:** Bearer JWT. **Permissions:** `webhook.view`.
**Query:** `endpoint_id?`, `event_id?`, `delivered?`, `since?`, `until?`

## 18.5 POST `/v1/webhooks/deliveries/{id}/replay`

**Auth:** Bearer JWT. **Permissions:** `webhook.manage`.
**Effect:** Re-queues the delivery.

# 19. Admin and Operational Endpoints

## 19.1 POST `/v1/admin/break-glass/requests`

**Auth:** Bearer JWT.
**Body:** `{ reason, requested_scope, requested_duration_minutes }`

## 19.2 POST `/v1/admin/break-glass/requests/{id}/approve`

**Auth:** Bearer JWT. **Permissions:** `break_glass.approve`. **MFA Step-Up** required.

## 19.3 GET `/v1/admin/status`

**Auth:** Bearer JWT. **Permissions:** `admin.status`.
**Returns:** Provider health (Twilio, SendGrid, IDV), queue depths, rotation pending, key version status, recent audit error rate.

## 19.4 GET `/v1/admin/metrics`

**Auth:** Bearer JWT. **Permissions:** `admin.metrics`.
**Returns:** Counts: active sessions, MFA enrollment rate, anonymization queue, legal hold count, impersonation sessions in progress, SMS spend day-to-date.

# 20. Health and Readiness

## 20.1 GET `/healthz`

**Auth:** None.
**Returns:** `200 OK` if the process is alive.

## 20.2 GET `/readyz`

**Auth:** None.
**Returns:** `200 OK` if the database is reachable, the master key is loaded, JWKS is loaded, and pg-boss is reachable. `503` otherwise.

# 21. Pagination, Sorting, and Filtering

Collection endpoints use cursor pagination. Standard query parameters:

| Parameter | Notes |
| --- | --- |
| `cursor` | Opaque cursor. First page omits. |
| `limit` | Default 25, max 200. |
| `sort` | Default `-created_at`. Endpoints document allowed sort columns. |

Responses include:

```json
{
  "data": [ ... ],
  "page": {
    "next_cursor": "...",
    "has_more": true
  }
}
```

# 22. Idempotency

POST endpoints that create resources accept `Idempotency-Key: <opaque-string>`. HEIMDALL Core caches the response for 24 hours keyed by `(client_id, idempotency_key)`. Replays return the original response.

# 23. Webhook Event Payloads

Every webhook delivery is signed using HMAC-SHA256 with the endpoint's signing secret. Headers:

```http
X-HEIMDALL-Signature: t=<timestamp>,v1=<hmac>
X-HEIMDALL-Event-Type: identity.created
X-HEIMDALL-Event-Id: <uuid>
X-HEIMDALL-Delivery-Id: <uuid>
X-HEIMDALL-Attempt: 1
```

Body:

```json
{
  "event_id": "...",
  "event_type": "identity.created",
  "created_at": "2026-05-19T18:43:00Z",
  "tenant_id": "...",
  "app_code": "pedl",
  "data": { ... },
  "correlation_id": "..."
}
```

Verification example (Node.js):

```ts
const expected = createHmac('sha256', signingSecret)
  .update(`${timestamp}.${rawBody}`)
  .digest('hex');
if (timingSafeEqual(Buffer.from(expected), Buffer.from(received))) { /* valid */ }
```

Retries follow exponential backoff (1m, 5m, 30m, 2h, 6h, 24h) with delivery abandoned after 24 hours. Manual replay always available via §18.5.

# 24. Rate Limits at the API Layer

In addition to Cloudflare edge limits, HEIMDALL Core enforces application-layer rate limits via the Fastify rate-limit plugin backed by Postgres counters. Defaults:

| Endpoint Class | Default Limit | Window |
| --- | --- | --- |
| `/oidc/auth`, `/oidc/token`, password login | 20/minute per IP, 100/hour per IP | sliding |
| Magic link, SMS OTP start | See Requirements §23.3 SMS caps | various |
| Identity read endpoints (`/v1/identities/me/*`) | 600/minute per parent | sliding |
| Identity write endpoints | 60/minute per parent | sliding |
| Secrets access endpoint | 600/minute per service account | sliding |
| Audit query endpoints | 60/minute per parent | sliding |
| Admin policy update | 30/hour per parent | sliding |

Exceeding limits returns `429 Too Many Requests` with `Retry-After` and a `rate_limit_policy_id` reference in the error envelope.

# 25. CORS

Hosted Login Pages and the HEIMDALL Admin UI on `*.myriad.com` are allowed origins. Tenant-domain origins are added dynamically per `tenant_domains` records once verified. All other origins are rejected.

Preflight responses cache for 5 minutes (`Access-Control-Max-Age: 300`). Credentialed CORS (`Access-Control-Allow-Credentials: true`) is enabled only for the Hosted Login Pages and Admin UI domains; tenant frontends use bearer JWT in Authorization headers and do not need credentialed CORS.

# 26. Frontend Integration Contract

The `@heimdall/client` TypeScript SDK provides:

| Component | Purpose |
| --- | --- |
| `HeimdallProvider` (React) | Context with current identity, persona, tenant, MFA state. |
| `useIdentity()` | Hook returning the current parent identity. |
| `usePersona()` | Hook returning the active persona and a switch function. |
| `useTenant()` | Hook returning the current tenant scope. |
| `useAuthorize(action, resource)` | Hook returning allow/deny by calling `/v1/authorize/check` with TTL cache. |
| `<ImpersonationBanner />` | Required component. Renders the red fixed banner when `act` claim is present. Mounts at app root. Undismissible. |
| `<ConsentReAcceptanceModal />` | Required component. Blocks app interaction when `/v1/disclosures/me/required-re-acceptance` returns nonempty. |
| `heimdallClient.fetch(...)` | Authenticated `fetch` wrapper that handles 401 step-up flow, 429 retry, and idempotency keys. |
| `heimdallClient.logout()` | Calls `/oidc/session/end`, clears local state, redirects. |

Integration of `ImpersonationBanner` and `ConsentReAcceptanceModal` is contractual: every HEIMDALL-consuming app must mount both at the app root. The Myriad release process for consuming apps includes a CI check that the components are mounted.

HEIMDALL — API Specification Companion | Myriad Technology Group
