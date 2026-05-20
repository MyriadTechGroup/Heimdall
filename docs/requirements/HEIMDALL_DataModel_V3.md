# HEIMDALL — Data Model and Schema

## Companion Document to HEIMDALL Requirements V3

Prepared for Myriad Technology Group | 2026-05-19

| Document Field | Value |
| --- | --- |
| Document Type | Data Model and Schema (companion to V3 Requirements) |
| Target ORM | Drizzle ORM |
| Target Migration Tool | Drizzle Kit |
| Target Database | PostgreSQL 15+ |
| Hosting Phase 1 | Supabase PostgreSQL (Postgres-standard SQL only) |
| Hosting Phase 2 | Amazon RDS PostgreSQL or Aurora PostgreSQL |
| Encryption Library | `pgcrypto` (built into Postgres) |
| Encryption Pattern | Envelope encryption: per-row Data Encryption Keys (DEKs) wrapped by Master Key |

# 1. Schema Organization

HEIMDALL uses three Postgres schemas:

| Schema | Purpose | Access |
| --- | --- | --- |
| `heimdall` | All HEIMDALL business tables. | HEIMDALL Core service role only. |
| `audit` | Append-only ledger tables (`audit_events`, `pii_change_events`, `communication_log`). | INSERT-only via triggers from `heimdall.*` and HEIMDALL Core; UPDATE and DELETE revoked from all roles except `compliance_role`. |
| `pgboss` | pg-boss background job queue. | HEIMDALL Core service role only. Managed by pg-boss library. |

Frontends never connect to PostgreSQL directly. There is no PostgREST or Supabase Data API exposure. HEIMDALL Core is the only path in.

# 2. Domain Map

| Domain | Tables |
| --- | --- |
| Application Registry | `applications`, `application_environments`, `application_minor_features` |
| Tenancy | `tenants`, `tenant_domains`, `tenant_branding`, `tenant_policies`, `tenant_policy_versions` |
| Identity Graph | `parent_identities`, `credential_identities`, `identity_link_edges`, `identity_link_edges_history`, `app_personas`, `persona_switch_events` |
| Membership and Authorization | `tenant_memberships`, `roles`, `permissions`, `role_permissions`, `role_permission_changes`, `persona_roles` |
| Guest and Offline | `guest_identities`, `guest_upgrade_events`, `offline_identity_events`, `offline_sync_batches`, `offline_conflict_records`, `device_installations` |
| Minors and Consent | `minor_profiles`, `guardian_relationships`, `age_policy_rules`, `application_minor_features` |
| Verification | `verification_sources`, `verification_records`, `verification_policy_rules` |
| Sessions and Devices | `session_records`, `trusted_devices`, `trusted_devices_2fa`, `device_risk_events` |
| MFA | `mfa_methods`, `mfa_backup_codes`, `mfa_challenges` |
| Secrets | `secret_collections`, `secret_items`, `secret_versions`, `secret_access_policies`, `secret_access_grants`, `secret_rotation_policies`, `secret_rotation_events` |
| Service Accounts | `service_accounts`, `service_account_credentials`, `service_account_grants` |
| Disclosures and Communications | `disclosures`, `disclosure_versions`, `disclosure_acceptances`, `communication_templates`, `communication_template_versions` |
| Anonymization | `anonymization_requests`, `anonymization_request_notifications` |
| Legal Holds | `legal_holds`, `legal_hold_blocked_actions` |
| Support and Impersonation | `impersonation_approvals`, `support_session_events` |
| OIDC Issuer | `idp_clients`, `idp_client_secrets`, `oidc_grants`, `oidc_sessions`, `oidc_authorization_codes`, `oidc_access_tokens`, `oidc_refresh_tokens` |
| Federation | `federated_identity_providers`, `federated_identity_links` |
| Webhooks and Integration | `api_clients`, `webhook_endpoints`, `webhook_deliveries` |
| Payments | `payment_provider_customer_refs` |
| Rate Limits | `rate_limit_policies`, `rate_limit_counters` |
| Audit (in `audit` schema) | `audit_events`, `pii_change_events`, `communication_log` |
| Break-Glass | `break_glass_requests`, `break_glass_sessions` |

# 3. Core Identity Tables

## 3.1 parent_identities

The Parent UUID is the central identity anchor. Every authenticated entity maps to exactly one Parent Identity. PII fields are encrypted via envelope encryption; tombstoned on anonymization.

```sql
CREATE TABLE heimdall.parent_identities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Identifying / display
  display_name_encrypted BYTEA,            -- envelope-encrypted display name
  display_name_dek_id UUID,                -- ref to data-encryption-key version
  legal_name_encrypted BYTEA,              -- optional, used only when collected
  legal_name_dek_id UUID,
  preferred_locale TEXT NOT NULL DEFAULT 'en-US',
  preferred_timezone TEXT,

  -- Lifecycle
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','suspended','anonymized','legal_hold','pending_anonymization')),
  status_reason TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_active_at TIMESTAMPTZ,

  -- Anonymization
  anonymized_at TIMESTAMPTZ,
  anonymized_reason TEXT,
  anonymization_request_id UUID,

  -- Provenance
  origin_app_code TEXT,                    -- app that originally created this identity
  origin_tenant_id UUID
);

CREATE INDEX idx_parent_identities_status ON heimdall.parent_identities (status);
CREATE INDEX idx_parent_identities_origin ON heimdall.parent_identities (origin_app_code, origin_tenant_id);
```

## 3.2 credential_identities

Every credential type (email, phone, social, WebAuthn passkey) is a row linked to a Parent UUID. PII fields are stored encrypted; HMAC-of-normalized-value enables dedup and lookup.

```sql
CREATE TABLE heimdall.credential_identities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id) ON DELETE RESTRICT,

  credential_type TEXT NOT NULL
    CHECK (credential_type IN (
      'email','phone','social_google','social_facebook','social_apple',
      'webauthn_passkey','password','magic_link','sms_passwordless'
    )),

  -- Value handling
  value_encrypted BYTEA,                   -- encrypted display value (email, phone, etc.)
  value_dek_id UUID,
  value_hmac BYTEA NOT NULL,               -- HMAC of normalized value, for lookup
  value_hmac_key_version INT NOT NULL,
  display_value TEXT,                      -- masked / partial value safe for UI ("***@example.com")
  external_subject TEXT,                   -- social provider subject ID (encrypted ref where applicable)

  -- Verification
  verified_at TIMESTAMPTZ,
  verified_via TEXT
    CHECK (verified_via IN ('email_link','sms_otp','voice_otp','social_idp','manual_admin','migration')),
  verification_attempt_count INT NOT NULL DEFAULT 0,
  last_verification_attempt_at TIMESTAMPTZ,

  -- Lifecycle
  is_primary BOOLEAN NOT NULL DEFAULT FALSE,
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','revoked','expired','pending_verification','tombstoned')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  revoked_at TIMESTAMPTZ,

  -- Password-specific (when credential_type='password')
  password_hash TEXT,                      -- Argon2id encoded string
  password_algorithm TEXT,                 -- 'argon2id'
  password_changed_at TIMESTAMPTZ,
  password_change_required BOOLEAN NOT NULL DEFAULT FALSE,

  -- WebAuthn-specific (when credential_type='webauthn_passkey')
  webauthn_credential_id BYTEA,
  webauthn_public_key BYTEA,
  webauthn_counter BIGINT,
  webauthn_aaguid UUID,
  webauthn_transports TEXT[],
  webauthn_friendly_name TEXT,

  UNIQUE (credential_type, value_hmac, value_hmac_key_version)
);

CREATE INDEX idx_credentials_parent ON heimdall.credential_identities (parent_id);
CREATE INDEX idx_credentials_lookup ON heimdall.credential_identities (credential_type, value_hmac);
CREATE INDEX idx_credentials_status ON heimdall.credential_identities (status) WHERE status = 'active';
```

