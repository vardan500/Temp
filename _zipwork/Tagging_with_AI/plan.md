# Azure Resource Tagging Strategy & Governance — Credit Union

## Revision Note 5 (app id retired — `app` is the chargeback key; zero AI-specific tags)
Every application has exactly **one calling identity, named exactly its `app` tag
value**, so `app id` was a pure alias of the existing default `app` tag and is
**retired** (`require-tag-app-id-on-shared.json` deleted). `consumer-map.csv` is now
keyed by `app` (columns: app, cost center, department, project, workload) and all KQL
splits by `app`. AI resources now carry **no AI-specific tags at all** — only the
default tags (identifying the platform team); the FinOps initiative is down to the two
format policies plus `audit-diagnostics-on-ai`. The load-bearing rule is the **naming
invariant**, enforced at consumer onboarding, not by Azure Policy (policy cannot see
APIM subscription / AAD app names): creating an identity = naming it the `app` value +
adding the consumer-map row in the same PR, before endpoint access. Nonzero unallocated
bucket ⇒ usually a naming violation. If an application ever needs a second identity
(second client, credential split, per-env), reintroduce `app id` as a distinct key —
see Revision Note 4's history for the pattern.

## Revision Note 4 (paygo-only, single shared platform, per-model split)
All AI billing is **pay-as-you-go** and all use cases run through a **single shared
platform** (one Foundry, shared Content Safety, etc.), so two more constant-value tags
are retired: `ai billing model` (always `paygo`; `allowed-values-ai-billing-model.json`
deleted) and `ai workload` (one shared stack serves every use case, so a per-use-case
resource tag cannot vary; `require-tag-ai-workload.json` deleted). The use-case
dimension moved to a `workload` column in `ai-chargeback/consumer-map.csv`; use-case
showback = split output grouped by `workload`. `app id` is now the only net-new AI tag.
The split formula is now **per model then summed** (model token prices differ; each
model's cost is read from its Cost Management meter lines), with a documented
price-weighted fallback and a prompt-vs-completion weighting note;
`monthly-split-summary.kql` implements the per-model split. If PTU/commitment is adopted
later, reintroduce `ai billing model` and an amortization method.

## Revision Note 3 (all AI resources are shared)
The AI chargeback model now assumes **every AI resource is a shared multi-tenant
endpoint** — there are no dedicated per-team AI resources. Consequences: the former
two-layer model collapses to a single usage-split path (telemetry → `app id` →
proportional split); resource-level cost tags on AI resources identify the platform team
and absorb unallocated usage; the `shared service` tag is **retired**;
`require-tag-app-id-on-shared` now applies unconditionally to AI resource types
(filename kept for initiative compatibility); new `audit-diagnostics-on-ai` policy
(AuditIfNotExists) verifies telemetry coverage since the split is the only attribution
path; the dedicated-deployment fallback for consumers without APIM/AAD identity is
removed — consumer identity is a prerequisite for endpoint access.

## Revision Note 2 (reorganized by three pillars)
Everything is now categorized by **FinOps**, **SecOps**, and **Operational Excellence**,
and includes all evaluation recommendations. Docs: `01` overview, `02` taxonomy (by
pillar), `03` finops, `04` secops, `05` operational-excellence, `06` governance, `07`
scope. Policies split into `policies/definitions/{finops,secops,operational-excellence}/`
(13 definitions), three per-pillar initiatives, and three per-pillar audit assignments.
Net-new tags added: `data classification`, `compliance`, `data steward`,
`criticality`, `backup`, `dr tier`, `managed by`, `maintenance window`, `createdOn`.
New enforcement: cost-center/owner format validation, data-classification cascade + require
on RGs, compliance-when-restricted, deny-public-network/require-HTTPS for restricted data,
require-criticality-on-prod. Existing default/functional tags reused unchanged.

## Revision Note 1 (reconciled with existing standard)
The organization already enforces and cascades **default tags** (`app`, `cost center`,
`department`, `env`, `owner`, `project`, `tier`) and **functional tags** (`auto`,
`expireOn`). This strategy was refactored to **reuse those as-is** and govern only the
**net-new tags**: `data classification` (required), `criticality` (optional), and
`compliance` (optional, un-enforced). Naming follows the existing lowercase convention.
Policy set reduced to 3 definitions + 1 supplemental initiative + 1 subscription audit
assignment (no Modify/identity needed — inheritance handled by the existing cascade).

## Problem Statement
A credit union needs a standardized Azure resource tagging strategy grounded in
FinOps principles and Microsoft Cloud Adoption Framework (CAF) recommended
practices, plus Azure Policy definitions to govern tag compliance. The strategy
must support cost allocation/chargeback, ownership accountability, and regulatory
compliance relevant to financial institutions (GLBA/Safeguards Rule, PCI-DSS,
NCUA guidance, SOC 2), while remaining practical to operate.

## Approach
Deliver a set of Markdown strategy documents plus Azure Policy definitions in
JSON. Enforcement follows a phased model: **audit-only first**, then tighten to
`deny`/`modify` once compliance baselines are met. Policies are authored to be
**assigned at the subscription scope**. The tag taxonomy covers the full set:
FinOps cost allocation, organizational/ownership, security/compliance, and
operations/data-classification.

## Deliverables

### Documentation (Markdown)
- `docs/01-tagging-strategy.md` — Executive summary, goals, FinOps + CAF
  alignment, roles/responsibilities, naming conventions (case, allowed
  characters, format rules).
- `docs/02-tag-taxonomy.md` — Full tag dictionary: each tag with name,
  purpose, required/optional, allowed values, examples, owner.
- `docs/03-governance-policy-plan.md` — Phased rollout (Phase 1 audit →
  Phase 2 remediate/inherit → Phase 3 deny), policy-to-tag mapping,
  assignment scope, exemption process, compliance reporting.
- `docs/04-finops-cost-allocation.md` — How tags map to cost views,
  chargeback/showback model, budgets/alerts, shared-cost handling.
- `docs/05-compliance-mapping.md` — Tag-to-regulation mapping matrix
  (GLBA, PCI-DSS, NCUA, SOC 2) and data-classification handling.
- `docs/06-policy-scope-guidance.md` — Which policies/tags are better
  candidates for management group vs subscription scope, with a
  recommended MG topology and migration path.
- `README.md` — Repo overview, structure, how to deploy policies.

### Azure Policy Definitions (JSON) — `policies/definitions/`
Custom policy definition JSON files (audit effect by default, parameterized
`effect` so the same file supports Audit → Deny promotion):
- `require-tag-Environment.json`
- `require-tag-CostCenter.json`
- `require-tag-Owner.json`
- `require-tag-Application.json`
- `require-tag-DataClassification.json`
- `require-tag-BusinessUnit.json`
- `allowed-values-Environment.json` — restrict to approved value list.
- `allowed-values-DataClassification.json` — restrict to approved list.
- `inherit-tag-from-rg-CostCenter.json` — modify/append inherit from RG.
- `inherit-tag-from-rg-Environment.json` — modify/append inherit from RG.
- (Note built-in alternatives where Microsoft already ships one.)

### Azure Policy Initiative & Assignments — `policies/initiatives/`, `policies/assignments/`
- `initiative-tagging-baseline.json` — policy set grouping all tag policies
  with a shared `effect` parameter for phased enforcement.
- `assignment-subscription-audit.json` — Phase 1 subscription assignment
  (effect = Audit / Manual), with parameters and identity for remediation.
- Example remediation task guidance for modify/append policies.

## Proposed Tag Taxonomy (summary — detailed in docs)
Required (governed):
- `Environment` — Prod | NonProd | Dev | Test | UAT | DR | Sandbox
- `CostCenter` — GL/cost-center code for chargeback
- `Owner` — accountable individual/team email (DL preferred)
- `Application` — application/workload name
- `BusinessUnit` — e.g., Lending, Deposits, Cards, DigitalBanking, IT, Corporate
- `DataClassification` — Public | Internal | Confidential | Restricted(PII/NPI)

Recommended (audit/optional):
- `Compliance` — GLBA | PCI-DSS | SOX | SOC2 | None (multi-value convention)
- `Criticality` — Mission-Critical | High | Medium | Low
- `ManagedBy` — team/MSP responsible for operations
- `Project` — project or initiative code
- `StartDate` / `EndDate` — lifecycle (for temp/sandbox cleanup)
- `MaintenanceWindow` — ops scheduling
- `Repository` / `DeployedBy` — IaC provenance

## Phased Governance Model
- **Phase 1 (Audit):** Assign initiative with effect=Audit at subscription
  scope. Build compliance dashboards, remediate manually. No blocking.
- **Phase 2 (Inherit/Remediate):** Enable `modify`/`append` inherit-from-RG
  policies with remediation tasks + managed identity to backfill tags.
- **Phase 3 (Deny):** Flip required-tag and allowed-values policies to
  effect=Deny for the critical subset (Environment, CostCenter, Owner,
  DataClassification); keep others at Audit.

## Notes & Considerations
- Prefer Microsoft **built-in** policies where equivalent exists (e.g.,
  "Require a tag on resources", "Inherit a tag from the resource group");
  custom JSON provided where value-restriction or bundling is needed.
- Tag values are case-sensitive in Azure — enforce a casing convention.
- Some resource types don't support tags / have limits (50 tags, length
  limits) — document exceptions.
- Data-classification `Restricted` implies NPI/PII (GLBA) handling controls —
  link to security baseline, not just a tag.
- Exemption process via Azure Policy exemptions with expiry + justification.
- Cost allocation depends on consistent `CostCenter` + `BusinessUnit`;
  shared/untagged costs handled via proportional split rules in FinOps doc.

## Todos
Tracked in SQL (see todos table).
