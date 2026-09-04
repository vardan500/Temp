# Migration Landing Zone Design

**Purpose:** Target landing zone for ~200 on-premises servers migrating to Azure.
**Version:** 1.0 — 2026-09-01
**Companion document:** [`existing-infrastructure-assessment.md`](./existing-infrastructure-assessment.md) — every current-state fact this design builds on.

---

## 1. Decisions taken

| Decision | Choice | Consequence this design must absorb |
|---|---|---|
| **Region** | **`westus`** | No availability zones. Resilience must come from Availability Sets + cross-region DR. |
| **Subscriptions** | **Existing `Production` and `Development`** | No new subscription boundary. Quota must be raised on a live prod subscription; existing policy applies immediately. |
| **Hub** | **Existing `tcu-p-usw-exprt-vnet-01`** — reused as-is | No new ExpressRoute circuit, connection, or Premium upgrade. |
| **Workload mix** | Windows domain-joined, SQL, app/web/middleware, **and Linux** | Four server tiers, and Azure domain controllers become mandatory. |

### Planning assumptions (adjust as inventory firms up)

- **~150 production / ~50 non-production** split. Both spokes are sized as `/16`, so any split works without re-addressing.
- **~40 TB** total replicated data (200 servers × ~200 GB average). Drives the bandwidth math in §12.
- Tier distribution: ~50 web, ~70 app/middleware, ~25 SQL, ~35 Linux, ~20 infra/mgmt/DC.

---

## 2. Placement model

Migrated servers land in **new dedicated spoke VNets** inside the existing subscriptions — not inside the existing `tcu-p-usw-vnet-01` / `tcu-np-usw-vnet-01` spokes.

```
Production subscription  f7f18245-3c4e-47f1-b465-e6bd0249f2c6
├── tcu-p-usw-exprt-vnet-01     10.209.0.0/16   HUB (existing, unchanged)
│     ├── ExpressRoute GW  tcu-p-usw-exprt-vnet-gw-01
│     └── Azure Firewall   tcu-p-usw-azurefw-01  @ 10.209.1.4
├── tcu-p-usw-vnet-01           10.211.0.0/16   existing prod spoke (untouched)
└── tcu-p-usw-mig-vnet-01       10.212.0.0/16   ★ NEW — migrated prod servers

Development subscription  029c65f2-4ae1-4e7b-b152-94e909b1d278
├── tcu-np-usw-vnet-01          10.210.0.0/16   existing non-prod spoke (untouched)
└── tcu-np-usw-mig-vnet-01      10.213.0.0/16   ★ NEW — migrated non-prod servers
```

**Why new spokes rather than extending `10.211`/`10.210`:**

1. The existing prod spoke carries ASE, APIM, Application Gateway, ISE and Databricks with delegated subnets. Adding 150 servers means repeated change windows against a VNet that fronts live customer-facing services.
2. Independent NSG, route table, and policy lifecycle for the migrated estate.
3. Clean cost and blast-radius separation without needing a new subscription.
4. Spoke VNets do **not** consume ExpressRoute circuit VNet links — only the hub does — so this costs nothing against the circuit's 10-link limit (1 in use).

---

## 3. Address plan

`10.212.0.0/16` and `10.213.0.0/16` were **verified free**: absent from all 204 on-prem BGP prefixes and from every Azure VNet. They are contiguous with the existing `10.209`/`10.210`/`10.211` Azure allocation.

| Range | Assignment | Status |
|---|---|---|
| `10.208.0.0/30` | ExpressRoute point-to-point | In use |
| `10.209.0.0/16` | Hub | In use |
| `10.210.0.0/16` | Non-prod spoke | In use |
| `10.211.0.0/16` | Prod spoke | In use |
| **`10.212.0.0/16`** | **Migration spoke — production** | **★ Proposed** |
| **`10.213.0.0/16`** | **Migration spoke — non-production** | **★ Proposed** |
| `10.214.0.0/16` | ⚠ Contains on-prem stray `10.214.65.64/32` | Avoid |
| `10.215.0.0/16` – `10.222.0.0/16` | Free | Reserve for growth |
| `10.223.0.0/16` | ⚠ Contains on-prem `10.223.4.0/24` | Avoid |

**Action:** on-prem BGP must be updated to accept `10.212.0.0/16` and `10.213.0.0/16` from Azure (AS path `65515`), matching how `10.209`–`10.211` are handled today.

---

## 4. Subnet plan

