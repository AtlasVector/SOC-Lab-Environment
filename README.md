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
- Windows-based infrastructure
- A testing host for experimentation
  
<img src="/resources/SOC-Lab-Environment.png" />

From an infrastructure perspective, the lab is built around two primary network zones:

- A WAN-facing network (`192.168.50.0/24`) connected via the `vmbr0` Linux bridge  
- An internal lab network (`10.10.10.0/24`) connected via the `vmbr1` Linux bridge and further segmented using VLANs  

The Proxmox host uses standard Linux bridges to connect virtual machines to these networks. This approach provides a simple and transparent networking model, which was helpful during the initial design and troubleshooting phases.

During later stages of the lab, I discovered that Linux bridges alone are not ideal for traditional port mirroring or SPAN traffic duplication. This limitation introduced challenges when attempting to provide reliable network visibility to the IDS. These challenges were solved  by implementing traffic mirroring using `tc` (traffic control), which allows packets to be mirrored at the bridge level. The details of this solution are documented in a later section.