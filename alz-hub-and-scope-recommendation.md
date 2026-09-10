# ALZ Hub & Scope — Balanced Recommendation

**Purpose:** Answer two questions ahead of the 200-server migration — *how much of the Azure Landing Zone (ALZ) reference architecture should TCU adopt now*, and *what should the hub look like*.
**Version:** 1.1 — 2026-09-09
**Changed in 1.1:** Azure Lighthouse delegation read in full (§2.5) — an external tenant holds Contributor, Virtual Machine Contributor and **User Access Administrator** at Production subscription scope. This reverses the domain-controller placement (H4) and adds **Identity** and **Management** platform subscriptions to the "adopt now" set. Connectivity and Security remain deferred.
**Author:** `vVashishtha@techcu.com`
**Reference architecture:** `enterprise-scale-architecture.pdf` (ALZ diagram set, version 2026-04-02) — "ALZ Hub & Spoke", "ALZ Virtual WAN", "ALZ Hierarchy", "ALZ Terminology".
**Companion documents:**
- [`existing-infrastructure-assessment.md`](./existing-infrastructure-assessment.md) — verified current state
- [`landing-zone-design.md`](./landing-zone-design.md) — the migration landing zone design this recommendation adjusts

> **This document does not modify either companion.** Where it revises a decision recorded in `landing-zone-design.md`, that is called out explicitly in §5.2 and §9.

---

## 1. The short version

| | Recommendation |
|---|---|
| **Topology** | **Keep hub-and-spoke. Do not adopt Virtual WAN.** |
| **Hub location** | **Leave the hub VNet, ExpressRoute gateway and firewall exactly where they are** — in the Production subscription. Do not attempt an ALZ Connectivity-subscription split before or during the migration. |
| **Hub content** | Adopt the ALZ *Connectivity subscription* **contents** in place: Firewall Policy, central private DNS, DNS Private Resolver, prod Bastion, SNAT headroom, DC placement. |
| **Governance scope** | **Adopt the ALZ management group hierarchy and move policy to MG scope now.** This is free, reversible, and fixes a verified control gap. |
| **Workload scope** | **Land the 200 migrated servers in two new landing zone subscriptions**, not in the existing Production/Development subscriptions. This is the single highest-value ALZ move available and the cheapest. |
| **Platform subscriptions** | **Create Identity and Management now.** Tier-0 assets — domain controllers and the platform log workspace — cannot sit in a subscription an external tenant holds Contributor and User Access Administrator over (§2.5). **Defer Connectivity and Security.** |

The shape of the recommendation is deliberate: **adopt the parts of ALZ that are cheap to do now and expensive to retrofit; defer the parts that are expensive now and no harder later.**

---

## 2. What changed since the assessment

Four items marked `NOT VERIFIABLE` or unexamined on 2026-09-01 were resolved today by querying Azure Resource Graph, which returns management-group ancestry without requiring `Microsoft.Management` permissions.

### 2.1 ✅ The management group hierarchy — now known

`az account management-group list` and direct ARM reads still fail with `AuthorizationFailed`. Resource Graph returns the ancestry chain regardless:

```
Tenant root group  (dcceaaae-204c-482d-9d1c-64b4f3a8fd60)
├── Corp-IT
│   ├── Platform      → Production   f7f18245-3c4e-47f1-b465-e6bd0249f2c6
│   └── Engineering   → Development  029c65f2-4ae1-4e7b-b152-94e909b1d278
└── (no MG)           → Visual Studio Enterprise  7260de92-5746-40b0-9a48-ad1a978ba813
```

A hierarchy **does** exist. Two findings fall straight out of it:

| # | Finding | Severity |
|---|---|---|
| **A** | The MG named **`Platform`** contains the **Production workload subscription**. In ALZ, `Platform` is reserved for Identity / Management / Connectivity / Security subscriptions, and workloads live under `Landing zones`. Any future ALZ policy set assigned to a MG called `Platform` will hit 360 production workload resources instead of platform services. | 🔴 High |
| **B** | The **Visual Studio Enterprise subscription sits directly under the tenant root** with no MG parent, and therefore inherits **no** governance. It holds `NetworkWatcherRG` and a `PolicyTest` RG in `canadacentral` — outside the `westus` estate entirely. | 🟠 Medium |

> ⚠ Resource Graph only returns ancestry for subscriptions this account can see. There may be additional management groups or subscriptions under `Corp-IT` that are not visible here. Confirm the full tree with an account holding `Management Group Reader` before executing §5.1.

### 2.2 🚨 Zero policy is assigned at management group scope

All **30** policy assignments in the tenant sit at **subscription scope**. Not one is at MG scope.

| Scope | Assignments |
|---|---|
| Production `f7f18245` | 10 |
| Development `029c65f2` | 19 |
| Visual Studio Enterprise `7260de92` | 1 |
| **Any management group** | **0** |

Because there is no inherited baseline, the two subscriptions have drifted — and drifted the wrong way. **Development is more tightly controlled than Production.**

| Control | Development | Production |
|---|---|---|
| Allowed locations | ✅ | ❌ **absent** |
| Enable diagnostic settings | ✅ | ❌ **absent** |
| SKU Restriction Policy Assignment | ✅ | ❌ **absent** |
| Linux VMs — Disk Encryption / EncryptionAtHost | ✅ | ❌ **absent** |
| Enforce TLS 1.2 for Azure Services | ✅ | ❌ **absent** |
| Enforce HTTPS only for App Services | ✅ | ❌ **absent** |
| App Service / Kudu SCM TLS ≥ 1.2 (Audit + Deny) | ✅ | ❌ **absent** |
| Tag governance on RG **and** resources | ✅ | ⚠ RG-level cascade only |
| Deny Windows VMs without EncryptionAtHost | ✅ | ✅ |
| `Microsoft.Compute/EncryptionAtHost` feature | **Registered** | **NotRegistered** |

The last two rows are the deadlock already documented in `landing-zone-design.md` §11. The rows above it are new, and they matter more for the ALZ question: **the production environment of a credit union has no `Allowed locations` guardrail and no enforced diagnostic settings, while its dev environment has both.** There is no mechanism in place today that would ever have caught that. MG-scope policy is that mechanism.

`Allowed locations` being absent on Production is directly relevant to the migration: nothing stops 200 servers, or an ASR failover, from landing in an unintended region.

### 2.3 Activity logs do not reach Log Analytics

| Subscription | Activity log diagnostic settings | Destination |
|---|---|---|
| Production | `diag-cslogact-prod` | Event Hub `evh-cslogact-prod-westus` (`evhns-cslog-kz5wah54kdpfi`, RG `rg-cs-prod`) |
| Production | `Rapid7InsightLogs` | Rapid7 |
| Development | `diag-cslogact-prod` | Same Event Hub, cross-subscription |
| Development | `APIM` | Log Analytics `tcu-np-logs` |

Activity logs are being streamed to an Event Hub and to Rapid7 — **but not into any Log Analytics workspace.** ALZ places platform activity logs in a central platform workspace in the Management subscription. Today there is nowhere in Azure to run a KQL query across control-plane activity.

