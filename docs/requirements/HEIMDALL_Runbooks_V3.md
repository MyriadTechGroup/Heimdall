# HEIMDALL — Security and Operations Runbooks

## Companion Document to HEIMDALL Requirements V3

Prepared for Myriad Technology Group | 2026-05-19

| Document Field | Value |
| --- | --- |
| Document Type | Security and Operations Runbooks (companion to V3 Requirements) |
| Audience | Myriad Engineering, Myriad Security, Myriad Legal, Myriad Operations |
| Refresh Cadence | Quarterly review minimum. Updates required after any material architectural change. |
| Drill Cadence | Each runbook in this document is rehearsed at least once per year unless otherwise noted. Master key recovery and breach response are rehearsed twice per year. |

# 1. Runbook Index

| # | Runbook | Trigger | Owner Role | Drill Cadence |
| --- | --- | --- | --- | --- |
| RB-01 | Master Key Rotation (Zero Downtime) | Scheduled (annual) or compromise | Myriad CTO | Annual |
| RB-02 | Master Key Recovery via Shamir Reconstruction | Master key lost | Myriad CTO + escrow holders | Semi-annual |
| RB-03 | Breach Response | Suspected or confirmed breach | Myriad CTO + Security Lead | Semi-annual |
| RB-04 | Anonymization Execution Audit | Quarterly compliance review or DSAR audit | Myriad Legal Officer | Quarterly |
| RB-05 | Legal Hold Placement and Dual-Control Override | Court order, regulator request, DPA | Myriad Legal Officer | Annual |
| RB-06 | Impersonation Approval Audit | Monthly review or specific incident | Myriad Security Auditor | Monthly |
| RB-07 | SMS Cost-Cap Response | Cost-cap alert triggered | On-call Engineer | As-needed |
| RB-08 | Credential Stuffing Response | Pattern detected at edge or app | On-call Engineer + Security Lead | Annual |
| RB-09 | Twilio Provider Outage | Twilio service degradation | On-call Engineer | Annual |
| RB-10 | SendGrid Provider Outage | SendGrid service degradation | On-call Engineer | Annual |
| RB-11 | Supabase Provider Outage | Supabase service degradation | On-call Engineer + CTO | Annual |
| RB-12 | AWS Migration Cutover | Phase 2 readiness | Myriad CTO + Engineering Lead | One-time, pre-rehearsed |
| RB-13 | Key Version Rollback | Bad key version detected | Myriad CTO | Annual |
| RB-14 | Audit Log Integrity Verification | Quarterly or post-incident | Myriad Security Auditor | Quarterly |
| RB-15 | Disclosure Republication and Re-Consent Wave | Legal-driven update | Myriad Legal Officer | As-needed |

# 2. Common Terms

| Term | Definition |
| --- | --- |
| Master Key | 32-byte symmetric key (AES-256) that wraps per-row Data Encryption Keys. Held in HEIMDALL Core process memory, sourced from `MASTER_KEY` environment variable (Phase 1) or AWS KMS (Phase 2). |
| DEK | Data Encryption Key. Per-row random key used to encrypt a single PII or secret value. Stored encrypted in `heimdall.data_encryption_keys`. |
| HMAC Key | Symmetric key used to produce searchable hashes of normalized PII (email, phone). Separate key for each field type, versioned. |
| Shamir Holder | A person who holds one share of the master key recovery scroll. Five holders, three required to reconstruct. |
| Break-Glass | Emergency elevated access procedure. Always time-bound and dual-controlled where possible. |
| Correlation ID | UUID propagated across HEIMDALL Core, audit events, communication log, and external provider calls for the same request or workflow. |

# 3. Shamir's Secret Sharing Escrow Plan

The Master Key and the HMAC key set are split using Shamir's Secret Sharing (3-of-5 scheme) at generation time. Shares are distributed to five designated holders:

| Holder | Role | Storage Form |
| --- | --- | --- |
| Holder 1 | Myriad CEO | Hardware security token (YubiKey HSM) in personal safe deposit box. |
| Holder 2 | Myriad CTO | Hardware security token (YubiKey HSM) in personal safe deposit box. |
| Holder 3 | Myriad General Counsel | Hardware security token (YubiKey HSM) in personal safe deposit box. |
| Holder 4 | Myriad Operations Lead | Hardware security token (YubiKey HSM) in office safe with dual-locking. |
| Holder 5 | External Law Firm (Engagement Partner) | Hardware security token (YubiKey HSM) in law firm's secured custody. |

