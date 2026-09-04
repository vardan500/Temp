# Azure — Existing Infrastructure Assessment

**Tenant:** Technology Credit Union (`techcu.onmicrosoft.com`, `dcceaaae-204c-482d-9d1c-64b4f3a8fd60`)
**Assessed:** 2026-09-01
**Assessed by:** `vVashishtha@techcu.com`
**Method:** Live enumeration via Azure CLI 2.88.0 against the tenant. Every figure below was read from the live environment, not inferred.

> **Scope note:** Two areas could not be read with the current account's permissions and are marked `NOT VERIFIABLE` throughout:
> - Management group hierarchy — `AuthorizationFailed` on `Microsoft.Management/register/action`
> - NIC effective route tables — `AuthorizationFailed` on `Microsoft.Network/networkInterfaces/effectiveRouteTable/action`

---

## 1. Subscriptions

| Subscription | ID | State | Resource groups |
|---|---|---|---|
| **Production** (default) | `f7f18245-3c4e-47f1-b465-e6bd0249f2c6` | Enabled | 44 |
| **Development** | `029c65f2-4ae1-4e7b-b152-94e909b1d278` | Enabled | 55 |
| **Visual Studio Enterprise** | `7260de92-5746-40b0-9a48-ad1a978ba813` | Enabled | 2 (`NetworkWatcherRG`, `PolicyTest`) |

**Delegated access (Azure Lighthouse):** the Production subscription reports two `managedByTenants` entries — `d69b3567-877c-4286-be00-05eb399ef140` and `2f4a9838-26b7-47ee-be60-ccc1fdec5953`. Two external tenants hold delegated rights into Production. Worth confirming both are current and intended.

**Management groups:** `NOT VERIFIABLE` — no `Microsoft.Management` permission on this account.

---

## 2. Region posture

Effectively the entire estate is in **`westus`**. Exceptions found:

| Resource | Region | Subscription |
|---|---|---|
| `vsNet` (Cato SASE VNet) | `westus2` | Production |
| `tcu-d-usw-consortium-egd-01` | `westus2` | Development |
| `cloud-shell-storage-southcentralus` | `southcentralus` | Production |
| `VisualStudioOnline-7DD0C588...` | `eastus2` | Development |
| `PolicyTest` | `canadacentral` | Visual Studio Enterprise |

### Availability zone support — verified by SKU query

| Region | Zones for `Standard_D4s_v5` |
|---|---|
| **`westus`** | **`[]` — none** |
| `westus2` | `1, 2, 3` |
| `westus3` | `1, 2, 3` |

`westus` is a legacy region with **no availability zone support**. The only intra-region resilience primitive available is the **Availability Set**, which in `westus` supports a **maximum of 3 fault domains** (verified: `MaximumPlatformFaultDomainCount = 3` for both `Aligned` and `Classic` SKUs).

---

## 3. Naming conventions in use

Consistent and worth preserving.

| Object | Pattern | Examples |
|---|---|---|
| Resource group | `tcu-<env>-usw-<workload>-rg-<nn>` | `tcu-p-usw-network-rg-01`, `tcu-np-usw-vmmgmt-rg-01` |
| Virtual machine | `<env><azw><workload><nn>` | `pazwsqldb01`, `qazwakcweb01`, `dazwakcweb02` |
| Subnet | `tcu-<env>-usw-<purpose>-snet-<nn>` | `tcu-p-usw-app-snet-01` |
| Route table | `tcu-<env>-usw-<purpose>-route-<nn>` | `tcu-p-usw-default-route-01` |
| VNet | `tcu-<env>-usw-vnet-<nn>` | `tcu-p-usw-vnet-01` |

**Environment codes observed:** `p` (prod), `np` (non-prod shared), `d` (dev), `qa`, `st` (stage), `uat`
**VM prefix codes observed:** `p` (prod), `q` (QA), `d` (dev); `azw` = Azure West

---

## 4. Virtual networks

