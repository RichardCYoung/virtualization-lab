# VMware ESXi VLAN Tagging and Trunk Troubleshooting

This document describes a sanitized VMware ESXi networking scenario involving physical switch trunks, native VLANs, tagged VLANs, and ESXi port groups.

The purpose is to demonstrate a structured troubleshooting approach when migrating ESXi hosts between switch ports with different VLAN tagging configurations.

## Scenario

Multiple VMware ESXi hosts were connected to managed Ethernet switches using 802.1Q trunk interfaces.

The environment contained two different physical switch configurations.

### Existing ESXi Hosts

The original ESXi hosts used VLAN 10 as the native VLAN on the physical switch trunk.

```text
Physical Switch

802.1Q Trunk
Native VLAN: 10
       |
       |
       v
ESXi Physical NIC
       |
       v
Standard vSwitch
       |
       v
Management Port Group
VLAN ID: 0
```

Because VLAN 10 was native on the physical switch, management traffic arrived at ESXi without an 802.1Q tag.

The ESXi port group therefore used:

```text
VLAN ID: 0
```

In VMware, VLAN ID 0 means that ESXi expects the traffic to be untagged.

## New ESXi Hosts

Additional ESXi hosts were connected to switch interfaces where VLAN 10 was carried as a tagged VLAN.

```text
Physical Switch

802.1Q Trunk
VLAN 10 Tagged
       |
       |
       v
ESXi Physical NIC
       |
       v
Standard vSwitch
       |
       v
Management Port Group
VLAN ID: 10
```

In this design, the physical switch sends VLAN 10 traffic to ESXi with an 802.1Q tag.

The ESXi port group must therefore be configured with:

```text
VLAN ID: 10
```

## The Important Difference

Although both designs place the management interface in VLAN 10, the tagging behavior is different.

```text
Design A

Physical Switch                   VMware ESXi
------------------------------------------------------
VLAN 10 native/untagged   --->    Port Group VLAN 0


Design B

Physical Switch                   VMware ESXi
------------------------------------------------------
VLAN 10 tagged            --->    Port Group VLAN 10
```

The VLAN number alone is therefore not enough to determine the correct ESXi configuration.

The network engineer must determine whether the physical switch sends the VLAN tagged or untagged.

## Why Connectivity Can Fail

Problems occur when the switch configuration and ESXi port-group configuration do not agree about who handles the VLAN tag.

### Mismatch Example 1

The switch sends VLAN 10 tagged:

```text
Switch
VLAN 10 TAGGED
      |
      v
ESXi Port Group
VLAN ID 0
```

ESXi receives tagged traffic on a port group that expects untagged traffic.

Connectivity can fail.

### Mismatch Example 2

The switch sends VLAN 10 untagged:

```text
Switch
VLAN 10 NATIVE
      |
      v
ESXi Port Group
VLAN ID 10
```

ESXi expects tagged VLAN 10 traffic but receives untagged Ethernet frames.

Connectivity can again fail.

## VM Migration Scenario

This becomes particularly important when virtual machines are moved between ESXi hosts.

Consider two groups of hosts:

```text
ESXi Hosts A/B
-------------------------------
Physical Switch:
VLAN 10 native

VMware:
Port Group VLAN ID 0


ESXi Hosts C/D
-------------------------------
Physical Switch:
VLAN 10 tagged

VMware:
Port Group VLAN ID 10
```

A VM can work correctly on Hosts A/B but lose connectivity after being moved to Hosts C/D if the destination port group does not match the VLAN tagging model used by the physical switch.

The VM itself may require no network configuration change.

The problem can exist entirely between:

```text
Physical Switch
       |
       v
ESXi Physical NIC
       |
       v
vSwitch
       |
       v
Port Group
```

## Troubleshooting Methodology

When a VM or ESXi management interface loses connectivity after a host migration, validate the network one layer at a time.

### 1. Verify the Physical Switch Port

Determine whether the ESXi-facing interface is configured as an access port or trunk.

For a trunk, determine:

- Native VLAN
- Tagged VLANs
- Allowed VLANs
- Interface state
- VLAN membership

Do not assume that two ESXi switch ports have identical configurations simply because both are configured as trunks.

## 2. Verify the ESXi Physical NIC

Confirm that the correct physical adapter is connected and operational.

Validate:

