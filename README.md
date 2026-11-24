# HADES Homelab --- Network Architecture & Security

> **Status:** Living documentation · **Audience:** Public GitHub
> repository\
> HADES is the domain and umbrella name for the complete homelab.
>
> This document describes the architecture and policies reported for the
> environment. Planned components are labelled. Credentials, public
> endpoints, VPN addressing, peer keys, and other operational secrets
> are intentionally excluded.

## 1. Overview

HADES is a segmented homelab built around a dedicated pfSense firewall.
It separates wireless user classes, server workloads, management access,
and emergency console access. Individual Proxmox containers have their
own network placement and firewall rules and are treated as separate
devices/services by Cerberus.

Design goals: - Explicit, least-privilege firewall rules. - Separate
ADMIN, HOME, GUEST, and IOT wireless networks. - A dedicated server
network. - Distinct WireGuard profiles for administrators and home
users. - Centralized monitoring, logging, and threat detection. -
Auditable infrastructure and minimal reliance on opaque services.

## 2. Infrastructure components

### 2.1 Core devices and services

The names in HADES follow the mythology of the underworld. Each name
identifies a distinct part of the infrastructure.

| Component | Mythological role | Function and status |
|---|---|---|
| **HADES** | The realm; the complete underworld. | Umbrella name for the homelab, its infrastructure, services, and internal domain. **In use.** |
| **Cerberus** | Guardian at the gates of Hades. | CWWK firewall appliance running pfSense CE. Handles routing, network segmentation, VPN termination, and firewall enforcement. Hardware: Intel N100, six Intel i226-V 2.5 GbE interfaces, 8 GB DDR5 RAM, and 128 GB SSD. **Deployed.** |
| **Minos** | Judge of the underworld. | AI server running Ollama, llama.cpp, Open WebUI, SearXNG, and FastAPI. **Deployed; software stack is evolving.** |
| **Styx** | The river connecting the realms. | Managed switch planned to expand the number of wired connections beyond the physical ports available on Cerberus. **Planned.** |
| **Charon** | Ferries users into Hades. | GL.iNet GL-AX1800 access point running a full OpenWrt configuration. Receives VLANs from Cerberus and connects wireless clients to the appropriate networks. **Deployed.** |
| **Persephone** | Keeper of the household in Hades. | Home Assistant instance responsible for smart-home automation and control. **Planned for the separate Proxmox server; deployment status to be verified.** |
| **Orpheus** | The one who crosses the boundary between worlds. | Website container; the public face of HADES and its communication point with the outside world. **Planned for the separate Proxmox server; deployment status to be verified.** |
| **Atlas** | The Titan who bears the heavens. | NAS and private cloud for data, archives, and backups. **Planned for the separate Proxmox server; deployment status to be verified.** |
| **Tartarus** | The deepest abyss. | Wazuh and Zabbix stack for centralized monitoring, log collection, incident analysis, and threat detection across the infrastructure. **Planned for the separate Proxmox server; deployment status to be verified.** |

**Charon design note:** Charon is an access point, not the primary
router. Cerberus provides the VLAN-backed networks; Charon maps wireless
clients into the appropriate VLAN.

**Cerberus hardware:** CWWK appliance based on Intel N100, six Intel
i226-V 2.5 GbE interfaces, 8 GB DDR5 RAM, and 128 GB SSD.

### 2.2 Separate Proxmox server

A separate physical server running **Proxmox** hosts containers (CTs).
Each CT has its own network placement and firewall policy. From
Cerberus's perspective, these workloads are treated as individual
devices/services, not as one unrestricted host.

-   **Persephone:** Home Assistant.
-   **Orpheus:** Public-facing website.
-   **Atlas:** NAS / private cloud / backup storage.
-   **Tartarus:** Wazuh and Zabbix monitoring stack.

The Proxmox host is physically separate from Minos. The intention is to
connect it to the server network alongside Minos. Do not infer that
every CT is already deployed or connected until verified on the live
system.

### 2.3 High-level topology