Three holders are required to reconstruct. No single holder can decrypt anything. Holder rotation requires re-generating the share set; the runbook for share rotation is identical to RB-02 in reverse (generate new master, redistribute, retire old shares). Annual rotation is recommended; mandatory rotation occurs upon role change of any holder.

# 4. RB-01: Master Key Rotation (Zero Downtime)

## 4.1 Triggers

- Scheduled annual rotation.
- Suspected master key compromise.
- Departure of any Shamir holder.
- Cryptographic library update that recommends rotation.

## 4.2 Preconditions

- HEIMDALL Core is healthy.
- `data_encryption_keys` table backup confirmed.
- A second on-call engineer is available throughout the rotation.

## 4.3 Procedure

1. **Generate the new master key.** On a single trusted workstation, run the master-key generation script (`packages/scripts/generate-master-key.ts`). Output is a 32-byte random key in hex. Workstation is wiped of any disk cache after the generation.

2. **Split the new master key using Shamir's Secret Sharing.** Use the standard tool with 3-of-5 threshold. Print share material to dedicated airgap printer. Distribute to the five Shamir holders in person or by trusted courier with signed receipt.

3. **Stage the new key.** Deploy a HEIMDALL Core build that knows both the current master key (version N) and the new master key (version N+1). The build reads `MASTER_KEY_CURRENT` and `MASTER_KEY_NEXT` from environment. New DEKs created during the staging window are encrypted with version N+1; existing DEKs continue to be decrypted with version N.

4. **Re-wrap all DEKs.** Run the pg-boss job `master_key.rewrap_deks` which iterates `data_encryption_keys`, unwraps each row with master key version N, re-wraps with master key version N+1, sets `master_key_version = N+1`. The job streams in batches of 500 rows and uses `SELECT ... FOR UPDATE SKIP LOCKED` to permit concurrent re-wrapping across multiple HEIMDALL Core instances.

5. **Verify.** Run a verification job that decrypts a sample of 1000 random encrypted columns end-to-end and compares the resulting plaintexts against fingerprints recorded before the rotation. Any mismatch halts the rotation.

6. **Promote.** Deploy a HEIMDALL Core build that reads only `MASTER_KEY_CURRENT = N+1`. Remove `MASTER_KEY_NEXT` from environment configuration.

7. **Retire old key.** After 72 hours of clean operation, mark all `data_encryption_keys` rows for version N as `retired_at = NOW()`. Delete the version-N material from any remaining holder backups, including the five Shamir-share backups, after a 90-day cooling period.

## 4.4 Rollback

If verification fails at step 5, the staged build at step 3 can decrypt with either key version. Revert by halting the re-wrap job and reverting `master_key_version` on any partially re-wrapped rows from a snapshot. Investigate root cause before re-attempting.

## 4.5 Audit

Every step generates `secret.master_key_rotation_step_completed` audit events. The post-completion review includes confirming an unbroken trail of these events plus the verification sample report.

# 5. RB-02: Master Key Recovery via Shamir Reconstruction

## 5.1 Triggers

- Master key environment variable lost (catastrophic infrastructure loss).
- HEIMDALL Core cannot decrypt secrets or PII (suspected key corruption).
- Disaster recovery activation.

## 5.2 Preconditions

- At least three Shamir holders are available and willing to participate.
- A trusted workstation is available for reconstruction. Workstation is airgapped (no network) during reconstruction.
- The reconstruction is conducted in person where possible. Video conferencing is permitted under exceptional circumstances with all participants on camera throughout.

## 5.3 Procedure

1. **Convene the reconstruction.** Myriad CTO or General Counsel initiates by contacting at least three Shamir holders. Time and location are recorded in `audit_events` under `security.key_recovery.initiated`.

2. **Verify each holder's identity.** Each holder presents government ID. The convener confirms.

3. **Read each share.** Each holder inserts their YubiKey HSM into the airgapped workstation in turn and reads out their share material. The workstation's reconstruction script consumes the shares without persisting them to disk.

4. **Reconstruct the master key.** The reconstruction script applies Shamir reconstruction and produces the 32-byte master key in memory.