| VNet | Resource group | Sub | Region | Address space | Subnets | Peerings |
|---|---|---|---|---|---|---|
| `tcu-p-usw-exprt-vnet-01` **(hub)** | `tcu-p-usw-exprt-rg-01` | Prod | westus | `10.209.0.0/16` | 6 | 2 |
| `tcu-p-usw-vnet-01` (prod spoke) | `tcu-p-usw-network-rg-01` | Prod | westus | `10.211.0.0/16` | 13 | 1 |
| `tcu-np-usw-vnet-01` (non-prod spoke) | `tcu-np-usw-network-rg-01` | Dev | westus | `10.210.0.0/16` | 20 | 1 |
| `workers-vnet` (Databricks managed) | `databricks-rg-tcubiadbprod-01` | Prod | westus | `10.139.0.0/16` | 2 | 0 |
| `PAZWTCUAPP01-vnet` | `tcu-p-usw-vm-apps-rg-01` | Prod | westus | `10.0.0.0/16` | 1 | 0 |
| `QAZWTCUAPP01-vnet` | `tcu-np-usw-vm-apps-rg-01` | Dev | westus | `10.0.0.0/16` | 1 | 0 |
| `vsNet` (Cato) | `TCU-INFRA-CATO-RG-01` | Prod | **westus2** | `172.16.0.0/22` | 2 | 0 |

### ⚠ Finding — duplicate and colliding `10.0.0.0/16`

`PAZWTCUAPP01-vnet` and `QAZWTCUAPP01-vnet` **both** use `10.0.0.0/16`. They are unpeered and isolated, so nothing is broken today — but:

- They collide **with each other**.
- `10.0.0.0/16` also collides with on-prem prefixes advertised over ExpressRoute: `10.0.4.0/22`, `10.0.6.0/30`, `10.0.20.0/22`, `10.0.101.0/24`.

These look like accidental portal-default VNets created alongside `PAZWTCUAPP01` / `QAZWTCUAPP01`. Neither can ever be peered to the hub as-is. Recommend re-addressing or decommissioning.

### Hub subnets — `tcu-p-usw-exprt-vnet-01` (`10.209.0.0/16`)

| Subnet | Prefix | NSG | Route table |
|---|---|---|---|
| `GatewaySubnet` | `10.209.0.16/28` | — | `tcu-p-usw-gw-route-01` |
| `AzureFirewallSubnet` | `10.209.1.0/24` | — | `tcu-p-usw-azurefw-route-01` |
| `tcu-p-usw-exprt-mgmt-snet-01` | `10.209.2.0/24` | `tcu-p-usw-exprt-nsg-01` | — |
| `tcu-p-usw-exprt-ad-snet-01` | `10.209.3.0/24` | — | — |
| `tcu-p-usw-exprt-containers-snet-01` | `10.209.4.0/24` | — | — |
| `tcu-p-usw-exprt-sharedsvcs-snet-01` | `10.209.8.0/24` | `tcu-p-usw-exprt-nsg-01` | `tcu-p-usw-hubdefault-route-01` |

**`tcu-p-usw-exprt-ad-snet-01` contains zero NICs.** A dedicated AD subnet was provisioned in the hub and never populated — there are no domain controllers in Azure.

### Prod spoke subnets — `tcu-p-usw-vnet-01` (`10.211.0.0/16`)

| Subnet | Prefix | Route table | NSG |
|---|---|---|---|
| `tcu-p-usw-agw-snet-01` | `10.211.0.0/24` | `tcu-p-usw-paas-route-01` | — |
| `tcu-p-usw-apim-snet-01` | `10.211.1.0/24` | `tcu-p-usw-paas-route-01` | `tcu-p-usw-nsg-01` |
| `tcu-p-usw-agw-snet-02` | `10.211.2.0/24` | `tcu-p-usw-paas-route-01` | — |
| `tcu-p-usw-databricks-public-snet-01` | `10.211.4.0/26` | `tcu-p-usw-dbs-route-01` | `tcu-p-usw-bi-dbs-nsg-01` |
| `tcu-p-usw-databricks-private-snet-01` | `10.211.5.0/26` | `tcu-p-usw-dbs-route-01` | `tcu-p-usw-bi-dbs-nsg-01` |
| `tcu-p-usw-ase-snet-01` | `10.211.7.0/24` | `tcu-p-usw-paas-route-01` | — |
| `tcu-p-usw-web-snet-01` | `10.211.8.0/21` | `tcu-p-usw-default-route-01` | `tcu-p-usw-nsg-01` |
| `tcu-p-usw-app-snet-01` | `10.211.16.0/21` | `tcu-p-usw-default-route-01` | `tcu-p-usw-nsg-01` |
| `tcu-p-usw-data-snet-01` | `10.211.24.0/21` | `tcu-p-usw-default-route-01` | `tcu-p-usw-nsg-01` |
| `tcu-p-usw-ise-snet-01` … `-04` | `10.211.32-35.0/24` | `tcu-p-usw-paas-route-01` | — |