`rg-cs-prod` contains only the Event Hub namespace and a managed identity `id-csscriptrunner-prod` — consistent with one of the two Azure Lighthouse delegations into Production (finding 13 of the assessment). **Confirm which delegated tenant owns this pipeline and that it is intended** before it becomes the evidence path for 200 more servers.

### 2.4 Other platform facts verified today

| Item | State |
|---|---|
| **Log Analytics workspaces** | **5**, all at **30-day** retention: `tcu-logs`, `tcu-cube-logs` (Prod); `tcu-np-logs`, `tcu-generic-devops-logs`, `DefaultWorkspace-029c65f2…` (Dev). No central platform workspace. |
| **Microsoft Sentinel** | **Not deployed anywhere in the tenant.** |
| **Defender for Cloud** | ✅ **Standard tier on 10 plans in both subscriptions**, consistently — VMs, SQL, SQL-on-VM, App Services, Storage, AKS, ACR, Key Vault, Discovery, FoundationalCspm. This is the strongest part of the current security posture. |
| **Virtual WAN** | None. |
| **Azure Firewall Policy** | None — firewall still on classic rules (6 network collections, 2 application, 4 NAT). |
| **DDoS Protection Plan** | None. |
| **NAT Gateway** | None. |
| **DNS Private Resolver** | None. |
| **Azure Bastion** | One, **Basic SKU**, Development only. |
| **ExpressRoute circuit authorizations** | **None** (`[]`). The circuit is currently usable only by the gateway in the Production subscription. |
| **ER gateway** | `Standard` tier, capacity 2, `activeActive: false`, **non-zonal**, 1 IP configuration. |
| **GatewaySubnet** | `10.209.0.16/28` — **the minimum size**. See §6.4. |
| **Resource counts** | Production 360 resources / 7 VMs; Development 576 / 9 VMs. |
| **Billing account** | ⚠ **Not verifiable with this account.** The only billing account visible is `Varadan` (`MicrosoftOnlineServicesProgram`) in a *different* tenant — this is the operator's personal account, not TCU's enrollment. **TCU's agreement type and subscription-creation rights are an open prerequisite for §5.2.** |

### 2.5 🚨 An external tenant holds Contributor and User Access Administrator over Production

Assessment finding 13 recorded two `managedByTenants` entries on Production and flagged them "worth confirming." The delegation has now been read in full, and it is the single most consequential fact in this document.

**`Softchoice LP (Shared Services)`** — tenant `d69b3567-877c-4286-be00-05eb399ef140` — holds the following at **Production subscription scope**, via the Lighthouse registration `Softchoice Services`:

| Role | Role definition ID | Delegated principal groups |
|---|---|---|
| **Contributor** | `b24988ac-6180-42a0-ab88-20f7382dd24c` | Softchoice Service Contributors; Softchoice Partner Management (USA) |
| **Virtual Machine Contributor** | `9980e02c-c2be-4d73-94e8-173b1dc7cf3c` | Softchoice Service Virtual Machine Contributors |
| **User Access Administrator** | `18d7d88d-d35e-4fb5-a5c3-7773c20a72d9` | Softchoice Service Automation and Management |
| Reader | `acdd72a7-3385-48ef-bd42-f606fba81ae7` | Service Readers; Monitoring and Events; Dashboard and Insights |

This is a legitimate and common managed-service arrangement. It becomes a problem the moment **tier-0 assets are placed inside that subscription** — which is exactly what `landing-zone-design.md` §7 proposes by putting domain controllers in the hub.

**What these roles do to a domain controller** — read from the live role definitions, not assumed:

| Capability | Mechanism |
|---|---|
| **Code execution as SYSTEM on a DC** | Virtual Machine Contributor carries `Microsoft.Compute/virtualMachines/*`, which includes `runCommand/action`. |
| **Offline extraction of every domain credential** | Contributor's `notActions` exclude only `Microsoft.Authorization/*` and a handful of unrelated operations — **snapshots are not excluded.** Snapshot a DC's OS disk, export it, read `NTDS.dit` offline. That yields every password hash in the domain, `krbtgt` included, and with it Golden Ticket capability. |
| **Self-escalation** | User Access Administrator can grant itself, or any other principal, any role at any scope beneath the subscription. |

#### Why a resource-group boundary does not help

**Azure RBAC is additive.** A role assignment at subscription scope inherits to every resource group and resource beneath it, and **cannot be subtracted at a lower scope.** The only subtractive mechanism in Azure is a deny assignment, which is not practically available (Azure Blueprints, its delivery vehicle, is deprecated).

So "place the domain controllers in a hardened resource group with tight RBAC" is **not a control** against these principals. Nor are resource locks — a `CanNotDelete` lock does not block `runCommand` or a disk snapshot. **The subscription is the boundary, and there is no smaller one.**

> ⚠ **The second delegated tenant is still unaccounted for.** `2f4a9838-26b7-47ee-be60-ccc1fdec5953` appears in `managedByTenants` but returns no readable registration definition with this account. Its granted roles are **unknown** and it may be scoped to a resource group rather than the subscription. Resolve this before G2 — it may be a second instance of the same exposure.

**Two ways to fix it.** They are not mutually exclusive, and the first does not depend on anyone outside TCU:

1. **Do not put tier-0 assets in Production** — create Identity and Management subscriptions (§5.2). Fast, unilateral, and the basis of this document's recommendation.
2. **Narrow the delegation itself** — scope Softchoice to resource groups rather than the subscription, and drop User Access Administrator. Architecturally the better fix and it addresses the root cause, but it is a commercial negotiation with a managed service provider and is therefore slower and not solely TCU's decision.

---

## 3. The two questions, framed

### Question 1 — Scope
> How much of the ALZ reference architecture does TCU adopt, and where do the 200 servers land?

ALZ prescribes an intermediate root MG, a `Platform` branch with **four** dedicated subscriptions (Identity, Management, Connectivity, Security), a `Landing zones` branch (Corp / Online), plus `Sandbox` and `Decommissioned`. TCU today has three subscriptions, two MGs, and no MG-scope policy.

### Question 2 — Hub
> Does the existing hub stay, change shape, or get rebuilt?

ALZ offers two connectivity patterns — **hub-and-spoke** (customer-managed hub VNet) and **Virtual WAN** (Microsoft-managed hub). TCU has a working, correctly-configured hub-and-spoke with gateway transit, symmetric firewall inspection, and disabled BGP propagation on server subnets. It is architecturally sound; it is simply under-equipped for 12× the server count.

These two questions are coupled, because ALZ's answer to "where does the hub live" is "in a dedicated Connectivity subscription" — which is a scope decision, not a network one.

---

## 4. Three postures, scored

