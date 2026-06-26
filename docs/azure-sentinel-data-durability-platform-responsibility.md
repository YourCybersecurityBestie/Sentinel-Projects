# Azure Sentinel Data Durability & Platform Responsibility

**Author:** Venicia Solomons, Cloud & AI Security SE  
**Date:** 2026-06-26  
**Audience:** Security Operations, Cloud Infrastructure, Compliance Teams

---

## Executive Summary

This document clarifies **Microsoft's platform responsibility vs. customer responsibility** for data durability in Azure Sentinel deployments. The key finding: **data loss is preventable with proper configuration**, but configuration choice remains a customer decision.

---

## Data Loss Scenarios

| Outage Duration | Data Loss | Recovery Time | Notes |
|---|---|---|---|
| **Single Zone / DC Failure (1-4 hours)** | **ZERO** | <5 min | Requires ZRS/GZRS; automatic failover |
| **Single Zone Extended (1+ days)** | **ZERO** | <5 min | Automatic failover restores access |
| **Entire Region (1 week+)** | **≤15 minutes** | 5-30 min | GZRS only; manual failover required |

---

## What Microsoft Guarantees

✅ **Data stored in Azure Storage** (backing all Sentinel Log Analytics workspaces)  
✅ **Zone-redundant replication** (automatic, synchronous within primary region)  
✅ **Geo-redundant backup** (asynchronous replication to secondary region, ≤15 min RPO)  
✅ **16 nines durability** (99.99999999999999% annual) with GZRS configuration  
✅ **Append-only, tamper-proof storage** (encryption at rest, Microsoft-managed keys by default)  
✅ **Automatic failover** (<5 min RTO) for zone-level failures  
✅ **Continuous data backups** and disaster recovery capability  

---

## What Customers Must Configure

⚠️ **Workspace Redundancy Level**

Three options available at deployment or upgrade:

### 1. **LRS (Locally Redundant Storage)** — Single Zone
- **Durability:** 11 nines (99.99999999%)
- **Zone Protection:** ❌ NONE
- **Regional Protection:** ❌ NONE
- **Risk:** Total data loss if single datacenter fails
- **Use Case:** Dev/test environments only (not recommended for production)

### 2. **ZRS (Zone-Redundant Storage)** — Multi-Zone Within Region
- **Durability:** 12 nines (99.9999999999%)
- **Zone Protection:** ✅ YES (automatic failover <5 min)
- **Regional Protection:** ❌ NONE
- **Risk:** Total data loss if entire region fails
- **Use Case:** Production deployments in single region

### 3. **GZRS (Geo-Zone-Redundant Storage)** — Multi-Zone + Geo-Redundant
- **Durability:** 16 nines (99.99999999999999%)
- **Zone Protection:** ✅ YES (automatic failover <5 min)
- **Regional Protection:** ✅ YES (≤15 min RPO, manual failover)
- **Risk:** Minimal — only ≤15 min data loss for catastrophic regional disaster
- **Use Case:** **Recommended for enterprise production** (SIEM/compliance workloads)

---

## Platform Responsibility Matrix

| Responsibility | Microsoft | Customer |
|---|---|---|
| **Data encryption at rest** | ✅ Handled | — |
| **Data replication within zone** | ✅ Handled | — |
| **Data backup/retention** | ✅ Handled | — |
| **Redundancy level selection** | — | ⚠️ **Must choose** |
| **Geo-replication configuration** | — | ⚠️ **Must enable** |
| **Regional failover decision** | — | ⚠️ **Manual action** |
| **Data retention period** | ✅ Configurable | ⚠️ **Customer sets** (default 30 days, up to 12 years) |
| **Encryption key management** | ✅ Microsoft-managed default | ⚠️ **Customer can upgrade to CMK** |

---

## Recovery Objectives

### **RTO (Recovery Time Objective)**
- **Zone failure:** <5 minutes (automatic)
- **Regional failure:** 5-30 minutes (manual account failover)

### **RPO (Recovery Point Objective)**
- **Zone failure:** 0 minutes (synchronous replication)
- **Regional failure:** ≤15 minutes (asynchronous geo-replication)

---

## How to Verify Current Configuration

### Azure Portal
1. Navigate to **Log Analytics Workspace** → **Properties**
2. Check "**Redundancy**" field
3. Current options: LRS, ZRS, GZRS (or STANDARD, PREMIUM if legacy)

### Azure CLI
```bash
az monitor log-analytics workspace show \
  --resource-group "<resource-group-name>" \
  --workspace-name "<workspace-name>" \
  --query "properties.sku.name" \
  --output tsv
```

### Expected Output
- `PerGB2018` (with redundancy configuration in separate field)
- Legacy: `Standard`, `Premium` (check redundancy separately)

---

## Recommendations for Sentinel Deployments

### **Minimum Production Standard: ZRS**
- ✅ Protects against single-zone failures
- ✅ Automatic failover, zero-loss recovery
- ⚠️ No protection against regional catastrophe
- 📌 Suitable for non-critical workloads or single-region deployments

### **Enterprise Recommended: GZRS**
- ✅ Zone-level protection (automatic, <5 min)
- ✅ Region-level protection (manual, ≤15 min RPO)
- ✅ Meets most compliance and SLA requirements
- ✅ 16 nines durability guarantee
- 📌 **Recommended for SIEM, compliance ingestion, regulated data**

---

## Cost Implications

| Storage Type | Relative Cost | RTO (Zone) | RTO (Region) |
|---|---|---|---|
| **LRS** | 1.0x (baseline) | ∞ (no protection) | ∞ (no protection) |
| **ZRS** | ~1.2x (20% premium) | <5 min | ∞ (no protection) |
| **GZRS** | ~1.3-1.5x (30-50% premium) | <5 min | 5-30 min |

*Exact pricing varies by region and data volume.*

---

## Next Steps

1. **Audit Current Configuration**
   - Query all Log Analytics workspaces
   - Document current redundancy settings
   - Identify any LRS-only deployments (high risk)

2. **Upgrade Production Workspaces to GZRS** (if not already)
   - Minimal downtime (can be done in-place)
   - Transparent to connected data sources
   - No breaking changes

3. **Document Failover Procedures**
   - Region-level failover is manual
   - Define runbooks and approval process
   - Test failover in pre-prod environment

4. **Monitor Replication Lag**
   - Use Azure Storage metrics to track geo-replication latency
   - Alert if RPO exceeds 15 minutes (indicates issues)

5. **Review Data Retention & Compliance**
   - Confirm retention period aligns with regulatory requirements
   - Consider long-term archive tier if required (up to 12 years)
   - Validate encryption key strategy (Microsoft-managed vs. customer-managed)

---

## References

- [Azure Storage Redundancy Documentation](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Azure Storage Geo Priority Replication (≤15 min RPO)](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy-priority-replication)
- [Azure Monitor Data Security & Platform Responsibility](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/best-practices-security#how-microsoft-secures-azure-monitor)
- [Disaster Recovery & Failover Guidance](https://learn.microsoft.com/en-us/azure/storage/common/storage-disaster-recovery-guidance)
- [Log Analytics Data Retention Configuration](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-retention-configure)
- [Microsoft Sentinel Data Connectors](https://learn.microsoft.com/en-us/azure/sentinel/connect-data-sources)

---

## Questions?

Contact: Venicia Solomons, Cloud & AI Security SE  
Internal Channel: [Security Architecture Team]