## 3.3 identity_link_edges and identity_link_edges_history

The merge graph. When two Parent UUIDs are linked, the "merged-into" parent becomes inactive; the "merged-from" parent becomes the surviving anchor. Reversible within the configured window.

```sql
CREATE TYPE heimdall.merge_type AS ENUM (
  'user_link','user_unlink','admin_merge','guardian_merge'
);

CREATE TABLE heimdall.identity_link_edges (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  surviving_parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  merged_parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  merge_type heimdall.merge_type NOT NULL,
  reason_code TEXT NOT NULL,
  reason_notes TEXT,
  initiated_by_parent_id UUID,             -- who initiated (user, support, admin)
  initiated_by_type TEXT NOT NULL
    CHECK (initiated_by_type IN ('self','support','admin','guardian','system')),
  merged_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  reversal_deadline_at TIMESTAMPTZ NOT NULL,
  reversed_at TIMESTAMPTZ,
  reversed_by_parent_id UUID,
  reversal_reason TEXT,
  permanently_merged_at TIMESTAMPTZ,        -- set when reversal_deadline_at passes
  CHECK (surviving_parent_id <> merged_parent_id)
);

CREATE INDEX idx_link_edges_surviving ON heimdall.identity_link_edges (surviving_parent_id);
CREATE INDEX idx_link_edges_merged ON heimdall.identity_link_edges (merged_parent_id);
CREATE INDEX idx_link_edges_pending_reversal
  ON heimdall.identity_link_edges (reversal_deadline_at)
  WHERE reversed_at IS NULL AND permanently_merged_at IS NULL;

-- History twin written by trigger on every INSERT/UPDATE
CREATE TABLE heimdall.identity_link_edges_history (
  history_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  edge_id UUID NOT NULL,
  operation TEXT NOT NULL CHECK (operation IN ('insert','update','delete')),
  snapshot JSONB NOT NULL,
  changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  changed_by_parent_id UUID
);
```

## 3.4 app_personas and persona_switch_events

A persona is a user's representation within a specific application and user-type context. PEDL has rider, driver, and admin personas. A single Parent UUID may have multiple personas across apps.

```sql
CREATE TABLE heimdall.app_personas (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  app_code TEXT NOT NULL,                  -- 'pedl','quest','myriad-dashboard'
  persona_type TEXT NOT NULL,              -- 'rider','driver','attendee','organizer','admin'
  tenant_id UUID,                          -- tenant scope where applicable
  display_name_encrypted BYTEA,
  display_name_dek_id UUID,
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','suspended','deactivated','anonymized')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  activated_at TIMESTAMPTZ,
  deactivated_at TIMESTAMPTZ,
  anonymized_at TIMESTAMPTZ,
  UNIQUE (parent_id, app_code, persona_type, tenant_id)
);

CREATE INDEX idx_personas_parent ON heimdall.app_personas (parent_id);
CREATE INDEX idx_personas_app ON heimdall.app_personas (app_code, persona_type);
CREATE INDEX idx_personas_tenant ON heimdall.app_personas (tenant_id);

CREATE TABLE heimdall.persona_switch_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL,
  session_id UUID,
  from_persona_id UUID,
  to_persona_id UUID,
  switched_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  ip_address INET,
  device_fingerprint TEXT,
  device_type TEXT
);
```

## 3.5 federated_identity_links

Records the linkage between HEIMDALL credentials and upstream IdP subject IDs.

```sql
CREATE TABLE heimdall.federated_identity_providers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  provider_code TEXT NOT NULL UNIQUE,      -- 'google','facebook','apple','okta','azure_ad'
  provider_type TEXT NOT NULL
    CHECK (provider_type IN ('oidc','oauth2','saml')),
  display_name TEXT NOT NULL,
  is_enabled BOOLEAN NOT NULL DEFAULT TRUE,
  config_jsonb JSONB,                      -- discovery URL, scopes, claims map, etc.
  client_id_secret_ref UUID,               -- ref into secret_items for the client_id (if sensitive)
  client_secret_secret_ref UUID,           -- ref into secret_items for the client_secret
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.federated_identity_links (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  credential_id UUID NOT NULL REFERENCES heimdall.credential_identities(id),
  provider_id UUID NOT NULL REFERENCES heimdall.federated_identity_providers(id),
  external_subject TEXT NOT NULL,          -- IdP subject claim
  external_email_hmac BYTEA,               -- HMAC of IdP-reported email
  external_email_verified BOOLEAN,
  claims_jsonb JSONB,                      -- last-known IdP claims (no PII plaintext)
  first_linked_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_used_at TIMESTAMPTZ,
  UNIQUE (provider_id, external_subject)
);

CREATE INDEX idx_fed_links_credential ON heimdall.federated_identity_links (credential_id);
```

# 4. Tenancy

```sql
CREATE TABLE heimdall.tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug TEXT NOT NULL UNIQUE,
  display_name TEXT NOT NULL,
  tier TEXT NOT NULL
    CHECK (tier IN ('myriad_internal','myriad_products','client','isolated_client')),
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deactivated_at TIMESTAMPTZ
);

CREATE TABLE heimdall.tenant_domains (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES heimdall.tenants(id),
  domain TEXT NOT NULL UNIQUE,
  is_verified BOOLEAN NOT NULL DEFAULT FALSE,
  is_primary BOOLEAN NOT NULL DEFAULT FALSE,
  dkim_status TEXT
    CHECK (dkim_status IN ('not_configured','pending','verified','failed')),
  spf_status TEXT
    CHECK (spf_status IN ('not_configured','pending','verified','failed')),
  verified_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.tenant_branding (
  tenant_id UUID PRIMARY KEY REFERENCES heimdall.tenants(id),
  logo_url TEXT,
  primary_color TEXT,
  secondary_color TEXT,
  email_from_address TEXT,                 -- defaults to noreply@myriad.com when null
  email_from_name TEXT,
  sms_sender_id TEXT,
  login_theme TEXT,                        -- theme key for the hosted login pages
  custom_css TEXT,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.tenant_policies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES heimdall.tenants(id),
  policy_category TEXT NOT NULL
    CHECK (policy_category IN (
      'auth_methods','mfa','social_providers','age_gate','disclosure',
      'rate_limit','communication','session','data_residency',
      'social_provider_credentials'
    )),
  current_version_id UUID,                 -- pointer to the active version
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (tenant_id, policy_category)
);

CREATE TABLE heimdall.tenant_policy_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  policy_id UUID NOT NULL REFERENCES heimdall.tenant_policies(id),
  version INT NOT NULL,
  effective_at TIMESTAMPTZ NOT NULL,
  superseded_at TIMESTAMPTZ,
  body_jsonb JSONB NOT NULL,
  change_summary TEXT,
  authored_by_parent_id UUID,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (policy_id, version)
);

CREATE INDEX idx_policy_versions_active
  ON heimdall.tenant_policy_versions (policy_id, effective_at)
  WHERE superseded_at IS NULL;

CREATE TABLE heimdall.tenant_memberships (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  tenant_id UUID NOT NULL REFERENCES heimdall.tenants(id),
  app_code TEXT,                           -- nullable for cross-app tenant access
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','suspended','revoked')),
  joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  revoked_at TIMESTAMPTZ,
  UNIQUE (parent_id, tenant_id, app_code)
);
```