| | **Posture 1 — Reuse as-is** | **Posture 2 — Balanced (recommended)** | **Posture 3 — Full ALZ** |
|---|---|---|---|
| MG hierarchy | Unchanged | ALZ-aligned under existing `Corp-IT` | ALZ-aligned, new intermediate root |
| Policy | Stays at subscription scope | **Moves to MG scope** | MG scope + full ALZ policy set (~200 definitions) |
| Servers land in | Existing Prod / Dev subs | **Two new landing zone subs** | Two new landing zone subs |
| Hub VNet | Stays in Production sub | **Stays in Production sub** | Moves to new Connectivity sub |
| ER gateway | Unchanged | **Unchanged** | Rebuilt in Connectivity sub + circuit authorization + cutover |
| Firewall | Classic rules | **Firewall Policy (+ Premium decision)** | Firewall Policy hierarchy, parent policy at platform scope |
| Platform subs | None | **Deferred, address space reserved** | Identity + Management + Connectivity + Security created now |
| Sentinel | No | Deferred, decision recorded | Yes, in Security subscription |
| **Delay to wave 1** | none | **~2–3 weeks, parallelisable** | **~3–4 months, serialised** |
| **Production outage required** | none | **none** | **yes** — ER gateway cutover |
| **Fixes the Prod/Dev control gap** | ❌ | ✅ | ✅ |
| **Fixes quota / blast-radius coupling** | ❌ | ✅ | ✅ |

### Why not Posture 1

Three verified reasons, in order of weight:

1. **Quota is a subscription-scoped ceiling.** `Total Regional vCPUs` on Production is **21 / 280**; the migration needs roughly **1,140**. Posture 1 means raising that ceiling to ~1,500 on the same subscription that runs ASE, APIM, ISE, Application Gateway, Databricks and `pazwsqldb01` — putting core-banking PaaS and 200 migrated servers into one quota pool with no isolation between them.
2. **Production has weaker policy than Development** (§2.2), and nothing at subscription scope will ever converge them. Adding 200 servers to that subscription adds 200 servers to the gap.
3. **Blast radius.** RBAC, resource locks, change windows, Defender alerting and cost attribution would all be shared between the migrated estate and live customer-facing services. The `cost center` tag is empty on every resource group sampled, so there is no compensating control for chargeback either.

### Why not Posture 3

1. **The ER gateway cannot be moved.** A VNet with a gateway and active peerings cannot be transferred between subscriptions. Posture 3 means building a *second* ER gateway in a new Connectivity subscription, creating a circuit authorization (there are **none** today), linking it, re-peering every spoke, re-pointing every UDR at a new firewall private IP, and cutting over — a planned production outage on **the single path to on-premises**, for an estate whose DNS already depends entirely on that path reaching `10.71.4.x`.
2. **It delivers nothing to the migration.** Every migration blocker — EncryptionAtHost, quota, 500 Mbps, DNS, no availability zones — is untouched by a Connectivity-subscription split.
3. **It is not harder later.** Once the estate is stable and the firewall runs Firewall Policy, the split becomes a scheduled platform change rather than a change competing with a migration. Deferring costs almost nothing; doing it now costs a production window and 3–4 months.

---

## 5. Recommendation A — Scope

### 5.1 Management group hierarchy

Build the ALZ tree **beneath the existing `Corp-IT` MG**. `Corp-IT` becomes the intermediate root (`Contoso` in the reference diagram). No new tenant-root child is required, which keeps the change small and avoids a tenant-root-scope permission request.

```
Tenant root group
└── Corp-IT                                  ← intermediate root (exists)
    ├── Platform                             ← RENAME the existing MG; see note
    │   ├── Identity        → tcu-plat-identity  ★ NEW — domain controllers
    │   ├── Management      → tcu-plat-mgmt      ★ NEW — platform Log Analytics
    │   ├── Connectivity                     (empty — hub stays in Production, §6.2)
    │   └── Security                         (empty — follows Sentinel, §7)
    ├── Landing zones
    │   └── Corp
    │       ├── Prod        → Production  f7f18245   (existing estate)
    │       │               → tcu-mig-prod  ★ NEW    (migrated servers)
    │       └── NonProd     → Development 029c65f2   (existing estate)
    │                       → tcu-mig-nonprod ★ NEW  (migrated servers)
    ├── Sandbox            → Visual Studio Enterprise 7260de92
    └── Decommissioned      (empty — for post-cutover source retirement)
```

**Deliberate choices:**

- **The existing `Platform` MG is renamed, not reused.** It currently holds the Production workload subscription. Rename it to `Corp` (or create `Landing zones/Corp/Prod`, move the subscription there, then repurpose the name). Doing this now, with three subscriptions, is a ten-minute operation with zero resource impact. Doing it after 200 servers land — with policy assigned to it — is a change-controlled event.
- **`Engineering` similarly becomes `Landing zones/Corp/NonProd`.** The name carries an org-chart meaning, not a platform meaning; ALZ MGs should reflect *policy requirements*, not team structure. If `Engineering` has meaning to the business, keep it as an RBAC group name, not an MG.
- **`Identity` and `Management` get real subscriptions immediately; `Connectivity` and `Security` are created empty.** The split is driven by §2.5, not by ALZ completeness — see §5.2. Creating the two empty MGs anyway costs nothing, makes the deferred phase 2 a subscription *move* rather than a hierarchy redesign, and gives the platform policy set a home from day one.
- **`Platform` needs a stricter policy baseline than `Landing zones`.** Once domain controllers live under it, assign the tier-0 controls there and nowhere else: deny public IP on NICs, deny inbound RDP/SSH from Internet, deny VM extensions outside an allow-list, deny snapshot/disk export, and require Bastion-only access. These are precisely the operations §2.5 identifies as the exposure — policy is the second layer behind the subscription boundary, not a substitute for it.
- **No `Online` MG.** ALZ splits `Corp` (connected to on-prem) from `Online` (internet-facing, no corporate connectivity). Every TCU workload is corp-connected. Add `Online` only if a genuinely isolated internet-facing workload appears.
- **`Decommissioned` matters here specifically.** 200 servers are being *retired* on-prem; if any source workload is lifted into Azure and then wound down, `Decommissioned` is where the subscription goes to have policy stripped and spend stopped.

> Moving a subscription between management groups does not touch resources, requires no downtime, and is reversible. It requires `Management Group Contributor` on both source and destination.

### 5.2 Subscription topology — the highest-value move

> **This revises a decision in `landing-zone-design.md` §1**, which selected *"Existing `Production` and `Development` — no new subscription boundary."*

**Recommendation: create four new subscriptions — two landing zone, two platform.**

| New subscription | Holds | ALZ role | Driver |
|---|---|---|---|
| `tcu-mig-prod` | `tcu-p-usw-mig-vnet-01` (`10.212.0.0/16`) + ~150 production servers | Landing zone — Corp / Prod | Quota, blast radius, EncryptionAtHost |
| `tcu-mig-nonprod` | `tcu-np-usw-mig-vnet-01` (`10.213.0.0/16`) + ~50 non-production servers | Landing zone — Corp / NonProd | Same |
| **`tcu-plat-identity`** | `tcu-p-usw-id-vnet-01` (`10.217.0.0/24`) + 2 domain controllers | **Platform — Identity** | **§2.5 — tier-0 RBAC boundary** |
| **`tcu-plat-mgmt`** | Central platform Log Analytics workspace (no VNet initially) | **Platform — Management** | **§2.5 — log integrity** |

**Nothing else in `landing-zone-design.md` changes.** The address plan, the `/21`-per-tier subnet layout, the routing model, the NSG rules, the naming convention, the RG structure and the wave plan are all carried over verbatim. Only the subscription the spokes live in changes.