5. **Verify the reconstructed key.** The script computes a fingerprint of the reconstructed key (HMAC-SHA256 of a known nonce, where the fingerprint was recorded at original generation time and stored in an envelope-sealed verification artifact). The fingerprint must match. If it does not, halt and investigate.

6. **Apply the key.** The reconstructed key is loaded into the HEIMDALL Core production environment via the platform-specific secure environment variable mechanism (Render, AWS Parameter Store, etc.) by the CTO or Operations Lead from the airgapped workstation. HEIMDALL Core is then restarted.

7. **Verify HEIMDALL Core health.** Confirm `/readyz` returns 200, sample a small number of secret decryptions, sample a small number of PII reads, all should succeed.

8. **Re-escrow the master key.** Within 30 days, run RB-01 to rotate the master key and re-establish a fresh Shamir share set. The reconstructed key is treated as transitional.

## 5.4 Rehearsal Cadence

Semi-annual. Rehearsal uses a non-production Shamir share set against a non-production master key. Holders practice the physical and procedural steps. The rehearsal is observed by the Security Auditor and a report is filed.

# 6. RB-03: Breach Response

## 6.1 Triggers

- Confirmed credential leak from a third-party service that may grant attacker entry.
- Suspected unauthorized access to `secrets` table or master key.
- Anomalous audit event pattern (mass token issuance, sudden secret access spike, sudden PII export volume).
- External notification (security researcher, abuse report, regulator).

## 6.2 Severity Levels

| Level | Definition | Response Time |
| --- | --- | --- |
| SEV-1 | Confirmed or highly likely breach of master key, secrets table, or production database. | Immediate. Pager-class. |
| SEV-2 | Confirmed or highly likely breach of a single secret, a single tenant's data, or a support agent account. | Within 1 hour. |
| SEV-3 | Suspicious activity that may indicate breach. | Within 4 hours. |

## 6.3 Immediate Response (SEV-1)

1. **Form the incident channel.** Slack channel `#incident-{timestamp}` created by on-call. CTO, Security Lead, Engineering Lead, Legal Officer, CEO joined within 15 minutes.

2. **Establish a scribe.** One person records timeline, decisions, and actions in real time.

3. **Triage scope.** Determine what is confirmed, what is suspected, what is excluded.

4. **Contain.**
   - Revoke all OIDC sessions if master key compromise is suspected: `UPDATE heimdall.session_records SET revoked_at = NOW(), revoked_reason = 'incident_response' WHERE revoked_at IS NULL;`
   - Revoke all impersonation approvals.
   - Place legal holds on any specifically affected parent identities.
   - Disable suspected compromised admin accounts.
   - If a specific secret is compromised, rotate it immediately using RB-01-style procedure for that secret.
   - If master key compromise is confirmed, execute RB-01 against the entire DEK set as fast as safely possible (this may take hours; document the window).

5. **Notify.**
   - Internal: CEO and General Counsel notified immediately.
   - Regulators: General Counsel determines GDPR (72-hour clock), CCPA, state breach laws (varying timelines), and contractual notification obligations. The General Counsel owns regulator and customer notification, not Engineering.
   - Customers: Customer notification is drafted by Legal and Communications, reviewed by CTO for technical accuracy, sent through the Communications Registry to capture an immutable record of who received what notification when.

6. **Preserve evidence.** Snapshot affected database state. Export relevant audit log windows to immutable storage. Preserve Cloudflare logs (request through 30-day Cloudflare log retention or longer via Logpush). Do not modify or delete anything.

7. **Investigate.** Forensics by internal Security Lead plus, where appropriate, an external incident response firm.

8. **Eradicate.** Root-cause fix deployed. New keys, new credentials, new sessions, patched vulnerability.

9. **Recover.** Service restored to full operation. Communications confirming resolution sent.

10. **Post-incident review.** Within 14 days, blameless post-mortem written. Action items tracked. Runbook updated.

## 6.4 SEV-2 and SEV-3

Same shape, scaled down. SEV-3 may proceed with engineering investigation alone unless escalation criteria are met.

# 7. RB-04: Anonymization Execution Audit

## 7.1 Purpose

Verify quarterly that:

- Every `anonymization_requests` row with `status='executed'` has corresponding `audit_events` entries for `anonymization.executed`, the four required notification entries in `anonymization_request_notifications`, and matching `pii_change_events` rows of type `anonymize` on the affected tables.
- No request scheduled for execution was missed.
- No request was executed before its `scheduled_execution_at` unless `execute-now` was invoked with a legal reference.