# 5. Roles, Permissions, and Authorization

```sql
CREATE TABLE heimdall.roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code TEXT NOT NULL,                      -- 'myriad_super_admin','tenant_owner','pedl_driver'
  display_name TEXT NOT NULL,
  description TEXT,
  scope TEXT NOT NULL
    CHECK (scope IN ('global','tenant','app','app_tenant')),
  tenant_id UUID REFERENCES heimdall.tenants(id), -- nullable for global/system roles
  app_code TEXT,                           -- nullable for non-app-scoped roles
  is_system BOOLEAN NOT NULL DEFAULT FALSE, -- system roles cannot be deleted
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (code, tenant_id, app_code)
);

CREATE TABLE heimdall.permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code TEXT NOT NULL UNIQUE,               -- 'user.view','secret.read','ride.accept'
  display_name TEXT NOT NULL,
  description TEXT,
  category TEXT,                           -- 'identity','tenant','secret','audit','app:pedl','app:quest'
  is_sensitive BOOLEAN NOT NULL DEFAULT FALSE -- requires MFA step-up and extra audit
);

CREATE TABLE heimdall.role_permissions (
  role_id UUID NOT NULL REFERENCES heimdall.roles(id),
  permission_id UUID NOT NULL REFERENCES heimdall.permissions(id),
  granted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  granted_by_parent_id UUID,
  PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE heimdall.role_permission_changes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  role_id UUID NOT NULL,
  permission_id UUID NOT NULL,
  change_type TEXT NOT NULL CHECK (change_type IN ('grant','revoke')),
  changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  changed_by_parent_id UUID,
  reason TEXT
);

CREATE TABLE heimdall.persona_roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  persona_id UUID NOT NULL REFERENCES heimdall.app_personas(id),
  role_id UUID NOT NULL REFERENCES heimdall.roles(id),
  granted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  granted_by_parent_id UUID,
  expires_at TIMESTAMPTZ,
  UNIQUE (persona_id, role_id)
);

CREATE INDEX idx_persona_roles_persona ON heimdall.persona_roles (persona_id);
CREATE INDEX idx_persona_roles_role ON heimdall.persona_roles (role_id);
```

# 6. Minors and Guardians

```sql
CREATE TABLE heimdall.minor_profiles (
  parent_id UUID PRIMARY KEY REFERENCES heimdall.parent_identities(id),
  date_of_birth_encrypted BYTEA,
  date_of_birth_dek_id UUID,
  age_at_signup INT,                       -- age band at signup, no exact DOB stored unencrypted
  jurisdiction_code TEXT NOT NULL,         -- 'US-CA','US-FL','EU-DE','US' for default
  declared_minor BOOLEAN NOT NULL DEFAULT FALSE,
  attestation_method TEXT NOT NULL
    CHECK (attestation_method IN ('self_attestation','guardian_attestation','dob_verified','idv_verified')),
  age_gate_status TEXT NOT NULL DEFAULT 'compliant'
    CHECK (age_gate_status IN ('compliant','blocked','pending_guardian','transitioned')),
  age_transitioned_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.guardian_relationships (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  guardian_parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  minor_parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  relationship_type TEXT NOT NULL
    CHECK (relationship_type IN ('parent','legal_guardian','foster','authorized_adult')),
  status TEXT NOT NULL DEFAULT 'active'
    CHECK (status IN ('active','revoked','lapsed_age_transition')),
  established_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  revoked_at TIMESTAMPTZ,
  revoked_reason TEXT,
  CHECK (guardian_parent_id <> minor_parent_id)
);

CREATE TABLE heimdall.age_policy_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_code TEXT NOT NULL,
  jurisdiction_code TEXT NOT NULL,         -- 'US-COPPA','US-CA','US-FL','EU-DE','GLOBAL'
  minimum_age INT NOT NULL,
  requires_guardian_consent BOOLEAN NOT NULL DEFAULT FALSE,
  guardian_age_minimum INT,                -- minimum age for the guardian
  restricted_features TEXT[],              -- features blocked for minors
  disclosure_categories_required TEXT[],   -- e.g. ['coppa_consent']
  effective_at TIMESTAMPTZ NOT NULL,
  superseded_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (app_code, jurisdiction_code, effective_at)
);

CREATE TABLE heimdall.application_minor_features (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_code TEXT NOT NULL,
  feature_code TEXT NOT NULL,              -- 'public_profile','direct_messaging','purchase_flow'
  jurisdiction_code TEXT NOT NULL,
  minor_access TEXT NOT NULL
    CHECK (minor_access IN ('blocked','restricted','allowed_with_consent','allowed')),
  effective_at TIMESTAMPTZ NOT NULL,
  superseded_at TIMESTAMPTZ,
  UNIQUE (app_code, feature_code, jurisdiction_code, effective_at)
);
```

# 7. Guests and Offline

```sql
CREATE TABLE heimdall.guest_identities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_code TEXT NOT NULL,
  device_fingerprint TEXT NOT NULL,
  device_platform TEXT NOT NULL
    CHECK (device_platform IN ('web','ios','android','mobile_web')),
  tenant_id UUID,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_active_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ,                  -- configurable per app
  claimed_by_parent_id UUID,
  claimed_at TIMESTAMPTZ,
  UNIQUE (app_code, device_fingerprint)
);

CREATE TABLE heimdall.guest_upgrade_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  guest_id UUID NOT NULL REFERENCES heimdall.guest_identities(id),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  upgraded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  preserved_data_summary JSONB             -- what was carried forward from guest state
);

CREATE TABLE heimdall.device_installations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID REFERENCES heimdall.parent_identities(id),
  guest_id UUID REFERENCES heimdall.guest_identities(id),
  app_code TEXT NOT NULL,
  device_fingerprint TEXT NOT NULL,
  device_platform TEXT NOT NULL,
  install_id TEXT,                         -- vendor install ID
  push_token TEXT,                         -- for future push messaging
  app_version TEXT,
  os_version TEXT,
  first_seen_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_seen_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (app_code, device_fingerprint)
);

CREATE TABLE heimdall.offline_identity_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  device_installation_id UUID NOT NULL REFERENCES heimdall.device_installations(id),
  event_type TEXT NOT NULL,                -- 'persona_action','consent_grant','profile_update'
  client_event_id TEXT NOT NULL,           -- idempotency key
  client_updated_at TIMESTAMPTZ NOT NULL,
  server_received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  applied_at TIMESTAMPTZ,
  payload_jsonb JSONB NOT NULL,
  conflict_record_id UUID,
  UNIQUE (device_installation_id, client_event_id)
);

CREATE TABLE heimdall.offline_sync_batches (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  device_installation_id UUID NOT NULL,
  started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMPTZ,
  event_count INT NOT NULL DEFAULT 0,
  conflict_count INT NOT NULL DEFAULT 0,
  status TEXT NOT NULL DEFAULT 'in_progress'
    CHECK (status IN ('in_progress','completed','failed'))
);

CREATE TABLE heimdall.offline_conflict_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id UUID NOT NULL REFERENCES heimdall.offline_identity_events(id),
  loser_payload_jsonb JSONB NOT NULL,
  winner_payload_jsonb JSONB NOT NULL,
  resolution_strategy TEXT NOT NULL DEFAULT 'last_write_wins',
  reviewed_at TIMESTAMPTZ,
  reviewed_by_parent_id UUID
);
```