Highest allocation is `10.211.35.255`; `10.211.36.0`–`10.211.255.255` is unused inside the prod spoke.

**The `/21`-per-server-tier convention (web / app / data) is the established pattern and is carried into the proposed design.**

### Non-prod spoke subnets — `tcu-np-usw-vnet-01` (`10.210.0.0/16`)

20 subnets. Notable: `AzureBastionSubnet` `10.210.3.0/26`, `tcu-np-usw-vm-test-01` `10.210.3.64/26`, and the same `/21` web/app/data trio at `10.210.8.0/21`, `10.210.16.0/21`, `10.210.24.0/21`.

### Peerings

| VNet | Peering | State | `allowGatewayTransit` | `useRemoteGateways` | `allowForwardedTraffic` |
|---|---|---|---|---|---|
| Hub | `peer-hub-to-prod` | Connected | **true** | false | true |
| Hub | `peer-hub-to-spoke` (→ non-prod) | Connected | **true** | false | true |
| Prod spoke | `peer-prod-to-hub` | Connected | false | **true** | true |

Textbook hub-and-spoke with gateway transit. Spokes are **not** peered to each other.

---

## 5. ExpressRoute

| Property | Value |
|---|---|
| Circuit | `tcu-p-usw-att-exprt-01` (`tcu-p-usw-exprt-rg-01`) |
| Provider | **AT&T Netbond** |
| Peering location | **Silicon Valley** |
| Bandwidth | **500 Mbps** |
| SKU | **Standard** / `MeteredData` |
| Provisioning | `Provisioned` / `Enabled` |
| Global Reach | Disabled |
| Peerings configured | **`AzurePrivatePeering` only — no Microsoft peering** |
| Peer ASN | 8030 (AT&T) |
| VLAN ID | 1 |
| Primary peer subnet | `10.208.0.0/30` |
| Gateway | `tcu-p-usw-exprt-vnet-gw-01`, SKU **Standard**, **non-zonal** |
| Connection | `tcu-p-usw-exprt-vnet-gw-con-01`, BGP `False`, FastPath `False` |

### Circuit headroom

| Limit (Standard SKU) | Allowed | In use |
|---|---|---|
| VNet connections per circuit | 10 | **1** |
| Private peering route prefixes | 4,000 | **204** |

Standard SKU reaches **all Azure regions within the same geopolitical region**, so every US region is already in scope from the Silicon Valley peering location. Premium would only be required to cross geopolitical boundaries.

### BGP — 204 routes learned from on-prem

**On-prem advertises a default route `0.0.0.0/0` via `10.208.0.1`.** Customer ASNs observed: `65001` (bulk), `65002`, `65003`.

On-prem address blocks in use:

| Block | Prefixes | Notes |
|---|---|---|
| `10.0.0.0/16` – `10.3.0.0/16` | 11 | **Collides with the two orphan `10.0.0.0/16` VNets** |
| `10.10.0.0/16` | 4 | |
| `10.23.0.0/16`, `10.24.0.0/16` | 3 | `easesilverlake`, `easeprod`, `easenonprod` |
| `10.41.0.0/16` | 1 | Full /16 — routed via Cato |
| **`10.71.0.0/16`** | **78** | Primary on-prem datacenter range; DCs live here |
| `10.72`–`10.74`, `10.80`, `10.98` | 8 | |
| `10.100`, `10.115`, `10.184` | 10 | Mostly /32 loopbacks |
| `10.214.65.64/32`, `10.223.4.0/24` | 2 | Isolated strays in the Azure band |
| `172.16`–`172.29` | 14 | Includes Cato `172.16.0.0/21` |
| `192.168.0.0/16` | 19 | |

Azure prefixes advertised back to on-prem (AS path `65515`): `10.209.0.0/16`, `10.210.0.0/16`, `10.211.0.0/16`.

### ✅ Verified free address space

Within the Azure allocation band, **`10.212.0.0/16` and `10.213.0.0/16` are entirely absent** from both the BGP table and every Azure VNet. They are contiguous with the existing `10.209`/`10.210`/`10.211` allocation. Nearest neighbours are the strays `10.214.65.64/32` and `10.223.4.0/24`.

---

## 6. Azure Firewall

