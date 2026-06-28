# Network Topology

## Architecture Diagram

```
                          ┌───────────────────────────┐
                          │        Internet / WAN      │
                          └─────────────┬───────────────┘
                                        │
                              ┌─────────┴─────────┐
                              │   Router/Firewall   │
                              │   192.168.1.1/24    │
                              └─────────┬─────────┘
                                        │
                       ┌────────────────┼────────────────┐
                       │                │                │
              ┌────────┴───────┐ ┌──────┴──────┐ ┌────────┴────────┐
              │   DC / DNS /    │ │  WDS / MDT  │ │  Client Endpoint │
              │   DHCP Server   │ │   Server    │ │   (Test VM/PC)   │
              │ 192.168.1.10/24 │ │192.168.1.20 │ │  DHCP-assigned   │
              └─────────────────┘ └─────────────┘ └─────────────────┘
```

## Lab Environment Overview

This lab simulates a small enterprise network used to design, build, and
test a Windows endpoint deployment pipeline using WDS (Windows Deployment
Services) and MDT (Microsoft Deployment Toolkit).

### Components

- **Router/Firewall (192.168.1.1/24)** — Provides routing between the lab
  network and the internet, and isolates the lab subnet from other
  networks.
- **Domain Controller / DNS / DHCP (192.168.1.10/24)** — Hosts Active
  Directory Domain Services, DNS resolution for the lab domain, and DHCP
  scope/options (including PXE boot options) used by client endpoints
  during network boot.
- **WDS/MDT Server (192.168.1.20/24)** — Hosts Windows Deployment Services
  and the Microsoft Deployment Toolkit. Serves boot images over PXE and
  drives the task sequences used to deploy Windows images to clients.
- **Client Endpoint (DHCP-assigned)** — Test VM or physical PC that PXE
  boots against the WDS server to receive an automated OS deployment.

### Network Flow

1. The client endpoint powers on and initiates a PXE boot request.
2. DHCP (on the DC) responds with an IP lease and PXE boot options
   pointing to the WDS server.
3. The client downloads the WDS boot image (Windows PE) over TFTP/PXE.
4. Windows PE loads the MDT deployment wizard, which connects to the MDT
   deployment share on the WDS/MDT server.
5. The selected task sequence runs, applying the reference image, drivers,
   and post-install configuration to the client.

### IP Addressing Plan

| Host                  | IP Address       | Role                          |
|-----------------------|-------------------|-------------------------------|
| Router/Firewall       | 192.168.1.1/24    | Gateway / NAT                 |
| DC / DNS / DHCP       | 192.168.1.10/24   | Domain, name resolution, DHCP |
| WDS / MDT Server      | 192.168.1.20/24   | Image deployment              |
| Client Endpoint       | DHCP-assigned     | Deployment target             |
