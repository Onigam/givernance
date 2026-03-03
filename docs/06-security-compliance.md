# 06 — Security & Compliance (EU/GDPR)

## Control baseline
- Data residency: EU-only regions by default
- Encryption at rest (AES-256) + in transit (TLS 1.3 minimum)
- Tenant isolation via `org_id` + Postgres RLS
- RBAC + permission scopes by capability
- Immutable audit log for privileged actions
- Secrets in vault; no plaintext secrets in DB

## GDPR-by-design
- Lawful basis per contact/communication
- Consent ledger (channel + purpose + timestamp)
- DSAR tooling: export, rectify, erase
- Pseudonymization for deleted PII while preserving finance integrity
- Data retention policies per object class

## Access model
- Roles: super_admin, org_admin, fundraising_manager, program_manager, volunteer_coordinator, data_entry, finance_viewer, volunteer, beneficiary, report_only
- Privileged operations require step-up auth + reason field
- Break-glass access logged and time-limited

## Security operations
- SAST/DAST in CI
- Dependency scanning + SBOM
- Quarterly access review
- Incident runbooks (P1/P2)
- Backup policy: daily full + PITR, restore drills monthly