``` mermaid
flowchart TB
    NET((Internet))
    ISP["ISP network"]
    CPE["ISP router / ONT<br/>Bridge mode"]
    FW["Cerberus<br/>pfSense CE<br/>Firewall · routing · VPN"]
    SW["Styx<br/>Managed switch<br/>(planned)"]
    AP["Charon<br/>GL-AX1800 · OpenWrt"]

    subgraph WIFI["Wireless networks carried as VLANs"]
      ADMIN["WIFI_ADMIN"]
      HOME["WIFI_HOME"]
      GUEST["WIFI_GUEST"]
      IOT["WIFI_IOT"]
    end

    subgraph SERVERS["Server network"]
      MINOS["Minos<br/>AI services"]
      PVE["Separate Proxmox server"]
      PER["Persephone<br/>Home Assistant CT"]
      ORP["Orpheus<br/>Website CT"]
      ATL["Atlas<br/>NAS / private cloud CT"]
      TAR["Tartarus<br/>Wazuh + Zabbix CT"]
      PVE --> PER
      PVE --> ORP
      PVE --> ATL
      PVE --> TAR
    end

    MGMT["Management network<br/>Dedicated test port"]
    CONSOLE["CONSOLE<br/>Emergency / debugging access"]

    NET --> ISP --> CPE --> FW
    FW --> SW
    SW --> AP
    AP --> ADMIN
    AP --> HOME
    AP --> GUEST
    AP --> IOT
    FW --> SERVERS
    SW -. future expansion .-> SERVERS
    FW --> MGMT
    FW -. separate access path .-> CONSOLE
```

**Topology description**

1.  Internet → ISP network → ISP router/ONT in **bridge mode** →
    Cerberus. PPPoE is established by Cerberus.
2.  Cerberus is the central firewall and router.
3.  Styx will expand the number of wired connections when the available
    firewall ports are no longer sufficient.
4.  Charon receives the wireless VLANs from Cerberus and provides four
    wireless networks: ADMIN, HOME, GUEST, and IOT.
5.  A separate server-network link runs from Cerberus and/or Styx. Minos
    is currently the only server reported as connected there. The
    separate Proxmox server and its CT workloads are intended to join
    this network.
6.  The management network uses a separate physical port. It is a
    deliberately powerful test network intended for experiments and
    access to infrastructure.
7.  CONSOLE is a separate debugging and emergency path to the firewall
    administration interface; it is not a normal client VLAN.

## 3. Cerberus --- pfSense interface map

The following logical interface assignments were provided:

  -------------------------------------------------------------------------
  pfSense name              Device / parent         Purpose
  ------------------------- ----------------------- -----------------------
  `WAN`                     `igc0`                  WAN; PPPoE

  `LAN`                     `igc1`                  Parent interface for
                                                    wireless VLANs

  `WIFI_ADMIN`              VLAN on `igc1`          ADMIN wireless network

  `WIFI_HOME`               VLAN on `igc1`          HOME wireless network

  `WIFI_GUEST`              VLAN on `igc1`          GUEST wireless network

  `WIFI_IOT`                VLAN on `igc1`          IOT wireless network

  `SRVPORT1`                `igc2`                  First physical
                                                    server-network port

  `SVRPORT2`                `igc3`                  Second physical
                                                    server-network port

  `SERVERS`                 `BRIDGE0`               Bridge joining
                                                    `SRVPORT1` and
                                                    `SVRPORT2`

  `CABLE_CONNECTION_PORT`   `igc4`                  Separate wired
                                                    connection;
                                                    endpoint/purpose to be
                                                    documented privately

  `VPN_ADMIN`               `tun_wg0`               Assigned WireGuard
                                                    ADMIN interface

  `VPN_HOME`                `tun_wg1`               Assigned WireGuard HOME
                                                    interface

  Management                Separate physical port  Dedicated
                                                    test/management
                                                    network; interface name
                                                    and address omitted

  CONSOLE                   Dedicated console       Debugging and emergency
                            access                  firewall access
  -------------------------------------------------------------------------

> **VLAN tag clarification:** The source notes identify four separate
> wireless VLAN interfaces on `igc1`, but repeat the same numeric VLAN
> tag for each. Distinct networks on one parent interface require
> distinct VLAN IDs. This public document therefore records the logical
> interface names and parent only. Add verified tag IDs after checking
> **Interfaces → Assignments** and **Interfaces → VLANs** in pfSense.

`SERVERS` is a bridge (`BRIDGE0`) joining `SRVPORT1` and `SVRPORT2`.
Verify bridge membership and filtering behavior in the live
configuration before changing it.

## 4. Network segmentation

