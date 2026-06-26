# Microsoft Sentinel Data Durability & Platform Responsibility

**Author:** Venicia Solomons, Cloud & AI Security SE  
**Date:** 2026-06-26  
**Audience:** Security Operations, Cloud Infrastructure, Compliance Teams

---

## Executive Summary

This guide clarifies **Microsoft's platform responsibilities vs. customer responsibilities** for data durability in Microsoft Sentinel deployments. 

**Key Finding:** Data loss is preventable, but you must choose the right redundancy configuration.

- ✅ **No Sentinel data is lost** if you configure **ZRS** or **GZRS**
- ❌ **All data is lost** in single datacenter failure if you use **LRS** (not recommended for production)
- 📋 **Microsoft handles** replication; **you choose** the redundancy level

---

## Data Loss Scenarios by Configuration

| Scenario | LRS | ZRS | GZRS |
|---|---|---|---|
| **Single datacenter fails** | ❌ **Total loss** | ✅ Zero loss | ✅ Zero loss |
| **Single zone fails (1-4 hrs)** | ❌ **Total loss** | ✅ Auto failover <5 min | ✅ Auto failover <5 min |
| **Entire region fails** | ❌ **Total loss** | ❌ **No protection** | ⚠️ ≤15 min loss (manual failover) |
| **Cost vs. LRS** | 1.0x (baseline) | 1.2x (20% more) | 1.3-1.5x (30-50% more) |

**Recommended for Sentinel:** **GZRS** (production) or **ZRS** (single-region only)

---

## What Microsoft Guarantees

**Reference:** [Azure Storage Data Redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)

Microsoft Sentinel data is stored in **Azure Storage** and managed by **Azure Monitor**. Microsoft guarantees:

✅ **Multiple replicas** of all your data (synchronous within a zone)  
✅ **Zone-level failover** in <5 minutes (automatic via DNS repointing)  
✅ **Geo-replication** to a secondary region 100+ miles away  
✅ **16 nines durability** (99.99999999999999% annual) with GZRS  
✅ **Append-only, tamper-proof** logs (immutable, encryption at rest)  
✅ **Automatic backups** and disaster recovery infrastructure  

