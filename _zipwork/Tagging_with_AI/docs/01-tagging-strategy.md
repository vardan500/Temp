# Azure Resource Tagging Strategy — Overview

## 1. Purpose
This strategy defines Azure resource tagging for **[Credit Union Name]**, organized around
**three pillars**:

1. **FinOps** — cost allocation, chargeback/showback, budget accountability.
2. **SecOps** — data classification, regulatory scope, and classification-driven controls.
3. **Operational Excellence** — resilience, DR/backup, and operational routing.

It aligns with the **Microsoft Cloud Adoption Framework (CAF)** naming/tagging guidance,
the **FinOps Foundation** framework, and the **Azure Well-Architected Framework** pillars.

## 2. Relationship to the Existing Standard
The organization already **enforces and cascades** a set of default tags (RG → resources)
and provides functional tags. This strategy **reuses those as-is** and adds net-new tags
and, crucially, **policies that connect tags to enforcement**.

| Existing default tags | Existing functional tags |
|-----------------------|--------------------------|
| `app`, `cost center`, `department`, `env`, `owner`, `project`, `tier` | `auto`, `expireOn` |

Net-new tags added by this strategy (by pillar): see `02-tag-taxonomy.md`.

## 3. Three-Pillar Model
| Pillar | Primary tags | Governs | Doc |
|--------|--------------|---------|-----|
| **FinOps** | `cost center`, `department`, `app`, `project`, `createdOn` | Cost-tag value integrity & allocation | `03-pillar-finops.md` |
| **SecOps** | `data classification`, `compliance`, `data steward` | Classification, regulatory scope, security controls | `04-pillar-secops.md` |
| **Operational Excellence** | `criticality`, `backup`, `dr tier`, `managed by`, `maintenance window`, `env`, `tier` | Resilience, DR, ops routing | `05-pillar-operational-excellence.md` |

`owner` and `env` are **cross-cutting** — used by all three pillars.

## 4. Naming Convention (follows existing convention)
- **Tag names:** lowercase, spaces allowed (e.g., `cost center`, `data classification`).
  Do **not** introduce a competing PascalCase scheme — names are case-sensitive and mixing
  fragments cost/compliance reports.
- **Tag values:** lowercase for enumerations (`prod`, `restricted`, `mission-critical`).
- **Allowed characters:** letters, numbers, spaces, `-`, `_`, `.`. Avoid `<>%&\?/`.

### Azure tag constraints
- Max **50 tags** per resource/RG; name ≤ 512 chars, value ≤ 256 chars.
- Some resource types don't support tags or surface them in Cost Management.
- Inheritance is via policy — default tags already cascade; `data classification` gains a
  new cascade policy in the SecOps pillar.

## 5. Governance Model (all pillars)
Enforcement is **phased and per-pillar**: each pillar ships an **initiative** and an
**Audit** subscription assignment. Promotion Audit → Deny is a **parameter change**.
See `06-governance-policy-plan.md` for the rollout and `07-policy-scope-guidance.md` for
management-group vs subscription placement.

## 6. Roles & Responsibilities
| Role | Owns |
|------|------|
| **Cloud/Platform Team** | Taxonomy, initiatives, assignments, FinOps & Ops policies. |
| **FinOps / Finance** | `cost center` mapping, chargeback, budgets. |
| **Security / Compliance (CISO)** | `data classification`, `compliance`, security controls. |
| **App / Resource Owners** | Apply/maintain tags on their workloads. |
| **Governance/Audit** | Compliance dashboards, exemption review. |

## 7. Document Map
- `02-tag-taxonomy.md` — tag dictionary (categorized by pillar).
- `03-pillar-finops.md` · `04-pillar-secops.md` · `05-pillar-operational-excellence.md`
- `06-governance-policy-plan.md` — phased enforcement, all policies by pillar.
- `07-policy-scope-guidance.md` — management group vs subscription placement.
