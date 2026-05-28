# AI Platform Evaluation — Vertex AI vs Azure AI Foundry

> **Page properties**
> | | |
> |---|---|
> | **Status** | `RECOMMENDATION READY` |
> | **Owner** | Architecture |
> | **Audience** | Architecture review, AI platform working group |
> | **Workload** | Internal AI chat app + reusable internal AI API |
> | **Environment** | All-in on Microsoft Azure |
> | **Decision sought** | Platform selection for the internal AI API |
> | **Version** | 1.0 |
> | **Last reviewed** | 2026-05-27 |
> | **Re-evaluation cadence** | Annual + on listed triggers (see Section 10) |

---

> ℹ️ **TL;DR**
>
> **Recommendation: build on Azure AI Foundry**, behind an Azure API Management (APIM) AI Gateway designed with a **portable backend abstraction** so a second backend (e.g., Vertex AI) can be added later for a specific workload without rewriting consumers.
>
> Foundry is selected on the strength of single-cloud data path, native Entra identity, existing compliance evidence pack, and adequate model coverage for the named workload. No Vertex-exclusive capability has been identified as a requirement for the initial internal chat + reusable AI API.

> ✅ **Decision panel**
>
> **Platform:** Azure AI Foundry
> **Pattern:** APIM AI Gateway in front, model-routing abstraction inside, Private Endpoints throughout
> **Optionality preserved:** Vertex AI can be added as a second backend behind the same gateway if a future workload justifies it
> **Excluded by this decision:** standing up Vertex AI now as a primary or co-primary platform

---

## Table of Contents

1. Context
2. Decision framing (why this comparison is asymmetric)
3. Side-by-side comparisons
4. Scorecard
5. Recommendation
6. Rationale
7. Risks accepted
8. Implementation roadmap
9. Cost shape
10. Re-evaluation triggers
11. Open questions and dependencies
12. Related pages

---

## 1. Context

The company is initiating an internal generative AI capability with two coupled goals:

- An **internal AI chat application** for employees (call center support, internal Q&A, policy lookup, drafting assistance).
- A **reusable internal AI API** that other internal systems can call (e.g., document classification, summarization, intent routing, lightweight scoring).

The customer environment is **all-in on Azure**: Entra ID for identity, Azure data platforms (ADLS / Fabric / OneLake / SharePoint) for content, Microsoft 365 for collaboration, and Microsoft Sentinel for SIEM. There is no existing GCP footprint and no operational team experienced with running GCP workloads.

Two platforms were evaluated: **Azure AI Foundry** and **Google Vertex AI**.

---

## 2. Decision framing (why this comparison is asymmetric)

> ⚠️ **Frame this correctly**
>
> This is not a symmetric platform bake-off. The choice is between **staying single-cloud on Foundry** and **adopting a second cloud provider (Vertex) for AI**. Those are different magnitudes of decision and should not be scored on equal footing.

The comparison answers two questions, in order:

1. **Can Foundry meet the bar for this workload?**
2. **If yes, does Vertex offer something specific and material enough to justify a second-vendor posture** (egress, dual identity, dual compliance evidence, second on-call surface, partner interconnect)?

The default is Foundry. The burden of proof sits on Vertex to demonstrate a material advantage on a criterion that matters for this workload.

---

## 3. Side-by-side comparisons

### 3.1 Platform overview

| Dimension | Azure AI Foundry | Google Vertex AI |
|---|---|---|
| **Position relative to customer** | Native — inside existing Azure tenant, EA, identity, network, compliance pack | External — net-new cloud presence, identity federation, compliance evidence |
| **Model catalog (headline)** | OpenAI GPT-4o / 4.1 / o-series, Anthropic Claude, Meta Llama, Mistral, Cohere, Microsoft Phi | Google Gemini, Anthropic Claude (via Model Garden), Meta Llama, third-party via Model Garden |
| **Inference services** | Foundry-managed endpoints, serverless and provisioned throughput | Vertex-managed endpoints, online and batch prediction |
| **RAG / grounding (native)** | Azure AI Search, Fabric / OneLake, ADLS, SharePoint connectors, Purview lineage | Vertex AI Search, BigQuery, GCS, Vertex RAG Engine |
| **Safety / responsible AI** | Azure AI Content Safety, groundedness, jailbreak detection, PII detection | Safety Filters, Model Armor, responsible-AI tooling |
| **Prompt ops / evals** | Prompt Flow, Foundry evals, Azure ML integration | Vertex AI Studio, Vertex Pipelines, eval services |

### 3.2 Network and data path