Server tiers use **`/21`** to match the established `10.211.8|16|24.0/21` convention, giving **2,043 usable addresses per tier** — roughly 10× headroom over the planned population.

### Production spoke — `tcu-p-usw-mig-vnet-01` (`10.212.0.0/16`)

| Subnet | Prefix | Usable | Purpose | Route table |
|---|---|---|---|---|
| `AzureBastionSubnet` | `10.212.0.0/26` | — | Bastion (prod has none today) | — |
| `tcu-p-usw-mig-mgmt-snet-01` | `10.212.0.64/27` | 27 | Jumpboxes, management tooling | `default` |
| `tcu-p-usw-mig-dc-snet-01` | `10.212.1.0/26` | 59 | Azure domain controllers | `default` |
| `tcu-p-usw-mig-stage-snet-01` | `10.212.2.0/24` | 251 | Azure Migrate appliance, replication | `stage` |
| `tcu-p-usw-mig-pe-snet-01` | `10.212.3.0/24` | 251 | Private endpoints | none |
| `tcu-p-usw-mig-web-snet-01` | `10.212.8.0/21` | 2,043 | IIS / web tier | `default` |
| `tcu-p-usw-mig-app-snet-01` | `10.212.16.0/21` | 2,043 | App / middleware | `default` |
| `tcu-p-usw-mig-data-snet-01` | `10.212.24.0/21` | 2,043 | SQL Server / data tier | `default` |
| `tcu-p-usw-mig-linux-snet-01` | `10.212.32.0/21` | 2,043 | Linux estate | `default` |
| `tcu-p-usw-mig-infra-snet-01` | `10.212.40.0/24` | 251 | File, print, batch, infra | `default` |
| *reserved* | `10.212.48.0/20` + | — | Growth / future tiers | — |

### Non-production spoke — `tcu-np-usw-mig-vnet-01` (`10.213.0.0/16`)

Identical layout at `10.213.x`, with `np` in place of `p`. Bastion optional — `tcu-np-usw-vnet-01-bastion` already exists in the Development subscription and can be reused via peering.

---

## 5. Routing and egress

**The existing model is correct and is reproduced exactly.** No hub change required.

### `tcu-{p|np}-usw-mig-default-route-01` — all server subnets

| Route | Prefix | Next hop |
|---|---|---|
| `all-thru-firewall` | `0.0.0.0/0` | `VirtualAppliance` → `10.209.1.4` |

**`disableBgpRoutePropagation: true`** — mirroring `tcu-p-usw-default-route-01`.

This is the crux of the design: with BGP propagation disabled, both internet-bound *and* on-prem-bound traffic leave via the single default route to the Azure Firewall, which then reaches on-prem through the hub's ExpressRoute gateway. It also means the `0.0.0.0/0` that on-prem advertises over ExpressRoute does **not** hijack Azure internet egress.

### `tcu-{p|np}-usw-mig-stage-route-01` — Azure Migrate staging subnet only

| Route | Prefix | Next hop |
|---|---|---|
| `onprem-thru-firewall` | `10.71.0.0/16` | `VirtualAppliance` → `10.209.1.4` |
| `direct-to-internet` | `0.0.0.0/0` | `Internet` |

Kept separate so replication traffic can reach Azure public endpoints without forcing 40 TB through the firewall, and so the migration path can be retired without touching steady-state server routing. See §12 for the Microsoft-peering / private-endpoint decision this depends on.

### Peering

| Side | Peering | `allowGatewayTransit` | `useRemoteGateways` | `allowForwardedTraffic` |
|---|---|---|---|---|
| Hub → mig-prod | `peer-hub-to-mig-prod` | **true** | false | true |
| mig-prod → Hub | `peer-mig-to-hub` | false | **true** | true |
| Hub → mig-nonprod | `peer-hub-to-mig-np` | **true** | false | true |
| mig-nonprod → Hub | `peer-mig-to-hub` | false | **true** | true |

The hub-side peering for the non-prod spoke is a **cross-subscription** operation (hub in Production, spoke in Development) and must be created from the Production side. Create the hub side first — a spoke peering with `useRemoteGateways: true` will not reach `Connected` until `allowGatewayTransit` is set on the hub side.

**Also required:** add `10.212.0.0/16` and `10.213.0.0/16` to `tcu-p-usw-gw-route-01` on the hub `GatewaySubnet`, so on-prem→spoke traffic is forced through the firewall the same way `10.210`/`10.211` are today. Without this, return traffic bypasses inspection and you get asymmetric routing.

