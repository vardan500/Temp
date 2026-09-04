# AI Resource Chargeback (FinOps)

How to attribute **Azure OpenAI / AI Foundry** and **Azure AI Services** (Cognitive
Services, Document Intelligence, Speech, AI Search) cost to the business, extending the
existing FinOps tagging chargeback (`03-pillar-finops.md`).

**Primary chargeback dimensions:** `cost center` + `department`. **Join key:** `app` —
each application has exactly **one calling identity, named exactly its `app` tag value**,
so telemetry yields the application directly. **Secondary:** `project`.

## 1. Why AI needs more than resource tags
A resource's `cost center` tag attributes **100%** of that resource's bill to one owner.
In this environment **every AI resource is a shared multi-tenant endpoint**, consumed by
many teams and billed by **tokens / PTUs / transactions** — so no AI cost can be attributed
by resource tags alone. All AI spend is attributed by a single **usage-based split** path:

```
telemetry → per-application usage (app) → proportional split of the resource's bill
```

Resource-level cost tags (`cost center`, `department`, …) remain required on AI resources,
but they identify the **platform team** that owns the endpoint — and receive the
**unallocated** remainder of the split — not a consuming business unit. The only
AI-adjacent cost still attributed purely by resource tags is GPU / ML compute (§9).

## 2. AI tags (see `02-tag-taxonomy.md §1a`)
There are **no AI-specific tags**. AI resources carry only the existing default tags
(`cost center`, `department`, `app`, `project`, `owner`, `env`), which identify the
platform team (see §1), not the consumer. With a single shared platform (one Foundry,
shared Content Safety, etc.), any AI-specific resource tag would hold one constant
value, so every chargeback dimension lives where it can vary — in telemetry and the
consumer map: consumer = the calling identity, whose name **is** the consuming
application's `app` value (§4); use case = `workload` column of `consumer-map.csv` (§5);
billing is uniformly **pay-as-you-go**, split per model (§6). (Retired tags from earlier
revisions: `shared service`, `ai workload`, `ai billing model`, `app id`.)

**Policies (FinOps pillar):** `audit-diagnostics-on-ai` — bundled in
`initiative-finops.json` (group `ai-cost-allocation`). The naming invariant
(identity name = `app` value) is enforced operationally at consumer onboarding (§4),
not by Azure Policy — policy cannot see APIM subscription or AAD app names.

## 3. Telemetry sources
| Source | Provides | Notes |
|--------|----------|-------|
| Azure OpenAI **diagnostic logs** → Log Analytics | Requests, token counts, model, deployment | Enable `RequestResponse` / `Audit` categories via diagnostic settings |
| Azure Monitor **metrics** | `TokenTransaction`, `ProcessedPromptTokens`, `GeneratedTokens`, `ProcessedInferenceTokens` | Good for totals, weaker per-consumer identity |
| **APIM** in front of the endpoint | Per-consumer identity (subscription key / JWT `sub` / header) + token metrics via `llm`/`azure-openai` policies | **Recommended** for multi-tenant shared endpoints |
| AI Services (Speech, Doc Intelligence, Search) diagnostics | Transaction/operation counts | Split by transactions instead of tokens |

Because the split is the **only** attribution path, an endpoint without diagnostics
produces cost that cannot be allocated at all. Diagnostics coverage is therefore audited
by policy (`audit-diagnostics-on-ai`) from Phase 1.