## 7.2 Procedure

1. Pull a SQL report for the quarter covering all anonymization requests, their statuses, scheduled vs actual execution times, notification log, and matching audit events.
2. Spot-check 10% of executed requests by `parent_id`: query `parent_identities`, `credential_identities`, `app_personas`, `verification_records` to confirm tombstoning was applied per Requirements §16.3.
3. For every `status='executed'` row, confirm `communication_log` shows initial, day-15, final reminder, and executed notifications were sent (or, where contact channels were invalid, that the failed deliveries are logged).
4. For every `status='blocked_legal_hold'` row, confirm a matching active legal hold record exists.
5. For every `execute-now` invocation, confirm `legal_reference` is non-empty and the audit event includes the approver.
6. Sign off the quarterly report. File in compliance archive.

## 7.3 Failure Modes

Any discrepancy raises a SEV-2 incident. Likely causes: failed pg-boss job (notification not sent), trigger misfire (anonymization run without audit event), or operator error (execute-now without reference).

# 8. RB-05: Legal Hold Placement and Dual-Control Override

## 8.1 Placement

1. Myriad Legal Officer receives the legal trigger (court order, regulator letter, internal litigation hold memo).
2. Legal Officer signs into HEIMDALL Admin UI with hardware MFA.
3. Legal Officer creates the hold via `/v1/legal-holds` POST, supplying authority, reference_id, reason_code, reason_notes.
4. HEIMDALL Core revokes all sessions for the held parent_id, queues any pending anonymization requests, blocks future logins.
5. The held user, on next login attempt, sees the generic "account temporarily unavailable, contact support" message. The legal hold reason is never disclosed to the user.
6. If the user contacts support, support sees the legal hold flag and routes the inquiry to Legal. Support agents do not discuss the hold's existence with the user.

## 8.2 Override Procedure (When a Held Account Must Take an Action)

Override is needed when, for example, court-ordered disclosure requires the Myriad team to export data from the held account, or when a regulator-supervised investigation requires controlled access.

1. `myriad_super_admin` (Admin A) creates an override request via `/v1/legal-holds/{hold_id}/override-requests` with reason and requested action.
2. A different `myriad_super_admin` (Admin B) reviews the request and co-signs via `/v1/legal-holds/override-requests/{override_id}/cosign` within 60 minutes of the original request.
3. Co-signing requires hardware MFA from Admin B.
4. The override executes the requested action and immediately closes. Every step generates `legal_hold.override_requested`, `legal_hold.override_approved` (the co-sign), and event(s) for the action taken.
5. Override expires automatically if Admin B does not co-sign within 60 minutes.
6. Self-override (Admin A co-signing their own request) is impossible by API design and additionally audited as an anomaly if ever attempted.

## 8.3 Release

1. Legal Officer confirms the legal trigger has ended.
2. Release recorded via `/v1/legal-holds/{hold_id}/release` POST with reason.
3. Queued anonymization requests, if any, are reactivated with their remaining timeline.

# 9. RB-06: Impersonation Approval Audit

## 9.1 Cadence

Monthly. Reviews the prior month's `impersonation_approvals` and `support_session_events`.

## 9.2 Procedure

1. Pull all `impersonation_approvals` for the period.
2. For each approved approval, confirm:
   - The requester and approver are different people.
   - The approver held the required role (`myriad_support_l2` or `tenant_admin` per scope).
   - The justification is non-empty and references a real ticket.
   - The session, if started, completed within the granted window.
3. For each session, review `support_session_events`:
   - Spot-check that the actions taken align with the stated justification.
   - Confirm the impersonation banner was rendered (confirmed via app-side telemetry).
4. For each denied approval, confirm reason is documented.
5. For each expired approval (requester did not start within 30 min), no review required; logged only.
6. Compute aggregates: approval rate, average session duration, top approvers, top requesters.
7. Sign off. Anomalies escalate to Security Lead.

## 9.3 Red Flags

- Self-approval attempts (should be impossible by API; alert if found).
- Approver and requester have personal relationship (HR-checked separately).
- Multiple sessions on a single user from different agents within a short window without ticket linkage.
- Sessions outside business hours without on-call justification.

# 10. RB-07: SMS Cost-Cap Response

## 10.1 Alerts

