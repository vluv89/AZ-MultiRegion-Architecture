# Architecture Diagrams

A closer look at how everything connects, and what the failover test actually looked like step by step.

## 1. Network & Infrastructure Topology

```mermaid
graph TB
    User[("Global Users")] --> TM["Azure Traffic Manager<br/>Profiles: dealer (Priority) / dealer2 (Geographic)"]

    subgraph CUS["Central US — RG-CentralUS"]
        direction TB
        VNetCUS["VNet-CentralUS<br/>10.1.0.0/16"] --> SubCUS["Subnet-CentralUS"]
        SubCUS --> NSGCUS["NSG-CentralUS"]
        SubCUS --> VMCUS["VM-CentralUS<br/>Windows Server 2016 (Standard B2s)<br/>Dealer Portal Site (IIS)"]
        VMCUS --> AVSETCUS["AVSET-CentralUS<br/>2 Fault Domains / 5 Update Domains"]
        VMCUS --- PIPCUS["Public IP: 13.89.72.193<br/>vm-centralus-d3745b.centralus.cloudapp.azure.com"]
    end

    subgraph STI["South India — RG-SouthIndia"]
        direction TB
        VNetSTI["VNet-SouthIndia<br/>10.2.0.0/16"] --> SubSTI["Subnet-SouthIndia"]
        SubSTI --> NSGSTI["NSG-SouthIndia"]
        SubSTI --> VMSTI["VM-SouthIndia<br/>Windows Server 2016 (Standard B2s)<br/>Dealer Portal Site (IIS)"]
        VMSTI --> AVSETSTI["AVSET-SouthIndia<br/>2 Fault Domains / 5 Update Domains"]
        VMSTI --- PIPSTI["Public IP: 135.13.16.54<br/>vm-southindia-da2f7b.southindia.cloudapp.azure.com"]
    end

    TM -->|Priority 2 / Secondary| PIPCUS
    TM -->|Priority 1 / Primary + Geo-mapped Asia/AU| PIPSTI
    VNetCUS <-. VNet Peering (bidirectional, Connected) .-> VNetSTI

    subgraph ID["Australia East — Identity Tier"]
        DC["DCVM1<br/>AD DS + DNS Server<br/>AD1DC1-vnet (10.0.0.0/16)"]
    end

    subgraph SVC["Shared Services"]
        Blob["Storage Account: dealerdataorg<br/>Container: dealer-docs<br/>Public access: disabled<br/>Access: SAS token only"]
        LA["Log Analytics Workspace<br/>DealerMonitoringWorkspace<br/>+ Alert Rules (CPU, Mem, Disk, Net, Availability)"]
    end

    VMCUS -. metrics/logs .-> LA
    VMSTI -. metrics/logs .-> LA
    DC -. identity/DNS .-> VNetCUS
    DC -. identity/DNS .-> VNetSTI
```

---

## 2. What happened during the failover test

This is the sequence from the failover test described in [`../docs/troubleshooting.md`](../docs/troubleshooting.md) — South India was taken offline on purpose to see how Traffic Manager would react.

```mermaid
sequenceDiagram
    participant Client as Client (ping / nslookup)
    participant TM as Traffic Manager (dealer)
    participant STI as VM-SouthIndia (Priority 1)
    participant CUS as VM-CentralUS (Priority 2)

    Note over TM,STI: Normal state — South India is primary
    TM->>STI: HTTP health probe
    STI--xTM: No response (endpoint down)
    TM->>TM: Mark SouthIndiaEP = Degraded

    Client->>TM: nslookup dealer.trafficmanager.net
    TM->>CUS: HTTP health probe
    CUS->>TM: 200 OK (healthy)
    TM->>Client: Resolve to vm-centralus-d3745b.centralus.cloudapp.azure.com (13.89.72.193)

    Client->>STI: ping 135.13.16.54
    STI--xClient: Request timed out (100% loss)

    Client->>CUS: Connects to Central US endpoint
    CUS->>Client: Serves Dealer Portal site
```