| Property | Foundry (single-cloud) | Vertex (cross-cloud via PSC + Partner Interconnect) |
|---|---|---|
| Caller → AI hop | Private Endpoint on Microsoft backbone | Private Service Connect over ExpressRoute + Partner Interconnect (Megaport / Equinix) |
| Member data path | Stays inside Azure trust boundary | Crosses cloud boundary on every call |
| Egress cost | $0 | Per-GB interconnect egress + fixed monthly circuit + partner fabric |
| Latency overhead (caller → AI) | <10 ms intra-region | +10–30 ms typical |
| DNS | Private DNS zones, native | Split-horizon, PSC private endpoint mapping |
| Failure domains | One (Azure region) | Azure region + GCP region + interconnect partner + two private endpoints |
| Examiner narrative | "AI runs in Azure, like everything else" | Net-new cross-cloud data-flow explanation required |

### 3.3 Identity and access

| Property | Foundry | Vertex |
|---|---|---|
| Identity source | Native Entra ID | Entra → Workload Identity Federation → GCP service account → Vertex IAM |
| Workload auth | Managed identities, Entra RBAC, Conditional Access, PIM | WIF tokens; CA enforceability requires explicit design |
| Key management | None in flight (managed identity) | None in flight (WIF), but service-account lifecycle is net-new to manage |
| Operational burden | Existing team, existing skills | Second identity model, second admin surface |

### 3.4 Compliance and governance

| Property | Foundry | Vertex |
|---|---|---|
| Audit / examiner evidence pack | Existing (SOC 2, ISO, PCI in current binder) | Existing pack + GCP evidence pack + interconnect partner attestations |
| Data lineage | Purview, end-to-end native | Purview (Azure) + Dataplex (GCP), manually bridged at the boundary |
| Data residency | Azure regions in scope; no cross-cloud movement | Cross-cloud movement is intrinsic to the path |
| Net-new artifacts for compliance | None | Cross-cloud data-flow diagram, WIF trust documentation, GCP shared-responsibility narrative, interconnect partner attestations, GCP-side log retention policy |

### 3.5 Operational surface

| Property | Foundry | Vertex |
|---|---|---|
| Clouds operated | 1 | 2 |
| Control planes monitored | 1 | 2 |
| On-call surface | Existing Azure rotation | Existing + GCP + interconnect partner |
| SIEM ingestion | App Insights → Log Analytics → Sentinel, native | Cloud Logging → Pub/Sub → Sentinel connector, net-new pipeline |
| Skills required | Existing | Existing + GCP IAM + Vertex services + cross-cloud networking |

### 3.6 Cost shape (qualitative)

| Cost component | Foundry | Vertex |
|---|---|---|
| Token spend (per model) | Comparable on equivalent models; varies month-to-month | Comparable on equivalent models; varies month-to-month |
| Network / egress | $0 (Private Endpoint on MS backbone) | Interconnect egress + fixed monthly circuit + partner fees |
| Identity / IAM ops | Absorbed in existing Azure ops | Net-new (WIF + GCP IAM admin) |
| Compliance evidence ops | No delta | Annualized cost of producing second evidence pack |
| On-call / SRE | No delta | Expanded on-call surface |
| Discounting | Existing Microsoft EA / MCA terms apply | Net-new commercial relationship |

> ⚠️ **Don't anchor on token pricing.** Per-1k-token rates shift monthly and are roughly comparable between equivalent models on each platform. The cost difference is dominated by **egress + ops + compliance evidence**, not by token list price.

### 3.7 Gateway pattern (APIM AI Gateway)

| Property | In front of Foundry | In front of Vertex |
|---|---|---|
| APIM AI Gateway policies (`llm-token-limit`, `llm-semantic-cache-*`, `llm-content-safety`, `llm-emit-token-metric`) | Native, well-trodden pattern | Same APIM policies apply; only the backend URL and auth differ |
| Backend auth | APIM managed identity → Foundry RBAC | APIM managed identity → WIF → GCP IAM → Vertex |
| What changes per backend | Backend URL, auth mode | Backend URL, auth mode |
| What stays the same | Throttling, semantic cache, PII redaction, prompt logging, model routing, key vaulting | Identical |

> ✅ **Key insight:** the APIM AI Gateway investment is **portable between backends**. It is the right place to invest first, regardless of which platform is chosen for inference. Designed correctly, it allows adding Vertex later as a second backend without rewriting any consumer.

---

## 4. Scorecard

Weights:
- **C — Critical.** A clear loss blocks the platform for this workload.
- **H — High.** Meaningful but not blocking on its own.
- **M — Medium.** Tiebreaker.