#### 5.2.1 Why Identity and Management, but not Security or Connectivity

The two platform subscriptions are **not** recommended for ALZ completeness. They exist to answer one question: *which assets about to be created are tier-0, and are they landing inside a subscription an external tenant holds Contributor and User Access Administrator over?* (§2.5)

| Platform subscription | Verdict | Reasoning |
|---|---|---|
| **Identity** | ✅ **Create now** | The design places **two domain controllers** in Azure. Under §2.5, subscription-scope Virtual Machine Contributor grants `runCommand` on them, and Contributor can snapshot and export a DC's OS disk for offline `NTDS.dit` extraction — full domain credential compromise including `krbtgt`. No RG-level control mitigates an inherited subscription-scope assignment. **A separate subscription is the only available boundary.** |
| **Management** | ✅ **Create now** | The central platform Log Analytics workspace (§8, G3) has the identical exposure: a subscription-scope Contributor can delete it or reduce its retention — the workspace whose whole purpose is evidencing what happened. Separately, **a Log Analytics workspace is genuinely painful to relocate once ~200 agents are reporting to it**, so its home should be decided before wave 1, not after. |
| **Security** | ⏸ **Defer** | ALZ's Security subscription exists to host **Sentinel** and the security workspace. TCU has no Sentinel (§2.4), so it would be an empty container. Meaningful separation already exists in a stronger form — activity logs stream to an Event Hub and Rapid7, **outside Azure RBAC entirely** (§2.3). Until Sentinel is funded, the security workspace lives in `tcu-plat-mgmt` with its own RBAC. Create Security when Sentinel is created, not before. |
| **Connectivity** | ⏸ **Defer** | Unchanged from §4/§6.2. Requires a second ExpressRoute gateway, a circuit authorization and a production cutover on the only path to on-prem. The hub is not a tier-0 credential store, so §2.5 does not apply with the same force — and locks plus RG-scoped RBAC (§6.2) are proportionate here even though they are not for DCs. |

#### 5.2.2 The constraint this introduces

**A virtual machine must live in the same subscription as its virtual network.** Consequently, moving the domain controllers into `tcu-plat-identity` means **they cannot use the empty hub AD subnet** `tcu-p-usw-exprt-ad-snet-01`. They need their own VNet.

| Item | Value |
|---|---|
| Identity VNet | `tcu-p-usw-id-vnet-01`, **`10.217.0.0/24`** |
| DC subnet | `tcu-p-usw-id-dc-snet-01`, `10.217.0.0/26` |
| Platform block reserved | **`10.217.0.0/16`** — so future platform VNets carve from it without a further on-prem BGP change |
| Peering | To hub, `useRemoteGateways: true` (hub side `allowGatewayTransit: true`, already set) |
| Routing | Identical to the spokes — `0.0.0.0/0` → `10.209.1.4`, `disableBgpRoutePropagation: true` |
| Hub AD subnet | Left empty, or repurposed. It is no longer the DC target. |

This reverses **H4 / Option A** in §6.3 and closes `landing-zone-design.md` §7 in favour of a **third** option that document did not consider: DCs in a dedicated identity VNet, in a dedicated subscription, peered to the hub.

**Cost of the reversal is small.** One extra VNet, one peering, one route table — and Defender for Cloud bills per resource, not per subscription, so two DCs cost the same wherever they sit. Against that: the DCs stop being reachable by an external tenant's standing Contributor.

**Why this is the right trade:**

| Argument | Detail |
|---|---|
| **Quota isolation** | `Total Regional vCPUs` is a per-subscription, per-region cap. A dedicated subscription gets its own pool. The ~1,500 vCPU request lands on a subscription with nothing else in it, instead of on the one running core banking. A quota request is required either way — this changes *which* ceiling moves, not *whether* one does. |
| **Escapes the EncryptionAtHost deadlock cleanly** | The deny policy and the `NotRegistered` feature are both **subscription-scoped artefacts of Production**. A new subscription starts clean: register `Microsoft.Compute/EncryptionAtHost` **first**, then inherit the same deny control from MG scope. You keep the control and lose the deadlock — without weakening, exempting or disabling a legitimate encryption policy on Production. |
| **Blast radius** | Independent RBAC, resource locks, change windows and Defender scope. The migrated estate cannot take out ASE, APIM or Databricks, and vice versa. |
| **Chargeback** | A subscription boundary gives cost attribution immediately, which matters because the `cost center` tag is empty on every RG sampled. Fix the tags anyway (§10) — but do not depend on them for the migration's cost story. |
| **Cost of the change** | **Zero.** Subscriptions carry no charge. No data moves. No downtime. |
| **Capability already required** | The current design already peers cross-subscription (hub in Production, non-prod spoke in Development). Nothing new is needed. |

**What it costs:**

| Cost | Mitigation |
|---|---|
| Defender for Cloud plans must be enabled on 4 new subscriptions | Defender **bills per resource, not per subscription**, so the two platform subscriptions add almost nothing — the cost sits with the ~200 migrated servers and would exist wherever they landed. Enforce enablement via MG-scope DINE policy so a new subscription cannot be missed. |
| Azure Migrate must target the new subscription | Configure once, before wave 0. Cheaper than re-targeting after wave 1. |
| **Azure Lighthouse delegations do not follow to new subscriptions** | **This is the point, not a side effect** (§2.5). The migrated estate and the domain controllers start outside Softchoice's delegated scope. Re-delegate deliberately and narrowly *if* the managed service is required there — do not re-create a subscription-scope Contributor + User Access Administrator grant by default. |
| An extra VNet and peering for the identity subscription | §5.2.2. One VNet, one peering, one route table. |
| More subscriptions to govern | Precisely why §5.3 moves policy to MG scope first. |
| 🚨 **Subscription creation rights are unverified** | The billing account visible to this operator is personal, not TCU's (§2.4). **Confirm agreement type and creation rights before committing to this path.** |

**Fallback if subscription creation is blocked.** If TCU's agreement or approval process makes new subscriptions impractical inside the migration timeline, fall back to `landing-zone-design.md` as written — the migrated estate lands in the existing subscriptions — but then §5.3 (MG-scope policy) and §6 (hub work) become **more** important, not less, because the subscription boundary is no longer available as a control. Reserve the subscription split as a phase-2 item alongside the Connectivity split.

### 5.3 Policy — move the baseline to MG scope

This is free, reversible, and closes the gap in §2.2.

| Step | Action |
|---|---|
| 1 | **Build the corp baseline at `Corp-IT`.** Everything both subscriptions should have: `Allowed locations` (**westus, westus2, eastus** — eastus for ASR), disk encryption / EncryptionAtHost for Windows **and** Linux, TLS 1.2 enforcement, Key Vault firewall, managed-disk and snapshot public-access denial, storage infrastructure encryption, Guest Configuration prerequisites, `Enable diagnostic settings`. |
| 2 | **Reconcile the drift**, using Development's 19 assignments as the starting inventory — it is the stricter of the two. Every control Development has and Production lacks is a candidate for the baseline. |
| 3 | **Assign in `DoNotEnforce` first** at `Corp-IT`, read the compliance result across all 360 + 576 existing resources, then flip to `Default`. Assigning deny policies enforced-first across a live estate of 936 resources is how you break a production change window. |
| 4 | **Retire the duplicated subscription-scope assignments** once the MG baseline is enforcing, leaving only genuinely subscription-specific ones. |
| 5 | **Landing-zone-specific policy at `Landing zones/Corp/Prod`** — the migrated estate's rules (required tags including `cost center` and `migration-wave`, allowed VM SKUs, mandatory backup, mandatory Availability Set membership). |
| 6 | **Keep `Sandbox` loose but bounded** — `Allowed locations` and a cost budget, nothing more. The `PolicyTest` RG in `canadacentral` is exactly what a Sandbox MG is for. |