---

## 6. Segmentation

**NSG at the subnet boundary, firewall for east–west and egress policy.**

`tcu-{p|np}-usw-mig-server-nsg-01` applied to all server subnets:

| Priority | Rule | Direction | Action | Source → Destination |
|---|---|---|---|---|
| 100 | `Allow-OnPrem-Inbound` | In | Allow | On-prem prefixes → spoke |
| 110 | `Allow-Intra-Spoke` | In | Allow | Spoke → spoke |
| 120 | `Allow-Hub-Inbound` | In | Allow | `10.209.0.0/16` → spoke |
| 4096 | `Deny-All-Inbound` | In | Deny | `*` → `*` |

On-prem source prefixes: `10.0.0.0/16`, `10.1.0.0/16`, `10.2.0.0/16`, `10.3.0.0/16`, `10.10.0.0/16`, `10.23.0.0/16`, `10.24.0.0/16`, `10.41.0.0/16`, `10.71.0.0/16`, `10.72.0.0/16`, `10.73.0.0/16`, `10.74.0.0/16`, `192.168.0.0/16`.

**Tier-to-tier policy (web→app→data) belongs on the Azure Firewall, not the NSGs** — that keeps the 200-server ruleset in one auditable place. This depends on the Firewall Policy migration in §11.

---

## 7. Identity and DNS

Today every Azure VNet resolves DNS to on-prem DCs `10.71.4.15 / .45 / .30` over ExpressRoute. Migrating 200 domain-joined servers onto that dependency turns a latent single point of failure into an operational one: if the circuit drops, 200 servers lose name resolution and domain authentication simultaneously.

### Target

1. **Deploy two domain controllers into Azure**, in different Availability Set fault domains, as a replica of the on-prem domain. Static private IPs.
2. **Repoint VNet DNS** on both migration spokes to the Azure DCs first, on-prem DCs second:
   ```
   Primary   : <Azure DC 1>
   Secondary : <Azure DC 2>
   Tertiary  : 10.71.4.15   (on-prem fallback)
   ```
3. Create an AD **Site** for Azure with the `10.212.0.0/16` and `10.213.0.0/16` subnets registered, so domain-joined servers bind to the Azure DCs rather than traversing ExpressRoute for every authentication.

### Where the DCs go — one hub decision required

The hub already contains **`tcu-p-usw-exprt-ad-snet-01` (`10.209.3.0/24`), purpose-built for this and completely empty.** Domain controllers are a shared service consumed by both spokes, so architecturally that subnet is the right home — but it is the **one change this design asks of the hub**.

| Option | Placement | Trade-off |
|---|---|---|
| **A (preferred)** | Hub `tcu-p-usw-exprt-ad-snet-01` | Correct shared-service placement; subnet already exists for exactly this. Requires a hub change. |
| **B** | Spoke `tcu-{p|np}-usw-mig-dc-snet-01` | Zero hub changes; allows environment-isolated DCs. Duplicates DCs per spoke. |

The `dc` subnet is included in both spokes regardless, so Option B remains available without re-addressing.

### Private DNS

Only `privatelink.blob.core.windows.net` and `privatelink.database.windows.net` exist today. Build out the zones the migrated estate needs (Key Vault, Storage file/queue/table, Recovery Services, Monitor) and link them to both migration spokes.

---

## 8. Resilience without availability zones

`westus` offers **no availability zones**. Verified maximum in this region: **3 fault domains** per Availability Set (`Aligned` and `Classic`).

### Availability Sets

- **`Aligned` SKU** (required for managed disks), **3 fault domains**, **20 update domains**.
- **One Availability Set per application tier per resource group.** Availability Sets cannot span resource groups, which is why the RG layout in §9 is tier-aligned.
- Maximum **200 VMs per Availability Set** — comfortable at this scale.

> ⚠ **An Availability Set can only be assigned at VM creation time.** A VM cannot be added to one afterwards without being recreated. Azure Migrate lets you specify the target Availability Set at failover — **this must be set in the migration plan, not fixed later.** Getting this wrong means rebuilding servers post-cutover.

### Proximity Placement Groups

For latency-sensitive app↔SQL pairs, place both tiers in a shared PPG. Note the tension: a PPG pins VMs closer together, which narrows fault-domain spread. Use it selectively, not estate-wide.

### Cross-region DR

`westus` pairs with **`eastus`**. With no in-region zone protection, cross-region DR carries the whole resilience story:

