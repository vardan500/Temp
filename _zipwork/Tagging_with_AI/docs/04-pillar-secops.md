# Pillar: SecOps

Data classification, regulatory scope, and **classification-driven security controls**.
This is the highest-value pillar for a credit union: it not only tags sensitivity but
**wires tags to enforcement**.

## 1. Tags (see `02-tag-taxonomy.md §2`)
`data classification` (required), `compliance` (required when restricted),
`data steward` — all net-new.

## 2. Policies (`policies/definitions/secops/`)
| Policy | Effect (default) | What it does |
|--------|------------------|--------------|
| `require-tag-data-classification.json` | Audit | Requires the tag on resources **and resource groups** (mode All). |
| `allowed-values-data-classification.json` | Audit | Restricts to `public`/`internal`/`confidential`/`restricted`. |
| `inherit-tag-from-rg-data-classification.json` | Modify | Cascades the tag from the RG (like default tags). Needs managed identity + Tag Contributor. |
| `require-compliance-when-restricted.json` | Audit | Requires `compliance` when `data classification = restricted`. |
| `deny-public-network-when-restricted.json` | Audit→**Deny** | For `restricted` Storage/SQL/Key Vault/Cosmos, blocks public network access. |
| `require-secure-transfer-when-restricted.json` | Audit→**Deny** | For `restricted` storage, requires HTTPS-only. |

Bundled in **`initiative-secops.json`** (two effect params: `auditDenyEffect`,
`inheritEffect`), assigned via **`assignment-secops-audit.json`** which includes a
**system-assigned identity** so the Modify cascade can be enabled without re-creating it.

## 3. Applicable Frameworks
| Framework | Relevance |
|-----------|-----------|
| **GLBA / FTC Safeguards Rule** | Protection of members' NPI. |
| **NCUA Part 748** | Security program & incident response. |
| **PCI-DSS** | Payment card data. |
| **SOC 2** | Trust Services Criteria. |
| **SOX** (if applicable) | Financial reporting integrity. |

## 4. Tag-to-Framework Matrix
| Tag | GLBA | NCUA 748 | PCI-DSS | SOC 2 | Role |
|-----|:----:|:--------:|:-------:|:-----:|------|
| `data classification` | ✅ | ✅ | ✅ | ✅ | Scope & control driver |
| `compliance` | ✅ | ✅ | ✅ | ✅ | Regulatory scope flag |
| `owner` / `data steward` | ✅ | ✅ | ✅ | ✅ | Accountability / IR |
| `env` | | ✅ | ✅ | ✅ | Prod/non-prod separation |

## 5. Data Classification → Enforced Controls
| Value | Meaning | Controls (tag-enforced + baseline) |
|-------|---------|------------------------------------|
| `public` | Publishable | Standard. |
| `internal` | Internal-use | Access control, encryption at rest. |
| `confidential` | Sensitive business data | + restricted access, logging, TLS. |
| `restricted` | **Member NPI / cardholder data** | + **deny public network** (policy), **HTTPS-only** (policy), **require `compliance` tag** (policy), private endpoints, CMK/HSM, DLP, monitored access. |

Unlike a label-only approach, `restricted` now triggers **actual Azure Policy controls**
(public-network deny, HTTPS-only). Extend with encryption/private-endpoint built-ins as
needed.

## 6. Audit Scoping (Resource Graph)
```kusto
resources
| where tags['data classification'] == 'restricted'
      or tags['compliance'] contains 'PCI-DSS'
| project name, type, resourceGroup, subscriptionId,
          owner=tags['owner'], classification=tags['data classification']
```

## 7. Exemptions
Use Azure Policy exemptions with justification + expiry. **Do not** broadly exempt
`restricted` / PCI resources.

## 8. Phasing
Phase 1 Audit → Phase 2 enable Modify cascade + remediate → Phase 3 Deny (target Prod).
See `06-governance-policy-plan.md`.

## 9. Disclaimer
Guidance only. Validate the control set with compliance, legal, and audit stakeholders;
obligations depend on your data flows and charter.