> ⚠ **`Allowed locations` needs `eastus` in the allow-list** before ASR replication is configured (`landing-zone-design.md` §8). A location policy that omits the DR region silently blocks failover — and you discover it during the failover test, or worse, during a real one.

**Do not adopt the full ALZ policy set (~200 definitions) in one step.** It is designed for greenfield. Retrofitting it across 936 existing resources will generate a compliance report nobody can act on and will stall the migration. Take the ~15 controls above, prove the mechanism, and expand after cutover.

---

## 6. Recommendation B — Hub

### 6.1 Hub-and-spoke, not Virtual WAN

The reference architecture offers both. **Hub-and-spoke is the correct choice for TCU, and it is not close.**

| Factor | TCU reality | Verdict |
|---|---|---|
| Regions | 1 (`westus`), + `eastus` for DR | vWAN's core value is many-region transit. Not applicable. |
| ExpressRoute circuits | 1, single peering location (Silicon Valley) | vWAN's multi-circuit routing intent adds nothing. |
| Spokes | 3 today, 5 after migration | Manual peering is entirely manageable at this count. |
| Branch / SD-WAN | **Already solved by Cato Networks** — `vSocket-Infra-CATO`, `vsNet` in westus2, `10.41.0.0/16` routed via Cato | vWAN's branch-connectivity value is already delivered by an incumbent SASE platform. |
| Shared services in the hub | DCs planned for `tcu-p-usw-exprt-ad-snet-01`; `sharedsvcs` subnet in use | 🚫 **A vWAN hub is a Microsoft-managed VNet. You cannot deploy domain controllers or arbitrary VMs into it.** This alone disqualifies vWAN without a redesign of `landing-zone-design.md` §7. |
| Existing routing model | BGP propagation disabled + `0.0.0.0/0` → `10.209.1.4`, symmetric inspection | vWAN replaces this wholesale with routing intent. A working, understood model would be discarded for no gain. |
| Migration risk | Already gated on 3 blockers | vWAN migration means a new ER connection, a new firewall, and every UDR rewritten. |

**Record this as a decision, not an omission.** vWAN is the right answer for a multi-region, many-branch estate; TCU is neither. If TCU later expands to a second production region or retires Cato, revisit.

### 6.2 The hub stays where it is

Keep `tcu-p-usw-exprt-vnet-01` (`10.209.0.0/16`) in the Production subscription, with its existing ER gateway, firewall and peerings. Do not move it, do not rebuild it, do not add a second hub in `westus`.

**But bound it as if it were already a Connectivity subscription.** This is the mechanism that makes the deferred split cheap:

| Control | Action |
|---|---|
| **RBAC** | `tcu-p-usw-exprt-rg-01` gets its own role assignments — Network Contributor for the platform team only. No workload team, and no migration team, holds write on the hub RG. |
| **Locks** | `CanNotDelete` on the ExpressRoute circuit, the gateway, the connection, the firewall, the hub VNet and the four firewall public IPs. |
| **Hygiene** | Move `tcu-np-usw-azlos-temp-pip-01` off the production firewall, or rename it. A non-prod-named, temp-named public IP on the production perimeter is an audit finding waiting to happen (assessment finding 14). |
| **Change process** | Hub changes follow a platform change window, separate from migration waves. §5 of `landing-zone-design.md` is explicit that the migration asks almost nothing of the hub — hold that line. |
| **Naming** | Treat `tcu-p-usw-exprt-rg-01` as the Connectivity boundary. Everything the ALZ Connectivity subscription would hold goes in this RG and nowhere else. |

When phase 2 arrives, the split becomes "move one well-bounded resource group's worth of function into a new subscription", not "work out which of 360 resources are platform".

### 6.3 Hub work items — in priority order

| # | Item | Why | Effort |
|---|---|---|---|
| **H1** | 🚨 **Migrate the firewall to Firewall Policy** | Prerequisite for *everything else* — Premium, IP Groups, rule hierarchy, and any future parent policy at platform scope. Classic rule collections (47 network + 4 application rules) will not survive a 200-server ruleset. Migrate the existing rules into a policy, verify parity, then attach. | Medium |
| **H2** | 🔴 **Decide Standard vs Premium — explicitly** | Standard has **no IDPS and no TLS inspection**. Threat intel is `Alert`-only. For a credit union under FFIEC/GLBA adding 200 servers, this needs a recorded decision with a named owner, not a default. **Recommend Premium for the production hub.** Requires a Premium *policy* (a Standard policy cannot attach to a Premium firewall) and a firewall stop/start — **plan a maintenance window**. | Medium, + recurring cost |
| **H3** | 🔴 **SNAT port headroom** | 4 public IPs ≈ **9,984 SNAT ports**. At 200 servers that is ~50 ports per server. Fine at 16 VMs; a hard wall at 200 — and SNAT exhaustion presents as intermittent, unexplained connection failures, the worst possible failure mode mid-migration. **Either add public IPs (linear, simple) or attach a NAT Gateway to `AzureFirewallSubnet` (64k ports per IP).** NAT Gateway integration is incompatible with zone-redundant firewalls — this firewall is non-zonal, so it is available; **verify against current product support before committing.** Instrument SNAT port utilisation *before* wave 1 either way. | Low–Medium |
| **H4** | 🔴 **Domain controllers → dedicated identity VNet in `tcu-plat-identity`, *not* the hub** | ⚠ **Revised.** The hub AD subnet `tcu-p-usw-exprt-ad-snet-01` is the intuitive home — it exists, it is empty, and it was provisioned for exactly this. **Do not use it.** Placing DCs in the hub places them in the Production subscription, where an external tenant holds Contributor, Virtual Machine Contributor and User Access Administrator (§2.5) — meaning `runCommand` on a DC and disk-snapshot export of `NTDS.dit`. Because a VM must share a subscription with its VNet, the DCs need their own VNet: `tcu-p-usw-id-vnet-01` (`10.217.0.0/24`), peered to the hub, same routing model (§5.2.2). Two DCs, different fault domains, static IPs. | Medium |
| **H5** | 🟠 **One Bastion in the hub, Standard SKU** | `landing-zone-design.md` §4 places `AzureBastionSubnet` in each migration spoke. **A single Bastion in the hub serves every peered spoke** — one deployment, one set of logs, one audit surface, less cost. Use **Standard**, not Basic (the Dev Bastion is Basic): Standard adds host scaling, native client and IP-based connection, all of which matter at 200 servers. Production has no Bastion today and reaches prod via the `pazwazrjb01` jumpbox. | Low |
| **H6** | 🟠 **Centralise private DNS zones in the hub RG** | Only two zones exist (`privatelink.blob…`, `privatelink.database…`), both in `tcu-p-usw-network-rg-01` — the *prod spoke's* network RG, not the hub's. Build the rest (Key Vault, storage file/queue/table, Recovery Services, Monitor) **in `tcu-p-usw-exprt-rg-01`** and link every VNet. This matches the ALZ Connectivity subscription and makes the phase-2 split clean. | Low |
| **H7** | 🟠 **Azure DNS Private Resolver in the hub** | Complements the DCs rather than replacing them. Handles private-link zone resolution and gives on-prem a supported conditional-forwarding target that is not a domain controller. Reduces the estate-wide dependency on `10.71.4.x` being reachable for *Azure* name resolution (assessment finding 4). Needs a delegated subnet, `/28` minimum. | Low–Medium |
| **H8** | 🟡 **DDoS Network Protection — decide and record** | ALZ places a DDoS plan in the Connectivity subscription. It is a material monthly cost, and TCU's internet exposure is limited (firewall PIPs, AGW). **Recommend: decline for now, record the rationale, revisit if internet-facing exposure grows.** An explicit "no" is an audit-defensible answer; silence is not. | Decision only |
| **H9** | 🟡 **Reserve capacity for ER + VPN coexistence** | See §6.4. | Planning |

