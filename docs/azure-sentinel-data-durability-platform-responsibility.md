# Microsoft Sentinel Data Durability & Platform Responsibility

**Author:** Venicia Solomons, Cloud & AI Security SE  
**Date:** 2026-06-26  
**Audience:** Security Operations, Cloud Infrastructure, Compliance Teams

---

## Executive Summary

Microsoft Sentinel is a **platform service** that provides **built-in data protection and redundancy out of the box**.

**What you get automatically:**
✅ **Zone-redundant replication** across availability zones in your region  
✅ **Automatic failover** (<5 min) if a zone fails  
✅ **Encryption at rest** by default  
✅ **Append-only, tamper-proof logs** for compliance  
✅ **Continuous backups** and disaster recovery infrastructure  

**Optional enhancements (for maximum resilience):**
📋 **Geo-redundancy** — replicate to a secondary region 100+ miles away (≤15 min RPO)  
📋 Choose between **ZRS** (zone-level protection) or **GZRS** (zone + region protection)  

**Bottom line:** Your Sentinel data is **protected and resilient by default**. Geo-redundancy is an optional enhancement for organizations with extreme disaster recovery requirements.

---

## Data Protection by Redundancy Level

By default, your Sentinel deployment is **zone-redundant** (ZRS). Here's what each level provides:

| Feature | What's Included | Zone Failure | Region Failure | Cost |
|---|---|---|---|---|
| **Default (ZRS)** | ✅ Multi-zone replication, auto failover <5 min, zero loss | ✅ Protected | — | Included |
| **Enhanced (GZRS)** | ✅ ZRS + geo-backup to secondary region | ✅ Protected | ⚠️ ≤15 min potential loss | +30-50% |

**Translation:** You're protected by default. GZRS is optional for extreme disaster scenarios.

---

## Redundancy Configuration Options

---

### 1. **ZRS (Zone-Redundant Storage)** — Default Out-of-Box ✅

**What it does:** Replicates data synchronously across 3+ zones in your region. If a zone fails, automatic failover in <5 minutes.

| Metric | Value |
|---|---|
| **What you get** | Built-in (no action needed) |
| **Zone failure protection** | ✅ Automatic, zero data loss |
| **Region disaster protection** | Not needed for most orgs |
| **Failover time** | <5 minutes (automatic) |
| **Durability** | 12 nines annual |
| **Data loss risk** | ✅ None for zone-level events |

**Best for:** Most organizations' Sentinel deployments (default choice)

---

### 2. **GZRS (Geo-Zone-Redundant Storage)** — Optional Enhancement for Max Resilience

**What it does:** Same as ZRS, PLUS asynchronous backup to a secondary region 100+ miles away.

| Metric | Value |
|---|---|
| **What you get** | ZRS + regional backup |
| **Zone failure protection** | ✅ Automatic, zero data loss |
| **Region disaster protection** | ✅ Manual failover, ≤15 min RPO |
| **Failover time** | Auto <5 min (zone) + 5-30 min manual (region) |
| **Durability** | 16 nines annual |
| **Cost** | ~30-50% storage premium |

**Best for:** Mission-critical environments with extreme resilience requirements

---

## Getting Started with Sentinel

### New Deployment — You're Protected by Default

When you create a new Sentinel workspace, you immediately get **zone-redundant protection out of the box** — no extra configuration needed.

**If you want enhanced regional resilience**, you can optionally enable GZRS:

**Via Azure Portal:**
1. Create Log Analytics Workspace (standard process)
2. Go to workspace → **Storage account** → **Redundancy**
3. Click **"Change to Geo-zone-redundant storage"** (optional)
4. Confirm (cost: ~30-50% storage premium)

**Via Terraform / Bicep (IaC) — Recommended for Enterprise:**