| Property | Value |
|---|---|
| Name | `tcu-p-usw-azurefw-01` (`tcu-p-usw-exprt-rg-01`) |
| Private IP | `10.209.1.4` |
| SKU / tier | `AZFW_VNet` / **Standard** |
| Zones | **None** (non-zonal) |
| Firewall Policy | **None — classic rule collections** |
| Threat intel mode | `Alert` |
| Public IPs | **4** |
| Network rule collections | 6 (**47 rules**) |
| Application rule collections | 2 (**4 rules**) |
| NAT rule collections | 4 |

Public IPs attached: `tcu-p-usw-azurefw-01-pip-01` (`ip-configuration`), `-pip-02` (`appian-cloud`), `-pip-03` (`appian-cloud-prod`), and `tcu-np-usw-azlos-temp-pip-01` (`azlos-np-temp`).

**Notes:**
- 4 public IPs ≈ **9,984 SNAT ports** total (2,496 per IP). Sufficient today at 16 VMs; needs monitoring at 200+.
- **Classic rules, not Firewall Policy.** Firewall Policy is a prerequisite for the Premium tier, IP Groups, and rule hierarchy — all of which matter for managing a 200-server ruleset.
- **Standard tier has no IDPS and no TLS inspection.** For a credit union under FFIEC/GLBA examination this is worth a deliberate, documented decision.
- One non-prod-named public IP (`azlos-np-temp`) is attached to the production firewall.

---

## 7. Routing and egress model

| Route table | Applied to | Routes | BGP propagation |
|---|---|---|---|
| `tcu-p-usw-gw-route-01` | `GatewaySubnet` | `10.210.0.0/16` → `10.209.1.4`<br>`10.211.0.0/16` → `10.209.1.4` | — |
| `tcu-p-usw-azurefw-route-01` | `AzureFirewallSubnet` | `0.0.0.0/0` → `Internet` | — |
| `tcu-p-usw-hubdefault-route-01` | Hub shared services | `0.0.0.0/0` → `10.209.1.4` | — |
| **`tcu-p-usw-default-route-01`** | **prod web / app / data subnets** | `0.0.0.0/0` → `10.209.1.4` | **Disabled** |
| `tcu-p-usw-paas-route-01` | PaaS subnets (AGW, APIM, ASE, ISE) | on-prem prefixes → `10.209.1.4`<br>`0.0.0.0/0` → `Internet` | **Disabled** |
| `tcu-p-usw-dbs-route-01` | Databricks subnets | — | — |

**The effective model:** server subnets disable BGP propagation and send *everything* — internet **and** on-prem — to the Azure Firewall via a single `0.0.0.0/0` route. The firewall then reaches on-prem through the hub's ExpressRoute gateway. Traffic from on-prem into the spokes is forced through the firewall by the `GatewaySubnet` route table. Inspection is symmetric.

Because BGP propagation is disabled on server subnets, the `0.0.0.0/0` advertised from on-prem does **not** override Azure egress. Internet egress leaves via the Azure Firewall, not via on-prem.

---

## 8. DNS

- **All VNets resolve to on-prem domain controllers:** `10.71.4.15`, `10.71.4.45`, `10.71.4.30` (verified on both the hub and the prod spoke).
- **No domain controllers exist in Azure.** The hub's `tcu-p-usw-exprt-ad-snet-01` is empty.
- Private DNS zones in Production — only two:

| Zone | Resource group | VNet links | Records |
|---|---|---|---|
| `privatelink.blob.core.windows.net` | `tcu-p-usw-network-rg-01` | 1 | 2 |
| `privatelink.database.windows.net` | `tcu-p-usw-network-rg-01` | 1 | 4 |

**Risk:** every Azure workload's name resolution depends on ExpressRoute reaching `10.71.4.x`. If the circuit drops, Azure DNS resolution fails estate-wide. This is a single point of failure independent of any migration.

---

## 9. Compute inventory

**16 VMs total.** A 200-server migration is roughly a **12× increase**.

### Production (7)

| VM | Resource group | Size | OS |
|---|---|---|---|
| `pazwazragt01` | `TCU-P-USW-AZRAGT-RG-01` | `Standard_B4ms` | Windows |
| `pazwsqldb01` | `TCU-P-USW-DATA-RG-01` | `Standard_D8s_v3` | Windows |
| `pazwdatadog01` | `TCU-P-USW-DATADOG-RG-01` | `Standard_B1s` | Linux |
| `pazwdatadog02` | `TCU-P-USW-DATADOG-RG-01` | `Standard_B2s` | Windows |
| `pazwazrjb01` | `TCU-P-USW-JB-RG-01` | `Standard_B4ms` | Windows |
| `PAZWTCUAPP01` | `TCU-P-USW-VM-APPS-RG-01` | `Standard_E2s_v3` | Windows |
| `vSocket-Infra-CATO` | `TCU-INFRA-CATO-RG-01` | `Standard_D2s_v5` | Linux |

