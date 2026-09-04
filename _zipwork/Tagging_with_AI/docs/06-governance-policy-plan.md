# Governance & Policy Plan (All Pillars)

This plan governs the **net-new** tag policies across the three pillars. Existing
default-tag enforcement/cascade (`app`, `cost center`, `department`, `env`, `owner`,
`project`, `tier`) and functional tags (`auto`, `expireOn`) are unchanged. Each pillar
ships its own **initiative** and **Audit** subscription assignment.

## 1. Initiatives & Assignments
| Pillar | Initiative | Assignment | Effect param(s) |
|--------|-----------|------------|-----------------|
| FinOps | `initiative-finops.json` | `assignment-finops-audit.json` | `effect` |
| SecOps | `initiative-secops.json` | `assignment-secops-audit.json` | `auditDenyEffect`, `inheritEffect` |
| Ops Excellence | `initiative-operational-excellence.json` | `assignment-operational-excellence-audit.json` | `effect` |

## 2. All Policies by Pillar
### FinOps
| Policy | Phase 1 | Deny target |
|--------|---------|-------------|
| `match-format-cost-center` | Audit | Deny |
| `match-format-owner` | Audit | Audit |
| `audit-diagnostics-on-ai` (AI types) | AuditIfNotExists | AuditIfNotExists |

### SecOps
| Policy | Phase 1 | Deny target |
|--------|---------|-------------|
| `require-tag-data-classification` (incl. RGs) | Audit | **Deny** |
| `allowed-values-data-classification` | Audit | **Deny** |
| `inherit-tag-from-rg-data-classification` (Modify) | Disabled | Modify (Phase 2) |
| `require-compliance-when-restricted` | Audit | **Deny** |
| `deny-public-network-when-restricted` | Audit | **Deny** |
| `require-secure-transfer-when-restricted` | Audit | **Deny** |

### Operational Excellence
| Policy | Phase 1 | Deny target |
|--------|---------|-------------|
| `allowed-values-criticality` | Audit | Audit |
| `require-criticality-on-prod` | Audit | **Deny** |
| `allowed-values-backup` | Audit | Audit |
| `allowed-values-dr-tier` | Audit | Audit |
| `require-tag-managed-by` | Audit | Audit |

## 3. Phased Rollout
| Phase | FinOps | SecOps | Ops Excellence |
|-------|--------|--------|----------------|
| **1 — Audit** | `effect=Audit` | `auditDenyEffect=Audit`, `inheritEffect=Disabled` | `effect=Audit` |
| **2 — Inherit/Remediate** | (n/a) | `inheritEffect=Modify` + remediation task | (n/a) |
| **3 — Deny** | `effect=Deny` | `auditDenyEffect=Deny` (target Prod) | `effect=Deny` (critical subset) |

**Cadence:** 30–60 days per phase, gated on ≥95% compliance before promotion.

## 4. SecOps Managed Identity & Remediation (Phase 2)
Only the SecOps assignment needs an identity (it contains the Modify cascade):
```bash
# Grant the assignment identity rights to modify tags
PID=$(az policy assignment show --name tagging-secops-audit --scope "/subscriptions/<subId>" --query identity.principalId -o tsv)
az role assignment create --assignee-object-id "$PID" --assignee-principal-type ServicePrincipal \
  --role "Tag Contributor" --scope "/subscriptions/<subId>"

# Enable Modify cascade
az policy assignment update --name tagging-secops-audit --scope "/subscriptions/<subId>" \
  --params '{ "auditDenyEffect": {"value":"Audit"}, "inheritEffect": {"value":"Modify"} }'

# Backfill existing resources
az policy remediation create --name remediate-classification \
  --policy-assignment tagging-secops-audit \
  --definition-reference-id inherit-data-classification \
  --resource-discovery-mode ReEvaluateCompliance
```
FinOps and Ops Excellence assignments contain only Audit/Deny policies — **no identity
required**.

## 5. Built-in vs Custom
Prefer Microsoft built-ins where equivalent (e.g., *Require a tag on resources*). Custom definitions here provide value
restriction, conditional (classification-driven) logic, and bundling the built-ins don't.

## 6. Exemptions
Azure Policy exemptions only (`Waiver`/`Mitigated`) with justification + **expiry** +
approver. Reviewed quarterly. Never broadly exempt `restricted`/PCI resources.

## 7. Compliance Reporting
- Azure Policy compliance blade per assignment.
- Resource Graph for missing net-new tags:
```kusto
resources
| where isnull(tags['data classification'])
     or (tags['env'] == 'prod' and isnull(tags['criticality']))
| project name, type, resourceGroup, subscriptionId, tags
```

## 8. Change Control
Taxonomy/allowed-value changes go through the Cloud/Platform Team (SecOps values via
Security/Compliance) as version-controlled PRs. Existing default-tag policies are out of
scope for changes here.