### 6.4 Hub address plan — additions and one constraint

The hub is at `10.209.0.0/16` with six subnets. Utilisation is trivial; there is ample room.

| Subnet | Prefix | Status |
|---|---|---|
| `GatewaySubnet` | `10.209.0.16/28` | ⚠ **In use — see constraint below** |
| `AzureFirewallSubnet` | `10.209.1.0/24` | In use |
| `tcu-p-usw-exprt-mgmt-snet-01` | `10.209.2.0/24` | In use |
| `tcu-p-usw-exprt-ad-snet-01` | `10.209.3.0/24` | ⚠ **Empty — and stays empty.** No longer the DC target (H4, §5.2.2). Repurpose or leave reserved. |
| `tcu-p-usw-exprt-containers-snet-01` | `10.209.4.0/24` | In use |
| **`AzureBastionSubnet`** | **`10.209.5.0/26`** | ★ Proposed (H5) |
| **`tcu-p-usw-exprt-dnspr-in-snet-01`** | **`10.209.6.0/28`** | ★ Proposed — DNS Private Resolver inbound, delegated `Microsoft.Network/dnsResolvers` (H7) |
| **`tcu-p-usw-exprt-dnspr-out-snet-01`** | **`10.209.6.16/28`** | ★ Proposed — outbound endpoint, if conditional forwarding to on-prem is required (H7) |
| *reserved* | `10.209.7.0/24` | ★ Hold for `AzureFirewallManagementSubnet` (needed only if forced tunnelling is ever adopted) |
| `tcu-p-usw-exprt-sharedsvcs-snet-01` | `10.209.8.0/24` | In use |
| *free* | `10.209.9.0/24` – `10.209.255.0/24` | Available |

#### ⚠ Constraint — `GatewaySubnet` is `/28`, the minimum

`/28` is the smallest Azure permits and it is fully consumed by the ExpressRoute gateway. Two consequences:

1. **ExpressRoute + VPN gateway coexistence in the hub requires `/27` minimum (`/26` recommended).** A site-to-site VPN as ExpressRoute backup is the standard mitigation for "the 500 Mbps circuit is a single point of failure for all 200 servers and for estate-wide DNS" — and at `/28` that door is closed.
2. `10.209.0.0/28` is **free and adjacent**, so `10.209.0.0/27` is addressable on paper. In practice, resizing a `GatewaySubnet` with a gateway deployed is generally not supported — **assume it requires a gateway rebuild and a maintenance window; verify against current product behaviour before planning.**

**Recommendation:** do not attempt this during the migration. Record it as a known constraint on the ER-backup option, and fold the resize into whichever future change already rebuilds the gateway — the phase-2 Connectivity split is the natural candidate, since that rebuilds the gateway anyway.

### 6.5 The missing hub — `eastus` DR

`landing-zone-design.md` §8 makes ASR to `eastus` the entire resilience story for a region with no availability zones, and calls it "the item most likely to draw examiner attention." **That story has no hub.** A failed-over tier-1 VM in `eastus` needs a VNet, on-prem connectivity, DNS, and inspected egress. None exists.

| Item | Recommendation |
|---|---|
| **Circuit capacity** | ✅ **No new circuit, no Premium upgrade required.** The Standard SKU reaches every region in the same geopolitical area, and the circuit is at **1 of 10** VNet connections. An `eastus` hub connects to the *same* circuit via a second gateway. |
| **Circuit authorization** | There are **none** today. If the DR hub lands in a different subscription, it needs one. Trivial to create; easy to forget until failover day. |
| **Address space** | Reserve **`10.215.0.0/16`** (DR hub) and **`10.216.0.0/16`** (DR spoke) now. Both are inside the verified-free `10.215`–`10.222` band. Avoid `10.214` (on-prem stray `10.214.65.64/32`) and `10.223` (on-prem `10.223.4.0/24`). |
| **On-prem BGP** | Add `10.215.0.0/16` and `10.216.0.0/16` to the same BGP acceptance change already required for `10.212`/`10.213`, **plus `10.217.0.0/16` for the platform block** (§5.2.2). **One change, five prefixes — do it once.** |
| **Zones** | `eastus` **has** availability zones. The DR hub's gateway and firewall can be zone-redundant even though the primary's cannot. Worth stating plainly to examiners: the DR posture is *more* resilient than the primary. |
| **Build timing** | Design and reserve now; build at phase 7 of the existing roadmap, before the failover test — not before wave 1. |

> **Reserving the address space is the urgent part.** Everything else can wait. If `10.215`/`10.216` get consumed by something else in the interim, the DR design starts with a re-addressing exercise.

---

## 7. Explicitly deferred or declined

Recording these as decisions matters as much as the recommendations. An examiner asking "why is there no Sentinel?" needs an answer better than *nobody considered it*.