# 8. Verification

```sql
CREATE TABLE heimdall.verification_sources (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  provider_code TEXT NOT NULL UNIQUE,      -- 'stripe_identity','persona','onfido','idme','manual_admin'
  display_name TEXT NOT NULL,
  is_enabled BOOLEAN NOT NULL DEFAULT TRUE,
  config_jsonb JSONB,
  credentials_secret_ref UUID,             -- ref into secret_items
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.verification_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  app_code TEXT,                           -- nullable for global verifications
  verification_source_id UUID NOT NULL REFERENCES heimdall.verification_sources(id),
  provider_verification_id TEXT NOT NULL,
  verification_level TEXT NOT NULL
    CHECK (verification_level IN ('email','phone','government_id','government_id_with_selfie','knowledge_based','biometric')),
  status TEXT NOT NULL
    CHECK (status IN ('pending','passed','failed','expired','revoked')),
  claims_jsonb JSONB,                      -- verified name, verified DOB ref, verified document type, etc.
  verified_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  revoked_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (verification_source_id, provider_verification_id)
);

CREATE INDEX idx_verification_parent ON heimdall.verification_records (parent_id);
CREATE INDEX idx_verification_status ON heimdall.verification_records (status);

CREATE TABLE heimdall.verification_policy_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_code TEXT NOT NULL,
  persona_type TEXT NOT NULL,
  required_verification_level TEXT NOT NULL,
  acceptable_providers TEXT[],
  effective_at TIMESTAMPTZ NOT NULL,
  superseded_at TIMESTAMPTZ
);
```

# 9. Sessions, Devices, and MFA

```sql
CREATE TABLE heimdall.session_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  app_code TEXT NOT NULL,
  tenant_id UUID,
  active_persona_id UUID,
  oidc_session_id UUID,
  authentication_methods TEXT[],           -- 'password','social_google','sms_otp','totp','webauthn'
  mfa_level TEXT NOT NULL DEFAULT 'none'
    CHECK (mfa_level IN ('none','single_factor','multi_factor','strong_multi_factor')),
  ip_address INET,
  user_agent TEXT,
  device_fingerprint TEXT,
  geo_country TEXT,
  geo_region TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_active_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL,
  revoked_at TIMESTAMPTZ,
  revoked_reason TEXT
);

CREATE INDEX idx_sessions_parent_active
  ON heimdall.session_records (parent_id)
  WHERE revoked_at IS NULL;

CREATE TABLE heimdall.trusted_devices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  device_fingerprint TEXT NOT NULL,
  friendly_name TEXT,
  first_trusted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL,
  revoked_at TIMESTAMPTZ,
  last_seen_ip INET,
  last_seen_at TIMESTAMPTZ,
  UNIQUE (parent_id, device_fingerprint)
);

CREATE TABLE heimdall.trusted_devices_2fa (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  device_fingerprint TEXT NOT NULL,
  friendly_name TEXT,
  trusted_for_methods TEXT[],              -- which MFA methods this device satisfies
  expires_at TIMESTAMPTZ NOT NULL,
  revoked_at TIMESTAMPTZ,
  UNIQUE (parent_id, device_fingerprint)
);

CREATE TABLE heimdall.device_risk_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID,
  device_fingerprint TEXT,
  ip_address INET,
  signal_type TEXT NOT NULL,               -- 'new_country','sim_swap','impossible_travel','tor_exit','known_bot'
  signal_severity TEXT NOT NULL CHECK (signal_severity IN ('info','warn','high','critical')),
  signal_payload_jsonb JSONB,
  detected_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.mfa_methods (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  method_type TEXT NOT NULL
    CHECK (method_type IN ('totp','sms','email_otp','webauthn','backup_codes','fido2_hardware')),
  is_primary BOOLEAN NOT NULL DEFAULT FALSE,
  friendly_name TEXT,
  -- TOTP-specific
  totp_secret_encrypted BYTEA,
  totp_secret_dek_id UUID,
  totp_algorithm TEXT,                     -- 'SHA1','SHA256','SHA512'
  totp_digits INT,
  totp_period INT,
  -- SMS-specific
  credential_id UUID REFERENCES heimdall.credential_identities(id),  -- which phone
  -- WebAuthn linked via credential_identities row
  webauthn_credential_id UUID REFERENCES heimdall.credential_identities(id),
  enrolled_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_used_at TIMESTAMPTZ,
  revoked_at TIMESTAMPTZ
);

CREATE INDEX idx_mfa_methods_parent ON heimdall.mfa_methods (parent_id) WHERE revoked_at IS NULL;

CREATE TABLE heimdall.mfa_backup_codes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  code_hash TEXT NOT NULL,                 -- Argon2id or bcrypt of the backup code
  generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  used_at TIMESTAMPTZ
);

CREATE TABLE heimdall.mfa_challenges (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL,
  session_id UUID,
  method_id UUID,
  method_type TEXT NOT NULL,
  challenge_token TEXT NOT NULL UNIQUE,
  expires_at TIMESTAMPTZ NOT NULL,
  attempts INT NOT NULL DEFAULT 0,
  max_attempts INT NOT NULL DEFAULT 5,
  succeeded_at TIMESTAMPTZ,
  failed_at TIMESTAMPTZ,
  ip_address INET,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

# 10. Secrets

```sql
CREATE TABLE heimdall.secret_collections (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code TEXT NOT NULL UNIQUE,               -- 'twilio','sendgrid','stripe','social-google'
  display_name TEXT NOT NULL,
  description TEXT,
  scope TEXT NOT NULL
    CHECK (scope IN ('global','app','tenant','app_tenant','provider','database','webhook')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.secret_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  collection_id UUID NOT NULL REFERENCES heimdall.secret_collections(id),
  key TEXT NOT NULL,                       -- 'api_key','client_secret','signing_key'
  environment TEXT NOT NULL
    CHECK (environment IN ('dev','staging','prod')),
  app_code TEXT,
  tenant_id UUID,
  description TEXT,
  current_version_id UUID,
  rotation_policy_id UUID,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (collection_id, key, environment, app_code, tenant_id)
);

CREATE TABLE heimdall.secret_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  secret_item_id UUID NOT NULL REFERENCES heimdall.secret_items(id),
  version INT NOT NULL,
  value_encrypted BYTEA NOT NULL,          -- envelope-encrypted secret value
  value_dek_id UUID NOT NULL,
  value_fingerprint BYTEA,                 -- HMAC of plaintext (for "did this value change?" without decrypting)
  master_key_version INT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_by_parent_id UUID,
  activated_at TIMESTAMPTZ,
  retired_at TIMESTAMPTZ,
  UNIQUE (secret_item_id, version)
);

CREATE TABLE heimdall.secret_access_policies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  secret_item_id UUID NOT NULL REFERENCES heimdall.secret_items(id),
  principal_type TEXT NOT NULL CHECK (principal_type IN ('role','service_account','parent')),
  principal_id UUID NOT NULL,
  allowed_actions TEXT[] NOT NULL,         -- ['read','rotate','revoke']
  granted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  granted_by_parent_id UUID,
  expires_at TIMESTAMPTZ
);

