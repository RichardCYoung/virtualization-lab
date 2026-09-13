# Proxmox VE Networking - Linux Bridges and VLANs

This document describes a sanitized Proxmox VE networking lab focused on Linux bridges, physical interfaces, VLAN tagging, and virtual machine connectivity.

The goal is to demonstrate how traditional Ethernet switching concepts map into a Linux-based virtualization environment.

## Overview

Proxmox VE uses Linux networking components rather than VMware-style virtual switches and port groups.

A common Proxmox network design consists of:

```text
Physical Ethernet Switch
          |
          | 802.1Q trunk
          |
          v
    Physical NIC
       eno1
          |
          v
        vmbr0
   VLAN-aware bridge
          |
          +------ VM 1 - VLAN 10
          |
          +------ VM 2 - VLAN 20
          |
          +------ VM 3 - VLAN 30
```

The Linux bridge performs a role similar to a virtual Ethernet switch.

## Physical Network

The physical switch interface connected to the Proxmox host can be configured as an 802.1Q trunk.

Example:

```text
Switch Port
    |
    | 802.1Q trunk
    |
    +--- VLAN 10 - Management
    +--- VLAN 20 - Servers
    +--- VLAN 30 - Lab
    +--- VLAN 40 - Storage
    |
    v
Proxmox Physical NIC
```

The exact switch syntax depends on the network vendor.

The important requirement is that the VLAN configuration on the physical switch agrees with the configuration on the Proxmox host.

## Linux Bridge

A typical Proxmox bridge is named:

```text
vmbr0
```

The physical interface is attached to the bridge:

```text
eno1
  |
  v
vmbr0
```

Virtual machine interfaces can then attach to `vmbr0`.

Conceptually:

```text
                vmbr0
                  |
        +---------+---------+
        |         |         |
        v         v         v
       VM1       VM2       VM3
```

This is similar to connecting several physical servers to an Ethernet switch.

## Example Proxmox Network Configuration

Proxmox networking is commonly defined in:

```text
/etc/network/interfaces
```

Example sanitized configuration:

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

This configuration creates a VLAN-aware Linux bridge using `eno1` as the physical uplink.

## Configuration Explanation

### Physical Interface

```text
iface eno1 inet manual
```

The physical interface does not require its own IP address when it is being used as a bridge port.

Instead, Layer-3 configuration is normally assigned to the bridge or an appropriate VLAN interface.

### Bridge Port

```text
bridge-ports eno1
```

This attaches the physical NIC to `vmbr0`.

The traffic path becomes:

```text
VM
 |
 v
vmbr0
 |
 v
eno1
 |
 v
Physical Switch
```

### VLAN Awareness

```text
bridge-vlan-aware yes
```

This enables VLAN filtering and VLAN-aware forwarding on the Linux bridge.

Multiple VLANs can therefore traverse the same physical interface.

### Allowed VLAN Range

```text
bridge-vids 2-4094
```

This defines the VLAN IDs available to the bridge.

In a more restrictive design, only the VLANs actually required by the host could be allowed.

For example:

```text
bridge-vids 10 20 30 40
```

Restricting the VLAN list can make the intended network design clearer.

## VM VLAN Assignment

A virtual machine connected to `vmbr0` can be assigned a VLAN tag in Proxmox.

For example:

```text
VM Network Device

Bridge: vmbr0
VLAN Tag: 20
```

Traffic from that VM is associated with VLAN 20.

Conceptually:

```text
VM
|
| untagged inside guest
|
v
Proxmox Virtual NIC
|
| VLAN 20 assigned
|
v
vmbr0
|
| VLAN 20 tagged
|
v
eno1
|
v
Physical Switch Trunk
```

The guest operating system does not necessarily need to understand VLAN tagging.

Proxmox can handle the VLAN association on behalf of the VM.

## Multiple Virtual Machines

Different VMs can use different VLANs while sharing the same physical NIC.

```text
                 Proxmox Host

              +---------------+
              |     vmbr0     |
              +---------------+
                |     |     |
                |     |     |
              VLAN10 VLAN20 VLAN30
                |     |     |
                v     v     v
               VM1   VM2   VM3

                     |
                     v
                   eno1
                     |
                802.1Q trunk
                     |
                     v
              Physical Switch
```

This allows network segmentation without requiring a dedicated physical NIC for every VLAN.

## Proxmox Management Network

Management traffic can also be placed on a VLAN.

There are several possible designs.

### Untagged Management

The physical switch sends management traffic untagged.

Example concept:

```text
Switch Native VLAN
       |
       | untagged
       |
       v
      eno1
       |
       v
     vmbr0
       |
       v
Proxmox Management IP
```

### Tagged Management

Management can instead use an explicitly tagged VLAN.

Conceptually:

```text
Physical Switch
       |
       | VLAN 10 tagged
       |
       v
      eno1
       |
       v
     vmbr0
       |
       v
   vmbr0.10
       |
       v
Proxmox Management IP
```

Example sanitized configuration:

```text
auto vmbr0
iface vmbr0 inet manual
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes

auto vmbr0.10
iface vmbr0.10 inet static
        address 192.168.x.x/24
        gateway 192.168.x.1
```

This explicitly places the Proxmox management interface in VLAN 10.

## Native Versus Tagged VLANs

The same VLAN tagging principles that apply to VMware ESXi also apply to Proxmox.

The physical switch and hypervisor must agree on whether traffic is tagged.

For example:

```text
Physical Switch                 Proxmox
-------------------------------------------------

VLAN 10 native/untagged   --->  Untagged bridge traffic

VLAN 10 tagged            --->  VLAN-aware bridge / VLAN 10
```

A mismatch can result in loss of connectivity.

## Troubleshooting VM Connectivity

When a Proxmox VM cannot reach the network, troubleshoot the complete path.

```text
Guest OS
   |
   v
Virtual NIC
   |
   v
VM VLAN Assignment
   |
   v
Linux Bridge
   |
   v
Physical NIC
   |
   v
Physical Switch
   |
   v
Gateway / Destination
```

Each layer should be validated independently.

## Check Proxmox Interfaces

Display interface configuration:

```bash
ip addr
```

Display link state:

```bash
ip link
```

Display routing:

```bash
ip route
```

These commands help confirm the host's Layer-2 and Layer-3 configuration.

## Check Linux Bridges

Display bridge information:

```bash
bridge link
```

Display VLAN information:

```bash
bridge vlan show
```

This is particularly useful when troubleshooting VLAN-aware bridges.

Example:

```text
port              vlan-id
eno1              10
                  20
                  30
vmbr0             10
                  20
                  30
```

## Check the Physical Interface

Confirm the physical NIC has link:

```bash
ip link show eno1
```

Additional interface information may be available using:

```bash
ethtool eno1
```

Important values include:

- Link detected
- Speed
- Duplex
- Interface state

## Check the Bridge

Verify that the physical NIC is attached to the expected bridge:

```bash
bridge link
```

The physical interface should appear as a member of `vmbr0`.

## Check VM Configuration

Proxmox VM configuration can be inspected from the CLI.

Example:

```bash
qm config <vmid>
```

Look for the VM network adapter.

Example sanitized output:

```text
net0: virtio=<mac-address>,bridge=vmbr0,tag=20
```

Important values include:

```text
bridge=vmbr0
tag=20
```

This indicates that the VM is connected to `vmbr0` and assigned to VLAN 20.

## Verify the Physical Switch

The physical switch interface should also be checked.

Validate:

- Interface state
- Trunk configuration
- Native VLAN
- Allowed VLANs
- Tagged VLANs
- MAC address learning

A correctly configured Proxmox host cannot compensate for a VLAN that is missing from the physical switch trunk.

## MAC Address Troubleshooting

MAC address tables can help determine how far traffic is travelling.

On the physical switch, verify that the VM MAC address is learned on the expected Proxmox uplink.

On Linux, bridge forwarding information can be viewed with:

```bash
bridge fdb show
```

This can help determine whether Layer-2 traffic is reaching the bridge.

## Packet Capture

Linux provides powerful packet-capture capabilities for troubleshooting virtual networking.

For example:

```bash
tcpdump -i eno1
```

Capture traffic on the bridge:

```bash
tcpdump -i vmbr0
```

Capture a specific VLAN:

```bash
tcpdump -i eno1 vlan 20
```

This can help determine whether VLAN tags are present on the physical uplink.

For example:

```text
VM
 |
 v
vmbr0
 |
 | capture here
 |
 v
eno1
 |
 | capture here
 |
 v
Physical Switch
```

Comparing captures at different points can help isolate where traffic is being lost.

## VMware and Proxmox Comparison

Although VMware ESXi and Proxmox use different terminology, many of the networking concepts are similar.

| VMware ESXi | Proxmox VE |
| --- | --- |
| Physical NIC / vmnic | Physical NIC / enoX |
| Standard vSwitch | Linux Bridge |
| Port Group | Bridge + VM network configuration |
| Port Group VLAN ID | VM VLAN Tag |
| VMkernel Interface | Host bridge/VLAN interface |
| Trunk uplink | VLAN-aware bridge uplink |
| vNIC | TAP / virtual NIC |

The underlying Ethernet concepts remain the same.

## Troubleshooting Example

Consider a VM assigned to VLAN 20.

```text
VM
VLAN 20
   |
   v
vmbr0
   |
   v
eno1
   |
   | VLAN 20 tagged
   |
   v
Physical Switch
```

If the VM cannot communicate, verify:

1. VM NIC is connected
2. VM uses the correct bridge
3. VLAN tag is 20
4. `vmbr0` is VLAN-aware
5. Physical NIC is part of `vmbr0`
6. Physical interface has link
7. VLAN 20 is allowed on the physical trunk
8. VLAN 20 exists on the physical network
9. Gateway is reachable
10. VM IP configuration is correct

This prevents troubleshooting from becoming focused on only one component.

## Network Engineering Perspective

Proxmox networking demonstrates that virtualization does not replace traditional networking concepts.

The same concepts remain important:

- Ethernet
- MAC addresses
- VLANs
- 802.1Q
- Trunks
- Layer-2 forwarding
- IP addressing
- Routing
- MTU
- Packet capture

The difference is that part of the network now exists inside the Linux host.

A useful troubleshooting mindset is therefore:

```text
The hypervisor is also part of the network.
```

## Future Lab Work

Future Proxmox lab work may include:

- Dedicated management VLAN
- Storage VLAN
- Multiple physical uplinks
- Linux bonding
- LACP
- Bridge redundancy
- Proxmox clustering
- VM migration
- Ceph networking
- MTU and jumbo-frame testing
- Virtual firewall deployment
- Kubernetes VMs
- Network automation
- Monitoring

## Security and Sanitization

This document contains lab-built and sanitized examples.

IP addresses, MAC addresses, hostnames, credentials, and environment-specific information have been removed or generalized.

No employer, customer, or production configuration data is included.