| Item | Decision | Rationale | Revisit |
|---|---|---|---|
| **Virtual WAN** | ❌ Declined | Single region, single circuit, few spokes; Cato already covers branch/SD-WAN; a managed hub cannot host the planned DCs. | If a second production region appears, or Cato is retired. |
| **Connectivity subscription** | ⏸ Deferred | Requires a second ER gateway, a circuit authorization and a production cutover on the only path to on-prem. Delivers nothing to the migration. No harder later. The hub is not a tier-0 credential store, so RG-scoped RBAC and locks (§6.2) are proportionate here. | Phase 2, post-cutover. Bundle with the `GatewaySubnet` resize. |
| **Identity subscription** | ✅ **Adopted now** | Reversed from an earlier draft. §2.5 — domain controllers cannot sit in a subscription an external tenant holds Contributor, VM Contributor and User Access Administrator over. No RG-level control mitigates inherited subscription-scope RBAC. | — |
| **Management subscription** | ✅ **Adopted now** | Same RBAC argument applied to the platform Log Analytics workspace, plus a workspace is painful to relocate once ~200 agents report to it. | — |
| **Security subscription** | ⏸ Deferred | It exists to host Sentinel, and there is no Sentinel. An empty container adds governance surface for no control. Security workspace lives in `tcu-plat-mgmt` meanwhile. | Create it when Sentinel is funded — same change. |
| **Microsoft Sentinel** | ⏸ Deferred, **decision required** | Not deployed anywhere. Defender for Cloud is at Standard on 10 plans in both subscriptions and is doing real work; Rapid7 and Datadog are already ingesting, and activity logs already leave Azure RBAC via Event Hub. Sentinel is a significant recurring cost and a real project. **But 30-day Log Analytics retention is short of what FFIEC/GLBA examination typically expects, and that gap is independent of Sentinel.** | **Fix retention now** (§8, G3). Decide on Sentinel as a separate, funded workstream. |
| **Full ALZ policy set (~200 definitions)** | ❌ Declined for now | Designed for greenfield. Retrofitting across 936 existing resources produces an unactionable compliance report and stalls the migration. | Expand incrementally from the ~15-control baseline after cutover. |
| **DDoS Network Protection** | ❌ Declined, recorded | Material monthly cost against limited internet exposure. | If internet-facing footprint grows. |
| **Subscription vending / ALZ IaC accelerator** | ⏸ Deferred | Four subscriptions do not justify a vending platform. Deploy the MG hierarchy and policy as code regardless, so it is reproducible and reviewable. | If the estate passes ~10 subscriptions. |
| **`Online` landing zone MG** | ❌ Not created | Every TCU workload is corp-connected. | If a genuinely isolated internet-facing workload appears. |

---

## 8. Sequenced plan

Mapped onto the roadmap already in `landing-zone-design.md` §13. **Nothing here moves wave 1 to the right** — the governance work runs in parallel with discovery, which is the longest pole regardless.

| Phase | ALZ work | Runs alongside | Gate |
|---|---|---|---|
| **G0 — Verify** | Confirm the full MG tree with `Management Group Reader`. Confirm TCU's billing agreement and subscription-creation rights. **Identify the second delegated tenant `2f4a9838…` and read its granted roles (§2.5).** Confirm ownership of the `rg-cs-prod` Event Hub pipeline. | Roadmap phase 0 (unblock) | Full tree known; subscription path confirmed or fallback (§5.2) invoked; **both delegations fully enumerated** |
| **G1 — Hierarchy** | Build `Platform/{Identity,Management,Connectivity,Security}`, `Landing zones/Corp/{Prod,NonProd}`, `Sandbox`, `Decommissioned` under `Corp-IT`. Rename `Platform`→`Corp` and `Engineering`. Move all three existing subscriptions into place. | Roadmap phase 1 (discover) | Every subscription has a correct MG parent |
| **G2 — Subscriptions** | Create `tcu-mig-prod`, `tcu-mig-nonprod`, **`tcu-plat-identity`, `tcu-plat-mgmt`**. **Register `Microsoft.Compute/EncryptionAtHost` on the migration subscriptions before anything else.** Enable Defender plans on all four. Submit the vCPU quota request against the migration subscriptions. **Do not re-create the Lighthouse delegation on any of them by default (§2.5).** | Roadmap phase 1 | Test Windows VM creates successfully with `encryptionAtHost = true`; no subscription-scope external delegation on the new subscriptions |
| **G3 — Policy & platform logging** | Assign the corp baseline at `Corp-IT` in `DoNotEnforce`; review compliance; flip to `Default`. Assign the **tier-0 baseline at `Platform`** (§5.1). Retire duplicated subscription-scope assignments. Stand up the central platform Log Analytics workspace **in `tcu-plat-mgmt`**, route activity logs from **all** subscriptions to it, **and raise retention to ≥12 months.** | Roadmap phase 2 (foundation) | Prod/Dev control gap closed; `Allowed locations` includes `eastus`; activity log queryable in KQL |
| **G4 — Hub** | H1 Firewall Policy → H2 Premium decision → H3 SNAT → H6 private DNS zones → H5 hub Bastion → H7 DNS Private Resolver. Apply hub RBAC and locks (§6.2). | Roadmap phases 2–4 | Firewall Policy attached with rule parity verified; SNAT instrumented |
| **G5 — Identity** | H4 — build `tcu-p-usw-id-vnet-01` (`10.217.0.0/24`) in `tcu-plat-identity`, peer to hub, then two DCs in different fault domains with static IPs. Repoint spoke DNS. Create the Azure AD Site with `10.212`/`10.213` registered. | Roadmap phase 3 (identity) | Domain join succeeds from a spoke without traversing ExpressRoute; **no external tenant holds standing write access to either DC** |
| **G6 — Address reservation** | Reserve `10.215.0.0/16` / `10.216.0.0/16` (DR) and `10.217.0.0/16` (platform). Fold all three, plus `10.212`/`10.213`, into a **single** on-prem BGP change. | Roadmap phase 2 | On-prem accepts all five prefixes |
| **G7 — Migrate** | No ALZ work. Waves run. | Roadmap phases 5–6 | Per-wave sign-off |
| **G8 — DR** | Build the `eastus` DR hub (zone-redundant gateway and firewall) + ASR. | Roadmap phase 7 | Tested failover with connectivity and DNS proven |
| **G9 — Phase 2 (post-cutover)** | Connectivity / Management / Identity / Security subscription split. `GatewaySubnet` resize to `/27`+ and ER+VPN coexistence. Sentinel decision. Expand the policy set. | After roadmap phase 8 | — |

---

## 9. What this changes in `landing-zone-design.md`

For traceability. That document is **not** edited; these are the deltas this recommendation introduces.

| § | Current decision | This recommendation | Impact |
|---|---|---|---|
| §1 | Subscriptions: existing `Production` and `Development` | **Four new subscriptions** — two landing zone, two platform (Identity, Management) | Address plan, subnets, routing, NSGs, naming, RG layout and waves **all unchanged** — only the containing subscription differs. Fallback in §5.2 if blocked. |
| §4 | `AzureBastionSubnet` in each migration spoke | **One Standard Bastion in the hub** | Frees `10.212.0.0/26` and `10.213.0.0/26`; one deployment instead of two. |
| §7 | DC placement — Option A (hub AD subnet) vs Option B (spoke DC subnet), decision open | ❌ **Neither.** A third option: **dedicated identity VNet `10.217.0.0/24` in a dedicated `tcu-plat-identity` subscription**, peered to the hub. Both original options place DCs inside a subscription an external tenant holds Contributor + User Access Administrator over (§2.5). | Closes the open decision, against both alternatives that were on the table. Hub AD subnet stays empty; spoke `dc` subnets become unnecessary. |
| §11 P0 | Register `EncryptionAtHost` on Production | **Register on the new subscriptions first.** Production still needs it if anything ever lands there. | Removes the deadlock from the migration's critical path. |
| §11 P4 | Firewall Policy migration at P4 | **Promoted to the first hub work item (H1)** | It gates Premium, IP Groups and rule hierarchy. |
| §8 | ASR to `eastus` for tier-1 | **Adds the missing DR hub, and reserves `10.215`/`10.216` now** | Closes a gap in the resilience story. |
| §11 P5 | "Log Analytics — extend retention" | **Central platform workspace in `tcu-plat-mgmt` + activity logs into it + ≥12-month retention** | Activity logs currently reach only an Event Hub and Rapid7 (§2.3); a workspace in Production is deletable by an external tenant's Contributor (§2.5). |
| §3 | Address plan reserves `10.212`/`10.213`, marks `10.215`–`10.222` "reserve for growth" | **Allocates `10.215` (DR hub), `10.216` (DR spoke), `10.217` (platform block)** | Five prefixes in one on-prem BGP change instead of three changes. |
| — | *(not covered)* | **`Allowed locations` must include `eastus`** before ASR is configured | Otherwise policy silently blocks failover. |
| — | *(not covered)* | **Lighthouse delegation is subscription-scope Contributor + VM Contributor + User Access Administrator** | Governs where every tier-0 asset in this design is allowed to live (§2.5). |