CREATE TABLE heimdall.secret_access_grants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  secret_item_id UUID NOT NULL,
  parent_id UUID NOT NULL,
  granted_by_parent_id UUID NOT NULL,
  reason TEXT NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  revoked_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.secret_rotation_policies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  rotation_interval_days INT NOT NULL,
  alert_before_expiry_days INT NOT NULL DEFAULT 7,
  auto_rotate BOOLEAN NOT NULL DEFAULT FALSE,
  rotation_handler_code TEXT,              -- which rotation procedure to invoke
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.secret_rotation_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  secret_item_id UUID NOT NULL,
  previous_version INT,
  new_version INT,
  rotation_type TEXT NOT NULL CHECK (rotation_type IN ('manual','scheduled','emergency')),
  initiated_by_parent_id UUID,
  reason TEXT,
  succeeded_at TIMESTAMPTZ,
  failed_at TIMESTAMPTZ,
  failure_reason TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

# 11. Disclosures and Communications

```sql
CREATE TABLE heimdall.disclosures (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  category TEXT NOT NULL
    CHECK (category IN (
      'terms_of_service','privacy_policy','tenant_terms','coppa_consent',
      'gdpr_data_processing','sms_consent','marketing_opt_in','app_specific_eula',
      'community_guidelines','acceptable_use_policy','cookie_consent'
    )),
  scope_layer TEXT NOT NULL
    CHECK (scope_layer IN ('global','app','tenant')),
  app_code TEXT,
  tenant_id UUID REFERENCES heimdall.tenants(id),
  display_name TEXT NOT NULL,
  current_version_id UUID,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CHECK (
    (scope_layer = 'global' AND app_code IS NULL AND tenant_id IS NULL) OR
    (scope_layer = 'app' AND app_code IS NOT NULL AND tenant_id IS NULL) OR
    (scope_layer = 'tenant' AND tenant_id IS NOT NULL)
  )
);

CREATE TABLE heimdall.disclosure_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  disclosure_id UUID NOT NULL REFERENCES heimdall.disclosures(id),
  version TEXT NOT NULL,                   -- 'v1.0', '2026-05-19', etc.
  effective_at TIMESTAMPTZ NOT NULL,
  superseded_at TIMESTAMPTZ,
  body_markdown TEXT NOT NULL,
  body_html TEXT,
  summary_of_changes TEXT,
  requires_re_acceptance BOOLEAN NOT NULL DEFAULT FALSE,
  locale TEXT NOT NULL DEFAULT 'en-US',
  authored_by_parent_id UUID,
  approved_by_parent_id UUID,
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (disclosure_id, version, locale)
);

CREATE INDEX idx_disclosure_versions_active
  ON heimdall.disclosure_versions (disclosure_id, locale, effective_at)
  WHERE superseded_at IS NULL;

CREATE TABLE heimdall.disclosure_acceptances (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  disclosure_version_id UUID NOT NULL REFERENCES heimdall.disclosure_versions(id),
  accepted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  accepted_via TEXT NOT NULL
    CHECK (accepted_via IN ('signup','re_consent_flow','admin_recorded','migrated_legacy')),
  ip_address INET,
  user_agent TEXT,
  device_fingerprint TEXT,
  geo_country TEXT,
  correlation_id UUID,
  UNIQUE (parent_id, disclosure_version_id)
);

CREATE INDEX idx_disclosure_acceptances_parent ON heimdall.disclosure_acceptances (parent_id);

CREATE TABLE heimdall.communication_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code TEXT NOT NULL UNIQUE,               -- 'anonymization_initial','disclosure_re_consent','mfa_enrolled'
  display_name TEXT NOT NULL,
  description TEXT,
  channel TEXT NOT NULL CHECK (channel IN ('email','sms','in_app','push')),
  is_transactional BOOLEAN NOT NULL DEFAULT TRUE,
  current_version_id UUID,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.communication_template_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  template_id UUID NOT NULL REFERENCES heimdall.communication_templates(id),
  version INT NOT NULL,
  locale TEXT NOT NULL DEFAULT 'en-US',
  subject TEXT,                            -- for email
  body_text TEXT NOT NULL,
  body_html TEXT,
  variables JSONB,                         -- declared substitution variables
  effective_at TIMESTAMPTZ NOT NULL,
  superseded_at TIMESTAMPTZ,
  authored_by_parent_id UUID,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (template_id, version, locale)
);

-- communication_log lives in the audit schema (Section 14)
```

# 12. Anonymization, Legal Holds, Support

