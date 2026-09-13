# Virtualization Lab

Hands-on virtualization lab covering VMware ESXi, VMware vSphere, Proxmox VE, virtual networking, VLANs, virtual machines, and infrastructure troubleshooting.

This repository documents practical virtualization work performed in a private lab environment, with a particular focus on the interaction between physical networking and virtual infrastructure.

## Lab Overview

The environment is used to explore and troubleshoot technologies including:

- VMware ESXi
- VMware vSphere / vCenter Server
- Proxmox VE
- Linux
- Virtual machines
- Virtual switches
- Linux bridges
- 802.1Q VLAN tagging
- Trunk and access ports
- Virtual NICs
- VM migration
- Virtual network appliances
- Hypervisor networking
- Infrastructure troubleshooting

The goal is not only to deploy virtual machines, but to understand the complete network path between the physical infrastructure, hypervisor, virtual network, and guest operating system.

## Virtualization Networking Architecture

The lab demonstrates how the same Ethernet and VLAN concepts are implemented differently by VMware ESXi and Proxmox VE.

![Virtualization Networking Architecture](screenshots/virtualization-networking.png)

At a high level:

```text
                     Physical Network
                            |
                       802.1Q Trunk
                            |
              +-------------+-------------+
              |                           |
              v                           v
         VMware ESXi                  Proxmox VE
              |                           |
           vSwitch                      vmbr0
              |                     VLAN-aware
         Port Groups                      |
              |                       VM VLAN Tags
              |                           |
              v                           v
             VMs                         VMs
```

Although the implementation differs between hypervisors, the underlying networking principles remain the same.

## VMware vSphere Lab

The VMware environment consists of multiple ESXi hosts managed through VMware vCenter Server.

![VMware vSphere Lab](screenshots/vsphere.png)

The environment includes a mixture of Linux, Windows, routing, and virtual network appliance workloads.

The VMware lab is used to test and troubleshoot:

- Multi-host ESXi networking
- VMware vCenter management
- Standard virtual switches
- Physical uplinks
- VMkernel networking
- Port groups
- VLAN tagging
- 802.1Q trunks
- Native VLANs
- VM network assignment
- VM migration between hosts
- Virtual network appliances
- Physical-to-virtual network connectivity

## VMware ESXi Networking

A major focus of the VMware lab is understanding where VLAN tagging occurs between the physical switch and ESXi.

For example:

```text
Physical Switch                    VMware ESXi
---------------------------------------------------------
VLAN 10 native / untagged   --->   Port Group VLAN 0

VLAN 10 tagged              --->   Port Group VLAN 10
```

Both configurations place traffic in VLAN 10, but the Ethernet frames are handled differently.

With a native VLAN, the physical switch transmits the traffic without an 802.1Q tag.

With a tagged VLAN, the VLAN information is carried across the trunk and ESXi handles the VLAN assignment through the port group.

A mismatch between the physical switch and VMware configuration can result in loss of management or VM connectivity.

A detailed lab case study is available here:

```text
vmware/esxi-vlan-troubleshooting.md
```

## ESXi VLAN Troubleshooting

The VMware case study documents a scenario involving ESXi hosts connected to physical switch trunks using different VLAN tagging models.

The troubleshooting process follows the complete traffic path:

```text
Physical Switch
      |
      v
802.1Q Trunk
      |
      v
ESXi Physical NIC
      |
      v
vSwitch
      |
      v
Port Group
      |
      v
Virtual NIC
      |
      v
Virtual Machine
```

The key lesson is:

> A VLAN number and a VLAN tag are not the same thing.

Two ESXi environments can both use VLAN 10 while requiring different port-group configurations depending on whether VLAN 10 arrives tagged or untagged.

## Proxmox VE

The Proxmox portion of the lab focuses on Linux-based virtualization and networking.

Topics include:

- Proxmox VE networking
- Linux bridges
- Physical NIC integration
- VLAN-aware bridges
- 802.1Q trunks
- VM VLAN assignment
- Tagged and untagged traffic
- Linux network troubleshooting
- Virtual machine connectivity

A typical Proxmox networking design looks like:

```text
Physical Ethernet Switch
          |
          | 802.1Q Trunk
          |
          v
      Physical NIC
         eno1
          |
          v
        vmbr0
   VLAN-aware bridge
          |
     +----+----+----+
     |         |    |
     v         v    v
  VLAN 10   VLAN 20 VLAN 30
     |         |    |
     v         v    v
    VM1       VM2  VM3
```

The Linux bridge performs a role similar to a virtual Ethernet switch.

## Proxmox VLAN-Aware Networking

A typical sanitized Proxmox bridge configuration might look like:

```text
auto lo
iface lo inet loopback

iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.x.x/24
        gateway 192.168.x.1
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-4094
```

Virtual machines can then be assigned VLAN tags while sharing the same physical uplink.

For example:

```text
VM Network Device

Bridge:   vmbr0
VLAN Tag: 20
```

The resulting traffic path is:

```text
Virtual Machine
      |
      v
Virtual NIC
      |
   VLAN 20
      |
      v
    vmbr0
      |
      v
     eno1
      |
 VLAN 20 tagged
      |
      v
Physical Switch
```