Exact IP subnets, gateway addresses, DHCP pools, and VLAN tags are
omitted from this public version. The logical zones are:

  -----------------------------------------------------------------------
  Network                 Purpose                 Trust level
  ----------------------- ----------------------- -----------------------
  `WIFI_ADMIN`            Trusted administrator   Highest user-network
                          wireless devices        trust

  `WIFI_HOME`             Household wireless      Trusted for household
                          devices                 use, not infrastructure
                                                  administration

  `WIFI_GUEST`            Guest devices           Untrusted; isolated
                                                  from internal networks

  `WIFI_IOT`              Smart-home and IoT      Restricted to
                          devices                 explicitly permitted
                                                  services

  `SERVERS`               Minos and the Proxmox   Server zone; explicit
                          host/CT workloads as    inter-zone policy
                          deployed                

  Management              Dedicated wired         Highly privileged;
                          test/management         physically controlled
                          connection              

  CONSOLE                 Emergency/debugging     Recovery access,
                          path                    separate from client
                                                  VLANs
  -----------------------------------------------------------------------

The management port is not an ordinary user network. Because it is
intended to provide broad access for testing, keep it physically
controlled and disconnected when not needed.

## 5. DNS

-   Cerberus provides internal DNS resolution for the HADES environment.
-   Internal aliases use the HADES namespace, for example
    `cerberus.hades`.
-   DNS aliases provide stable names for infrastructure devices and
    services.
-   Upstream DNS currently uses **Cloudflare DNS over HTTPS (DoH)** as a
    temporary solution.
-   The long-term plan is to deploy a self-hosted DNS service and
    replace this temporary upstream dependency.

Avoid publishing a complete internal hostname inventory if it reveals
services that are not intended to be public.

## 6. Firewall policy

Cerberus enforces segmentation through interface-level rules. This table
summarizes the stated policy intent; actual rule order and aliases must
be checked in the live pfSense configuration.

### 6.1 WAN

-   Block bogon networks.
-   Apply pfBlockerNG filtering.
-   Permit inbound traffic required for the WireGuard tunnels.
-   Do not expose other internal services directly on WAN without a
    separately documented rule.

### 6.2 Wireless interfaces

  -----------------------------------------------------------------------
  Interface               Permitted traffic       Restrictions
  ----------------------- ----------------------- -----------------------
  `WIFI_ADMIN`            Access everywhere for   Limit membership to
                          administration          trusted admin devices

  `WIFI_HOME`             DNS; communication      Block firewall,
                          within `WIFI_HOME`      ISP-router, and AP
                          subnets                 administration panels;
                                                  block access to
                                                  `WIFI_ADMIN`

  `WIFI_GUEST`            DNS                     Block administration
                                                  panels, `SERVERS`,
                                                  `WIFI_ADMIN`, and
                                                  `WIFI_HOME`

  `WIFI_IOT`              DNS and access to       Block other IPv4
                          Persephone (Home        traffic, including WAN
                          Assistant)              and other
                                                  VLANs/networks
  -----------------------------------------------------------------------

HOME permits communication between devices within its own network while
preventing access to the administrator network and infrastructure
management interfaces.

IOT is intentionally restrictive. If an IoT device needs an additional
destination (for example, NTP or a local controller), add a narrow
exception instead of general outbound access.

### 6.3 Server network

`SERVERS` policy:

-   Permit DNS.
-   Block connections to `WIFI_ADMIN`, `WIFI_HOME`, and `WIFI_GUEST`.
-   Apply explicit service-specific rules for traffic to other zones.
-   Treat Proxmox containers as individual workloads with their own
    firewall policy.

Do not infer unrestricted communication between Minos and the Proxmox
CTs merely because they share the server zone.

### 6.4 WireGuard

Common **WireGuard** group rules:

-   Permit DNS access for both tunnels.

Assigned interface rules:

  -----------------------------------------------------------------------
  Interface                           Policy
  ----------------------------------- -----------------------------------
  `VPN_ADMIN`                         Permit access to administration
                                      panels and `SERVERS` subnets

  `VPN_HOME`                          Block administration panels; permit
                                      access to `SERVERS` subnets
  -----------------------------------------------------------------------

Keep shared WireGuard group rules narrow. A broad pass rule on the group
tab can match before rules on assigned tunnel interfaces and undermine
the intended per-tunnel policy.

### 6.5 pfBlockerNG

**pfBlockerNG** is installed on Cerberus and is used to filter network
traffic/requests according to configured lists and feeds. Exact feeds,
update schedules, modes, and exceptions are intentionally not included
in this public overview.

## 7. WireGuard remote access

HADES has two separately assigned WireGuard tunnels:

-   **VPN-ADMIN** --- remote administrative access.
-   **VPN-HOME** --- remote access for ordinary home use, with a more
    restrictive firewall policy.

