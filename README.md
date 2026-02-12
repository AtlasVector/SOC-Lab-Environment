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

## Network Design, Segmentation & Routing
The lab network is designed to support isolation, controlled communication, and experimentation with visibility and access boundaries. Segmentation is implemented using a combination of Linux bridges, VLANs, and a central firewall/router.

At the Proxmox level, the environment is built around two primary Linux bridges:

- **`vmbr0` – WAN-facing bridge**  
  Subnet: `192.168.50.0/24`  
  Addressing: DHCP from the upstream network  (Home Router)

- **`vmbr1` – Internal lab bridge**  
  Subnet: `10.10.10.0/24`  
  Purpose: Carries internal lab traffic and VLANs  

<img src="/resources/PVE-NICs.png" />

The internal lab network is further segmented using VLANs, which are trunked over `vmbr1` and terminated on pfSense. This allows multiple isolated networks to share the same physical bridge while remaining logically separated.
### Internal Network Segmentation
The following internal networks are defined:

**VLAN 10 (vtnet1.10) – EmployeesVLAN10** 
	- **Subnet**: `10.10.10.0/24` 
	- **Subnet Range:** `10.10.10.1` - `10.10.10.254`
	- **Gateway:** `10.10.10.254`
	- **DHCP range:** `10.10.10.10 - 10.10.10.30
	- **DNS Server:** 1`0.10.10.254`
**Purpose:** Windows workstations and user activity

**VLAN 20 (vtnet1.20)– ServerVLAN20**
	- **Subnet:** `10.10.20.0/24`
	- **Subnet Range:** `10.10.20.1` - `10.10.20.254`
	- **Gateway:** `10.10.20.254`
	- **DHCP range:** `10.10.20.10`- `10.10.20.30`
**Purpose:** Server-side services and infrastructure

**VLAN 30 (vtnet1.30)– ManegmentVLAN30**
	- **Subnet:** `10.10.30.0/24`
	- **Subnet Range:** `10.10.30.1` - `10.10.30.254`
	- **Gateway:** `10.10.30.254`
	- **DHCP range:** `10.10.30.10 - 10.10.30.30
	- **Purpose:** Management, monitoring, and security components

In addition to VLAN-based segmentation, a dedicated network is used for testing and attack simulation:

**ATTACKERLAN40**  
	- **Subnet:** `10.10.40.0/24`
	- **Gateway:** `10.10.40.254`
	- **Connection type:** Dedicated pfSense interface (`vtnet2`)
	- **DHCP range:** `10.10.40.1` - `10.10.40.40`
	- **Purpose:** Isolated testing host for attack and detection validation
### Firewall & Routing
pfSense acts as the central routing and control point for the lab. All internal networks are routed through pfSense, making it responsible for:
- Inter-VLAN routing
- Enforcing network isolation
- Applying firewall policies
- Serving as the default gateway for all segments

This design allows traffic between internal segments to be explicitly routed and filtered, while also supporting controlled testing of lateral movement, isolation boundaries, and monitoring coverage.

<img src="/resources/Pfsense-net.png" />
## Traffic Mirroring and Network Visibility
  
While attempting to mirror SPAN traffic to the Suricata IDS, it became apparent that **standard Linux bridges in Proxmox do not support native port mirroring**. This limitation surfaced during testing rather than design, requiring an alternative approach to maintain full network visibility.

As a workaround, traffic mirroring was implemented at the **virtual NIC (tap interface) level** using Linux **traffic control (`tc`)**. This allowed ingress traffic from the monitored VM interface to be mirrored directly to the Suricata monitoring interface.

<img src="/resources/tc-mirroring.png" />

To confirm that traffic was successfully mirrored to the correct interface on the Suricata VM, packet capture was performed directly on the monitoring interface.  

In this lab, the Suricata monitoring interface is `enp6s19`. Visibility was verified using:

```Bash
sudo tcpdump -i enp6s19
```


<img src="/resources/suricata-mirroed.png" />

To ensure the mirroring remains active after host reboots, the configuration was wrapped in a **systemd service** on the Proxmox host.

<img src="/resources/systemd-tc-mirror.png" />

As a future improvement, the lab will explore **Open vSwitch (OVS)** to evaluate its native SPAN and mirroring capabilities compared to the current tc-based solution.

## Suricata Configuration and Alert Validation

Once traffic visibility was confirmed at the interface level, the next step was ensuring Suricata was correctly ingesting and processing mirrored traffic.

Suricata was configured to listen on the dedicated monitoring interface: `enp6s19`

<img src="/resources/suricata-interface.png" />


Suricata configuration is tested before proceeding further using the below command
`
```Bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```

<img src="/resources/suricata-test.png" />

#### **Service Validation**
The Suricata service status was verified to confirm it was actively monitoring the interface:
<img src="/resources/Suricata-Validation.png" />


