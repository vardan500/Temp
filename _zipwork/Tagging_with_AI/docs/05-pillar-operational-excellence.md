# Pillar: Operational Excellence

Resilience, disaster recovery/backup, and operational routing. Ensures every workload —
especially production — declares its criticality, recovery posture, and operating owner.

## 1. Tags (see `02-tag-taxonomy.md §3`)
`criticality`, `backup`, `dr tier`, `managed by`, `maintenance window` (net-new) plus
existing `env`, `tier`, `auto`, `expireOn`.

## 2. Policies (`policies/definitions/operational-excellence/`)
| Policy | Effect (default) | What it does |
|--------|------------------|--------------|
| `allowed-values-criticality.json` | Audit | Restricts `criticality` to `mission-critical`/`high`/`medium`/`low`. |
| `require-criticality-on-prod.json` | Audit | Requires `criticality` when `env = prod`. |
| `allowed-values-backup.json` | Audit | Restricts `backup` to `daily`/`weekly`/`monthly`/`none`. |
| `allowed-values-dr-tier.json` | Audit | Restricts `dr tier` to `tier1`/`tier2`/`tier3`/`none`. |
| `require-tag-managed-by.json` | Audit | Requires `managed by` (operating team/MSP). |

Bundled in **`initiative-operational-excellence.json`**, assigned via
**`assignment-operational-excellence-audit.json`** (subscription scope, Audit).

> `maintenance window` is free-text (no allowed-values policy); recommended but not
> strictly enforced. Add a `require`-style policy later if desired.

## 3. Why These Tags
| Tag | Operational value |
|-----|-------------------|
| `criticality` | Prioritize DR/BCP, incident severity, and change risk. Distinct from architectural `tier`. |
| `dr tier` | Encodes RTO/RPO expectations for recovery runbooks. |
| `backup` | Declares backup-frequency intent; reconcile against actual Backup policy. |
| `managed by` | Routes operational tickets/alerts to the operating team (vs accountable `owner`). |
| `maintenance window` | Schedules patching/maintenance without surprising owners. |
| `auto` / `expireOn` | Existing: off-hours shutdown and auto-cleanup. |

## 4. Operational Signals
| Signal | Tags | Action |
|--------|------|--------|
| Prod without recovery posture | `env=prod` missing `criticality`/`dr tier` | Backfill; block in Deny phase. |
| Backup intent vs reality mismatch | `backup` vs Azure Backup state | Remediate protection. |
| Unrouted resource | missing `managed by` | Assign operating team. |
| Stale/temp resources | `expireOn` past | Auto-delete. |

## 5. Alignment with WAF Operational Excellence
- **Observability & ownership:** `managed by`, `owner` establish clear operational lines.
- **Safe deployment/maintenance:** `maintenance window`, `env`, `tier`.
- **Resilience:** `criticality`, `dr tier`, `backup` feed DR planning and testing.

## 6. Phasing
Audit → Deny for the critical subset (e.g., `require-criticality-on-prod`) once prod
coverage is high. See `06-governance-policy-plan.md`.