### Development (9)

| VM | Resource group | Size | OS |
|---|---|---|---|
| `tcuduswtestvm01` | `RG-DEVOPSTEST-01` | `Standard_D2s_v3` | Windows |
| `dazwakcweb02` | `TCU-D-USW-AKCWEB-RG-01` | `Standard_D2s_v3` | Windows |
| `dazwdevsql01` | `TCU-NP-USW-DEVDATA-RG-01` | `Standard_E2s_v3` | Windows |
| `QAZWTCUAPP01` | `TCU-NP-USW-VM-APPS-RG-01` | `Standard_D4s_v3` | Windows |
| `qazwakcapp02` | `TCU-QA-USW-AKCAPP-RG-01` | `Standard_D4s_v3` | Windows |
| `qazwakcimm01` | `TCU-QA-USW-AKCAPP-RG-01` | `Standard_B2ms` | Windows |
| `qazwakcsql01` | `TCU-QA-USW-AKCDATA-RG-01` | `Standard_E2ds_v4` | Windows |
| `qazwakcweb01` | `TCU-QA-USW-AKCWEB-RG-01` | `Standard_D2s_v3` | Windows |
| `qazwakcweb02` | `TCU-QA-USW-AKCWEB-RG-01` | `Standard_D2s_v3` | Windows |

---

## 10. Compute quota — Production, `westus`

| Quota | Used | Limit |
|---|---|---|
| **Total Regional vCPUs** | **21** | **280** |
| Standard BS Family vCPUs | 11 | 350 |
| Standard DSv3 Family vCPUs | 8 | **100** |
| Standard ESv3 Family vCPUs | 2 | **100** |
| Standard Dsv6 / Ddsv6 / Dasv7 / … | 0 | 350 |
| Total Regional Low-priority vCPUs | 0 | 350 |

**`Total Regional vCPUs` at 280 is the binding constraint.** It caps the whole subscription regardless of per-family headroom. 200 servers will require roughly 800–1,200 vCPU.

---

## 11. Azure Policy — Production

10 assignments, all `enforcementMode: Default` (enforced):

| Assignment | Type |
|---|---|
| Cascade Default Tags Production | Custom initiative `94c350c2-e9df-4f6f-821d-d21b90a7edf9` |
| ASC DataProtection | Built-in initiative |
| ASC Default | Built-in initiative |
| Managed disks should disable public network access | Built-in |
| Snapshots should disable public network access (Deny) | Built-in |
| Storage accounts should have infrastructure encryption | Built-in |
| Windows virtual machines should enable Azure Disk Encryption or EncryptionAtHost | Built-in `3dc5edcd-002d-444c-b216-e123bbfa37c0` |
| Deploy prerequisites to enable Guest Configuration policies on VMs | Built-in DINE |
| **Deny Windows VMs without EncryptionAtHost** | **Custom `bb4ab29614a65a26accce8f7`** |
| Azure Key Vault should have firewall enabled or public network access disabled | Built-in |

### 🚨 Blocker — EncryptionAtHost deadlock on Production

Verified across three separate reads:

1. Custom policy `bb4ab29614a65a26accce8f7` denies any VM where `securityProfile.encryptionAtHost != true`, scoped to Windows images **and** to VMs where `imageSKU` does not exist — which is exactly the shape of a VM created by Azure Migrate from replicated managed disks.
2. Its `effect` parameter has **`defaultValue: "Deny"`**, the assignment passes **no override** (`parameters: {}`), `enforcementMode` is `Default`, and `notScopes` is **empty**.
3. `Microsoft.Compute/EncryptionAtHost` on Production is **`NotRegistered`**.

| Subscription | `EncryptionAtHost` feature |
|---|---|
| **Production** | **`NotRegistered`** |
| Development | `Registered` |

**The result is a deadlock:** on Production, a Windows VM must set `encryptionAtHost = true` to satisfy the policy, but the subscription cannot set it because the feature is not registered. Every Windows server landing in Production fails at create time until the feature is registered. The existing 7 prod VMs predate the assignment.