| # | Criterion | Weight | Foundry | Vertex | Notes |
|---|---|---|---|---|---|
| 1 | Model fit for named intents | C | ✅ Meets | ✅ Meets | No Vertex-exclusive capability identified as a requirement for the initial workload. Both platforms cover internal chat and the planned reusable AI API. |
| 2 | Data path / network | C | ✅ Strong | ⚠️ Adequate w/ B2 only | Single-cloud trumps cross-cloud for member-adjacent data. Vertex requires PSC + Partner Interconnect to be acceptable. |
| 3 | Identity and access | C | ✅ Strong (native Entra) | ⚠️ Adequate w/ WIF | Second identity model = net-new admin burden. |
| 4 | Grounding / data integration | H | ✅ Strong (Fabric / ADLS / Search / Purview) | ⚠️ Requires data copy or sync | No business reason to duplicate the grounding corpus into GCP today. |
| 5 | LLM gateway pattern | H | ✅ Native APIM AI Gateway over Private Endpoint | ✅ Same APIM in front, cross-cloud backend | Pattern is portable; favors Foundry only because backend is closer. |
| 6 | Compliance / examiner evidence | H | ✅ No delta | ⚠️ Net-new pack required | Recurring annual cost. |
| 7 | Responsible AI tooling | M | ✅ Content Safety, groundedness, jailbreak | ✅ Safety Filters, Model Armor | At parity for this workload. |
| 8 | MLOps / prompt ops | M | ✅ Prompt Flow, Azure ML in skillset | ⚠️ Net-new (Vertex Studio, Pipelines) | No team experience with Vertex. |
| 9 | Cost (run-rate + ops) | M | ✅ Single bill, no egress | ⚠️ Egress + interconnect + ops | Token list price is a wash; network and ops dominate. |
| 10 | Vendor concentration risk | M | ⚠️ Heavy Microsoft concentration | ✅ Diversification benefit | The only criterion where Vertex meaningfully wins on its own. Insufficient on its own to flip the decision. |

**Critical-weight outcome:** Foundry passes all three. Vertex meets the bar with caveats but does not exceed Foundry on any C-weight criterion.

**Result:** Foundry is selected.

---

## 5. Recommendation

> ✅ **Recommended: Azure AI Foundry**
>
> Build the internal AI chat application and the reusable internal AI API on **Azure AI Foundry**, fronted by an **Azure API Management AI Gateway** designed with a **portable backend abstraction**. Use Private Endpoints throughout. Use existing Entra ID, Sentinel, and Purview integrations as-is.
>
> Preserve the option to add **Vertex AI as a second backend** behind the same gateway if a future workload demonstrates a Vertex-specific need (e.g., a verified, durable Gemini-exclusive capability requirement; a BigQuery-colocated ML workload; a board-level diversification mandate).

### What this does NOT mean

- Not a recommendation against Google Cloud as a platform generally.
- Not a permanent exclusion of Vertex from the credit union's AI strategy.
- Not a commitment to a single model — Foundry's catalog (OpenAI, Anthropic, Meta, Mistral, Cohere, Microsoft) provides material breadth.

---

## 6. Rationale

The recommendation rests on five points, in order of weight:

1. **Cross-cloud cost is real and recurring, single-cloud savings are real and recurring.** For an internal AI API expected to serve multiple consumer systems with member-adjacent data, every prompt would cross a cloud boundary in the Vertex path. The cost is not only egress — it is the **ongoing operational and compliance overhead** of running a second cloud presence.
2. **No Vertex-exclusive capability is currently named as a requirement** for the initial workload. Foundry's catalog covers internal chat and the planned reusable AI API with credible model options (OpenAI, Anthropic, Meta). Choosing Vertex today would be on speculation of a future need rather than a known one.
3. **Identity, data, and governance are already coherent in Azure.** Entra ID, Purview lineage, Sentinel ingestion, Conditional Access, and managed identities reach the AI service natively in the Foundry path. The Vertex path replaces native integration with a federated equivalent — workable, but a net-new operational surface that must be staffed forever.
4. **The compliance evidence delta is permanent, not one-time.** Adding Vertex means producing and maintaining a second evidence pack for examiners every year. The marginal annual cost is non-trivial for a credit union risk function.
5. **The APIM AI Gateway investment preserves optionality.** Because the gateway pattern is portable, choosing Foundry today does not foreclose adding Vertex tomorrow for a specific, justified workload. The decision is reversible at the workload level without rewriting consumers.

---

## 7. Risks accepted

> ⚠️ **Risks of choosing Foundry — and how they are mitigated**
>
> | Risk | Mitigation |
> |---|---|
> | Microsoft AI vendor concentration | APIM gateway abstraction preserves the option to add a second backend (Vertex or self-hosted) per-workload without consumer rewrites. |
> | Lag on a future Google-exclusive model capability | Annual re-evaluation; explicit re-evaluation triggers listed in Section 10. |
> | Internal API consumers becoming Azure-coupled | Mitigated by routing all consumers through APIM with a logical model identifier (not an Azure resource ID) in the request contract. |
> | Foundry model availability shifting | Periodically verify the model catalog in the customer's tenant and region; don't assume static availability. |

