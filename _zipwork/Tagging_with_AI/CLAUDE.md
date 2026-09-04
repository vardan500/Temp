# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An Azure resource tagging strategy and governance package for a credit union. There is no application code, build system, or test suite — the deliverables are Markdown strategy docs (`docs/`), Azure Policy JSON (`policies/`), and KQL queries + a CSV mapping for AI cost chargeback (`ai-chargeback/`).

Everything is organized around **three pillars** — **FinOps**, **SecOps**, and **Operational Excellence** — and that structure is mirrored across `docs/03–05`, `policies/definitions/{finops,secops,operational-excellence}/`, the three initiatives, and the three assignments. When adding or renaming anything, keep all four layers (docs, definitions, initiatives, assignments) and the structure listing in `README.md` in sync.

`plan.md` is the original project plan plus revision notes; its early proposal (PascalCase tags like `CostCenter`, a single initiative) is **superseded** — `docs/` and `policies/` are authoritative.

## Validation

There is no linter. Policy JSON is validated by parsing:

```powershell
Get-ChildItem policies -Recurse -Filter *.json | ForEach-Object { Get-Content $_.FullName -Raw | ConvertFrom-Json | Out-Null; $_.Name }
```

## Deployment

Full Azure CLI sequence is in `README.md`. Order matters: definitions → initiatives (`az policy set-definition create`) → assignments. Phase promotion (Audit → Deny) is a parameter change on the assignment, never a new policy file.

## Cross-file coupling (easy to break)

- Initiatives reference definitions by **file basename**: `[concat(subscription().id, '/providers/Microsoft.Authorization/policyDefinitions/<basename>')]`. Renaming a definition file breaks its initiative; deployment uses `basename "$f" .json` as the policy name.
- Assignments reference initiatives the same way (`policySetDefinitions/initiative-<pillar>`). The subscription GUID in assignment files is a `00000000-…` placeholder.
- `ai-chargeback/consumer-map.csv` (keyed by `app`) joins telemetry to cost dimensions; any `app` in telemetry but not in the CSV lands in the unallocated bucket (`queries/unallocated-usage.kql`).

## Conventions

- **Tag names are lowercase and may contain spaces** (`cost center`, `data steward`, `data classification`) — this follows the organization's existing convention. Azure tag names/values are case-sensitive; do not "fix" the casing or the spaces.
- The org's existing default tags (`app`, `cost center`, `department`, `env`, `owner`, `project`, `tier`) and functional tags (`auto`, `expireOn`) are **reused as-is**. Do not author policies that re-enforce their presence or cascade — the existing default-tag policy handles that. This repo only governs **net-new tags** and **value integrity** (formats, allowed values) on existing ones.
- Every policy definition parameterizes its effect (`Audit`/`Deny`/`Disabled`, default `Audit`) so Audit → Deny promotion is a parameter flip. Initiatives expose a shared `effect` parameter, with two exceptions: **SecOps** has `auditDenyEffect` + `inheritEffect` (it contains the tag-inherit Modify cascade — its assignment is the only one needing a managed identity), and **FinOps** has `effect` + `diagnosticsEffect` (the AI diagnostics check is `AuditIfNotExists`/`Disabled` and can't share the tag-policy effect values).
- Definition metadata carries `"category": "Tags"` and a `"pillar"` field; initiatives group definitions via `policyDefinitionGroups`.
- Prefer Microsoft built-in policies where an equivalent exists; custom definitions exist only for value restriction and classification-driven logic.

## AI chargeback model

Every AI resource is a **shared multi-tenant paygo endpoint** on a single shared platform (see `docs/08-ai-chargeback.md`), and **no AI-specific tags exist** — AI resources carry only the default tags, which identify the platform team and absorb unallocated usage; they never attribute consumer cost. All AI consumer cost flows through one usage-split path keyed on `app`: each application has exactly one calling identity **named exactly its `app` tag value** (the load-bearing naming invariant — enforced at consumer onboarding, not by Azure Policy), so telemetry → KQL in `ai-chargeback/queries/` → `consumer-map.csv` (keyed by `app`; `workload` column = use-case rollup) → split **per model** (token prices differ; each model's actual cost comes from its Cost Management meter lines), reconciled back to Cost Management. Retired tags from earlier revisions (see plan.md revision notes for reintroduction conditions): `shared service`, `ai workload`, `ai billing model`, `app id`. The KQL column names (`properties_app_id_s` etc.) are placeholders that must be adjusted to the actual APIM/diagnostic log schema.
