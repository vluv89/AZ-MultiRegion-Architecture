# Lessons Learned & What's Next

## What I learned doing this

- DNS-based failover (Traffic Manager) works, but it's not instant — it depends on DNS caching and TTL on the client side, not just how fast Traffic Manager detects the outage. If someone asks about recovery time, that's the honest answer.
- Priority routing and Geographic routing solve different problems even though they're both "Traffic Manager routing methods." Priority is about availability (failover when something breaks). Geographic is about sending people to the right place regardless of health (compliance, latency, etc.). I hadn't really separated those two ideas in my head until I configured both side by side.
- Subscription quotas are a real constraint, not just a settings screen. Hitting the free-tier vCPU limit forced me to rethink the design, and the result (Traffic Manager handling inter-region failover instead of an internal load balancer handling intra-region balancing) ended up being a reasonable multi-region pattern anyway.
- It's worth actually testing things instead of trusting the portal's status icons — the SAS token / public access test and the alert-firing test were both cases where I proved something worked instead of assuming it did.

## What I'd do differently next time

- Build this with Terraform instead of clicking through the Portal. That's a separate project I'm planning, so I can compare doing it manually vs. as code.
- Look at Azure Front Door or Application Gateway instead of (or alongside) Traffic Manager — DNS failover is fine but a real load balancer would fail over faster.
- If I had a paid subscription with more quota, scale each region to more than one VM, or move to VM Scale Sets, so there's actual load balancing within a region and not just failover between them.
- Swap SAS tokens for Azure AD-based access (RBAC + Managed Identity) on the storage account, so there's no shared secret floating around.
- Add Azure Backup or Site Recovery for VM-level recovery on top of the DNS failover that's already there.
- Build a simple dashboard (Workbook or Grafana) on top of the Log Analytics data instead of checking each metric individually.