Detailed Proxmox networking documentation is available here:

```text
proxmox/proxmox-networking.md
```

## VMware and Proxmox Comparison

The terminology differs between VMware and Proxmox, but many of the underlying networking concepts are equivalent.

| Networking Component | VMware ESXi | Proxmox VE |
| --- | --- | --- |
| Physical NIC | vmnic | enoX |
| Virtual switching | Standard vSwitch | Linux Bridge |
| VM network assignment | Port Group | Bridge / VM NIC |
| VLAN assignment | Port Group VLAN ID | VM VLAN Tag |
| Host network interface | VMkernel Interface | Bridge / VLAN Interface |
| Trunk connectivity | vSwitch uplink | VLAN-aware bridge uplink |
| Virtual NIC | vNIC | Virtual NIC / TAP |

This makes virtualization networking a natural extension of traditional network engineering.

## Network Engineering Perspective

Virtualization does not remove traditional networking concepts.

It extends the network into the hypervisor.

The same technologies remain important:

- Ethernet
- MAC addressing
- VLANs
- IEEE 802.1Q
- Trunking
- Layer-2 forwarding
- IP addressing
- Routing
- MTU
- Link aggregation
- Network redundancy
- Packet capture

A useful troubleshooting model is:

```text
Physical Network
       |
       v
Physical Switch
       |
       v
Hypervisor Uplink
       |
       v
Virtual Switch / Linux Bridge
       |
       v
Port Group / VLAN
       |
       v
Virtual NIC
       |
       v
Guest Operating System
```

The hypervisor should therefore be treated as part of the end-to-end network rather than as an isolated server platform.

## Troubleshooting Methodology

When troubleshooting virtualization networking, the lab uses a layered approach.

### Physical Layer

Validate:

- Physical link state
- Interface speed
- Duplex
- Cabling
- Switch interface state

### Switching Layer

Validate:

- Access or trunk mode
- Native VLAN
- Tagged VLANs
- Allowed VLANs
- VLAN existence
- MAC address learning

### Hypervisor Layer

Validate:

- Physical uplink
- vSwitch or Linux bridge
- VLAN configuration
- Port-group assignment
- Bridge membership
- VM network configuration

### Virtual Machine Layer

Validate:

- Virtual NIC state
- Port-group or bridge assignment
- VLAN assignment
- Guest IP configuration
- Default gateway
- Routing
- DNS

The complete path is tested rather than assuming the problem exists only in the VM or only on the physical switch.

## Lab Use Cases

The virtualization environment supports several areas of infrastructure testing.

### Network Infrastructure

Virtual routers, switches, firewalls, and Linux systems can be deployed to test network designs and troubleshooting scenarios.

### Network Automation

Virtual network devices can provide repeatable test targets for Python and API-based automation.

### Kubernetes

Virtual machines can be used to host Linux and Kubernetes nodes for container and networking labs.

### Security

Virtual firewalls and segmented VLANs can be used to test network security designs in an isolated environment.

### Troubleshooting

The environment allows configuration changes and failure scenarios to be reproduced without affecting production infrastructure.

## Repository Structure

```text
virtualization-lab/
├── README.md
│
├── vmware/
│   └── esxi-vlan-troubleshooting.md
│
├── proxmox/
│   └── proxmox-networking.md
│
└── screenshots/
    ├── virtualization-networking.png
    └── vsphere.png
```

## Related Projects

This lab complements other hands-on infrastructure projects in my GitHub portfolio.

### Network Automation

Multi-vendor network automation using Python, Netmiko, YAML, configuration auditing, inventory collection, and compliance workflows.

Repository:

```text
network-automation
```

### Kubernetes Lab

Hands-on Kubernetes and MicroK8s work covering cluster deployment, Kubernetes networking, Helm, NetBox, and troubleshooting.

Repository:

```text
kubernetes-lab
```

Together, these projects demonstrate the relationship between:

```text
Physical Networking
        |
        v
Virtualization
        |
        v
Linux
        |
        v
Containers / Kubernetes
        |
        v
Network Automation
```

## Future Lab Work

Planned additions include:

- Additional ESXi networking scenarios
- VMkernel networking
- VMware NIC teaming
- Multiple ESXi uplinks
- Proxmox Linux bonding
- LACP
- Dedicated management VLANs
- Storage VLANs
- VM migration testing
- Proxmox clustering
- Shared storage
- Ceph networking
- MTU and jumbo-frame testing
- Virtual firewall deployment
- Kubernetes VM hosting
- Infrastructure monitoring
- Network automation integration

## Security and Lab Environment

This repository contains examples, documentation, diagrams, and screenshots from a private, non-production lab environment.

Screenshots and documentation may include internal lab hostnames, private IP addressing, device names, VLAN IDs, virtual machine names, and other lab-specific details where they help demonstrate the environment or troubleshooting scenario.

Credentials, passwords, API keys, authentication tokens, private keys, certificates, public IP addresses, and other sensitive information are not intentionally published.

All systems and configurations shown are part of a personal lab environment unless explicitly identified as generalized examples.

No employer, customer, or production configuration data is included.