```sql
CREATE TABLE heimdall.anonymization_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  scope TEXT NOT NULL CHECK (scope IN ('full_account','per_app','per_persona')),
  scope_target JSONB,                      -- {app_code} or {persona_id}
  initiator_type TEXT NOT NULL
    CHECK (initiator_type IN ('self','support','legal_compliance','admin')),
  initiator_parent_id UUID,
  legal_reference TEXT,                    -- DPA/GDPR/court order reference
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending','cancelled','executed','blocked_legal_hold','failed')),
  requested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  scheduled_execution_at TIMESTAMPTZ NOT NULL,
  notification_1_sent_at TIMESTAMPTZ,
  notification_2_sent_at TIMESTAMPTZ,
  final_reminder_sent_at TIMESTAMPTZ,
  executed_at TIMESTAMPTZ,
  cancelled_at TIMESTAMPTZ,
  cancelled_by_parent_id UUID,
  cancelled_reason TEXT,
  blocked_by_legal_hold_id UUID,
  notes TEXT
);

CREATE INDEX idx_anonymization_due
  ON heimdall.anonymization_requests (scheduled_execution_at)
  WHERE status = 'pending';

CREATE TABLE heimdall.anonymization_request_notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  request_id UUID NOT NULL REFERENCES heimdall.anonymization_requests(id),
  notification_phase TEXT NOT NULL CHECK (notification_phase IN ('initial','day_15','final','executed','cancelled','blocked_legal_hold')),
  channel TEXT NOT NULL,
  communication_log_id UUID,
  sent_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.legal_holds (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  authority TEXT NOT NULL,                 -- 'internal_legal','court_order','regulator','dpa_request'
  reference_id TEXT NOT NULL,              -- legal matter ID
  placed_by_parent_id UUID NOT NULL,       -- myriad_legal_officer
  reason_code TEXT NOT NULL,
  reason_notes TEXT,
  placed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  effective_from TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  released_at TIMESTAMPTZ,
  released_by_parent_id UUID,
  release_reason TEXT
);

CREATE INDEX idx_legal_holds_active ON heimdall.legal_holds (parent_id) WHERE released_at IS NULL;

CREATE TABLE heimdall.legal_hold_blocked_actions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  legal_hold_id UUID NOT NULL REFERENCES heimdall.legal_holds(id),
  attempted_action TEXT NOT NULL,          -- 'login','anonymization_execute','merge','credential_change'
  attempted_by_parent_id UUID,
  attempted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  ip_address INET,
  user_agent TEXT
);

CREATE TABLE heimdall.impersonation_approvals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  requester_parent_id UUID NOT NULL,       -- support agent requesting
  target_parent_id UUID NOT NULL,          -- user being impersonated
  ticket_reference TEXT NOT NULL,
  justification TEXT NOT NULL,
  requested_duration_minutes INT NOT NULL CHECK (requested_duration_minutes <= 60),
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending','approved','denied','expired','consumed')),
  requested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  approval_window_expires_at TIMESTAMPTZ NOT NULL,  -- requester has 30 min after approval to start
  approved_by_parent_id UUID,
  approved_at TIMESTAMPTZ,
  denied_by_parent_id UUID,
  denied_at TIMESTAMPTZ,
  denial_reason TEXT,
  consumed_at TIMESTAMPTZ
);

CREATE INDEX idx_impersonation_pending
  ON heimdall.impersonation_approvals (status)
  WHERE status = 'pending';

CREATE TABLE heimdall.support_session_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  approval_id UUID NOT NULL REFERENCES heimdall.impersonation_approvals(id),
  support_parent_id UUID NOT NULL,
  impersonated_parent_id UUID NOT NULL,
  event_type TEXT NOT NULL,                -- 'session_started','action_taken','session_ended','token_renewed'
  action_descriptor TEXT,                  -- description of action taken
  request_path TEXT,
  http_method TEXT,
  ip_address INET,
  device_fingerprint TEXT,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

# 13. OIDC Issuer Tables

The `oidc-provider` library uses an adapter pattern. HEIMDALL implements the adapter against the following tables.

```sql
CREATE TABLE heimdall.idp_clients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id TEXT NOT NULL UNIQUE,
  display_name TEXT NOT NULL,
  client_type TEXT NOT NULL CHECK (client_type IN ('public','confidential')),
  app_code TEXT,
  tenant_id UUID,
  redirect_uris TEXT[] NOT NULL,
  post_logout_redirect_uris TEXT[],
  grant_types TEXT[] NOT NULL,             -- 'authorization_code','refresh_token','client_credentials'
  response_types TEXT[] NOT NULL,
  token_endpoint_auth_method TEXT NOT NULL,
  scopes_allowed TEXT[] NOT NULL,
  require_pkce BOOLEAN NOT NULL DEFAULT TRUE,
  require_signed_request_object BOOLEAN NOT NULL DEFAULT FALSE,
  access_token_lifetime_seconds INT NOT NULL DEFAULT 900,
  refresh_token_lifetime_seconds INT NOT NULL DEFAULT 2592000,
  id_token_lifetime_seconds INT NOT NULL DEFAULT 900,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.idp_client_secrets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id UUID NOT NULL REFERENCES heimdall.idp_clients(id),
  secret_hash TEXT NOT NULL,               -- hashed; plaintext returned once at creation, never stored
  secret_fingerprint TEXT NOT NULL,        -- first 4 chars for display ("abcd...")
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  revoked_at TIMESTAMPTZ
);

-- Storage for oidc-provider models (Grant, Session, AuthorizationCode, AccessToken, RefreshToken,
-- DeviceCode, ClientCredentials, RegistrationAccessToken, Interaction, ReplayDetection, PushedAuthorizationRequest, BackchannelAuthenticationRequest)
-- Unified table per oidc-provider's recommended pattern:
CREATE TABLE heimdall.oidc_payloads (
  id TEXT NOT NULL,
  type INT NOT NULL,                       -- oidc-provider model type code
  payload JSONB NOT NULL,
  grant_id TEXT,
  user_code TEXT,
  uid TEXT,
  expires_at TIMESTAMPTZ,
  consumed_at TIMESTAMPTZ,
  PRIMARY KEY (id, type)
);

CREATE INDEX idx_oidc_payloads_grant ON heimdall.oidc_payloads (grant_id);
CREATE INDEX idx_oidc_payloads_user_code ON heimdall.oidc_payloads (user_code);
CREATE INDEX idx_oidc_payloads_uid ON heimdall.oidc_payloads (uid);
CREATE INDEX idx_oidc_payloads_expires ON heimdall.oidc_payloads (expires_at) WHERE expires_at IS NOT NULL;
```

# 14. Audit Schema (Append-Only Ledgers)

```sql
CREATE SCHEMA audit;