Development is unaffected — the feature is registered there.

---

## 12. Tagging

The `Cascade Default Tags Production` initiative cascades this taxonomy:

`app` · `cost center` · `department` · `env` · `owner` · `project` · `tier`

Application is inconsistent:

| Resource group | Tag state |
|---|---|
| `tcu-p-usw-data-rg-01` | Populated — `app=none`, `department=Platform Services`, `env=prod`, `owner=kezeh@techcu.com`, `project=Azure`, `tier=infra` |
| `tcu-p-usw-vm-apps-rg-01` | **All seven keys present but empty** |

`cost center` was empty on every resource group sampled. For a 200-server migration where chargeback matters, this needs fixing before the estate lands.

---

## 13. Monitoring, backup, and security tooling

**Log Analytics**

| Workspace | Resource group | Retention | SKU |
|---|---|---|---|
| `tcu-logs` | `tcu-p-usw-exprt-rg-01` | **30 days** | `pergb2018` |
| `tcu-cube-logs` | `tcu-p-usw-bi-rg-01` | **30 days** | `pergb2018` |

30-day retention is short of what a credit union typically needs to evidence under FFIEC/GLBA examination.

**Recovery Services Vaults**

| Vault | Resource group | Region |
|---|---|---|
| `tcu-p-usw-azragt-bckup-01` | `tcu-p-usw-azragt-rg-01` | westus |
| `tcu-p-usw-sql-bckup-01` | `tcu-p-usw-data-rg-01` | westus |

Both are workload-specific. There is no general-purpose VM backup vault sized for a 200-server estate.

**Third-party tooling present**

| Tool | Evidence |
|---|---|
| Datadog | `tcu-p-usw-datadog-rg-01`, `pazwdatadog01/02`, Azure-native Datadog resource |
| Rapid7 | `tcu-p-usw-rapid7` |
| Cato Networks (SASE) | `TCU-INFRA-CATO-RG-01`, `vSocket-Infra-CATO`, `vsNet` in westus2; `10.41.0.0/16` routed via Cato |
| Databricks | Managed RGs in both Production and Development |

**Azure Bastion**

| Bastion | Subscription |
|---|---|
| `tcu-np-usw-vnet-01-bastion` | Development |

**No Bastion in Production.** Prod access today is via the jumpbox `pazwazrjb01`.

**Azure Migrate:** no `Microsoft.Migrate/migrateProjects` and no `Microsoft.Migrate/assessmentProjects` exist in either subscription. Nothing has been started.

---

## 14. Summary of findings

| # | Finding | Severity |
|---|---|---|
| 1 | **EncryptionAtHost deadlock on Production** — Deny policy active, feature `NotRegistered`. Blocks every Windows VM at create time. | 🚨 Blocker |
| 2 | `westus` has **no availability zones**; 200 production servers would share one datacenter fault domain set (max 3 FDs). | 🔴 High |
| 3 | **`Total Regional vCPUs` 21/280** — insufficient by roughly 4×. `DSv3`/`ESv3` capped at 100 each. | 🔴 High |
| 4 | **All DNS depends on on-prem `10.71.4.x` over ExpressRoute.** No Azure DCs; hub AD subnet empty. | 🔴 High |
| 5 | **ExpressRoute is 500 Mbps.** At ~40 TB this is ~7.4 days of fully saturated transfer; realistically far longer. | 🔴 High |
| 6 | **No Microsoft peering** — Azure Migrate replication to public endpoints has no ExpressRoute path. | 🟠 Medium |
| 7 | Azure Firewall is **Standard with classic rules** — no Firewall Policy, no IDPS, no TLS inspection. | 🟠 Medium |
| 8 | Two orphan VNets both on `10.0.0.0/16`, colliding with each other and with on-prem. | 🟠 Medium |
| 9 | ER gateway and firewall are both **non-zonal** and single-instance. | 🟠 Medium |
| 10 | Log Analytics retention **30 days**. | 🟠 Medium |
| 11 | Tag taxonomy exists but is **largely unpopulated**; `cost center` empty everywhere sampled. | 🟡 Low |
| 12 | No Bastion in Production. | 🟡 Low |
| 13 | Two external tenants hold **Lighthouse delegation** into Production. | 🟡 Verify |
| 14 | Non-prod-named public IP attached to the production firewall. | 🟡 Low |
