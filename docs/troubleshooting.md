# Troubleshooting & Root Cause Narratives

This document captures the real engineering decisions, constraints, and validation work done during the build — not just "what was deployed" but **how it was tested and what went wrong along the way**.

---

## Incident 1: Validating Traffic Manager DR Failover (South India → Central US)

### Symptom
Needed to prove that if the **primary regional endpoint (South India)** became unavailable, the **`dealer` Traffic Manager profile** would automatically reroute traffic to the **secondary endpoint (Central US)** — without manual intervention.

### Evidence Gathering
From a Windows client, the South India VM's public endpoint (`135.13.16.54`) was taken offline (VM stopped/unreachable), then connectivity and DNS resolution were tested:

```cmd
C:\Windows\System32>ping 135.13.16.54

Pinging 135.13.16.54 with 32 bytes of data:
Request timed out.
Request timed out.
Request timed out.
Request timed out.

Ping statistics for 135.13.16.54:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss)
```

```cmd
C:\Windows\System32>nslookup dealer.trafficmanager.net
Server:  UnKnown
Address:  fe80::1227:f5ff:fe82:f518

Non-authoritative answer:
Name:    vm-centralus-d3745b.centralus.cloudapp.azure.com
Address:  13.89.72.193
Aliases:  dealer.trafficmanager.net
```

### Investigation
- In the **Traffic Manager → Endpoints** blade for profile `dealer`, the **`SouthIndiaEP`** endpoint status changed from `Enabled / Online` to **`Enabled / Degraded`** — confirming Traffic Manager's HTTP health probe detected the outage.
- `CentralUSEP` remained `Online`.
- Despite South India being **Priority 1** (the primary), the **DNS answer for `dealer.trafficmanager.net` resolved to the Central US endpoint** (`13.89.72.193` / `vm-centralus-d3745b.centralus.cloudapp.azure.com`).

### Root Cause
South India VM/endpoint was unreachable → Traffic Manager's health checks (run on a configurable interval) detected repeated probe failures → endpoint automatically marked **Degraded** and removed from the active resolution set → Traffic Manager fell back to the next-healthy endpoint by priority (Central US).

### Resolution / Validation
- No manual DNS changes were required — **this is exactly the intended behaviour of Priority routing**.
- Confirmed recovery path: once the South India VM/endpoint is healthy again and passes probes, Traffic Manager will mark it `Online` again and resume routing to it as Priority 1 (Traffic Manager re-checks endpoints on its configured probe interval).

### Lessons Learned
- **DNS-based failover has a propagation delay** — driven by the DNS TTL set on the Traffic Manager profile and any client/resolver caching. This is an important trade-off vs. an L4/L7 load balancer (instant re-routing) and should be called out when discussing RTO (Recovery Time Objective) for this design.
- Always validate failover **from the client's perspective** (DNS resolution + actual connectivity), not just by looking at green/red status icons in the portal — the portal status can lag slightly behind real-world DNS propagation.
- `ping` + `nslookup` are simple but effective tools for proving HA/DR behaviour during a demo or interview — they produce evidence you can screenshot and reference later.

---

## Incident 2: Subscription Quota Limits Forced an Architecture Pivot

### Symptom
The original design called for **multiple VMs per region inside each Availability Set**, fronted by an **internal Azure Load Balancer**, to demonstrate intra-region load balancing in addition to cross-region failover.

### Investigation
- Attempting to deploy a second VM into `AVSET-CentralUS` / `AVSET-SouthIndia` failed due to **CPU core quota limits on the free-tier/trial subscription** (each region has a default vCPU quota that a B-series VM plus a second instance would exceed).
- Confirmed via the Azure Portal deployment error referencing the subscription's regional vCPU quota, and via `Subscriptions → Usage + quotas`.

### Root Cause
A **trial/free-tier Azure subscription enforces low default vCPU quotas per region** (commonly low single digits), which is sufficient for one VM per region but not for a second instance in the same Availability Set, nor for the extra compute an internal Load Balancer's backend pool would need.

### Decision & Resolution
Rather than abandon the HA/DR requirement, the design was **adapted to a pattern that doesn't depend on multiple VMs per region**:

| Original plan | What was deployed instead | Why it still satisfies the requirement |
|---|---|---|
| 2+ VMs per region behind an internal Load Balancer | 1 VM per region, each in its own Availability Set | Availability Sets are still configured correctly (2 FD / 5 UD) and are ready to scale out when quota allows |
| Internal Load Balancer for intra-region balancing | Azure Traffic Manager for **inter-region** failover/routing | Traffic Manager satisfies the "route around a failed region" requirement at the DNS layer, and adds a second capability (geographic routing) the original design didn't have |

