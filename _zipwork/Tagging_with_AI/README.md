# Azure Resource Tagging Strategy & Governance — Credit Union

A tagging strategy organized around **three pillars** — **FinOps**, **SecOps**, and
**Operational Excellence** — aligned with Microsoft CAF, the FinOps Foundation framework,
and the Azure Well-Architected Framework. It **reuses the organization's existing default
and functional tags** and adds net-new tags plus Azure Policy that **connects tags to
enforcement**.

## Existing tags reused (unchanged)
Default (enforced & cascaded RG→resources): `app` · `cost center` · `department` · `env` ·
`owner` · `project` · `tier`  Functional: `auto` · `expireOn`

## Net-new tags by pillar
- **FinOps:** `createdOn` (+ value integrity on `cost center`, `owner`). AI chargeback needs **no AI-specific tags**: all AI resources are shared paygo endpoints on a single platform, attributed by a per-model usage split keyed on `app` (each application's single calling identity is named its `app` tag value); use case comes from the consumer map's `workload` column.
- **SecOps:** `data classification` · `compliance` · `data steward`
- **Ops Excellence:** `criticality` · `backup` · `dr tier` · `managed by` · `maintenance window`

See `docs/02-tag-taxonomy.md` for the full dictionary and reconciliation.

## Repository Structure
```
.
├── plan.md
├── docs/
│   ├── 01-tagging-strategy.md              # Overview + three-pillar model
│   ├── 02-tag-taxonomy.md                  # Tag dictionary categorized by pillar
│   ├── 03-pillar-finops.md
│   ├── 04-pillar-secops.md
│   ├── 05-pillar-operational-excellence.md
│   ├── 06-governance-policy-plan.md        # Phased enforcement, all policies by pillar
│   ├── 07-policy-scope-guidance.md         # MG vs subscription placement by pillar
│   └── 08-ai-chargeback.md                 # AI resource chargeback (shared-endpoint split)
├── ai-chargeback/                          # Usage-split telemetry & allocation assets
│   ├── consumer-map.csv                    # app -> cost dims + workload (use-case rollup)
│   └── queries/                            # KQL: per-consumer usage, unallocated, split
└── policies/
    ├── definitions/
    │   ├── finops/
    │   │   ├── match-format-cost-center.json
    │   │   ├── match-format-owner.json
    │   │   └── audit-diagnostics-on-ai.json
    │   ├── secops/
    │   │   ├── require-tag-data-classification.json
    │   │   ├── allowed-values-data-classification.json
    │   │   ├── inherit-tag-from-rg-data-classification.json
    │   │   ├── require-compliance-when-restricted.json
    │   │   ├── deny-public-network-when-restricted.json
    │   │   └── require-secure-transfer-when-restricted.json
    │   └── operational-excellence/
    │       ├── allowed-values-criticality.json
    │       ├── require-criticality-on-prod.json
    │       ├── allowed-values-backup.json
    │       ├── allowed-values-dr-tier.json
    │       └── require-tag-managed-by.json
    ├── initiatives/
    │   ├── initiative-finops.json
    │   ├── initiative-secops.json
    │   └── initiative-operational-excellence.json
    └── assignments/
        ├── assignment-finops-audit.json
        ├── assignment-secops-audit.json
        └── assignment-operational-excellence-audit.json
```

## Phased Enforcement (per pillar)
| Phase | FinOps | SecOps | Ops Excellence |
|-------|--------|--------|----------------|
| 1 Audit | `effect=Audit` | `auditDenyEffect=Audit`, `inheritEffect=Disabled` | `effect=Audit` |
| 2 Inherit | — | `inheritEffect=Modify` + remediation | — |
| 3 Deny | `effect=Deny` | `auditDenyEffect=Deny` (Prod) | `effect=Deny` (critical subset) |

Promotion is a **parameter change**. Only the **SecOps** assignment needs a managed
identity (it contains the Modify cascade).

## Deployment (Azure CLI)

> Replace `<subId>`; run `az account set --subscription <subId>` first. `jq` used for portability.

### 1. Create all policy definitions
```bash
for f in policies/definitions/**/*.json; do
  name=$(basename "$f" .json)
  mode=$(jq -r '.properties.mode' "$f")
  az policy definition create \
    --name "$name" \
    --rules "$(jq -c .properties.policyRule "$f")" \
    --params "$(jq -c .properties.parameters "$f")" \
    --mode "$mode" \
    --display-name "$(jq -r .properties.displayName "$f")" \
    --subscription <subId>
done
```

### 2. Create the three initiatives
```bash
for f in policies/initiatives/*.json; do
  name=$(basename "$f" .json)
  az policy set-definition create \
    --name "$name" \
    --definitions "$(jq -c .properties.policyDefinitions "$f")" \
    --params "$(jq -c .properties.parameters "$f")" \
    --definition-groups "$(jq -c .properties.policyDefinitionGroups "$f")" \
    --display-name "$(jq -r .properties.displayName "$f")" \
    --subscription <subId>
done
```

### 3. Assign each pillar (Phase 1 - Audit)
```bash
# FinOps
az policy assignment create --name tagging-finops-audit \
  --policy-set-definition initiative-finops \
  --scope "/subscriptions/<subId>" \
  --params '{ "effect": {"value":"Audit"}, "diagnosticsEffect": {"value":"AuditIfNotExists"} }'

# SecOps (system-assigned identity for the Modify cascade in later phases)
az policy assignment create --name tagging-secops-audit \
  --policy-set-definition initiative-secops \
  --scope "/subscriptions/<subId>" \
  --params '{ "auditDenyEffect": {"value":"Audit"}, "inheritEffect": {"value":"Disabled"} }' \
  --mi-system-assigned --location eastus

# Operational Excellence
az policy assignment create --name tagging-ops-audit \
  --policy-set-definition initiative-operational-excellence \
  --scope "/subscriptions/<subId>" \
  --params '{ "effect": {"value":"Audit"} }'
```

### 4. Phase 2 (SecOps only) — enable cascade + remediate
See `docs/06-governance-policy-plan.md §4` for the identity role assignment and remediation task.

### 5. Phase 3 — tighten to Deny
```bash
az policy assignment update --name tagging-secops-audit --scope "/subscriptions/<subId>" \
  --params '{ "auditDenyEffect": {"value":"Deny"}, "inheritEffect": {"value":"Modify"} }'
az policy assignment update --name tagging-finops-audit --scope "/subscriptions/<subId>" \
  --params '{ "effect": {"value":"Deny"} }'
```
> For Deny-in-Prod / Audit-in-NonProd, assign initiatives at management-group scope —
> see `docs/07-policy-scope-guidance.md`.

## Notes
- Tag names use the **existing lowercase convention**; names/values are case-sensitive.
- Prefer Microsoft **built-in** policies where equivalent; custom definitions add value
  restriction and classification-driven logic.
- Validate the SecOps compliance/control mapping (`docs/04-pillar-secops.md`) with your
  compliance, legal, and audit stakeholders.
```

All 20 policy JSON files are schema-valid (`ConvertFrom-Json`).
