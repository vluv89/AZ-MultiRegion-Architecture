# Lessons Learned & Future Improvements

## Lessons Learned

1. **DNS-based failover (Traffic Manager) is a real, low-cost DR pattern** — but it's not instant. RTO is bounded by DNS TTL and resolver/client caching, not by Traffic Manager's probe interval alone. Any HA/DR design using Traffic Manager needs to state this trade-off explicitly.

2. **Two routing methods, two different problems.** Configuring both a **Priority** profile (`dealer`) and a **Geographic** profile (`dealer2`) on the same backend infrastructure made the distinction concrete:
   - *Priority* = "always prefer Region A unless it's down" → availability-driven.
   - *Geographic* = "send users to the region closest/most relevant to them" → performance/compliance-driven (e.g. data residency).

3. **Subscription quotas are part of the architecture, not just an annoyance.** Hitting the free-tier vCPU quota forced a pivot from "multiple VMs + internal Load Balancer per region" to "single VM per region + Traffic Manager for inter-region failover." The resulting design is arguably **more representative of a real multi-region DR pattern** than the original plan would have been.

4. **Security defaults are worth testing, not just configuring.** Reproducing the `PublicAccessNotPermitted` error before issuing a SAS token proved the storage account's secure-by-default posture was actually in effect — not just selected in a dropdown.

5. **An alert rule isn't "done" until it has fired at least once.** Deliberately causing `VMDown_CUS` to fire validated the entire monitoring chain end-to-end (metric → rule → alert → Log Analytics).

---

## Future Improvements

### Reliability / HA
- Move from **Availability Sets** to **Availability Zones + Virtual Machine Scale Sets (VMSS)** for stronger intra-region resiliency once subscription limits allow.
- Add **Azure Backup** and/or **Azure Site Recovery** for VM-level disaster recovery in addition to the application-layer DNS failover already in place.
- Introduce **Azure Front Door** or **Application Gateway** for L7 global routing — faster failover than DNS TTL, plus WAF and SSL offload.

### Security
- Replace **account-key-based SAS tokens** with **Azure AD–based access** (RBAC + Managed Identity, or user-delegation SAS) to remove long-lived shared secrets from the document distribution flow.
- Apply **Azure Policy** to enforce "public blob access disabled" and "NSG required on every subnet" at the subscription level, rather than relying on per-resource configuration.
- Enable **Microsoft Defender for Cloud** recommendations across both resource groups and remediate the top findings.

### Cost Optimization
- Right-size the B2s VMs based on actual CPU/memory utilization data from the Log Analytics workspace once real traffic patterns exist.
- Evaluate **Azure Hybrid Benefit** for the Windows Server licensing if this were extended with on-prem licenses.
- Consider **Auto-shutdown schedules** for non-production VMs (already available as a VM setting) to control dev/test spend.

### Observability
- Build a **Workbook or Grafana dashboard** on top of `DealerMonitoringWorkspace` for a single-pane-of-glass view of both regions (CPU, memory, disk, network, availability, Traffic Manager endpoint health).
- Add **Application Insights** if/when the static site is replaced with a real application tier, for request-level tracing.

### Infrastructure as Code
- This project was built manually via the Azure Portal to focus on understanding each service in isolation. The natural next step — **rebuilding this exact architecture in Terraform** — is tracked as a **separate follow-up project** (see proposal document provided alongside this repo) to demonstrate IaC, state management, and repeatable deployments.
