# Interview Talking Points

Likely questions this project prepares you for, grouped by topic, with the angle you should take when answering.

---

## Networking

**Q: How does VNet Peering work, and what are its limits?**
- Talk through: non-transitive peering (A↔B and B↔C doesn't mean A↔C), no overlapping address spaces allowed, peering vs. VPN Gateway (peering = backbone, no encryption overhead; VPN Gateway = needed for on-prem connectivity or transitive routing via NVA).
- Reference: you configured `VNet-CentralUS` (10.1.0.0/16) ↔ `VNet-SouthIndia` (10.2.0.0/16) peering and verified "Connected" status on both sides.

**Q: What's the difference between an NSG and a firewall/Azure Firewall?**
- NSG = stateful, rule-based filtering at the subnet/NIC level (L3/L4). Azure Firewall = managed, centralized, supports FQDN filtering, threat intelligence, and works across VNets/hubs.
- You used per-region NSGs (`NSG-CentralUS`, `NSG-SouthIndia`) — a good entry point to discuss when you'd escalate to a hub-and-spoke + Azure Firewall design.

---

## High Availability & Disaster Recovery

**Q: Walk me through how Traffic Manager failover actually works.**
- Explain the health-probe model (HTTP/HTTPS/TCP probes on an interval), endpoint status states (Online/Degraded/Disabled), and that **Traffic Manager operates at the DNS layer** — it doesn't proxy traffic.
- Use your validated example: South India endpoint went `Degraded`, `dealer.trafficmanager.net` started resolving to the Central US VM, confirmed via `nslookup`.
- Be ready to discuss **why this isn't instant** (DNS TTL, client/resolver caching) and how that compares to a Load Balancer or Front Door failover (much faster, but operates differently — L4 vs L7 vs DNS).

**Q: Priority vs Geographic vs Performance vs Weighted routing — when would you use each?**
- Priority = active/passive DR.
- Geographic = data residency / compliance / "send EU users to EU region."
- Performance = latency-based, "send users to the lowest-latency healthy endpoint."
- Weighted = canary releases / gradual traffic shifting.
- You implemented both **Priority** (`dealer`) and **Geographic** (`dealer2`) on the same backend — good story for "I understand these aren't interchangeable, they solve different problems."

**Q: What is an Availability Set, and how is it different from an Availability Zone or VMSS?**
- Availability Set = fault domains (shared power/network) + update domains (patching groups) **within a single datacenter**.
- Availability Zone = physically separate datacenters within a region (protects against datacenter-level failure).
- VMSS = managed group of identical VM instances with autoscaling, can span zones.
- Be honest about the constraint: you configured Availability Sets correctly (2 FD / 5 UD) but a free-tier quota limited you to one VM per set — and explain what you'd do differently with more quota.

---

## Storage & Security

**Q: How does a SAS token work, and why not just share the storage account key?**
- SAS = a signed URL granting scoped permissions (read/write/list), over a defined protocol (HTTPS-only), within a defined time window — without exposing the account key.
- Account key = full control over the entire storage account; if leaked, everything is compromised and must be rotated.
- You can describe reproducing `PublicAccessNotPermitted` first, then issuing a read-only, time-bound SAS — i.e., **deny by default, grant minimum necessary access**.

**Q: What's the difference between a SAS token and Azure AD-based access (RBAC)?**
- SAS = shared secret model (anyone with the URL has access until expiry/revocation).
- Azure AD/RBAC = identity-based, auditable per-principal access, supports Managed Identities (no secrets at all).
- Flag this as a known improvement area for this project — shows self-awareness about production-readiness.

---

## Identity

**Q: What does "migrating AD to Azure" actually involve, and what are the options?**
- Differentiate: **AD DS on an Azure VM** (what you built — lift-and-shift of the directory itself) vs. **Azure AD Connect / hybrid identity** (syncing on-prem AD to Azure AD for SaaS/M365 access) vs. **Azure AD DS** (managed domain service, no VM management).
- You stood up `DCVM1` running AD DS + DNS — be ready to discuss when a full AD DS VM makes sense vs. Azure AD DS vs. just using Azure AD natively.

---

## Monitoring

**Q: How do you know your alerts actually work?**
- This is your strongest story: you didn't just create alert rules — you **triggered `VMDown_CUS`**, confirmed it appeared in Monitor → Alerts with the correct severity, and cross-checked the Log Analytics activity log.
- Extend it: discuss alert severity design (Sev 0–4), action groups (email/SMS/webhook/runbook), and noise reduction for production environments.

---

## "Tell me about a time you hit a constraint and had to adapt"

This is the **subscription quota / architecture pivot** story (Incident 2 in `docs/troubleshooting.md`):
1. Original design needed multiple VMs per Availability Set + internal Load Balancer.
2. Hit a free-tier vCPU quota limit.
3. Re-scoped to single-VM-per-region + Traffic Manager for inter-region failover — which still met the HA/DR requirement (arguably more realistically).
4. Documented the trade-off and what you'd do with a production subscription (request quota increase, add VMSS).

This answers behavioral questions about problem-solving, adaptability, and communicating trade-offs — without needing a "real job" story.
