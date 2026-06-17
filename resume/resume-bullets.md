# Resume Bullet Points

**Project title for resume:** *Azure Multi-Region High Availability & Disaster Recovery Architecture* (Personal Cloud Project)

> Place under a "Personal Cloud Projects" / "Independent Infrastructure Projects" section — not under employment history.

---

## Full bullet set (pick 3–5 depending on role)

- Designed and deployed a dual-region (US Central / South India) Azure infrastructure for a customer-facing web application, including VNets, subnets, NSGs, and bidirectional VNet Peering for secure cross-region connectivity.

- Configured Azure Traffic Manager with **Priority** and **Geographic** routing profiles to provide automatic regional failover and latency/region-aware traffic distribution; validated live failover behaviour using `ping` and `nslookup` after simulating a regional outage.

- Migrated on-premises Active Directory user/group management and DNS to an Azure-hosted domain controller (AD DS + DNS Server), reducing reliance on on-prem identity infrastructure.

- Implemented secure document distribution via Azure Blob Storage by disabling public access at the account/container level and issuing scoped, HTTPS-only, time-limited SAS tokens for external access.

- Built monitoring and alerting across two Azure regions using Azure Monitor and a Log Analytics workspace, covering CPU, memory, disk IOPS, network throughput, and VM availability — and validated the alert pipeline by triggering and confirming a live alert.

- Diagnosed and resolved a subscription vCPU quota limitation by re-architecting the planned multi-VM/internal-load-balancer design into a Traffic Manager–based multi-region failover pattern, without compromising the original availability requirement.

- Configured Availability Sets (fault domains / update domains) per region to align VM placement with Azure's hardware fault-isolation model.

---

## Condensed 2-line version (for tight resume space)

> Designed and deployed a multi-region (Central US / South India) Azure architecture with peered VNets, Availability Sets, and Traffic Manager–based DR failover (Priority + Geographic routing), validated via live failover testing; secured document distribution with SAS-scoped Blob Storage and built cross-region Azure Monitor / Log Analytics alerting.

---

## Skills keyword bank (for ATS matching)

Azure Virtual Network (VNet), VNet Peering, Network Security Groups (NSG), Azure Virtual Machines, Availability Sets, Azure Traffic Manager, DNS, Disaster Recovery (DR), High Availability (HA), Active Directory Domain Services (AD DS), Azure Blob Storage, Shared Access Signature (SAS), Azure Monitor, Log Analytics, Azure AD, Windows Server 2016, Multi-Region Architecture, Failover Testing, Cloud Security