Enable diagnostics (example):
```bash
az monitor diagnostic-settings create \
  --name ai-chargeback \
  --resource "<aiResourceId>" \
  --workspace "<logAnalyticsWorkspaceId>" \
  --logs '[{"category":"RequestResponse","enabled":true},{"category":"Audit","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

## 4. Consumer identity strategy
The split needs a reliable **consumer key** per call. Every application has exactly
**one calling identity**, and that identity is **named exactly the application's `app`
tag value** — so the key in telemetry *is* `app`, the same dimension the rest of the
tagging standard already uses. Since every resource is shared, per-call identity is a
**hard prerequisite** — there is no dedicated-resource fallback:
1. **Preferred — APIM:** each application gets one APIM subscription/product, named its
   `app` value; APIM emits it into logs.
2. **AAD app / managed identity:** one app registration per application, named its
   `app` value.

The **naming invariant** (identity name = `app` value) is what makes this work, and it
is enforced at onboarding, not by policy: creating a consumer identity means (a) naming
it exactly the portfolio `app` value and (b) adding the row to `consumer-map.csv` in the
same PR — **before** granting endpoint access. Traffic with a missing or misnamed
identity cannot be attributed: it lands in the platform's unallocated bucket (§5) and is
tracked as a KPI (a nonzero bucket usually means a naming violation).

`app` is the single join key used everywhere (default tag + telemetry + mapping table).

## 5. Consumer → allocation mapping
Maintain `ai-chargeback/consumer-map.csv` (version-controlled, PR-reviewed), keyed by
`app`:
```
app,cost center,department,project,workload
online-banking,CC-4100,DigitalBanking,DAO,member-chatbot
loan-origination,CC-5200,Lending,AKC,doc-intelligence-loans
fraud-analytics,CC-3300,RiskFraud,Consortium,fraud-copilot
```
The `workload` column is the **use-case rollup** (formerly the `ai workload` resource
tag): with one shared platform serving every use case, the use-case dimension can only
come from per-consumer data. Use-case showback = split output grouped by `workload`.
Any `app` seen in telemetry but **absent** from the map → **unallocated** bucket
(platform cost center — the `cost center` tag on the resource itself), tracked as a KPI
until mapped or the identity's name is corrected. With all AI resources shared, this map
is the **sole source** of AI consumer chargeback — treat changes to it with the same
rigor as a policy change (PR review by the platform + FinOps owners).

## 6. Split formula (paygo, per model)
All billing is **pay-as-you-go**, and token prices differ per model — raw token totals
would make consumers of cheap models subsidize consumers of expensive ones. So the split
runs **per model, then sums**. For shared resource `R`, model `m`, period `P`:

```
consumer_share(c, m) = tokens(c, m) / total_tokens(R, m, P)
consumer_cost(c)     = Σ over m of  consumer_share(c, m) × actual_cost(R, m, P)
```

- **`actual_cost(R, m, P)` — read, don't estimate:** Azure OpenAI paygo bills **per-model
  meter lines**, so each model's actual cost comes straight from the Cost Management
  usage detail / export filtered to the resource.
- **Fallback (no meter-level detail):** maintain a version-controlled model price table
  (governed like `consumer-map.csv`), compute each consumer's estimated dollars from list
  prices, use those as split weights, and scale so the total reconciles to the actual.
- **Prompt vs completion:** completion tokens are typically several times more expensive
  than prompt tokens. Where consumers differ sharply in shape (e.g., prompt-heavy RAG vs
  generation-heavy drafting), weight the two token types by their prices instead of using
  the raw `prompt + completion` sum.
- **AI Services (transactions):** `usage` = transaction/operation counts per `app`
  (Content Safety, Document Intelligence, Speech, Search), split per meter the same way.

Always **reconcile** the sum of `consumer_cost` back to Cost Management — the delta is
the unallocated bucket.

## 7. KQL (in `ai-chargeback/queries/`)
Tokens per application **and model** (the model dimension is required for the per-model
split in §6; the identity name in telemetry equals the `app` tag value):
```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.COGNITIVESERVICES"
| where TimeGenerated between (startofmonth(now()) .. now())
| extend app = tostring(properties_app_id_s)   // shape to your APIM/log schema
| extend model = tostring(properties_model_s)
| summarize prompt=sum(toint(promptTokens_d)), completion=sum(toint(completionTokens_d)) by app, model
| extend totalTokens = prompt + completion
| order by totalTokens desc
```
See the folder for `transactions-per-consumer`, `unallocated-usage`, and
`monthly-split-summary` (implements the per-model split) queries.

## 8. Showback → Chargeback phasing
| Phase | Goal | Enforcement |
|-------|------|-------------|
| **1 — Instrument & Audit** | Diagnostics on every AI resource; identities renamed/issued per the naming invariant (§4); publish **showback** per `app`; build map | FinOps policies `effect=Audit`, `diagnosticsEffect=AuditIfNotExists` |
| **2 — Attribute** | Monthly split; reconcile unallocated; identity coverage ≥95% and rising toward ~100% (the split is the only attribution path); validate vs Cost Management | Policies still Audit; remediate diagnostics & identity naming |
| **3 — Chargeback + Deny** | Post allocations to GL; formal chargeback | Pillar value-integrity policies → `Deny` per `06`; AI attribution enforcement stays **operational** (naming invariant at onboarding + diagnostics audit) — there is no AI tag left to Deny |

Cadence 30–60 days per phase, gated on coverage/reconciliation accuracy (mirrors
`06-governance-policy-plan.md`).

## 9. Special cases
- **GPU / ML compute (VMs, AKS GPU nodes, ML compute):** out of primary scope but attribute
  via existing cost tags on the compute resource; note as a future extension.
- **Data sensitivity:** keep split dimensions to non-content metadata (ids, counts, model,
  tokens) — never prompt/response content. Align AI resources with SecOps
  `data classification`.
- **Untagged/unmapped usage:** platform `cost center` bucket + KPI; drive to zero.
- **PTU / commitment later:** if a reservation is ever adopted for a stable heavy
  workload, reintroduce the `ai billing model` tag and an amortization method (usage
  share vs provisioned share) — see plan.md revision history.

## 10. Worked example (per-model split)
Shared Azure OpenAI endpoint, actual monthly cost **$10,000** — Cost Management meters:
**gpt-4o $8,000**, **gpt-4o-mini $2,000**. Split each model's cost by that model's tokens:

| Model (actual cost) | app | tokens | share | charge |
|---------------------|-----|--------|-------|--------|
| gpt-4o ($8,000) | online-banking | 6,000,000 | 60% | $4,800 |
| | fraud-analytics | 3,000,000 | 30% | $2,400 |
| | (unmapped) | 1,000,000 | 10% | $800 |
| gpt-4o-mini ($2,000) | online-banking | 5,000,000 | 25% | $500 |
| | loan-origination | 15,000,000 | 75% | $1,500 |

Summed per application:
| app | charge | allocation |
|-----|--------|------------|
| online-banking | $5,300 | CC-4100 / DigitalBanking (`member-chatbot`) |
| fraud-analytics | $2,400 | CC-3300 / RiskFraud (`fraud-copilot`) |
| loan-origination | $1,500 | CC-5200 / Lending (`doc-intelligence-loans`) |
| (unmapped) | $800 | platform bucket (KPI — likely a misnamed identity, see §4) |
| **Total** | **$10,000** | ✓ reconciles |

Note why per-model matters: `online-banking` generated 37% of all tokens (11M of 30M)
but owes 53% of the cost, because most of its tokens hit the expensive model. A raw
token split would have under-charged it and over-charged `loan-origination`.