| Threshold | Alert |
| --- | --- |
| 50% of daily global SMS cost cap | Slack notification to on-call. |
| 75% of daily global SMS cost cap | Slack + email to on-call and Engineering Lead. |
| 90% of daily global SMS cost cap | Pager-class to on-call. |
| 100% reached | HEIMDALL Core automatically halts new SMS sends. Pager-class to on-call and CTO. |
| Per-country cap reached | SMS to that country halted. Alert to on-call. |

## 10.2 Investigation

When an alert fires:

1. Check `audit_events` and `rate_limit_counters` for SMS volume distribution by country, by tenant, by phone number prefix.
2. Distinguish legitimate volume spike from suspected fraud:
   - **Legitimate** — known marketing event, new tenant launch, holiday seasonality. Discuss with Engineering Lead whether to raise cap for the day.
   - **Pumping** — concentrated traffic to expensive country prefixes, often premium-rate numbers. Concentrated traffic from a single IP or single tenant. Twilio Verify carrier risk score elevated. Identify pattern; apply blocks.
3. Mitigations available:
   - Reduce per-tenant SMS quota in `rate_limit_policies` for the offending tenant.
   - Block specific country code via `rate_limit_policies` with country_code scope.
   - Block specific IP via Cloudflare WAF.
   - Block specific phone-number prefix via HEIMDALL Core code check.
   - Disable SMS for the affected app or tenant entirely; switch to email or in-app OTP.
4. Notify the affected tenant if mitigation degrades their service.
5. Post-incident review if the cap was actually reached.

## 10.3 Documenting

Every cap event generates `rate_limit.cap_reached` audit events including the policy hit and the scope_key. Post-incident review filed in compliance archive.

# 11. RB-08: Credential Stuffing Response

## 11.1 Detection Signals

- Cloudflare Bot Management or WAF flags concentrated login attempts.
- HEIMDALL Core's failed-login metric crosses threshold for a single tenant or IP range.
- `audit_events` show a spike in `auth.login.failure` events with diverse `parent_id` targets from few sources.

## 11.2 Immediate Mitigations

1. **Edge-level.** Lower Cloudflare rate limit thresholds for `/v1/auth/login/*` and `/oidc/auth`. Enable Cloudflare Turnstile challenge on every login attempt (default is challenge-on-suspicion). Consider geo-blocking if traffic comes from a single non-customer geography.
2. **App-level.** Increase per-IP and per-account login attempt limits temporarily.
3. **Account-level.** For any account showing 5+ failed attempts, automatic temporary lockout (15 minutes) plus email/SMS notification to the legitimate user.
4. **Tenant-level.** If a single tenant is targeted, optionally raise their MFA policy temporarily to require MFA on every login (rather than first device).

## 11.3 Investigation

1. Identify whether actual account compromises occurred (correlate failed-then-successful login patterns).
2. If credentials were validated against HEIMDALL, those are confirmed compromised externally; force password reset on the affected accounts.
3. Determine source list characteristics. Report to law enforcement only with General Counsel approval and direction.

## 11.4 Communication

1. Affected users receive a "suspicious login activity" email with steps to secure their account (change password, enable MFA, review trusted devices).
2. If a significant subset of users is affected, a tenant-level or app-level disclosure is prepared by Legal and Communications.

# 12. RB-09: Twilio Provider Outage

## 12.1 Detection

- Twilio Verify API returns 5xx or times out beyond the configured 30-second envelope.
- HEIMDALL Core's `SmsProvider` health metric drops below 80% success rate over 5 minutes.

## 12.2 Mitigations

1. **Graceful degradation.** HEIMDALL Core emits banner to active sessions: "SMS verification temporarily unavailable. Please use TOTP, email, or WebAuthn." Active SMS challenges are not invalidated.
2. **Failover path (Phase 2+).** If a secondary SMS provider is configured (e.g., AWS End User Messaging), HEIMDALL Core rotates `SmsProvider` to the secondary via configuration update. Account onboarding flows that require SMS get the secondary path.
3. **Disclosure.** If outage exceeds 30 minutes, post status on the tenant admin dashboard.

## 12.3 Recovery

1. Re-enable Twilio Verify as primary once Twilio status returns to green.
2. Re-test by issuing a verification to a known good phone.
3. Reconcile any users locked out during the outage; offer recovery via alternate factor.