---

## 10. Decisions required

| # | Decision | Owner | Blocks | Default if undecided |
|---|---|---|---|---|
| 1 | **New landing zone subscriptions — yes or no?** | Platform + Finance | G2, and therefore the quota request | Fallback to existing subscriptions (§5.2) — **weaker, and hard to reverse after 200 servers land** |
| 2 | **Azure Firewall Premium — yes or no?** | Security / CISO | H2, and the maintenance window | Standard persists by default. **For FFIEC/GLBA this must be an explicit, recorded choice.** |
| 3 | **Microsoft Sentinel — fund, or document the alternative?** | Security / CISO | Nothing in the migration | No Sentinel, no recorded rationale — **the weakest possible audit position** |
| 4 | **Log Analytics retention target** | Compliance | G3 | 30 days — **short of typical examination expectations** |
| 5 | **Rename `Platform` and `Engineering` MGs** | Platform | G1 | Semantic collision persists and compounds |
| 6 | 🚨 **Is subscription-scope Contributor + User Access Administrator for Softchoice still intended?** (§2.5) | Security / CISO | G0 | An external tenant retains `runCommand` and disk-snapshot rights over everything in Production — **including anything tier-0 placed there** |
| 7 | **Should the migrated estate and the platform subscriptions be in Lighthouse scope at all?** | Security | G2 | New subscriptions are excluded by default — **decide deliberately, do not re-delegate by reflex**. If the managed service is needed, scope it to resource groups and drop User Access Administrator |
| 7a | **Who is `2f4a9838-26b7-47ee-be60-ccc1fdec5953`, and what does it hold?** | Security | G0 | A second delegated tenant with **unknown** rights over Production |
| 7b | **Identity + Management subscriptions — approve?** | Platform + Security | G2, G5 | DCs and the platform log workspace default into Production, inside the §2.5 exposure |
| 8 | **ER backup path — accept the single circuit, or plan VPN coexistence?** | Network | G9 (`GatewaySubnet` resize) | Single circuit remains a single point of failure for 200 servers *and* estate-wide DNS |
| 9 | **Populate `cost center` and the tag taxonomy before the estate lands** | Platform + Finance | G3 | Empty everywhere sampled; retrofitting across 200 servers is materially harder |

---

## 11. Open items requiring elevated permissions

These could not be verified with `vVashishtha@techcu.com` and should be confirmed before G1/G2 execute.

| Item | Blocked by | Needed for |
|---|---|---|
| Full management group tree — MGs with no subscriptions, and any subscription not visible to this account | `Microsoft.Management/managementGroups/read` denied at `/providers/Microsoft.Management` and on `Corp-IT` / `Platform` directly | §5.1 — the hierarchy in this document is inferred from subscription ancestry only |
| Role assignments at MG scope | Same | Understanding who can already change the hierarchy |
| TCU's billing agreement type and subscription-creation rights | Only a personal `MicrosoftOnlineServicesProgram` account in tenant `748c0cb0…` is visible | §5.2 — **the primary recommendation depends on this** |
| Ownership of `rg-cs-prod` / `evhns-cslog-kz5wah54kdpfi` and the `id-csscriptrunner-prod` identity | Requires cross-tenant / Lighthouse visibility | §2.3 — this is the current control-plane evidence path |
| The second delegated tenant `2f4a9838-26b7-47ee-be60-ccc1fdec5953` — its registration definition, granted roles and scope | `az managedservices definition list` returns only the Softchoice delegation; ARG does not index `Microsoft.ManagedServices` proxy types. It may be resource-group-scoped rather than subscription-scoped. | §2.5 — **potentially a second instance of the same tier-0 exposure** |
| Subscription-scope Azure RBAC role assignments (native, non-Lighthouse) | `az role assignment list --scope /subscriptions/…` returned `MissingSubscription`; requires `Microsoft.Authorization/roleAssignments/read` | §2.5 — Lighthouse is confirmed, but **internal** subscription-scope Owner/Contributor grants are still unenumerated and carry the same inheritance problem |
| NIC effective route tables | `Microsoft.Network/networkInterfaces/effectiveRouteTable/action` denied | Confirming the routing model empirically rather than by configuration reading |

---

## 12. Summary

The migration is gated by five things — the EncryptionAtHost deadlock, a 4× vCPU quota shortfall, a 500 Mbps circuit, a DNS dependency on ExpressRoute, and a region with no availability zones. **Not one of them is solved by moving the hub into a Connectivity subscription**, which is why the full-ALZ posture is the wrong call right now.

Two of them — the quota ceiling and the EncryptionAtHost deadlock — *are* solved, cleanly and at zero infrastructure cost, by landing the migrated estate in **new landing zone subscriptions** under a **correct management group hierarchy** with **policy assigned at MG scope**.

The sixth constraint is the one that was not visible when the migration was planned. **An external tenant holds Contributor, Virtual Machine Contributor and User Access Administrator at Production subscription scope** (§2.5) — which means `runCommand` and disk-snapshot export against anything placed there. Since Azure RBAC is additive and inherited subscription-scope assignments cannot be subtracted lower down, no resource group, lock, or NSG mitigates it. The subscription is the only boundary available.

That reframes the platform-subscription question. It is not "how much ALZ should we adopt" — it is **"which assets in this design are tier-0, and are they about to be created inside that exposure?"** Two are: the **domain controllers** and the **platform log workspace**. Both get their own subscription, now. **Connectivity does not** — a hub is not a credential store, and moving it still costs a production window for no migration benefit. **Security does not** — it exists to hold Sentinel, and there is no Sentinel.

So the balance holds, with one correction: take the governance and workload-placement half of ALZ now because it is free and it compounds; take the two platform subscriptions that carry tier-0 assets because RBAC gives no alternative; leave the rest until the estate is stable.

The hub itself is not the problem. It is well-built and correctly routed. It is simply equipped for 16 servers, and 200 are coming — so re-equip it in place: **Firewall Policy first**, then a Premium decision, SNAT headroom, a hub Bastion, and centralised DNS. All of it inside the resource group that will one day become the Connectivity subscription. The one thing that does *not* go in the hub is the thing the hub has an empty subnet waiting for.