| Tier | Protection |
|---|---|
| Tier-1 (customer-facing, core banking) | Azure Site Recovery → `eastus`, documented RPO/RTO, tested failover |
| Tier-2 | GRS backup, rebuild-from-backup runbook |
| Tier-3 | GRS backup only |

**This is the item most likely to draw examiner attention.** The decision to run 200 production servers in a non-zonal region is defensible, but only with a documented, *tested* DR posture behind it. Recommend recording the rationale and the compensating controls explicitly.

---

## 9. Resource group layout

Tier-aligned, because Availability Sets cannot span resource groups.

**Production** (`f7f18245-…`)

| Resource group | Contents |
|---|---|
| `tcu-p-usw-mig-network-rg-01` | VNet, subnets, NSGs, route tables, Bastion |
| `tcu-p-usw-mig-web-rg-01` | Web tier VMs + Availability Set |
| `tcu-p-usw-mig-app-rg-01` | App / middleware VMs + Availability Set |
| `tcu-p-usw-mig-data-rg-01` | SQL VMs + Availability Set |
| `tcu-p-usw-mig-linux-rg-01` | Linux VMs + Availability Set |
| `tcu-p-usw-mig-infra-rg-01` | File, print, batch, DCs (Option B) |
| `tcu-p-usw-mig-bckup-rg-01` | Recovery Services Vault |
| `tcu-p-usw-mig-migrate-rg-01` | Azure Migrate project, appliance, replication storage |

**Development** (`029c65f2-…`) — same pattern with `tcu-np-usw-mig-*`.

---

## 10. Naming and tagging

Existing conventions are followed throughout — no new scheme is introduced.

| Object | Pattern | Example |
|---|---|---|
| Resource group | `tcu-<env>-usw-mig-<tier>-rg-01` | `tcu-p-usw-mig-app-rg-01` |
| VNet | `tcu-<env>-usw-mig-vnet-01` | `tcu-p-usw-mig-vnet-01` |
| Subnet | `tcu-<env>-usw-mig-<tier>-snet-01` | `tcu-p-usw-mig-data-snet-01` |
| Route table | `tcu-<env>-usw-mig-<purpose>-route-01` | `tcu-p-usw-mig-default-route-01` |
| Availability Set | `tcu-<env>-usw-mig-<tier>-avs-01` | `tcu-p-usw-mig-app-avs-01` |
| VM | `<p\|q\|d>azw<workload><nn>` | `pazwappsvr01` |

**Tags** — the `Cascade Default Tags Production` initiative already enforces the taxonomy: `app`, `cost center`, `department`, `env`, `owner`, `project`, `tier`.

Today these keys are present but **largely empty**, and `cost center` was blank on every resource group sampled. For a 200-server migration where chargeback matters, **populate the taxonomy before the estate lands** — retrofitting tags across 200 servers afterwards is materially harder. Add one migration-specific key: `migration-wave` (e.g. `wave-01`).

---

## 11. Prerequisites — ordered, with the blocker first

### 🚨 P0 — Resolve the EncryptionAtHost deadlock (Production only)

Production carries an enforced custom Deny policy, `Deny Windows VMs without EncryptionAtHost` (`bb4ab29614a65a26accce8f7`), with `defaultValue: "Deny"`, no parameter override, and an empty `notScopes`. Its rule matches Windows VMs **and** VMs where `imageSKU` does not exist — exactly the shape of a VM created by Azure Migrate from replicated managed disks.

Meanwhile `Microsoft.Compute/EncryptionAtHost` on Production is **`NotRegistered`**.

**Net effect: every Windows server landing in Production fails at create time.** The policy requires a property the subscription cannot set.

| | Production | Development |
|---|---|---|
| `EncryptionAtHost` feature | **`NotRegistered`** | `Registered` |

**Resolution — do this first:**
1. Register the feature on Production: `Microsoft.Compute/EncryptionAtHost`, then re-register the `Microsoft.Compute` provider. *(Registration is asynchronous — allow time before validating.)*
2. Configure Azure Migrate to set `encryptionAtHost = true` on every target VM.
3. Validate with a single throwaway Windows VM in `tcu-p-usw-mig-*` **before** any wave is scheduled.

Do **not** work around this by disabling or exempting the policy — it is a legitimate control, and for a credit union, weakening encryption-at-rest enforcement to enable a migration is the wrong trade. Register the feature instead.

### 🔴 P1 — Quota