# 13. RB-10: SendGrid Provider Outage

## 13.1 Detection

- SendGrid API returns 5xx or webhooks stop arriving for delivery confirmations.
- Bounce rates elevate suddenly across multiple tenants (suggesting upstream IP-reputation problem).

## 13.2 Mitigations

1. **Queue retention.** pg-boss `communication.delivery` job has retry with backoff. Messages persist in `communication_log` with `status='queued'`.
2. **Failover (Phase 2+).** If a secondary provider is configured (AWS SES), switch via `communication_policy` per tenant.
3. **Critical-only mode.** For SEV-1-class messages (anonymization notifications, security alerts), engineering may opt to deliver via a backup transactional ESP account.

## 13.3 Recovery

1. Re-enable SendGrid as primary once status is green.
2. Reconcile queued messages: re-send any that did not reach the user.

# 14. RB-11: Supabase Provider Outage

## 14.1 Detection

- HEIMDALL Core's `/readyz` returns 503 because the Postgres connection pool is degraded.
- Supabase Status Page reports incident.

## 14.2 Mitigations

1. **Active read-only mode.** If Supabase reports degraded writes but reads are healthy, HEIMDALL Core can flip to read-only mode. Existing sessions remain valid (in-memory JWT verification needs no DB). Logins and registrations are temporarily unavailable. Step-up MFA and impersonation are also unavailable. Active product usage continues for already-signed-in users (their apps degrade according to their own designs).
2. **Communication.** Status banner on Hosted Login Pages: "Authentication temporarily unavailable for new sessions. Please try again shortly."
3. **Read replica failover (if configured in Phase 2).** Promote a read replica to primary. Standby procedure documented as part of AWS migration runbook.

## 14.3 Catastrophic Loss

In the (very unlikely) event Supabase loses the primary database entirely:

1. Restore from the most recent nightly logical backup exported to non-Supabase storage.
2. Re-apply any audit log archive partitions from immutable object storage if hot Postgres lost them (audit only; never re-applied as state).
3. The data loss window equals the backup interval (24 hours). Communicate to users and tenants.
4. CTO and Engineering Lead jointly evaluate emergency migration to AWS RDS using RB-12 if Supabase recovery is uncertain.

## 14.4 Recovery

1. Once Supabase reports green, return to read-write mode.
2. Reconcile any in-flight requests that errored mid-write.
3. Post-incident review.

# 15. RB-12: AWS Migration Cutover

## 15.1 Purpose

Migrate HEIMDALL Core and its database from Phase 1 hosting (Render + Supabase + Backblaze B2) to Phase 2 hosting (AWS ECS Fargate + RDS/Aurora PostgreSQL + S3 Object Lock). The migration is rehearsed end-to-end in staging before any production cutover.

## 15.2 Pre-Migration Preparations

1. AWS account provisioned with required services: VPC, RDS/Aurora PostgreSQL cluster, ECS Fargate cluster, ALB, KMS keys, Secrets Manager, S3 buckets with Object Lock Compliance mode, CloudWatch log groups, EventBridge or SQS if adopted.
2. Database parameter compatibility verified: Aurora PostgreSQL major version matches Supabase's currently-running major version (or one major newer with verified migration path).
3. `pgcrypto` extension enabled on RDS/Aurora.
4. Master Key migrated: a fresh master key is generated in AWS KMS, and HEIMDALL Core is configured to wrap DEKs with KMS (envelope encryption via KMS GenerateDataKey API). The Phase 1 Master Key continues to wrap existing DEKs until re-wrapping completes.
5. HEIMDALL Core container image deployed to ECS Fargate in a staging environment. End-to-end smoke tests pass.
6. Cutover window scheduled. User communication sent 7 days in advance and 24 hours in advance.

## 15.3 Cutover Procedure

1. **T-24h.** Enable maintenance banner on Hosted Login Pages and HEIMDALL Admin UI: "Scheduled maintenance Sunday 2am–4am ET."
2. **T-1h.** Final dry-run of cutover script against staging. On-call team assembled.
3. **T-0h. Begin window.**
   a. Set HEIMDALL Core to read-only mode (sessions continue; no writes).
   b. Trigger final Supabase snapshot.
   c. Restore the snapshot into RDS/Aurora.
   d. Verify row counts match across all tables.
   e. Update DNS or load-balancer routing to point HEIMDALL Core endpoints to the AWS-hosted instances.
   f. Switch HEIMDALL Core configuration to read from AWS KMS for master key, AWS Secrets Manager for `MASTER_KEY` itself (Phase 2 replaces runtime env with Secrets Manager-fetched master key).
   g. Disable read-only mode.
   h. Run health checks; promote.