CREATE TABLE audit.audit_events (
  id UUID NOT NULL DEFAULT gen_random_uuid(),
  parent_id UUID,
  actor_parent_id UUID,
  actor_type TEXT,
  event_category TEXT NOT NULL,
  event_type TEXT NOT NULL,
  app_code TEXT,
  tenant_id UUID,
  ip_address INET,
  user_agent TEXT,
  device_fingerprint TEXT,
  device_type TEXT,
  locale TEXT,
  geo_country TEXT,
  geo_region TEXT,
  payload JSONB,
  correlation_id UUID,
  prev_hash BYTEA,                         -- hash of prior row for this aggregate
  row_hash BYTEA NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_events_parent ON audit.audit_events (parent_id, created_at);
CREATE INDEX idx_audit_events_actor ON audit.audit_events (actor_parent_id, created_at);
CREATE INDEX idx_audit_events_category ON audit.audit_events (event_category, event_type, created_at);
CREATE INDEX idx_audit_events_correlation ON audit.audit_events (correlation_id);

CREATE TABLE audit.pii_change_events (
  id UUID NOT NULL DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL,
  actor_parent_id UUID,
  actor_type TEXT NOT NULL,
  field_path TEXT NOT NULL,
  change_type TEXT NOT NULL CHECK (change_type IN ('create','update','delete','anonymize','view')),
  previous_value_hash BYTEA,
  new_value_hash BYTEA,
  ip_address INET,
  user_agent TEXT,
  device_fingerprint TEXT,
  device_type TEXT,
  locale TEXT,
  geo_country TEXT,
  geo_region TEXT,
  correlation_id UUID,
  prev_hash BYTEA,
  row_hash BYTEA NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_pii_change_parent ON audit.pii_change_events (parent_id, created_at);
CREATE INDEX idx_pii_change_actor ON audit.pii_change_events (actor_parent_id, created_at);

CREATE TABLE audit.communication_log (
  id UUID NOT NULL DEFAULT gen_random_uuid(),
  parent_id UUID,
  template_version_id UUID,
  channel TEXT NOT NULL,
  recipient_address_hash BYTEA,
  recipient_address_display TEXT,          -- masked / partial
  subject_snapshot TEXT,
  status TEXT NOT NULL,                    -- 'queued','sent','delivered','opened','clicked','bounced','failed'
  provider TEXT,
  provider_message_id TEXT,
  queued_at TIMESTAMPTZ,
  sent_at TIMESTAMPTZ,
  delivered_at TIMESTAMPTZ,
  opened_at TIMESTAMPTZ,
  clicked_at TIMESTAMPTZ,
  bounced_at TIMESTAMPTZ,
  failed_at TIMESTAMPTZ,
  failure_reason TEXT,
  correlation_id UUID,
  locale TEXT,
  prev_hash BYTEA,
  row_hash BYTEA NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_comm_log_parent ON audit.communication_log (parent_id, created_at);
CREATE INDEX idx_comm_log_correlation ON audit.communication_log (correlation_id);
CREATE INDEX idx_comm_log_status ON audit.communication_log (status, created_at);
```

# 15. Hash-Chain Trigger Implementation

```sql
CREATE OR REPLACE FUNCTION audit.compute_hash_chain()
RETURNS TRIGGER AS $$
DECLARE
  prev_row_hash BYTEA;
  payload_text TEXT;
BEGIN
  -- Find the prior row in this table (by created_at descending)
  EXECUTE format(
    'SELECT row_hash FROM %I.%I ORDER BY created_at DESC, id DESC LIMIT 1',
    TG_TABLE_SCHEMA, TG_TABLE_NAME
  ) INTO prev_row_hash;

  NEW.prev_hash := prev_row_hash;

  -- Compute row_hash from NEW.* as JSONB text plus prev_hash
  payload_text := COALESCE(NEW.payload::text, '') ||
                  COALESCE(NEW.event_type, '') ||
                  COALESCE(NEW.parent_id::text, '') ||
                  COALESCE(NEW.created_at::text, '');
  NEW.row_hash := digest(COALESCE(prev_row_hash, '\x00'::bytea) || payload_text::bytea, 'sha256');

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_events_hash
  BEFORE INSERT ON audit.audit_events
  FOR EACH ROW EXECUTE FUNCTION audit.compute_hash_chain();

-- Analogous triggers for pii_change_events and communication_log

-- Revoke UPDATE and DELETE on audit tables for all roles except compliance
REVOKE UPDATE, DELETE ON audit.audit_events FROM PUBLIC;
REVOKE UPDATE, DELETE ON audit.pii_change_events FROM PUBLIC;
REVOKE UPDATE, DELETE ON audit.communication_log FROM PUBLIC;
-- HEIMDALL Core service_role gets INSERT only on audit tables
```

# 16. PII Change Capture Triggers

Each PII-bearing table gets a trigger that writes to `audit.pii_change_events`. Example for `credential_identities`:

```sql
CREATE OR REPLACE FUNCTION heimdall.capture_credential_pii_change()
RETURNS TRIGGER AS $$
DECLARE
  ctx JSONB;
BEGIN
  -- ctx is set by HEIMDALL Core at the start of each transaction
  -- via SET LOCAL heimdall.request_context = '{...}'
  ctx := COALESCE(current_setting('heimdall.request_context', true)::jsonb, '{}'::jsonb);

  IF TG_OP = 'INSERT' THEN
    INSERT INTO audit.pii_change_events (
      parent_id, actor_parent_id, actor_type, field_path, change_type,
      new_value_hash, ip_address, user_agent, device_fingerprint, device_type,
      locale, geo_country, geo_region, correlation_id
    ) VALUES (
      NEW.parent_id,
      (ctx->>'actor_parent_id')::uuid,
      COALESCE(ctx->>'actor_type','system'),
      'credential_identities.' || NEW.credential_type,
      'create',
      NEW.value_hmac,
      (ctx->>'ip_address')::inet,
      ctx->>'user_agent',
      ctx->>'device_fingerprint',
      ctx->>'device_type',
      ctx->>'locale',
      ctx->>'geo_country',
      ctx->>'geo_region',
      (ctx->>'correlation_id')::uuid
    );
  ELSIF TG_OP = 'UPDATE' AND OLD.value_hmac IS DISTINCT FROM NEW.value_hmac THEN
    INSERT INTO audit.pii_change_events (
      parent_id, actor_parent_id, actor_type, field_path, change_type,
      previous_value_hash, new_value_hash, ip_address, user_agent,
      device_fingerprint, device_type, locale, geo_country, geo_region, correlation_id
    ) VALUES (
      NEW.parent_id,
      (ctx->>'actor_parent_id')::uuid,
      COALESCE(ctx->>'actor_type','system'),
      'credential_identities.' || NEW.credential_type,
      'update',
      OLD.value_hmac,
      NEW.value_hmac,
      (ctx->>'ip_address')::inet,
      ctx->>'user_agent',
      ctx->>'device_fingerprint',
      ctx->>'device_type',
      ctx->>'locale',
      ctx->>'geo_country',
      ctx->>'geo_region',
      (ctx->>'correlation_id')::uuid
    );
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_credential_pii_change
  AFTER INSERT OR UPDATE ON heimdall.credential_identities
  FOR EACH ROW EXECUTE FUNCTION heimdall.capture_credential_pii_change();
```

Analogous triggers exist for `parent_identities`, `minor_profiles`, `guardian_relationships`, `app_personas`, and `verification_records`.

# 17. Envelope Encryption Helpers

HEIMDALL Core uses `pgcrypto`'s `pgp_sym_encrypt` and `pgp_sym_decrypt` with per-row DEKs. The Master Key (32 bytes) is held in HEIMDALL Core memory (loaded from `MASTER_KEY` runtime env or AWS KMS). Per-row DEKs are random, stored encrypted in a `dek_table`, and referenced by `dek_id` columns on PII tables.

```sql
CREATE TABLE heimdall.data_encryption_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  wrapped_key BYTEA NOT NULL,              -- DEK wrapped by Master Key (envelope encryption)
  master_key_version INT NOT NULL,
  algorithm TEXT NOT NULL DEFAULT 'aes-256-gcm',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  retired_at TIMESTAMPTZ
);

CREATE INDEX idx_dek_master_version ON heimdall.data_encryption_keys (master_key_version);
```

HEIMDALL Core implements encryption in TypeScript using `node:crypto`:

1. Generate a random 32-byte DEK in memory.
2. Wrap the DEK with the Master Key using AES-256-GCM.
3. Insert wrapped DEK into `data_encryption_keys`; receive `dek_id`.
4. Encrypt plaintext with the DEK; store ciphertext in the target column; store `dek_id` in the companion column.
5. Discard plaintext DEK from memory.

On decryption, HEIMDALL Core fetches the wrapped DEK by `dek_id`, unwraps with Master Key, decrypts ciphertext, returns plaintext, and discards the unwrapped DEK.

# 18. HMAC Strategy for Searchable PII

Searchable PII fields (email, phone, social subject) store an HMAC-SHA256 of the normalized value alongside the encrypted display value. The HMAC key is itself versioned and held in HEIMDALL Core memory (separate from the Master Key but escrowed via the same Shamir scheme).

| Field | Normalization | HMAC Key |
| --- | --- | --- |
| Email | Lowercase, trim, NFC. | `hmac_key_email_v{N}` |
| Phone | E.164 normalization via `libphonenumber-js`. | `hmac_key_phone_v{N}` |
| Social subject | Provider-specific normalization (Google: lowercase ID; Apple: case-sensitive `sub`; Facebook: numeric ID). | `hmac_key_social_v{N}` |

HMAC key rotation requires re-HMACing all rows; performed as a background pg-boss job.

# 19. Rate Limit Policies

```sql
CREATE TABLE heimdall.rate_limit_policies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scope TEXT NOT NULL
    CHECK (scope IN ('per_phone','per_ip','per_account','per_tenant','per_country','global')),
  action TEXT NOT NULL,                    -- 'sms_send','login_attempt','password_reset','otp_verify'
  limit_count INT NOT NULL,
  window_seconds INT NOT NULL,
  cost_cap_usd NUMERIC(10,2),
  tenant_id UUID REFERENCES heimdall.tenants(id),  -- null = global default
  country_code TEXT,                       -- relevant for per_country
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.rate_limit_counters (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  policy_id UUID NOT NULL REFERENCES heimdall.rate_limit_policies(id),
  scope_key TEXT NOT NULL,                 -- normalized phone, IP, parent_id, etc.
  window_start TIMESTAMPTZ NOT NULL,
  count INT NOT NULL DEFAULT 0,
  cost_usd NUMERIC(10,2) NOT NULL DEFAULT 0,
  UNIQUE (policy_id, scope_key, window_start)
);

CREATE INDEX idx_rate_counters_lookup ON heimdall.rate_limit_counters (policy_id, scope_key, window_start);
```

# 20. Payments References

```sql
CREATE TABLE heimdall.payment_provider_customer_refs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_id UUID NOT NULL REFERENCES heimdall.parent_identities(id),
  app_code TEXT NOT NULL,
  provider TEXT NOT NULL CHECK (provider IN ('stripe','paypal','square','adyen','braintree','eventbrite')),
  provider_customer_id TEXT NOT NULL,
  display_brand TEXT,                      -- 'visa','mastercard','amex'
  display_last_four TEXT,                  -- '4242'
  display_exp_month TEXT,
  display_exp_year TEXT,
  is_default BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  removed_at TIMESTAMPTZ,
  UNIQUE (parent_id, app_code, provider, provider_customer_id)
);
```

# 21. Service Accounts and API Clients

```sql
CREATE TABLE heimdall.service_accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code TEXT NOT NULL UNIQUE,
  display_name TEXT NOT NULL,
  app_code TEXT,
  tenant_id UUID,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.service_account_credentials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  service_account_id UUID NOT NULL REFERENCES heimdall.service_accounts(id),
  credential_type TEXT NOT NULL CHECK (credential_type IN ('client_credentials_secret','signing_key')),
  secret_hash TEXT,
  signing_public_key BYTEA,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  revoked_at TIMESTAMPTZ
);

