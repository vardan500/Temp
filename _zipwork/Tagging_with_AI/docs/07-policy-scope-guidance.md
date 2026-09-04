# Policy Scope Guidance — Management Group vs Subscription (by Pillar)

Which pillar policies belong at the **management group (MG)** level vs the **subscription**
level. Complements `06-governance-policy-plan.md` (which assigns all three initiatives at
subscription scope for the initial pilot).

## 1. Guiding Principle
> Assign broad, stable, value-agnostic guardrails **as high as possible** (MG) so every
> current and future subscription inherits them. Keep **identity-bound**, **value-bearing**,
> and **remediation** work at the **subscription** level.

Azure Policy inherits downward, so high assignments reduce drift.

## 2. Suggested Management Group Topology
```
Root MG
└── CreditUnion (Corp) MG          <-- enterprise baseline (all 3 initiatives, Audit)
    ├── Platform MG
    ├── Landing Zones MG
    │   ├── Prod MG                <-- Deny enforcement (SecOps + critical Ops) here
    │   └── NonProd MG             <-- Audit here
    └── Sandbox MG
```

## 3. Better at **Management Group** Level
Universal, value-agnostic, stable — define once and inherit.

| Item | Pillar | Why MG |
|------|--------|--------|
| `require-tag-data-classification` (incl. RGs) | SecOps | Security is enterprise-wide. |
| `allowed-values-data-classification` | SecOps | Global classification scheme. |
| `deny-public-network-when-restricted` / `require-secure-transfer-when-restricted` | SecOps | Enterprise security controls; **Deny on Prod MG**. |
| `require-compliance-when-restricted` | SecOps | Regulatory scope is org-wide. |
| `match-format-cost-center` / `match-format-owner` | FinOps | Global value formats, rarely change. |
| `allowed-values-criticality` / `-backup` / `-dr tier` | Ops | Global enumerations. |
| `require-criticality-on-prod` | Ops | Best expressed by **Deny on Prod MG**. |
| All three **initiative definitions** | All | Define at Corp MG so all subs share one source of truth. |
| Naming/casing conventions | All | Uniform across the enterprise. |

**Key insight:** Environment-differentiated enforcement is best expressed by **MG
placement** (Prod MG = Deny, NonProd MG = Audit) rather than per-subscription toggling.

## 4. Better at **Subscription** Level
Context-specific, value-bearing, or identity-bound.

| Item | Pillar | Why subscription |
|------|--------|------------------|
| `inherit-tag-from-rg-data-classification` (Modify) | SecOps | Needs **managed identity + Tag Contributor** and remediation against concrete resources. |
| SecOps assignment **managed identity** | SecOps | Scoped rights per subscription are cleaner. |
| Remediation tasks | SecOps | Run against real resources in a subscription. |
| Existing default-tag cascade (Modify) | Core | Already implemented per subscription. |
| `cost center` / `department` default values | FinOps | Often map closely to a subscription. |
| `managed by` defaults | Ops | Aligns to subscription/team ownership. |
| Exemptions | All | Resource/subscription-specific with expiry. |
| Budgets & cost alerts | FinOps | Naturally scoped to subscription/department. |
| Pilot / early rollout | All | Start Phase 1 Audit on one subscription. |

## 5. Recommended Placement Summary
| Policy / concern | Scope | Effect |
|------------------|-------|--------|
| SecOps require/allowed/controls | **Corp MG** | Audit → (Prod MG) Deny |
| SecOps `inherit-data-classification` (Modify) + identity | **Subscription** | Modify + MI |
| FinOps value-format policies | **Corp MG** | Audit → Deny |
| Ops allowed-values | **Corp MG** | Audit |
| `require-criticality-on-prod` | **Prod MG** | Deny |
| Deny enforcement (SecOps critical) | **Prod MG** | Deny |
| Relaxed enforcement | **NonProd / Sandbox MG** | Audit |
| Remediation, exemptions, budgets, defaults | **Subscription** | n/a |

## 6. Migration Path from the Pilot
1. Validate the three initiatives + audit results on the pilot subscription.
2. Move initiative **definitions** to the Corp MG (or define there).
3. Re-assign FinOps, SecOps (audit/deny parts), and Ops initiatives at the **Corp MG**
   (Audit); remove redundant per-subscription audit assignments.
4. Add **Deny** assignments (SecOps + `require-criticality-on-prod`) at the **Prod MG**.
5. Keep the SecOps **Modify cascade** + identity and all remediation at **subscription**
   scope.

## 7. Trade-offs & Cautions
- Fewer, higher assignments = less drift but broader blast radius — always pilot in Audit.
- Deny at MG hits all child subs immediately; stage via Prod MG only after coverage ≥95%.
- Modify at MG needs the identity to have rights across all child subs — usually cleaner
  per-subscription.
- Consolidating into per-pillar **initiatives** keeps assignment counts within limits.
