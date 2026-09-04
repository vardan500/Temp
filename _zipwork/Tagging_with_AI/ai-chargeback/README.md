# AI Chargeback — telemetry & allocation assets

Assets that power the **usage-based split** — the sole attribution path for AI spend,
since every AI resource is a shared multi-tenant endpoint. See
`../docs/08-ai-chargeback.md` for the full method.

## Files
- `consumer-map.csv` — `app → cost center, department, project, workload`. The
  version-controlled join between shared-endpoint telemetry and the chargeback
  dimensions, keyed by `app`: each application has exactly one calling identity, named
  exactly its `app` tag value, so telemetry yields `app` directly. The `workload` column
  is the use-case rollup (there are no per-use-case resources to tag on a single shared
  platform). PR-reviewed; any `app` in telemetry but not here → unallocated (platform
  bucket) KPI — usually a misnamed identity.
- `queries/tokens-per-consumer.kql` — token usage per `app` and model (Azure OpenAI).
- `queries/transactions-per-consumer.kql` — transaction counts per `app` (AI Services).
- `queries/unallocated-usage.kql` — usage whose `app` is missing/unmapped.
- `queries/monthly-split-summary.kql` — per-application charge, split per model then
  summed (model prices differ; each model's Cost Management meter cost is split by that
  model's tokens).

## Usage
1. Enable diagnostics on shared AI resources → Log Analytics (see `docs/08 §3`).
2. Adjust the `extend`/column names in each query to match your log schema (APIM vs raw
   diagnostics differ).
3. Run monthly; join results to `consumer-map.csv`; apply the split formula in `docs/08 §6`;
   reconcile the total back to Cost Management.
