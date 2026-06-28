# Network Topology

## Architecture Diagram

```
                          ┌─────────────────────────────┐
                          │      Internet / Admin PC      │
                          └───────────────┬───────────────┘
                                          │  HTTPS (443)
                                  ┌───────┴────────┐
                                  │  Azure Bastion   │
                                  │  AzureBastion    │
                                  │  Subnet (no NSG  │
                                  │  needed; MSFT-   │
                                  │  managed)        │
                                  └───────┬────────┘
                                          │ RDP (tunneled)
                  ┌───────────────────────┼───────────────────────────┐
                  │           Azure Virtual Network (VNet)            │
                  │              10.0.0.0/16 — lab-vnet               │
                  │                                                    │
                  │   ┌─────────────────────────────────────────┐      │
                  │   │     Subnet: snet-infra (10.0.1.0/24)     │      │
                  │   │  ┌──────────────┐   ┌──────────────────┐ │      │
                  │   │  │ DC / DNS /   │   │  WDS / MDT       │ │      │
                  │   │  │ DHCP Server  │   │  Server          │ │      │
                  │   │  │ 10.0.1.4     │   │  10.0.1.5        │ │      │
                  │   │  └──────────────┘   └──────────────────┘ │      │
                  │   │           NSG: nsg-infra                  │      │
                  │   └─────────────────────────────────────────┘      │
                  │                                                    │
                  │   ┌─────────────────────────────────────────┐      │
                  │   │   Subnet: snet-clients (10.0.2.0/24)     │      │
                  │   │       ┌──────────────────────┐           │      │
                  │   │       │  Client Endpoint VM    │           │      │
                  │   │       │  (DHCP-assigned)       │           │      │
                  │   │       └──────────────────────┘           │      │
                  │   │           NSG: nsg-clients                │      │
                  │   └─────────────────────────────────────────┘      │
                  │                                                    │
                  │   ┌─────────────────────────────────────────┐      │
                  │   │ Subnet: AzureBastionSubnet (10.0.0.0/27) │      │
                  │   └─────────────────────────────────────────┘      │
                  └────────────────────────────────────────────────────┘
```

## Lab Environment Overview

This lab is hosted entirely in **Microsoft Azure** and simulates a small
enterprise network used to design, build, and test a Windows endpoint
deployment pipeline using WDS (Windows Deployment Services) and MDT
(Microsoft Deployment Toolkit). All infrastructure runs as Azure VMs inside
a single Virtual Network (VNet), with administrative access provided
through Azure Bastion rather than exposed public RDP.

### Components

- **Azure Virtual Network — `lab-vnet` (10.0.0.0/16)** — The software-defined
  network boundary for the entire lab, segmented into purpose-built subnets.
- **Subnet: `snet-infra` (10.0.1.0/24)** — Hosts the domain/infrastructure
  tier:
  - **DC / DNS / DHCP Server (10.0.1.4)** — Azure VM running Active
    Directory Domain Services, DNS resolution for the lab domain, and a
    DHCP scope/options (including PXE boot options) used by client
    endpoints during network boot.
  - **WDS/MDT Server (10.0.1.5)** — Azure VM hosting Windows Deployment
    Services and the Microsoft Deployment Toolkit. Serves boot images over
    PXE and drives the task sequences used to deploy Windows images to
    clients.
  - Protected by **NSG `nsg-infra`**, scoped to allow only the traffic
    required between subnets (DNS, DHCP, PXE/TFTP, SMB for the MDT
    deployment share, RDP from Bastion).
- **Subnet: `snet-clients` (10.0.2.0/24)** — Hosts the **Client Endpoint
  VM**, the test target that PXE boots against the WDS server to receive
  an automated OS deployment. Protected by **NSG `nsg-clients`**.
- **AzureBastionSubnet (10.0.0.0/27)** — Reserved subnet for **Azure
  Bastion**, providing browser-based RDP to lab VMs over HTTPS (443)
  without assigning any VM a public IP address.

### Network Flow

1. An administrator connects to a lab VM through **Azure Bastion** (HTTPS
   443) instead of a public RDP endpoint.
2. The client endpoint VM powers on and initiates a PXE boot request on
   `snet-clients`.
3. DHCP (on the DC, `10.0.1.4`) responds with an IP lease and PXE boot
   options pointing to the WDS server, with the NSGs permitting DHCP/PXE
   traffic between subnets.
4. The client downloads the WDS boot image (Windows PE) over TFTP/PXE from
   the WDS/MDT server (`10.0.1.5`).
5. Windows PE loads the MDT deployment wizard, which connects to the MDT
   deployment share on the WDS/MDT server.
6. The selected task sequence runs, applying the reference image, drivers,
   and post-install configuration to the client VM.

### IP Addressing Plan

| Resource                 | Address Space / IP   | Role                              |
|--------------------------|-----------------------|------------------------------------|
| VNet: `lab-vnet`          | 10.0.0.0/16           | Lab network boundary              |
| AzureBastionSubnet        | 10.0.0.0/27           | Azure Bastion (admin access)      |
| Subnet: `snet-infra`      | 10.0.1.0/24           | Domain/infrastructure tier        |
| DC / DNS / DHCP           | 10.0.1.4              | Domain, name resolution, DHCP     |
| WDS / MDT Server          | 10.0.1.5              | Image deployment                  |
| Subnet: `snet-clients`    | 10.0.2.0/24           | Deployment target tier            |
| Client Endpoint VM        | DHCP-assigned         | Deployment target                 |

### Azure-Specific Notes

- No VM in this lab has a **public IP address**; all administrative access
  goes through **Azure Bastion**.
- **NSGs** (`nsg-infra`, `nsg-clients`) are applied at the subnet level to
  restrict traffic to only what the deployment workflow requires (DHCP,
  DNS, PXE/TFTP, SMB, and Bastion-sourced RDP).
- DHCP for the lab is provided by the **Windows DC VM**, not Azure's
  built-in VNet DHCP, since custom PXE boot options (066/067) are required
  for WDS to function.