Production is at **21 / 280 Total Regional vCPUs**. `Total Regional vCPUs` caps the subscription regardless of per-family headroom.

| Tier | Servers | Indicative size | vCPU |
|---|---|---|---|
| Web | ~50 | `Standard_D4s_v5` | 200 |
| App / middleware | ~70 | `Standard_D8s_v5` | 560 |
| SQL / data | ~25 | `Standard_E8ds_v5` | 200 |
| Linux | ~35 | `Standard_D4s_v5` | 140 |
| Infra / mgmt / DC | ~20 | `Standard_D2s_v5` | 40 |
| **Total** | **~200** | | **~1,140** |

**Request `Total Regional vCPUs` → 1,500** (headroom for migration overlap, where source and target run concurrently). Also raise the per-family quotas for the chosen `Dsv5`/`Esv5` families; note **`DSv3` and `ESv3` are capped at 100 each** today, so avoid v3 sizes or raise those too.

Submit early — quota requests against a live production subscription can take days.

### 🔴 P2 — DNS and domain controllers

Deploy and promote the Azure DCs (§7) **before** the first domain-joined wave. Migrating servers onto an ExpressRoute-dependent DNS path and fixing it later means touching all 200 twice.

### 🟠 P3 — Hub prerequisites

| Item | Change |
|---|---|
| `tcu-p-usw-gw-route-01` | Add `10.212.0.0/16` and `10.213.0.0/16` → `10.209.1.4` |
| On-prem BGP | Accept `10.212.0.0/16` and `10.213.0.0/16` from Azure |
| Azure Firewall | Rules for the new server tiers |

### 🟠 P4 — Firewall Policy migration

`tcu-p-usw-azurefw-01` runs **classic rule collections** (47 network rules, 4 application rules) with **no Firewall Policy**. Classic rules do not scale to a 200-server ruleset and block adoption of IP Groups and rule hierarchy.

Migrating to Firewall Policy is also the prerequisite for **Premium tier**, which adds IDPS and TLS inspection. Standard has neither. For a credit union this is worth an explicit, documented decision — either upgrade, or record why Standard is accepted.

### 🟠 P5 — Supporting platform

| Item | Action |
|---|---|
| Recovery Services Vault | New vault per subscription, **GRS**, sized for the estate. Existing vaults are workload-specific. |
| Log Analytics | `tcu-logs` retains **30 days** — extend for examination evidence. Size ingestion for 200 servers. |
| Azure Bastion | Deploy in Production — none exists today. |
| Private DNS zones | Build out beyond the two that exist. |
| Datadog / Rapid7 | Extend onboarding to the migrated estate. |

---

## 12. Migration mechanics

**No Azure Migrate project exists in either subscription.** Nothing has been started.

### The 500 Mbps constraint

This is the schedule driver, and no landing-zone choice changes it.

| Metric | Value |
|---|---|
| Circuit bandwidth | **500 Mbps** (`MeteredData`) |
| Theoretical throughput | 62.5 MB/s ≈ 225 GB/hr ≈ **5.4 TB/day** |
| ~40 TB at 100% saturation | **~7.4 days** |
| ~40 TB at a realistic 30% allocation | **~25 days**, before delta sync |

You cannot hand a production circuit 100% of its capacity. Options:

1. **Increase circuit bandwidth** — can be done with no downtime, **but cannot be decreased**. Going 500 Mbps → 1 Gbps is a one-way door short of rebuilding the circuit. Same applies to `MeteredData` → `UnlimitedData`.
2. **Replicate over the internet** — leaves the circuit free for production. Requires the `stage` subnet's direct-internet route (§5).
3. **Azure Data Box for seeding** — bulk seed offline, delta-sync over the wire. Best fit for the largest file and SQL servers.
4. **Wave scheduling** — replicate off-hours, throttle during business hours.

Realistically this will be a **blend of 2, 3, and 4**.

### ⚠ No Microsoft peering

The circuit has **`AzurePrivatePeering` only**. Azure Migrate replication targets Azure public endpoints by default, and there is no ExpressRoute path to them. Decide before wave 1:

| Option | Implication |
|---|---|
| Replicate over the internet | Simplest; keeps the circuit clear for production. Needs egress allowed through the firewall or the `stage` direct-internet route. |
| Enable Microsoft peering | Replication rides ExpressRoute — but competes with production for the same 500 Mbps. |
| Private endpoints for Azure Migrate | Keeps replication entirely private over the circuit; most configuration, best alignment with the existing private-endpoint posture. |