**Microsoft does NOT guarantee:** Protection against choosing LRS for mission-critical workloads (that's a customer decision).

---

## What You Must Configure

**Reference:** [Disaster Recovery & Failover Guidance](https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance)

Three critical customer decisions:  

### 1. **LRS (Locally Redundant Storage)** — Single Datacenter

**How it works:** 3 copies stored in ONE datacenter. If DC fails = all copies lost.

| Property | Value |
|---|---|
| **Durability** | 11 nines (99.99999999%) |
| **Zone failure** | ❌ **Data lost** |
| **Region failure** | ❌ **Data lost** |
| **Cost** | Baseline (1.0x) |
| **Best for** | Dev/test only (NOT production) |

**Platform shows:** Storage account → Properties → Redundancy = "Locally-redundant storage"

---

### 2. **ZRS (Zone-Redundant Storage)** — Multi-Zone in Same Region

**How it works:** Synchronous replication across 3+ availability zones in primary region. If one zone fails, automatic failover <5 min.

| Property | Value |
|---|---|
| **Durability** | 12 nines (99.9999999999%) |
| **Zone failure** | ✅ **Auto failover <5 min** |
| **Region failure** | ❌ **No protection** |
| **Cost** | ~20% premium (1.2x) |
| **Best for** | Production (single region) |

**Platform shows:** Storage account → Properties → Redundancy = "Zone-redundant storage"

---

### 3. **GZRS (Geo-Zone-Redundant Storage)** — Multi-Zone + Geo-Backup ⭐ **RECOMMENDED**

**How it works:**
- **Primary region:** Synchronous replication across 3+ zones (like ZRS)
- **Secondary region:** Asynchronous backup 100+ miles away (≤15 min lag)
- **Zone fails:** Automatic failover <5 min, zero loss
- **Region fails:** Manual failover needed, ≤15 min of new data potentially lost

| Property | Value |
|---|---|
| **Durability** | 16 nines (99.99999999999999%) |
| **Zone failure** | ✅ **Auto failover <5 min** |
| **Region failure** | ⚠️ **Manual failover, ≤15 min RPO** |
| **Cost** | ~30-50% premium (1.3-1.5x) |
| **Best for** | **Enterprise production SIEM** |

**Platform shows:** Storage account → Properties → Redundancy = "Geo-zone-redundant storage"

---

## Platform Responsibility Matrix

**Reference:** [Azure Monitor Data Security & Platform Responsibility](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/best-practices-security#how-microsoft-secures-azure-monitor)

| Responsibility | Microsoft Handles | You Must Choose |
|---|---|---|
| **Data replication** | ✅ Automatic within zone(s) | — |
| **Encryption at rest** | ✅ Enabled by default (Microsoft-managed) | ⚠️ Can upgrade to customer-managed keys |
| **Zone failover** | ✅ Automatic <5 min if ZRS/GZRS | — |
| **Redundancy level** | — | ⚠️ **YOU must pick LRS, ZRS, or GZRS** |
| **Region failover** | — | ⚠️ **YOU must initiate manual failover** |
| **Backup retention** | ✅ Automatic (configurable 30-12 yrs) | ⚠️ You set retention policy |
| **Data purge** | ✅ Available on request (compliance) | ⚠️ You request when needed |

### What Does "YOU Must Choose" Mean?

**It means:**

1. **Default configuration is NOT enough.** Microsoft Sentinel installations default to LRS unless you upgrade.
2. **LRS = risky for production.** One datacenter failure = you lose everything.
3. **You must upgrade to ZRS or GZRS** before going live with compliance/production data.
4. **Regional failover is manual.** If a whole region fails, Microsoft won't fail over for you automatically. You must:
   - Monitor failover status
   - Update your applications to point to the secondary region
   - Follow your runbook

---

## How to Find Your Current Configuration

### Via Azure Portal (Easiest)

1. **Azure Portal** → **Log Analytics Workspace** → **Select your workspace**
2. **Properties** → Look for **"Redundancy"** field
3. **Note the value:**
   - `Locally-redundant` = LRS (⚠️ Risk)
   - `Zone-redundant` = ZRS (✅ Good)
   - `Geo-zone-redundant` = GZRS (✅ Best)

### Via Azure CLI

```bash
az monitor log-analytics workspace show \
  --resource-group "rg-name" \
  --workspace-name "workspace-name" \
  --query "properties.sku.name" \
  --output tsv
```

**Expected output:** `PerGB2018` (with redundancy set separately in properties)

### Via PowerShell

```powershell
Get-AzOperationalInsightsWorkspace `
  -ResourceGroupName "rg-name" `
  -Name "workspace-name" | Select-Object Sku, Location
```

---

## Recovery Objectives (RTO/RPO Explained)

**Reference:** [Geo Priority Replication & RPO](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy-priority-replication)

| Metric | Definition | ZRS | GZRS |
|---|---|---|---|
| **RTO** (Recovery Time Objective) | How long until you can use data again | <5 min (auto) | <5 min (auto) + 5-30 min (manual) for region fail |
| **RPO** (Recovery Point Objective) | How much new data might be lost | 0 min (sync) | 0 min (primary), ≤15 min (region fail) |

**Plain English:**
- **Zone fails** → Back online in <5 min, no data lost
- **Region fails** → Takes manual action (5-30 min), might lose ≤15 min of incoming logs (new writes not yet synced to secondary)

---

## How to Upgrade to GZRS

**Reference:** [Storage Account Upgrade Process](https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance#plan-for-failover)

**Important:** Upgrade can happen **in-place with minimal downtime** (typically 5-10 minutes).

### Steps

1. **Azure Portal** → **Storage Account** → **Redundancy**
2. **Change from:** LRS/ZRS → **To:** GZRS
3. **Confirm:** Review cost impact (typically 30-50% storage premium)
4. **Apply:** Click upgrade
5. **Wait:** 5-10 minutes for replication to complete

**No data loss. No downtime to queries (slight performance dip possible).**

---

## What If a Region Fails?

**Reference:** [Customer-Managed Failover Process](https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance#plan-for-failover)

### Automatic (Happens in <5 min)
✅ **Zone fails** (within same region) → Azure updates DNS → Secondary zone becomes active

### Manual (You must act)
⚠️ **Entire region fails** → You must:
1. Verify secondary region is available (via Azure Service Health)
2. **Azure Portal** → **Storage Account** → **Disaster recovery**
3. Click **"Failover to secondary"**
4. Confirm the action
5. Wait 5-30 minutes for failover to complete
6. Update your application connections (if hardcoded endpoints)

**During manual failover:** ≤15 minutes of new logs might be lost (writes not yet synced to secondary).

---

## Recommendations by Use Case

### **Option A: Single-Region SIEM (Cost-Conscious)**
- **Configuration:** ZRS
- **Protection:** Zone-level automatic failover
- **Risk:** Regional disaster = total loss
- **Best for:** Dev/test, non-critical monitoring, internal workloads
- **Cost:** ~20% premium over LRS

### **Option B: Enterprise Production SIEM (Recommended) ⭐**
- **Configuration:** GZRS
- **Protection:** Automatic zone failover + manual region failover
- **Risk:** Minimal (≤15 min loss only in catastrophic region failure)
- **Best for:** Production compliance workloads, 100GB/day+ ingestion, PCI/HIPAA/SOC 2
- **Cost:** ~30-50% premium over LRS

### **Option C: Mission-Critical (No Data Loss Tolerance)**
- **Configuration:** GZRS + Read-Access Secondary (RA-GZRS)
- **Protection:** Same as GZRS, plus ability to read from backup region
- **Cost:** Highest (~50-60% premium)
- **Best for:** 24/7 security operations, forensics preservation

---

## Next Steps

### **Immediate**
- [ ] Audit current redundancy configuration (LRS, ZRS, or GZRS?)
- [ ] Identify all Sentinel workspaces in your subscription
- [ ] Document which are production vs. dev/test

### **This Month**
- [ ] Upgrade production workspaces to GZRS
- [ ] Document cost impact in your budget
- [ ] Test upgrade in non-prod first

### **This Quarter**
- [ ] Create failover runbook for regional disaster
- [ ] Train operations team on failover procedure
- [ ] Schedule quarterly failover testing in dev/test environment
- [ ] Review data retention policies (30 days is default; consider long-term archive if needed)

### **Ongoing**
- [ ] Monitor replication lag via Azure Storage metrics (alert if RPO exceeds 15 min)
- [ ] Subscribe to Azure Service Health alerts for your primary region
- [ ] Document any regional failures and lessons learned

---

## References

1. **[Data Redundancy Options](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)** — Full details on LRS, ZRS, GRS, GZRS
2. **[Disaster Recovery & Failover Guidance](https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance)** — How to plan and execute failover
3. **[Geo Priority Replication (≤15 min RPO)](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy-priority-replication)** — RPO guarantees
4. **[Microsoft Sentinel Data Security](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/best-practices-security)** — Platform security practices
5. **[Log Analytics Workspace Data Retention](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-retention-configure)** — Retention policy configuration
6. **[What is Microsoft Sentinel?](https://learn.microsoft.com/en-us/azure/sentinel/overview)** — Sentinel overview and capabilities

---

## Questions?

**For technical questions:** Refer to [Microsoft Sentinel Support](https://learn.microsoft.com/en-us/azure/sentinel/overview)  
**For Azure Storage redundancy:** [Azure Storage Support](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)