### Lessons Learned
- **Subscription-level constraints are a normal part of real engineering** — knowing how to identify a quota error, locate the relevant quota in the portal, and re-architect around it is itself a demonstrable skill.
- This is a good interview talking point: *"I designed for X, hit a hard platform constraint, and re-scoped to Y while still meeting the original availability requirement — here's the trade-off I made and why."*
- In a production subscription, the fix would simply be to **request a quota increase** via Azure support — worth mentioning that the workaround was a constraint of the *learning environment*, not the architecture itself.

---

## Incident 3: Confirming Blob Storage Was Not Publicly Accessible, Then Granting Scoped Access

### Symptom
Needed to confirm that uploaded dealer documents (`Dealer_Handbook.pdf`) in the `dealer-docs` container were **not** accessible via a plain public URL, then provide a **controlled** way for external dealers to download them.

### Investigation
1. Copied the blob's direct URL (`https://dealerdataorg.blob.core.windows.net/dealer-docs/Dealer_Handbook.pdf`) and requested it directly in a browser.
2. Azure Storage returned an XML error response:

```xml
<Error>
  <Code>PublicAccessNotPermitted</Code>
  <Message>Public access is not permitted on this storage account.
  RequestId:46480bf6a-201e-0019-6671-a75ea3000000
  Time:2025-04-07T03:56:54</Message>
</Error>
```

### Root Cause
The storage account's **"Allow Blob public access"** setting was disabled (the secure default) and the container's access level was set to **Private (no anonymous access)** — so any unauthenticated request, even with a "correct" URL, is rejected at the storage service level.

### Resolution
- Used **Storage Browser → Generate SAS** on the `Dealer_Handbook.pdf` blob to create a **Shared Access Signature** scoped to:
  - **Permissions:** Read-only
  - **Allowed protocols:** HTTPS only
  - **Start/Expiry window:** a defined validity period (not indefinite)
  - **Signing key:** Key 1 (account key)
- Distributed the resulting **SAS URL** (containing a `sig=` token) instead of the bare blob URL.
- Re-tested: the SAS URL opened the PDF successfully in-browser, confirming scoped, time-limited access worked while the bare URL remained blocked.

### Lessons Learned
- This is a clean demonstration of **defense-in-depth for storage**: deny by default (no public access), then grant the *minimum necessary* access (read-only, HTTPS-only, time-bound) via SAS — rather than re-enabling public access or sharing the storage account key.
- For a future iteration, **Azure AD-based access (RBAC + Managed Identity / user delegation SAS)** would be a stronger pattern than account-key-based SAS, since it avoids long-lived shared secrets — noted in [Future Improvements](lessons-learned-and-future-improvements.md).

---

## Incident 4: Validating the Monitoring/Alerting Pipeline End-to-End

### Symptom
Alert rules were created (CPU, memory, disk IOPS, network, VM availability) but needed proof that the **full pipeline** — metric → alert rule → fired alert → Log Analytics activity log — actually worked, not just that the rules existed.

### Investigation
- Checked **Monitor → Alerts** with time range "Past 24 hours": showed **Total alerts = 1**, broken down as 1 Informational (Severity 3), 0 Critical/Error/Warning.
- The fired alert was **`VMDown_CUS`**, condition `VmAvailabilityMetric < 1`, affected resource `vm-centralus`, alert condition = **Fired**.
- Cross-checked the **Log Analytics Workspace (`DealerMonitoringWorkspace`) → Activity log**, which showed corresponding workspace operations (`Create Workspace`, `List Workspace Shared Keys`) succeeding, confirming the workspace was active and receiving data.

### Root Cause / Conclusion
Not a "failure" — this was a **deliberate validation step**. The `VMAvailabilityMetric < 1` alert firing for `VM-CentralUS` confirmed that:
1. The metric alert rule was correctly scoped to the right VM,
2. Azure Monitor correctly evaluated the condition,
3. The alert correctly fired and was visible in the Alerts blade with the right severity and timestamp.

### Lessons Learned
- **"Trust, but verify" applies to monitoring just as much as to networking** — an alert rule that has never fired hasn't been proven to work. Deliberately triggering each alert type (e.g., stopping a VM to trigger an availability alert) is good practice before relying on it in production.
- Severity levels matter for noise control: a VM availability blip during testing was correctly informational (Severity 3), not Critical — worth discussing how severity mapping should change for production alerts (e.g., paging on Sev 1/2 only).