The assigned pfSense interfaces are `VPN_ADMIN` (`tun_wg0`) and
`VPN_HOME` (`tun_wg1`). Tunnel endpoint addresses, listen ports, tunnel
subnets, and peer public keys are deliberately omitted.

### 7.1 Peer design and an important troubleshooting lesson

Each client device has its own key pair and a unique tunnel address.
Multiple devices can share a tunnel and inherit its policy; the separate
tunnels represent different trust levels, not one tunnel per user.

**Every server-side peer must have its own unique `Allowed IPs` host
route** (normally a `/32` for an IPv4 client). Do not assign the entire
tunnel subnet to multiple peers.

During troubleshooting, two HOME peers were found with the same
server-side `Allowed IPs` value covering the whole tunnel subnet.
Assigning a unique client address to each peer and correcting the
server-side peer entries resolved the connectivity problem.

Never commit private keys, preshared keys, peer public keys, QR codes,
or complete client configuration exports.

### 7.2 Intended access separation

  Capability                                 VPN-ADMIN   VPN-HOME
  ------------------------------------------ ----------- ---------------------------------------
  DNS through Cerberus                       Allowed     Allowed
  Firewall/router/AP administration panels   Allowed     Blocked
  Server network                             Allowed     Allowed, subject to server-side rules
  General infrastructure administration      Intended    Denied by default

## 8. Minos --- AI platform

Minos is the AI server and the "judge" of HADES. Its reported software
stack includes:

  Component        Purpose
  ---------------- ------------------------------------------------
  **Ollama**       Local model serving and API
  **llama.cpp**    LLM inference/runtime tooling
  **Open WebUI**   Web interface for interacting with models
  **SearXNG**      Self-hosted metasearch
  **FastAPI**      API layer for custom integrations and services

The stack is evolving. Keep exact bind addresses, published ports, model
names, API tokens, reverse-proxy configuration, and environment secrets
in private deployment documentation.

## 9. Proxmox workloads

The separate Proxmox server hosts (or is intended to host) these CT
workloads:

-   **Persephone:** Home Assistant.
-   **Orpheus:** Public-facing website.
-   **Atlas:** NAS/private cloud and backup storage.
-   **Tartarus:** Wazuh and Zabbix.

Each CT is intended to have its own network identity and firewall
policy. This lets Cerberus control which services can communicate across
network boundaries.

For public documentation, describe CT purpose and allowed service flows,
but omit CT IDs, internal addresses, credentials, storage paths, and
management URLs.

## 10. Management and recovery access

### Management network

-   Uses a dedicated physical port on Cerberus.
-   Provides a deliberately powerful test/management connection.
-   Is used for experiments and troubleshooting.
-   Is not intended for ordinary household or guest devices.
-   Must remain physically controlled and separate from user networks.

### CONSOLE

-   A separate console access path for debugging and emergency access to
    the firewall administration interface.
-   Intended for recovery if normal network access is unavailable or
    misconfigured.
-   Not the same thing as the management Ethernet network.

## 11. Public documentation: what is safe to disclose?

This repository explains architecture and design decisions; it is not a
copy of the live configuration.

### Reasonable to publish

-   Mythology-based component names and high-level roles.
-   Logical topology and trust boundaries.
-   Logical network names (ADMIN, HOME, GUEST, IOT, SERVERS).
-   Firewall policy intent.
-   Software stack and broad hardware class.
-   The existence of separate ADMIN and HOME WireGuard profiles.
-   Sanitized interface-role mapping, if desired.

### Keep private or sanitize

-   Public IPv4 addresses, WAN endpoint hostnames, DDNS records, and ISP
    account details.
-   WireGuard private keys, preshared keys, peer public keys, QR codes,
    and full client profiles.
-   Exact VPN tunnel addresses and UDP listen ports.
-   Full internal IP/subnet inventory if not needed for the public
    explanation.
-   Management and console addresses, URLs, and access procedures.
-   Full firewall exports, configuration backups, certificates, API
    keys, passwords, tokens, and secret environment variables.
-   Detailed pfBlockerNG feeds, allowlists, and exceptions if they
    reveal operational security choices.
-   Proxmox hostnames/IDs, CT IDs, storage paths, and backup locations.
-   A complete internal DNS inventory.

RFC1918 private addresses are not directly routable over the public
Internet, but a full internal address map still reveals network
structure. This document therefore favors logical names and policy
descriptions over exact addressing.
