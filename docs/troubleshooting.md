# Troubleshooting Notes

Things that didn't just work the first time, and what I did about it.

---

## Testing the failover actually works

Setting up Traffic Manager with two endpoints is easy. Trusting that it actually fails over correctly is a different thing, so I tested it instead of just assuming the green checkmarks in the portal meant it was working.

I stopped the South India VM so its endpoint would go offline, then checked what happened from a client machine.

First, a ping to the South India public IP:

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

That confirmed the VM was actually unreachable. Then I checked what `dealer.trafficmanager.net` (the Priority routing profile) was resolving to:

```cmd
C:\Windows\System32>nslookup dealer.trafficmanager.net
Server:  UnKnown
Address:  fe80::1227:f5ff:fe82:f518

Non-authoritative answer:
Name:    vm-centralus-d3745b.centralus.cloudapp.azure.com
Address:  13.89.72.193
Aliases:  dealer.trafficmanager.net
```

So instead of resolving to South India (which was Priority 1, the main region), it had switched over to the Central US VM. In the Traffic Manager portal, the South India endpoint showed as "Degraded" and Central US showed as "Online" — which matched what the DNS lookup showed.

This is just Traffic Manager doing its job — it runs health checks against each endpoint, and if one stops responding, it stops sending traffic there. No alarms, no manual intervention needed. The thing I actually learned from this is that failover here happens at the DNS level, which means it's not instant — it depends on DNS caching and TTL, not something that switches over the second the VM goes down. That's different from how a load balancer would behave, and it's something worth knowing if someone asks me about failover speed.

---

## Hitting a subscription quota wall

My original plan was to have multiple VMs per region behind an internal load balancer, so I could show both intra-region load balancing and inter-region failover. When I tried adding a second VM into the same Availability Set, it failed — turns out my free-tier subscription has a low default vCPU quota per region, and one B2s VM was already close to using it up.

I checked under Subscriptions → Usage + quotas and confirmed that was the issue, not something I'd misconfigured.

Rather than give up on the HA requirement, I changed the design: instead of load balancing within a region, I leaned fully on Traffic Manager to route between regions. One VM per region, each in its own Availability Set (still configured properly with fault domains and update domains, just not stretched across multiple VMs yet). It's a smaller setup than I originally planned, but it still meets the actual requirement — if one region goes down, traffic moves to the other one. On a paid subscription I'd just request a quota increase and add the extra VMs.

---

## Making sure blob storage was actually locked down

I wanted dealers to be able to download a handbook PDF, but I didn't want it to be publicly accessible to anyone with the link. Before generating a SAS token, I checked that public access really was off by just trying to open the blob's direct URL in a browser:

```xml
<Error>
  <Code>PublicAccessNotPermitted</Code>
  <Message>Public access is not permitted on this storage account.
  RequestId:46480bf6a-201e-0019-6671-a75ea3000000
  Time:2025-04-07T03:56:54</Message>
</Error>
```

Good — that confirmed the storage account setting was actually doing what I thought it was doing. Then I went into Storage Browser, selected the blob, and generated a SAS token scoped to read-only, HTTPS only, with a start and expiry time set. Tested that link and it opened the PDF fine, while the plain URL still didn't work.

The thing I'd improve here later: SAS tokens are still a shared secret — anyone with the link and before the expiry can access it. Longer term I'd look at Azure AD-based access (RBAC + Managed Identity) instead, since that's tied to an identity rather than a link.

---

## Making sure the alerts I set up actually fire

I created alert rules for CPU, memory, disk, network, and VM availability across both regions, but an alert rule that's never fired hasn't really been tested. So I stopped a VM on purpose to trigger the `VMDown_CUS` availability alert.

Checked Monitor → Alerts and saw it show up: 1 alert in the last 24 hours, Severity 3 (Informational), condition "Fired", on the right VM. Also checked the Log Analytics workspace activity log to confirm it was receiving data properly.

This was a good reminder that "I set up monitoring" and "I confirmed monitoring works" are two different claims, and it's worth being able to back up the second one.