CREATE TABLE heimdall.service_account_grants (
  service_account_id UUID NOT NULL REFERENCES heimdall.service_accounts(id),
  permission_id UUID NOT NULL REFERENCES heimdall.permissions(id),
  scope_descriptor JSONB,                  -- {tenant_id, app_code, etc.}
  granted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (service_account_id, permission_id)
);

CREATE TABLE heimdall.api_clients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  service_account_id UUID NOT NULL REFERENCES heimdall.service_accounts(id),
  display_name TEXT NOT NULL,
  api_key_hash TEXT NOT NULL,
  api_key_fingerprint TEXT NOT NULL,
  scope TEXT[] NOT NULL,
  rate_limit_policy_id UUID,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  revoked_at TIMESTAMPTZ
);
```

# 22. Webhooks and Outbox

```sql
CREATE TABLE heimdall.webhook_endpoints (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID REFERENCES heimdall.tenants(id),
  app_code TEXT,
  display_name TEXT NOT NULL,
  url TEXT NOT NULL,
  signing_secret_ref UUID NOT NULL,        -- ref into secret_items
  event_types TEXT[] NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE heimdall.webhook_deliveries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  endpoint_id UUID NOT NULL REFERENCES heimdall.webhook_endpoints(id),
  event_id UUID NOT NULL,
  payload JSONB NOT NULL,
  attempt_count INT NOT NULL DEFAULT 0,
  next_attempt_at TIMESTAMPTZ,
  delivered_at TIMESTAMPTZ,
  last_http_status INT,
  last_error TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhook_pending
  ON heimdall.webhook_deliveries (next_attempt_at)
  WHERE delivered_at IS NULL;
```

# 23. Break-Glass

```sql
CREATE TABLE heimdall.break_glass_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  requester_parent_id UUID NOT NULL,
  reason TEXT NOT NULL,
  requested_scope JSONB NOT NULL,          -- what elevated access is being requested
  requested_duration_minutes INT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending','approved','denied','expired','consumed','revoked')),
  requested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  approved_by_parent_id UUID,
  approved_at TIMESTAMPTZ,
  denied_at TIMESTAMPTZ,
  denial_reason TEXT
);

CREATE TABLE heimdall.break_glass_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  request_id UUID NOT NULL REFERENCES heimdall.break_glass_requests(id),
  parent_id UUID NOT NULL,
  started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL,
  ended_at TIMESTAMPTZ,
  end_reason TEXT
);
```

# 24. Partitioning Strategy

The three audit-schema tables are partitioned monthly by `created_at`. A pg-boss job rolls forward by creating the next two months' partitions weekly. Partitions older than 24 months are detached and archived to immutable object storage (Backblaze B2 / Cloudflare R2 in Phase 1; S3 Object Lock Compliance mode in Phase 2). Detached partitions remain queryable through a `pg_dump`-based restore-on-demand workflow for legal hold investigations.

# 25. Seed Data

The following are inserted by migration on initial deploy:

| Seed | Detail |
| --- | --- |
| Applications | `myriad-dashboard`, `heimdall-admin`, `heimdall-login`, `pedl`, `quest`. |
| Default tenant | `myriad-internal` (tier=`myriad_internal`), `myriad-products` (tier=`myriad_products`). |
| System roles | All Myriad and tenant role tiers from Requirements §13. |
| System permissions | Identity, tenant, secret, audit, support, and per-app permission codes. |
| Age policy rules | `(app=*, jurisdiction=US-COPPA, minimum_age=13, requires_guardian_consent=true, restricted_features=[...])` |
| Rate limit policies | Defaults from Requirements §23.3. |
| Disclosures (global) | `terms_of_service` v1.0, `privacy_policy` v1.0 (placeholder bodies; legal authors final content). |
| Verification sources | `stripe_identity` (disabled until credentials added). |
| Federated identity providers | `google`, `facebook`, `apple` (disabled until credentials and per-tenant policy added). |
| Communication templates | Anonymization initial, day-15, final, executed, cancelled; disclosure re-consent; MFA enrolled; password changed; suspicious login. |

# 26. Indexes Summary

Critical hot-path indexes:

| Table | Index | Reason |
| --- | --- | --- |
| `credential_identities` | `(credential_type, value_hmac)` | Login lookup, dedup checks. |
| `parent_identities` | `(status)` | Filtering active identities. |
| `persona_roles` | `(persona_id)` and `(role_id)` | Authorization fast path. |
| `tenant_memberships` | `(parent_id, tenant_id, app_code)` UNIQUE | Authorization scope resolution. |
| `session_records` | `(parent_id) WHERE revoked_at IS NULL` | Active session lookup. |
| `audit_events` | `(parent_id, created_at)`, `(correlation_id)` | Compliance queries. |
| `pii_change_events` | `(parent_id, created_at)` | GDPR right-to-know responses. |
| `anonymization_requests` | `(scheduled_execution_at) WHERE status='pending'` | pg-boss scheduler. |
| `legal_holds` | `(parent_id) WHERE released_at IS NULL` | Hot path on every login. |
| `oidc_payloads` | `(grant_id)`, `(user_code)`, `(uid)`, `(expires_at)` | `oidc-provider` adapter performance. |

# 27. Migration Strategy

Drizzle Kit generates SQL migrations into `packages/db/migrations/`. Each migration is reviewed and committed. The migration runner is invoked at HEIMDALL Core startup with an advisory lock to prevent concurrent migrations during multi-instance deploys. Seed data lives in `packages/db/seeds/` and runs after migrations. Data migrations (one-shot transformations) live in `packages/db/data-migrations/` and are run manually with explicit operator confirmation.

Backward-compatible migrations are required: every release deploys schema changes first, then code that uses them. Drop columns and tables in a separate later release after all code references are removed.

HEIMDALL — Data Model and Schema Companion | Myriad Technology Group