---

## 8. Implementation roadmap

> ℹ️ **Sequencing principle**
>
> Build the **gateway and the portable abstraction first**, then the chat app on top. The gateway is the durable investment; the chat app is the first consumer.

**Phase 1 — Foundation (gateway-first)**
- Stand up APIM (Standard v2 or Premium SKU; VNet integration required).
- Provision Foundry workspace with Private Endpoints; assign managed identity from APIM.
- Implement the APIM AI Gateway policy bundle: JWT validation (Entra), token throttling, semantic cache, PII redaction, content safety, prompt logging to Sentinel, model routing by logical name.
- Wire diagnostic settings to Log Analytics and Sentinel.
- Document the **request contract** (logical model identifier, not vendor resource ID).

**Phase 2 — Chat app (first consumer)**
- Build the internal chat application against the APIM endpoint, **not** directly against Foundry.
- Integrate Azure AI Search as the grounding store; index the agreed initial corpus.
- Configure Entra SSO for end users; managed identity for app-to-APIM.

**Phase 3 — Open the API to a second consumer**
- Onboard a second internal system to the same APIM endpoint.
- Validate that the contract (logical model identifier, throttling per consumer key, prompt logging tenant tag) holds.

**Phase 4 — Evaluation and tuning**
- Run a same-prompt, same-grounding eval over the top intents quarterly; track drift.
- Confirm the recommendation is still valid against re-evaluation triggers (Section 10).

---

## 9. Cost shape (qualitative budget structure)

This page does not commit to dollar figures (token prices and interconnect rates shift). The cost structure to forecast is:

- **Tokens** — per-model rates × forecast prompt + completion volume. Apply semantic-cache hit-rate assumption (typically 15–40% for chat workloads after a few weeks).
- **APIM SKU** — Standard v2 or Premium monthly base + capacity units as needed.
- **Azure AI Search** — replica + partition count for the index.
- **Storage** — ADLS / OneLake for source documents.
- **Observability** — Log Analytics / Sentinel ingestion volume (a portion of prompts and responses are logged).
- **Foundry runtime** — pay-as-you-go or provisioned throughput depending on burst pattern.

**Not budgeted** (since not in scope): ExpressRoute, Partner Interconnect, GCP project, Cloud Logging, second on-call rotation. These would only appear if the future trigger for adding Vertex fires.

---

## 10. Re-evaluation triggers

The platform decision should be **revisited when any of the following becomes true**, not on a fixed calendar alone:

- A specific business need emerges that **requires a Vertex-exclusive model capability** (e.g., Gemini long-context >1M tokens for a real intent, Vertex Search demonstrably outperforming Azure AI Search on a real corpus, BigQuery-colocated ML workload).
- A workload is proposed that is **naturally GCP-shaped** (e.g., heavy BigQuery analytics with embedded ML).
- A **risk / board mandate for vendor diversification** is issued.
- Microsoft makes a material adverse change to **pricing or terms** for Azure AI services.
- The annual review date falls and no other trigger has fired in the interim.

When a trigger fires, evaluate only the affected workload against the Vertex option — do not relitigate the entire platform decision.

---

## 11. Open questions and dependencies

> ❓ **Items to confirm with customer stakeholders before Phase 1**
>
> | # | Question | Owner |
> |---|---|---|
> | 1 | Confirmed list of top 3–5 intents for the chat app's initial scope | Product / business |
> | 2 | Source of grounding corpus for V1 (SharePoint sites, ADLS folders, Fabric items) | Data |
> | 3 | Azure regions in use and paired-region failover expectation | Infrastructure |
> | 4 | APIM SKU currently in use; capacity to add the AI Gateway workload | Platform |
> | 5 | Sentinel ingestion budget for prompt/response logging | Security |
> | 6 | PII redaction policy — what categories are scrubbed and how strictly | Risk / privacy |
> | 7 | Per-consumer throttle / quota policy for the reusable AI API | Architecture |
> | 8 | Evaluation eval set ownership (who builds and maintains the test prompts) | AI working group |

---

## 12. Related pages

- `01-decision-framework.md` — asymmetric framing, 10-criterion scorecard, evidence required per criterion, decision tree.
- `02-reference-architectures.md` — Foundry single-cloud architecture; Vertex three connectivity options; layer-by-layer comparison.
- `03-network-and-gateway-deepdive.md` — APIM AI Gateway pattern, network path analysis for both platforms, latency budget, things to verify.
- `CLAUDE.md` — project framing and conventions for future iterations.