```bicep
// This creates a Sentinel workspace with optional GZRS
resource logAnalyticsWorkspace 'Microsoft.OperationalInsights/workspaces@2021-12-01-preview' = {
  name: workspaceName
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'  // Sentinel pricing tier
    }
  }
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-09-01' = {
  name: storageAccountName
  location: location
  kind: 'StorageV2'
  sku: {
    name: 'Standard_GZRS'  // ← Optional: Add geo-redundancy
  }
}
```

### Existing Deployments — Check Your Protection Level

To verify you have the protection you expect:

```bash
# Check current redundancy
az storage account show \
  --resource-group "rg-name" \
  --name "storage-account-name" \
  --query "sku.name" \
  --output tsv
```

**You'll see:**
- `Standard_ZRS` = ✅ Protected (zone-level, included)
- `Standard_GZRS` = ✅✅ Protected (zone + region-level, optional)

**If you want to enhance to GZRS:**

```bash
az storage account update \
  --name "storage-account-name" \
  --resource-group "rg-name" \
  --sku Standard_GZRS
```

**Time required:** 5-10 minutes (minimal impact)

---

---

## Platform Responsibility Matrix

**Reference:** [Azure Monitor Data Security & Platform Responsibility](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/best-practices-security#how-microsoft-secures-azure-monitor)

| What | Microsoft Handles | You Optionally Choose |
|---|---|---|
| **Data replication within zone** | ✅ Automatic (ZRS default) | — |
| **Encryption at rest** | ✅ Enabled by default | ⚠️ Can upgrade to customer-managed keys |
| **Automatic zone failover** | ✅ <5 minutes | — |
| **Backup and recovery** | ✅ Built-in | ⚠️ Set retention policy (30-12 yrs) |
| **Data immutability** | ✅ Append-only logs | ⚠️ Can request purge for compliance |
| **Enhanced geo-redundancy (optional)** | — | ⚠️ Enable GZRS for region-level protection |
| **Regional failover (if GZRS enabled)** | — | ⚠️ Customer initiates if needed |

---

## Service Level Objectives (SLOs)

**Reference:** [Geo Priority Replication & RPO](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy-priority-replication)

When a failure occurs, here's what to expect:

| Scenario | Time to Recovery | Data Loss Risk | Notes |
|---|---|---|---|
| **Zone fails (default ZRS)** | <5 min (automatic) | ✅ None | Transparent failover via DNS |
| **Region fails (with GZRS)** | 5-30 min (manual failover) | ⚠️ ≤15 min of new logs | Customer initiates failover |
| **Normal operations** | — | — | Continuous protection, zero overhead |

**Translation:** Out of the box, you're protected for zone-level failures (most common). Regional disasters are rare and optional to protect against.

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

### **Standard: ZRS (Out-of-Box Default)**
- **What you get:** Zone-level protection, automatic failover
- **Best for:** Most organizations' Sentinel deployments
- **Cost:** Included (no premium)
- **Protection level:** ✅ Zone-level disasters (common)

### **Enhanced: GZRS (Optional Upgrade)**
- **What you get:** Zone-level + region-level protection
- **Best for:** Mission-critical environments, forensics data, regulatory requirements
- **Cost:** ~30-50% storage premium (typically <$200/month for most deployments)
- **Protection level:** ✅ Zone-level + ⚠️ Region-level (rare but catastrophic)

---

## Next Steps

### **For New Sentinel Deployments**
- [ ] Create your Log Analytics Workspace (zone-redundant protection is automatic)
- [ ] If your org requires regional resilience: Enable GZRS during or immediately after creation
- [ ] Document your redundancy choice in your deployment runbook

### **For Existing Sentinel Deployments**
- [ ] Verify your current protection level (`Standard_ZRS` or `Standard_GZRS`)
- [ ] If you want enhanced geo-redundancy: Upgrade to GZRS (5-10 min, minimal impact)
- [ ] Document your failover procedures in case of regional disaster

### **Ongoing**
- [ ] Subscribe to Azure Service Health alerts for your region
- [ ] Review data retention policies (30 days to 12 years configurable)
- [ ] Test failover procedures annually (dev/test environment)

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