### Wave approach

| Wave | Content | Purpose |
|---|---|---|
| **0** | Landing zone + DCs + 2–3 throwaway servers | Prove the EncryptionAtHost fix, DNS, routing, backup, monitoring end to end |
| **1** | Non-prod / low-criticality (~50) | Validate at scale in Development, where `EncryptionAtHost` is already registered |
| **2..n** | Production by application group | Keep app tiers together; migrate dependencies in the same wave |
| **Final** | Tier-1 core banking | Only after DR failover has been tested |

Run **Azure Migrate discovery and dependency analysis first.** The tier distribution in §1 is an assumption; discovery replaces it with fact — and dependency mapping is what stops a wave from severing a link between two servers that had to move together.

---

## 13. Roadmap

| Phase | Work | Gate |
|---|---|---|
| **0. Unblock** | Register `EncryptionAtHost` on Production; validate with a test Windows VM. Submit quota request. | Test VM creates successfully |
| **1. Discover** | Azure Migrate project + appliance; discovery and dependency analysis across all ~200 servers | Real inventory replaces §1 assumptions |
| **2. Foundation** | Both migration spokes, NSGs, route tables, peerings, hub route + BGP updates | On-prem ↔ spoke connectivity proven through the firewall |
| **3. Identity** | Azure DCs deployed and promoted; VNet DNS repointed; AD Site created | Domain join succeeds from the spoke without traversing ExpressRoute |
| **4. Platform** | RSV (GRS), Log Analytics, Bastion, private DNS zones, Datadog/Rapid7 onboarding | Backup and monitoring proven on a test VM |
| **5. Pilot** | Wave 0 — 2–3 non-critical servers, full lifecycle including a restore test | Restore succeeds; monitoring reports |
| **6. Migrate** | Waves 1..n by application group | Per-wave validation and sign-off |
| **7. DR** | ASR to `eastus` for tier-1; documented and **tested** failover | Successful failover test |
| **8. Optimize** | Right-size against actuals, Reserved Instances / Savings Plan, Azure Hybrid Benefit | Cost baseline agreed |

**Azure Hybrid Benefit is worth modelling early** — for ~150 Windows Server and ~25 SQL Server licences it is one of the largest single cost levers available, and it changes the business case.

---

## 14. Risks carried by this design

| Risk | Consequence | Mitigation |
|---|---|---|
| **No availability zones in `westus`** | A single-datacenter fault can affect all 200 servers | Availability Sets (3 FDs) + tested ASR to `eastus`; document the accepted risk |
| **500 Mbps circuit** | Migration measured in weeks | Data Box seeding, internet replication, wave scheduling |
| **Non-zonal, single-instance ER gateway and firewall** | Either is a single point of failure for all 200 servers | Accept and document, or plan zone-redundant replacements in a zonal region later |
| **Firewall SNAT capacity** | 4 public IPs ≈ 9,984 ports ≈ ~50 per server at 200 servers | Monitor SNAT port utilisation; add public IPs or a NAT Gateway before it binds |
| **Firewall Standard, classic rules** | No IDPS/TLS inspection; ruleset will not scale | Migrate to Firewall Policy; decide on Premium explicitly |
| **Blast radius shared with live prod** | Migrated servers sit alongside ASE, APIM, Databricks, SQL | RG-level RBAC, resource locks, separate change windows |
| **Orphan `10.0.0.0/16` VNets** | Collide with each other and with on-prem; can never be peered | Re-address or decommission `PAZWTCUAPP01-vnet` / `QAZWTCUAPP01-vnet` |
| **Lighthouse delegation** | Two external tenants hold rights into Production | Confirm both are current and intended before the estate grows 12× |

---

## 15. Open decisions

1. **Domain controller placement** — hub `tcu-p-usw-exprt-ad-snet-01` (Option A) or spoke `dc` subnets (Option B)?
2. **Replication path** — internet, Microsoft peering, or private endpoints for Azure Migrate?
3. **Circuit bandwidth** — temporarily increase for the migration, accepting that it cannot be reduced afterwards?
4. **Firewall Premium** — upgrade for IDPS/TLS inspection, or document acceptance of Standard?
5. **Prod/non-prod split** — the ~150/~50 assumption needs confirming against discovery output.
6. **Tier-1 definition** — which applications justify ASR to `eastus` versus backup-and-rebuild?
7. **Log Analytics retention** — what does TCU need to evidence under examination?
