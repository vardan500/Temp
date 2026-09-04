# Tag Taxonomy (Tag Dictionary) — Categorized by Pillar

Tag names use the existing convention: **lowercase, spaces allowed**. Names and values are
case-sensitive. Legend: **[E]** = existing (already enforced/cascaded, unchanged),
**[N]** = net-new (added by this strategy). "Governed" = enforced by an Azure Policy in
this repo.

## 0. Cross-Cutting (Core) Tags
Used by all three pillars.

| Tag | Src | Purpose | Allowed values / format |
|-----|-----|---------|-------------------------|
| `owner` | [E] | Accountable stakeholder/team | email / DL (FinOps validates format) |
| `env` | [E] | Environment | `dev` \| `qa` \| `nonprod` \| `staging` \| `prod` |
| `app` | [E] | Application/workload | free text (kebab preferred) |

---

## 1. FinOps Pillar Tags
Cost allocation, chargeback/showback, budget accountability.

| Tag | Src | Governed | Purpose | Allowed values / format |
|-----|-----|----------|---------|-------------------------|
| `cost center` | [E] | ✅ value format | Chargeback GL code | `CC-####` (e.g., `CC-4100`) |
| `department` | [E] | — (cascaded) | Cost rollup / org unit | e.g., `DigitalBanking`, `Lending` |
| `project` | [E] | — (cascaded) | Project/initiative | e.g., `DAO`, `AKC`, `Consortium` |
| `app` | [E] | — | App cost grouping | free text |
| `createdOn` | [N] | — (optional) | Provisioning date for aging/amortization | `YYYY-MM-DD` |

### 1a. AI Chargeback Tags (net-new)
Extend cost allocation to AI resources (Azure OpenAI / AI Foundry + Azure AI Services).
All AI resources are **shared multi-tenant paygo endpoints** on a single shared platform,
so no AI cost is attributed by resource tags alone — and **no AI-specific tags exist**.
AI resources carry only the default tags, which identify the platform team, not a
consumer.

Every chargeback dimension lives in telemetry and `ai-chargeback/consumer-map.csv`
(keyed by `app`): each application has exactly one calling identity, **named exactly its
`app` tag value**, so telemetry yields `app` directly; use case = the map's `workload`
column; billing is uniformly paygo, split per model. (Retired tags from earlier
revisions: `shared service`, `ai workload`, `ai billing model`, `app id`.) See
`08-ai-chargeback.md`.

Primary chargeback dimensions remain `cost center` and `department`; `project` is
secondary; `app` is the join key from telemetry to those dimensions.

**Policies:** `match-format-cost-center`, `match-format-owner`, `audit-diagnostics-on-ai`
(see `03-pillar-finops.md` and `08-ai-chargeback.md`). The identity naming invariant is
an onboarding rule, not an Azure Policy.

---

## 2. SecOps Pillar Tags
Data classification, regulatory scope, and classification-driven controls.

| Tag | Src | Governed | Purpose | Allowed values / format |
|-----|-----|----------|---------|-------------------------|
| `data classification` | [N] | ✅ required + values + cascade | Data sensitivity; drives controls | `public` \| `internal` \| `confidential` \| `restricted` |
| `compliance` | [N] | ✅ required when `restricted` | Regulatory scope flag | `GLBA`;`PCI-DSS`;`SOX`;`SOC2`;`None` (semicolon) |
| `data steward` | [N] | — (recommended) | Security/data contact (distinct from owner) | email / DL |

`restricted` denotes member NPI / cardholder data (GLBA/PCI).

**Policies:** `require-tag-data-classification`, `allowed-values-data-classification`,
`inherit-tag-from-rg-data-classification`, `require-compliance-when-restricted`,
`deny-public-network-when-restricted`,
`require-secure-transfer-when-restricted` (see `04-pillar-secops.md`).

---

## 3. Operational Excellence Pillar Tags
Resilience, DR/backup, and operational routing.

| Tag | Src | Governed | Purpose | Allowed values / format |
|-----|-----|----------|---------|-------------------------|
| `criticality` | [N] | ✅ values + required on prod | Business/DR criticality | `mission-critical` \| `high` \| `medium` \| `low` |
| `backup` | [N] | ✅ value restriction | Backup frequency intent | `daily` \| `weekly` \| `monthly` \| `none` |
| `dr tier` | [N] | ✅ value restriction | DR RTO/RPO tier | `tier1` \| `tier2` \| `tier3` \| `none` |
| `managed by` | [N] | ✅ required (audit) | Operating team/MSP (vs accountable owner) | team/alias |
| `maintenance window` | [N] | — (recommended) | Patch/change scheduling | free text (e.g., `Sun 02:00-04:00 CT`) |
| `tier` | [E] | — | Architectural tier | `app` \| `web` \| `data` \| `infra` |
| `auto` | [E] | — | VM auto start/stop off-hours | per existing guidance |
| `expireOn` | [E] | — | Auto-delete date (5 AM PST) | `YYYY-MM-DD` |

**Policies:** `allowed-values-criticality`, `require-criticality-on-prod`,
`allowed-values-backup`, `allowed-values-dr-tier`, `require-tag-managed-by`
(see `05-pillar-operational-excellence.md`).

---

## 4. Reconciliation — Earlier Proposal vs Final
| Earlier proposal | Resolution |
|------------------|------------|
| `Application` / `CostCenter` / `BusinessUnit` / `Environment` / `Owner` / `Project` | → existing `app` / `cost center` / `department` / `env` / `owner` / `project` |
| `EndDate` / auto-shutdown | → existing `expireOn` / `auto` |
| `DataClassification` | → **[N]** `data classification` (+ cascade + controls) |
| `Criticality` | → **[N]** `criticality` (+ required on prod) |
| `Compliance` | → **[N]** `compliance` (required when restricted) |
| — | → **[N]** added: `data steward`, `createdOn`, `backup`, `dr tier`, `managed by`, `maintenance window` |

## 5. Full Tag Block (example)
```json
{
  "app": "online-banking",
  "cost center": "CC-4100",
  "department": "DigitalBanking",
  "env": "prod",
  "owner": "digitalbanking-ops@creditunion.org",
  "project": "DAO",
  "tier": "web",
  "createdOn": "2026-01-15",
  "data classification": "restricted",
  "compliance": "PCI-DSS;GLBA",
  "data steward": "security-office@creditunion.org",
  "criticality": "mission-critical",
  "backup": "daily",
  "dr tier": "tier1",
  "managed by": "platform-team",
  "maintenance window": "Sun 02:00-04:00 CT",
  "expireOn": "2027-01-15"
}
```
