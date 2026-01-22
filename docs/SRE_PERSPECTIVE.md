# SRE Perspective: Ethereum Archive Node

## What is the SLO for archive queries?

**99.9% availability** (43 minutes downtime/month). Archive nodes serve forensic analysis and compliance queries—not real-time trading. Users tolerate scheduled maintenance if we communicate 48 hours ahead.

Performance-wise: **p99 latency < 5 seconds** for historical state queries. But honestly, **data integrity** is the real SLO. If we're serving corrupted state, the node is useless regardless of uptime.

**How the Operational Plan Addresses This:**
- **Scheduled maintenance windows** (first Saturday, 2-8 AM UTC) fit within the 43-minute budget
- **RAID 0 with Azure LRS** ensures data durability without sacrificing query performance (240,000 IOPS)
- **Premium SSD v2** delivers <0.5ms latency for historical lookups
- **Monitoring stack** (Azure Monitor + Prometheus) tracks `eth_blockNumber` sync status and query latency to ensure SLO compliance

---

## What breaks the error budget?

Two things: **bad configs** and **storage saturation**.

Config errors are the risk of single-VM simplicity. One bad `systemd` unit file takes down the whole service. We mitigate with Git-versioned Ansible playbooks—rollback in seconds if needed.

Storage is the bigger threat. Archive nodes hit 240,000 IOPS during query spikes. If we saturate disk throughput, latency balloons and we start timing out queries.

**How the Operational Plan Addresses This:**
- **Configuration Management via Ansible**: All changes go through Git; known-good playbooks enable instant rollback
- **RAID 0 (3x 8TB) + Premium SSD v2**: 240,000 total IOPS headroom prevents saturation(LRS already replicates data at the storage layer)
- **Azure Monitor disk metrics**: 60-second polling catches IOPS/throughput issues before they impact users
- **API rate limiting** (nginx, 100 req/min per IP): Prevents client-side DoS from burning error budget
- **Capacity alerts at 85%**: Gives time to expand storage via Terraform before hitting hard limits

---

## What failures are acceptable?

**Service restarts** are fine. Geth or Lighthouse crashing? `systemd` auto-restarts in <60 seconds. No big deal.

**Scheduled compaction** (monthly, 5-7 hours) is also acceptable. It's planned downtime—users know it's coming, and it keeps the disk from bloating.

What's **not** acceptable? Losing the 24TB of chaindata. Re-syncing from genesis takes weeks. That's a total service failure.

**How the Operational Plan Addresses This:**
- **Systemd auto-restart**: Built into service definitions; failures self-heal in <1 minute
- **3-tier backup strategy**:
  - **Hot**: Azure Managed Disk Snapshots (every 4 hours, 30-min RTO)
  - **Warm**: Weekly Azure Blob backups (8-hour RTO)
  - **Cold**: Monthly Archive tier backups (compliance, 24-48 hr RTO)
- **Quarterly DR drills**: Verify backups actually work; checksum validation after every backup
- **Scheduled compaction via Ansible**: `compact_scheduled.yml` automates the process, including status page updates

---

## What pages an on-call engineer?

Only what **Ansible EDA can't fix**. I've tuned PagerDuty to be conservative.

**Page-worthy scenarios:**
1. **Node down >5 minutes** after auto-restart fails (kernel panic, Azure outage)
2. **Disk at 85%** (needs manual Terraform expansion or emergency compaction)
3. **Sync stalled >1 hour** (network partition or client bug—needs human eyes)

Everything else (CPU spikes, low peers, high memory) goes to Slack. to checkt during business hours.

**How the Operational Plan Addresses This:**
- **Ansible Event-Driven Automation (EDA)**:
  - Listens to Azure Monitor webhooks
  - Auto-executes remediation playbooks (restart services, clear caches, reset peer connections)
  - Only escalates to PagerDuty if automation fails
- **Smart alerting thresholds**:
  - Critical: Node down (>5 min), disk >85%, sync stalled (>1 hour)
  - Warning: CPU >80%, peers <20, memory >90% → Slack only
- **Self-healing reduces pages by ~80%**: Transient issues (service crashes, memory spikes) resolve automatically
- **Terraform for disk expansion**: When 85% alert fires, on-call engineer runs `terraform apply` to attach larger disk—no manual VM reconfiguration