- Link state
- Speed
- Duplex
- Physical switch interface
- vSwitch uplink assignment

## 3. Verify the vSwitch

Determine which physical NICs are assigned to the virtual switch.

```text
Physical NIC
     |
     v
   vSwitch
     |
     +------ Port Group A
     |
     +------ Port Group B
```

Confirm that the expected uplink is active.

## 4. Verify the Port Group VLAN

Check the VLAN ID configured on the affected VMware port group.

Typical examples:

```text
VLAN ID 0
```

Traffic is expected to be untagged.

```text
VLAN ID 10
```

ESXi handles VLAN 10 tagging for that port group.

The VMware configuration must match the physical switch configuration.

## 5. Verify VM Network Assignment

Confirm that the virtual machine NIC is attached to the correct port group.

A VM can have the correct IP configuration and still have no connectivity if its virtual NIC is attached to the wrong VMware network.

Validate:

- Connected state
- Connect at power on
- Port-group assignment
- VLAN configuration
- Guest operating-system configuration

## 6. Compare Working and Non-Working Hosts

One of the most effective troubleshooting methods is to compare a working host with a non-working host.

Compare:

```text
Physical switch port
Native VLAN
Allowed VLANs
Tagged VLANs
ESXi uplink
vSwitch
Port group
Port-group VLAN ID
VM NIC assignment
```

This often reveals configuration differences much faster than troubleshooting the failed VM in isolation.

## End-to-End Traffic Path

The entire Layer-2 path should be considered.

```text
              Physical Network

             Ethernet Switch
                   |
                   | 802.1Q trunk
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
              VLAN Handling
                   |
                   v
              Virtual NIC
                   |
                   v
             Virtual Machine
```

A configuration mismatch at any point can interrupt connectivity.

## Native VLAN Considerations

Native VLANs can make troubleshooting more difficult because the traffic is transmitted without an 802.1Q tag.

For example:

```text
Switch configuration:

Native VLAN = 10
```

Traffic belonging to VLAN 10 can leave the switch interface untagged.

From the ESXi perspective, that traffic appears as ordinary untagged Ethernet traffic.

The associated ESXi port group would therefore normally use:

```text
VLAN ID = 0
```

This is fundamentally different from explicitly tagging VLAN 10 across the trunk.

## Tagged VLAN Design

An alternative is to carry VLAN 10 explicitly tagged:

```text
Switch
   |
   | VLAN 10 tagged
   |
   v
ESXi
Port Group VLAN 10
```

This makes VLAN membership explicit at both ends of the link.

Other VLANs can use the same trunk:

```text
Physical Switch
       |
       | 802.1Q Trunk
       |
       +--- VLAN 10
       +--- VLAN 20
       +--- VLAN 30
       +--- VLAN 40
       |
       v
      ESXi
       |
       v
     vSwitch
       |
       +--- Management - VLAN 10
       +--- Servers    - VLAN 20
       +--- Storage    - VLAN 30
       +--- Lab        - VLAN 40
```

## Key Lesson

The most important lesson from this scenario is:

> A VLAN number and a VLAN tag are not the same thing.

Both environments can use VLAN 10 while handling the Ethernet frames differently.

```text
VLAN 10 native
      =
VLAN 10 frames transmitted untagged


VLAN 10 tagged
      =
VLAN 10 frames transmitted with an 802.1Q tag
```

ESXi port-group configuration must match that behavior.

## Troubleshooting Principle

When troubleshooting virtual networking, avoid looking only at VMware or only at the physical switch.

Treat the physical and virtual network as one end-to-end system:

```text
Switch configuration
        +
Physical link
        +
ESXi uplink
        +
Virtual switch
        +
Port group
        +
Virtual NIC
        +
Guest OS
```

The configuration at each layer must agree with the next.

## Future Lab Work

Additional VMware lab documentation may include:

- Multiple VLANs over an ESXi trunk
- VMkernel networking
- Management VLAN migration
- Storage network separation
- NIC teaming
- Uplink failover
- vSwitch security settings
- MTU and jumbo frames
- Inter-VLAN routing
- Virtual firewall deployment
- ESXi network troubleshooting commands

## Security and Sanitization

This case study is based on hands-on lab troubleshooting.

Hostnames, IP addresses, MAC addresses, credentials, and other environment-specific information have been removed or generalized.

No employer, customer, or production configuration data is included.