4. **T+1h.** Re-wrap DEKs from Phase 1 Master Key to AWS KMS-wrapped Master Key per RB-01 procedure. This continues in the background.
5. **T+24h.** Phase 1 hosting (Render + Supabase) remains warm as fallback. Phase 1 secrets remain in place until re-wrap completes.
6. **T+7d.** If no issues, decommission Phase 1 hosting. Retain Supabase as a cold backup for the next 90 days.

## 15.4 Rollback

At any step before T+24h, HEIMDALL Core can be reverted to Phase 1 hosting by reverting DNS, restoring the writes that occurred post-cutover from the AWS-hosted database into Supabase, and resuming. The window for clean rollback narrows as users generate new data on the AWS side; a documented "point of no return" exists at T+24h.

# 16. RB-13: Key Version Rollback

## 16.1 Trigger

A new master key version (N+1) is suspected to have a flaw, but the rotation has already begun. Some DEKs are wrapped with N, some with N+1.

## 16.2 Procedure

1. Halt the re-wrap job.
2. Identify the affected rows: `SELECT id, master_key_version FROM heimdall.data_encryption_keys;` shows the split.
3. If only a small subset of DEKs are wrapped with N+1, re-wrap them back to N using the same staged build that knows both keys. Resume in halted state.
4. If a large subset are wrapped with N+1, the choice is: forward-fix (generate N+2 and rotate from N+1 to N+2) or backward-fix (rotate back to N). Forward-fix preserves more recent work; choice driven by why N+1 is suspected.
5. Generate the correction key, run rotation, verify, promote.

# 17. RB-14: Audit Log Integrity Verification

## 17.1 Cadence

Quarterly and after any incident response.

## 17.2 Procedure

1. Run the hash-chain verification script (`packages/scripts/audit-verify.ts`) which traverses each ledger table partition. For each row, the script:
   - Loads `prev_hash` and `row_hash`.
   - Recomputes the expected `row_hash` from row content plus `prev_hash`.
   - Reports any mismatch.
2. Compare row counts in `audit.audit_events` against operational metrics: number of logins, persona switches, secret accesses, etc., as known from external sources where available.
3. Verify that no partition is missing.
4. Spot-check cold archive partitions: download a sample partition from immutable storage, verify hash chain, verify a random sample of events match the warm-tier replicas (where applicable) or operational records.
5. File the verification report.

## 17.3 If Mismatches Are Found

Treat as SEV-2 minimum. Hash mismatches indicate either:

- Corruption (disk, replication, restoration).
- Tampering. Investigate immediately. Hash-chain protection is designed to make tampering detectable; an actual mismatch is a significant event.

# 18. RB-15: Disclosure Republication and Re-Consent Wave

## 18.1 Trigger

Legal Officer determines a disclosure (Terms of Service, Privacy Policy, COPPA notice, etc.) requires a new version with material change requiring re-acceptance.

## 18.2 Procedure

1. Legal Officer drafts the new version in HEIMDALL Admin UI via `/v1/disclosures/{disclosure_id}/versions` POST with `requires_re_acceptance=true`.
2. Engineering Lead reviews the diff via `/v1/disclosures/.../versions/.../diff` for technical accuracy (links, references).
3. Communications drafts the email and in-app notice that will accompany the publication.
4. CEO or General Counsel approves activation.
5. Activate via `/v1/disclosures/.../versions/.../publish`. Previous version `superseded_at` set.
6. HEIMDALL Core flags every active user requiring re-acceptance on their next login. The flag is computed on-demand by comparing `disclosure_acceptances` against active versions.
7. Communication campaign launches: notice email or SMS to all affected users. Tracked in `communication_log`.
8. Monitor re-acceptance rate over 30 days. If users have not re-accepted, their access to features requiring the disclosure is gated until they do.
9. After 90 days, escalate non-accepters: either grace extended (with Legal approval) or accounts moved to a restricted state.

## 18.3 Audit

