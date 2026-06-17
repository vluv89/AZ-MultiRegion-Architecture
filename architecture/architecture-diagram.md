# Architecture Diagrams

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

## 2. Disaster Recovery Failover Sequence (Validated)

This sequence reflects the failover behaviour observed and validated during testing —
see [`../docs/troubleshooting.md`](../docs/troubleshooting.md) for the full narrative.

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

---

## 3. Traffic Manager Routing Methods Comparison

```mermaid
graph LR
    subgraph Profile1["Profile: dealer (Priority Routing)"]
        A["Incoming request"] --> B{Is Priority 1<br/>(South India) healthy?}
        B -->|Yes| C[Route to South India]
        B -->|No| D[Failover to Central US<br/>Priority 2]
    end

    subgraph Profile2["Profile: dealer2 (Geographic Routing)"]
        E["Incoming request"] --> F{User region?}
        F -->|Asia: India, China<br/>Australia/Pacific: NSW| G[Route to South India - STI1]
        F -->|Other regions| H[Route to Central US - CUS1]
    end
```
