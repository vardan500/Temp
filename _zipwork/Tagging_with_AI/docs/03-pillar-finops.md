# Pillar: FinOps

Cost allocation, chargeback/showback, and budget accountability. This pillar **reuses the
existing default cost tags** and adds **value-integrity policies** so allocation isn't
fragmented by typos.

## 1. Tags (see `02-tag-taxonomy.md §1`)
`cost center`, `department`, `project`, `app` (existing, cascaded) + `createdOn` (net-new,
optional). `owner` is cross-cutting and validated here.

## 2. Policies (`policies/definitions/finops/`)
| Policy | Effect (default) | What it does |
|--------|------------------|--------------|
| `match-format-cost-center.json` | Audit | Flags `cost center` values not matching `CC-####`, preventing chargeback fragmentation. |
| `match-format-owner.json` | Audit | Flags `owner` values that aren't email-like, so budget alerts route correctly. |
| `audit-diagnostics-on-ai.json` | AuditIfNotExists | Flags AI resources without diagnostic settings — without telemetry their cost cannot be attributed to any consumer. |

Bundled in **`initiative-finops.json`**, assigned via **`assignment-finops-audit.json`**
(subscription scope, Audit). Presence/cascade of default tags remains owned by the
existing default-tag policy.

## 3. Tag-to-Cost-View Mapping
| Cost question | Tag(s) |
|---------------|--------|
| Cost per department | `department` |
| Cost per application | `app` |
| Which GL / cost center | `cost center` |
| Prod vs non-prod | `env` |
| Which project | `project` |
| Who owns the spend | `owner` |

### 3a. AI resource chargeback
AI resources (Azure OpenAI / AI Foundry + Azure AI Services) are billed by
**tokens / PTUs / transactions**, and **every AI resource is a shared multi-tenant
endpoint** — none can be attributed by a single `cost center` tag. All AI consumer
attribution is usage-based; the cost tags on the resource identify the platform team
that owns the endpoint (and absorb any unallocated remainder):

| AI cost question | Tag(s) / mechanism |
|------------------|--------------------|
| Which platform team owns the endpoint | existing cost tags (`cost center`, `department`, `app`, `project`) |
| Per-application cost | `app` from telemetry (identity name = `app` tag value) → per-model usage split (all billing is paygo) |
| Which use case | `workload` column in `consumer-map.csv`, grouped from the split output |
| Which model / at what price | telemetry `model` dimension × per-model Cost Management meters |

Full method — telemetry, consumer→cost-center mapping, split formula, KQL, and
showback→chargeback phasing — is in **`08-ai-chargeback.md`**; assets live in
`ai-chargeback/`.

Group by these in **Cost Management > Cost analysis**; save shared views per department.

## 4. Chargeback vs Showback
- **Showback:** report consumption to each department/app (no billing movement).
- **Chargeback:** allocate actual cost to `cost center` once value integrity is high.
- Default tags cascade from RGs, so coverage is strong — the FinOps risk is **bad values**,
  which the format policies address.

## 5. Budgets & Alerts
- Budgets filtered by `department` and `env`; thresholds 50/80/100% to `owner`.
- Separate `env = prod` vs non-prod budgets to surface non-prod waste early.

## 6. Shared & Untagged Cost Handling
- **Proportional split** of shared costs to departments by tagged spend.
- **Platform cost center** for unallocatable costs.
- **Untagged bucket** tracked as a KPI; usually indicates an RG missing tags — fix at the RG.
- **Shared AI endpoints:** cost is split by usage telemetry per `app` (identity name =
  `app` tag value), not by resource tag; unmapped usage goes to the platform bucket (see
  `08-ai-chargeback.md §5–6`).

## 7. Optimization Signals
| Signal | Tags | Action |
|--------|------|--------|
| Idle non-prod | `env` != prod | `auto` off-hours shutdown; rightsize |
| Expired temp | `expireOn` past | auto-deleted 5 AM PST |
| Low criticality on premium SKU | `criticality = low` | downgrade tier |
| Orphaned spend | missing/invalid `owner` | assign owner |

## 8. Phasing
Audit → Deny once cost-tag value integrity ≥95%. See `06-governance-policy-plan.md`.
