## Introduction
While this is not my first home server, it is my first lab built with a clear focus on security operations. As a hobby, I have previously run a small personal home lab for **home automation and self-hosted services,** which helped me build a solid foundation in Linux, networking, and troubleshooting.

I later built my first dedicated **cybersecurity lab** using VMware Fusion on a Windows host. That environment was essential for turning theory into practice, but as the lab grew I began to hit hardware limitations, particularly around RAM usage and scalability.

To remove those constraints and enable further experimentation, I decided to rebuild the lab on **Proxmox VE**. Rather than migrating the existing environment, I chose to start over, using the opportunity to apply what I had learned and intentionally face new technical challenges.

## Lab Goals
The primary goal of this lab is to create a learning environment where encountering problems is expected and encouraged. Instead of avoiding complexity, the lab is designed to introduce new challenges that require investigation, experimentation, and hands-on problem solving.

The lab emphasizes:
- Learning through troubleshooting and experimentation  
- Facing unfamiliar technical challenges  
- Validating ideas and assumptions by testing and observation  

This approach creates continuous opportunities to build understanding and confidence by solving real problems rather than following predefined paths.

---

## Environment Overview
The lab is hosted on a single Proxmox VE node and is composed of multiple virtual machines with clearly defined responsibilities. Services are intentionally separated to support troubleshooting, safe experimentation, and easier understanding of system behavior.

The environment includes:
- Network routing and control
- Segmented internal networks
- Traffic monitoring and inspection
- Centralized logging and analysis
- Windows based infrastructure
- A testing host for experimentation

<img src="/resources/SOC-Lab-Environment.png" />

From an infrastructure perspective, the lab is built around two primary network zones:
- A WAN-facing network (`192.168.50.0/24`)
- An internal lab network (`10.10.10.0/24`), further segmented using VLANs
  
<img src="/resources/PVE-NICs.png" />

## Network Design, Segmentation & Routing
The lab network is designed to support isolation, controlled communication, and experimentation with visibility and access boundaries. Segmentation is implemented using a combination of Linux bridges, VLANs, and a central firewall/router.

At the Proxmox level, the environment is built around two primary Linux bridges:

- **`vmbr0` – WAN-facing bridge**  
  Subnet: `192.168.50.0/24`  
  Addressing: DHCP from the upstream network  (Home Router)

- **`vmbr1` – Internal lab bridge**  
  Subnet: `10.10.10.0/24`  
  Purpose: Carries internal lab traffic and VLANs  

The internal lab network is further segmented using VLANs, which are trunked over `vmbr1` and terminated on pfSense. This allows multiple isolated networks to share the same physical bridge while remaining logically separated.
### Internal Network Segmentation
The following internal networks are defined:
- **VLAN 10 – EMPLOYEESLAN**  
  Subnet: `10.10.10.0/24`  
  Gateway: `10.10.10.254`  
  DHCP range: `[PLACEHOLDER – confirm range]`  
  Purpose: Windows workstations and user activity

- **VLAN 20 – SERVERSLAN**  
  Subnet: `10.10.20.0/24`  
  Gateway: `10.10.20.254`  
  DHCP range: `[PLACEHOLDER – confirm range]`  
  Purpose: Server-side services and infrastructure

- **VLAN 30 – MANAGEMENTLAN**  
  Subnet: `10.10.30.0/24`  
  Gateway: `10.10.30.254`  
  DHCP range: `[PLACEHOLDER – confirm range]`  
  Purpose: Management, monitoring, and security components

In addition to VLAN-based segmentation, a dedicated network is used for testing and attack simulation:

- **ATTACKERLAN40**  
  Subnet: `10.10.40.0/24`  
  Gateway: `10.10.40.254`  
  Connection type: Dedicated pfSense interface (`vtnet2`)  
  DHCP range: `[PLACEHOLDER – confirm range]`  
  Purpose: Isolated testing host for attack and detection validation
### Firewall & Routing
pfSense acts as the central routing and control point for the lab. All internal networks are routed through pfSense, making it responsible for:
- Inter-VLAN routing
- Enforcing network isolation
- Applying firewall policies
- Serving as the default gateway for all segments

pfSense is connected to:
- The WAN network via `vtnet0`
- The internal lab network via `vtnet1` (VLAN trunk)
- The attacker network via a dedicated interface (`vtnet2`)  

This design allows traffic between internal segments to be explicitly routed and filtered, while also supporting controlled testing of lateral movement, isolation boundaries, and monitoring coverage.

<img src="/resources/Pfsense-net.png" />
## Traffic Visibility & Mirroring (`tc`)
Ensuring reliable network visibility was one of the main challenges in this lab. While standard Linux bridges (`vmbr0`, `vmbr1`) provide a simple and transparent networking model, they do not offer native support for traditional port mirroring or SPAN.

To solve that, traffic mirroring was implemented on the Proxmox host using Linux **traffic control (`tc`)**. This allows packets traversing the internal bridge (`vmbr1`) to be mirrored to a dedicated monitoring interface.

The mirroring configuration is applied at the bridge level and made persistent using a `systemd service`, ensuring visibility is maintained across reboots.

As a next improvement, the lab will be extended to experiment with **Open vSwitch (OVS)** as an alternative approach to traffic mirroring and network visibility.