Every step generates `disclosure.published`, `disclosure.acceptance_required_wave`, `communication.sent`, `disclosure.accepted` events. The campaign is auditable end-to-end.

# 19. Incident Severity Matrix

| Category | SEV-1 | SEV-2 | SEV-3 |
| --- | --- | --- | --- |
| Authentication | Login unavailable for >5 min. | Login degraded (slow or partial). | Single-feature error (e.g., one social provider down). |
| Database | Production database unavailable. | Replica lag exceeds 30s. | Query slowdown without availability impact. |
| Secrets | Suspected master key compromise. | Single secret compromise. | Failed rotation attempt. |
| PII | Confirmed unauthorized PII export. | Suspected unauthorized PII view. | Unusual but explained PII access pattern. |
| External Provider | Critical provider down >15 min (Twilio Verify, SendGrid, Stripe Identity). | Critical provider degraded. | Non-critical provider down. |
| Audit | Hash-chain integrity violation. | Audit log lag exceeds 60s. | Partition rollover late. |

# 20. On-Call Expectations

| Role | Availability | Response SLA |
| --- | --- | --- |
| Primary on-call engineer | 24/7 during rotation week. | SEV-1: 15 min. SEV-2: 1 hour. SEV-3: next business day. |
| Secondary on-call engineer | 24/7 during rotation week. | Joins SEV-1 within 30 min of primary acknowledgement. |
| Engineering Lead | Business hours plus SEV-1 escalation. | SEV-1: 30 min. |
| CTO | Business hours plus SEV-1 escalation. | SEV-1: 1 hour. |
| Security Lead | Business hours plus security incidents. | SEV-1 security: 30 min. |
| General Counsel | Business hours plus legal incidents. | SEV-1 legal: 1 hour. |

# 21. Communications Templates for Incidents

The Communications Registry holds pre-drafted incident-comm templates for re-use:

| Template Code | Use |
| --- | --- |
| `incident_initial_status` | Initial public acknowledgement, scope-vague. |
| `incident_update_progress` | Updates during ongoing incident. |
| `incident_resolution` | All-clear announcement. |
| `breach_user_notice` | Per-user notification of confirmed personal-data breach. |
| `breach_regulator_notice_draft` | Starting point for regulator notification, finalized by Legal. |
| `outage_status_update` | Service-availability impact (non-security). |

Pre-drafted does not mean pre-published. Templates are reviewed and modified per incident by Legal and Communications.

# 22. Tooling

| Tool | Purpose |
| --- | --- |
| Slack | Incident channels, on-call notifications, impersonation approval alerts. |
| PagerDuty (or equivalent) | On-call paging. |
| HEIMDALL Admin UI | Most runbook actions can be initiated here with full audit. |
| `packages/scripts/` | CLI scripts for break-glass actions: master key rotation, DEK re-wrap, audit verify, key recovery reconstruction, anonymization execution dry-run. All scripts are itself audited and require an authenticated operator with appropriate permission. |
| Datadog or Grafana Cloud | Observability dashboards, alert rules. |
| Cloudflare Dashboard | Edge-level mitigations. |

# 23. Drill Calendar (Annual Template)

| Quarter | Drills |
| --- | --- |
| Q1 | RB-02 Master Key Recovery (rehearsal with non-prod shares); RB-04 Anonymization Audit; RB-06 Impersonation Approval Audit; RB-14 Audit Log Integrity Verification. |
| Q2 | RB-03 Breach Response (tabletop exercise SEV-1 scenario); RB-09 Twilio Outage; RB-10 SendGrid Outage. |
| Q3 | RB-01 Master Key Rotation (production-class rehearsal in staging); RB-08 Credential Stuffing tabletop; RB-04 Anonymization Audit; RB-06 Impersonation Approval Audit. |
| Q4 | RB-02 Master Key Recovery (rehearsal); RB-11 Supabase Outage tabletop; RB-15 Disclosure Republication tabletop; RB-14 Audit Log Integrity Verification. |

Phase 2-specific drills (RB-12 AWS Migration) are rehearsed at least three times in staging in the quarter preceding cutover.

# 24. Document Maintenance

Owner: Myriad CTO.
Review: quarterly.
Major rev: with any material architectural change.
Distribution: Engineering, Security, Legal, Operations, on-call rotation members, Shamir holders (RB-02 procedure copy only).

HEIMDALL — Security and Operations Runbooks Companion | Myriad Technology Group
