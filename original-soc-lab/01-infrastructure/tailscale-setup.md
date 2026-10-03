# Zero-Trust Network Implementation (Tailscale)

## Overview
Instead of relying on traditional fragile network bridges or port forwarding, this lab utilizes Tailscale to create a mesh network. This ensures that all nodes can communicate securely over an encrypted overlay network, regardless of their physical network location.

## Architecture Benefits
**Zero Trust:** No open ports on the public internet. All traffic is authenticated via user identity.
**Network Segmentation:** The lab environment (`100.x.x.x`) is completely isolated from the home LAN (`192.168.x.x`).
**Portability:** The entire lab can be moved to a different physical location (e.g., public Wi-Fi) without breaking connectivity.

## Configuration Steps

### 1. Installation (Linux/Debian)
The following script was executed on all Proxmox LXC/VM nodes (Kali, Ubuntu, Wazuh):

```bash
# Add Tailscale repository and install
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate the node
sudo tailscale up
```

## Network Verification

```bash
# From Attacker (Kali) -> Target (Ubuntu)
ping <TARGET_TAILSCALE_IP> -c 4

# From SIEM (Wazuh) -> Agent (Ubuntu)
/var/ossec/bin/agent_control -l
```

## Topology

```mermaid
graph TD
    subgraph Proxmox_Host ["🖥️ Proxmox Host (Home Lab)"]
        direction TB

        subgraph VM_Attacker ["🔴 Attacker Node"]
            Kali[Kali Linux]
            Tool1[Hydra / Nmap]
            IP_Kali["IP: 100.x.x.1"]
        end

        subgraph VM_Target ["🟢 Target Node"]
            Ubuntu[Ubuntu Server]
            Svc1[SSH / Apache]
            Agent[Wazuh Agent]
            IP_Ubu["IP: 100.x.x.2"]
        end

        subgraph VM_Defense ["🔵 Defense Node"]
            Wazuh[Wazuh SIEM]
            Docker[Docker Manager]
            IP_Waz["IP: 100.x.x.3"]
        end
    end

%% Networking Connections
    Kali -. "Encrypted Tunnel (WireGuard)" .-> Ubuntu
    Ubuntu <== "Log Shipping (Port 1514)" ==> Wazuh
    Kali -. "Brute Force Attack (SSH)" .-> Ubuntu

%% Tailscale Overlay
style Proxmox_Host fill:#f9f9f9,stroke:#333,stroke-width:2px
style VM_Attacker fill:#ffcccc,stroke:#cc0000
style VM_Target fill:#ccffcc,stroke:#009900
style VM_Defense fill:#ccccff,stroke:#0000cc
```