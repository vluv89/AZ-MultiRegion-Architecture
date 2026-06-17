# LinkedIn Project Description

**Project name:** Azure Multi-Region High Availability & Disaster Recovery Architecture
**Associated with:** Personal cloud engineering study (AZ-104 / Azure Administrator)

---

## Description (paste into LinkedIn "Projects" section)

Designed and built a multi-region Azure infrastructure for a customer-facing web application, simulating an on-prem-to-cloud migration with high availability and disaster recovery requirements.

**What I built:**
- Two peered VNets across Central US and South India, each with its own subnet, NSG, and Windows Server VM running an identical web app, placed in region-specific Availability Sets.
- Azure Traffic Manager configured with both Priority routing (for automatic DR failover) and Geographic routing (for region-aware traffic distribution) — and validated live failover by taking a regional endpoint offline and confirming DNS rerouted to the healthy region.
- Active Directory migration from an on-prem model to an Azure-hosted domain controller (AD DS + DNS).
- Secure document distribution via Blob Storage with public access disabled and SAS-token-based access.
- Cross-region monitoring and alerting via Azure Monitor and Log Analytics, covering compute, disk, network, and VM availability metrics — validated end-to-end by triggering and confirming a live alert.

**A real constraint I worked through:** a free-tier subscription's vCPU quota ruled out the original "multiple VMs + internal Load Balancer per region" design. I re-architected to a Traffic Manager–based inter-region failover model instead — which ended up being a more realistic representation of a multi-region DR pattern.

Full write-up, architecture diagrams, troubleshooting notes, and screenshots: [GitHub link]

---

## Shorter version (for a "Featured" link caption / headline area)

Built a 2-region Azure HA/DR architecture (peered VNets, Traffic Manager failover, SAS-secured storage, cross-region monitoring) and validated live failover with `ping`/`nslookup`. Full breakdown on GitHub →
