# 📘 Lab 12.9.2 - Configure IPv6 Addresses on Network Devices

![Cisco](https://img.shields.io/badge/Cisco-IOS-blue?style=for-the-badge&logo=cisco)
![IPv6](https://img.shields.io/badge/IPv6-Enabled-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)
![CCNA](https://img.shields.io/badge/CCNA-Lab-orange?style=for-the-badge)

## 📋 Table of Contents

- [Overview](#-overview)
- [Network Topology](#%EF%B8%8F-network-topology)
- [Objectives](#objectives)
- [Equipment](#equipment)
- [Lab Steps](#lab-steps)
- [Configuration Files](#configuration-files)
- [Verification](#verification)
- [Key Concepts](#key-concepts)
- [Troubleshooting](#troubleshooting)
- [Resources](#resources)

### 🎯 Overview

This laboratory demonstrates the configuration of IPv6 addresses on Cisco network devices, including:
- Manual IPv6 address assignment on router interfaces
- Link-local address configuration
- IPv6 routing enablement with SLAAC (Stateless Address Autoconfiguration)
- Switch management interface IPv6 configuration
- End-to-end connectivity verification

**Duration:** ~90 minutes  
**Difficulty:** Intermediate  
**Lab Number:** 12.9.2

### 🗺️ Network Topology

```
PC-B (.a::3)  ←→  [G0/0/0] R1 [G0/0/1]  ←→  S1  ←→  PC-A (.1::3)
     ↓                              ↓              ↓
Network A                      Network 1      VLAN 1 (.1::b)
(2001:db8:acad:a::/64)      (2001:db8:acad:1::/64)
```

### Addressing Table

| Device | Interface | IPv6 Address | Prefix | Default Gateway |
|--------|-----------|--------------|--------|-----------------|
| **R1** | G0/0/0 | `2001:db8:acad:a::1` | /64 | N/A |
| **R1** | G0/0/1 | `2001:db8:acad:1::1` | /64 | N/A |
| **R1** | Both | `fe80::1` (link-local) | /10 | N/A |
| **S1** | VLAN 1 | `2001:db8:acad:1::b` | /64 | `fe80::1` |
| **PC-A** | NIC | `2001:db8:acad:1::3` | /64 | `fe80::1` |
| **PC-B** | NIC | `2001:db8:acad:a::3` | /64 | `fe80::1` |

## 🎯 Objectives

### Part 1: Set Up Topology and Configure Basic Settings
- Cable the network according to topology
- Erase and reload router and switch
- Configure basic device settings (hostname, passwords, banner)
- **Configure SDM template for IPv6 support on switch** ⚠️ CRITICAL

### Part 2: Configure IPv6 Addresses Manually
- Assign IPv6 global unicast addresses to router interfaces
- Configure manual link-local addresses
- Enable IPv6 routing with `ipv6 unicast-routing`
- Configure IPv6 on switch management interface
- Assign static IPv6 addresses to PCs

### Part 3: Verify End-to-End Connectivity
- Test local connectivity with ping
- Verify inter-VLAN routing with traceroute
- Confirm SLAAC functionality

##  Equipment

- 1x Cisco Router 4221 (or equivalent) - **R1**
- 1x Cisco Catalyst 2960 Switch - **S1**
- 2x PCs (Windows) - **PC-A, PC-B**
- Console cables
- 3x Ethernet straight-through cables

## Lab Steps

### Part 1: Basic Configuration

#### Step 1: Router R1 Initial Setup

```cisco
Router> enable
Router# configure terminal

! Set hostname
Router(config)# hostname R1

! Secure privileged EXEC mode
R1(config)# enable secret class

! Configure MOTD banner
R1(config)# banner motd $Unauthorized access is prohibited!$

! Secure console line
R1(config)# line console 0
R1(config-line)# password ccna
R1(config-line)# login
R1(config-line)# exit

! Secure VTY lines
R1(config)# line vty 0 4
R1(config-line)# password ccna
R1(config-line)# login
R1(config-line)# exit

! Encrypt passwords
R1(config)# service password-encryption

! Disable DNS lookup
R1(config)# no ip domain-lookup

R1(config)# end
```

#### Step 2: Switch S1 Initial Setup

```cisco
! Apply the same basic configuration as R1
Switch> enable
Switch# configure terminal
Switch(config)# hostname S1
S1(config)# enable secret class
! ... (same pattern as R1)
```

#### Step 3:  CRITICAL - Enable IPv6 on Switch

```cisco
! Check current SDM template
S1# show sdm prefer

! Configure dual IPv4/IPv6 template
S1# configure terminal
S1(config)# sdm prefer dual-ipv4-and-ipv6 default
S1(config)# end

! Reload required
S1# reload
```

** Justification:** The default Catalyst 2960 SDM template does *NOT support IPv6*. Without this step, you CANNOT assign IPv6 addresses to VLAN 1 SVI.

### Part 2: IPv6 Configuration

#### Step 4: Configure IPv6 on R1 Interfaces

```cisco
R1# configure terminal

! Configure G0/0/0 (Network A)
R1(config)# interface gigabitEthernet 0/0/0
R1(config-if)# ipv6 address 2001:db8:acad:a::1/64
R1(config-if)# ipv6 address fe80::1 link-local
R1(config-if)# no shutdown

! Configure G0/0/1 (Network 1)
R1(config-if)# interface gigabitEthernet 0/0/1
R1(config-if)# ipv6 address 2001:db8:acad:1::1/64
R1(config-if)# ipv6 address fe80::1 link-local
R1(config-if)# no shutdown
R1(config-if)# end
```

**Verification:**

```cisco
R1# show ipv6 interface brief
R1# show ipv6 interface g0/0/0
```

**Expected Output:**
- Link-local: `FE80::1` (manually configured)
- Global: `2001:DB8:ACAD: A::1/64`
- Multicast groups: `FF02::1` (all-nodes), `FF02::1:FF00:1` (solicited-node)

#### Step 5: Enable IPv6 Routing (SLAAC)

```cisco
R1# configure terminal
R1(config)# ipv6 unicast-routing
R1(config)# exit
```

** CRITICAL:** This command:
1. Enables IPv6 routing between interfaces
2. Joins FF02::2 (all-routers multicast group)
3. Starts sending Router Advertisements (RA)
4. Enables hosts to use SLAAC

**Verification:**

```cisco
R1# show ipv6 interface g0/0/0
! Now shows FF02::2 in joined groups
```

#### Step 6: Configure IPv6 on S1

```cisco
S1# configure terminal

! Configure management interface
S1(config)# interface vlan 1
S1(config-if)# ipv6 address 2001:db8:acad:1::b/64
S1(config-if)# ipv6 address fe80::b link-local
S1(config-if)# no shutdown
S1(config-if)# end
```

**Verification:**

```cisco
S1# show ipv6 interface vlan 1
```

#### Step 7: Configure Static IPv6 on PCs

**PC-A Configuration:**
- IPv6 Address: `2001:db8:acad:1::3`
- Prefix Length: `64`
- Default Gateway: `fe80::1`

**PC-B Configuration:**
- IPv6 Address: `2001:db8:acad:a::3`
- Prefix Length: `64`
- Default Gateway: `fe80::1`

** Note:** Each PC will have TWO global IPv6 addresses:
1. Static address (manually configured)
2. SLAAC address (auto-generated from R1's RA)

### Part 3: Verification

#### From PC-A:

```powershell
# Test link-local connectivity to R1
C:\> ping fe80::1

# Test connectivity to S1
C:\> ping 2001:db8:acad:1::b

# Trace route to PC-B
C:\> tracert 2001:db8:acad:a::3
```

**Expected tracert output:**
```
1  2001:db8:acad:1::1  [R1]
2  2001:db8:acad:a::3  [PC-B]
```

#### From PC-B:

```powershell
# Test connectivity to PC-A
C:\> ping 2001:db8:acad:1::3

# Test link-local to R1
C:\> ping fe80::1
```

## Configuration Files

Complete device configurations are available in the [`configs/`](./configs/) directory:

- [`R1-final-config.txt`](./configs/R1-final-config.txt) - Router R1 complete configuration
- [`S1-final-config.txt`](./configs/S1-final-config.txt) - Switch S1 complete configuration
- [`PC-A-config.txt`](./configs/PC-A-config.txt) - PC-A IPv6 settings
- [`PC-B-config.txt`](./configs/PC-B-config.txt) - PC-B IPv6 settings

## Verification Checklist

- [ ] All interfaces show `[up/up]` status
- [ ] R1 has both global and link-local addresses on each interface
- [ ] R1 shows FF02::2 in multicast groups (router role)
- [ ] S1 VLAN 1 has IPv6 address and is up
- [ ] PC-A can ping fe80::1 (R1 link-local)
- [ ] PC-A can ping 2001:db8:acad:1::b (S1)
- [ ] PC-A can ping 2001:db8:acad:a::3 (PC-B)
- [ ] Traceroute shows proper path through R1
- [ ] PCs have both static and SLAAC addresses

## Key Concepts

### IPv6 Address Structure

For address `2001:db8:acad:1::3/64`:

| Part | Value | Description |
|------|-------|-------------|
| Global Routing Prefix | `2001:db8:acad` | 48 bits - /48 |
| Subnet ID | `1` | 16 bits - 4th hextet |
| Interface ID | `::3` | 64 bits - last 64 bits |
| Prefix Length | `/64` | Network portion length |

### Link-Local Address Reuse

**Question:** Why can the same link-local address (fe80::1) be assigned to both interfaces on R1?

**Answer:** Link-local addresses (fe80::/10) are designed for communication ONLY within a single network segment. Packets with link-local addresses are NEVER routed beyond the local link. Since G0/0/0 and G0/0/1 belong to completely separate network segments, there is no possibility of address conflict.

### SLAAC Process

1. Host generates link-local address (fe80:: + interface ID)
2. Host listens for Router Advertisements (sent to FF02::1)
3. Host extracts /64 prefix from RA (e.g., 2001:db8:acad:a::/64)
4. Host generates Interface ID (64 bits) using:
   - EUI-64 (based on MAC address), OR
   - Privacy extensions (random)
5. Host combines: Prefix (64 bits) + Interface ID (64 bits)
6. Host uses router's link-local as the default gateway

**Result:** No DHCP server needed for basic IPv6 addressing!

### IPv6 Multicast Groups

| Group | Purpose |
|-------|---------|
| `FF02::1` | All IPv6 nodes on local link |
| `FF02::2` | All IPv6 routers (enables RA transmission) |
| `FF02::1:FFxx:xxxx` | Solicited-node (used for Neighbor Discovery) |

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Cannot assign IPv6 to S1 VLAN 1 | `sdm prefer dual-ipv4-and-ipv6 default` + reload |
| PCs not getting SLAAC address | Enable `ipv6 unicast-routing` on R1 |
| Interface shows [admin down] | Use `no shutdown` on interface |
| No inter-VLAN routing | Verify R1 has correct /64 on both interfaces |
| PC can't ping default gateway | Check PC's gateway is fe80::1, verify link-local on R1 |

### Verification Commands

```cisco
! On Router
show ipv6 interface brief
show ipv6 interface g0/0/0
show ipv6 route

! On Switch
show sdm prefer
show ipv6 interface vlan 1

! On PC
ipconfig
ping <address>
tracert <address>
```

## Resources

- [Complete Lab Report (PDF)](./docs/Lab_12.9.2_IPv6_Configuration_Complete_Report.pdf)
- [Cisco IPv6 Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipv6_basic/configuration/xe-16/ip6b-xe-16-book.html)
- [RFC 3849 - IPv6 Address Prefix Reserved for Documentation](https://tools.ietf.org/html/rfc3849)
- [RFC 4291 - IPv6 Addressing Architecture](https://tools.ietf.org/html/rfc4291)

## Lab Status

**COMPLETED SUCCESSFULLY**

All objectives achieved:
- ✓ Basic device configuration completed
- ✓ IPv6 global unicast addresses configured
- ✓ Link-local addresses manually assigned
- ✓ IPv6 routing enabled with SLAAC
- ✓ Switch SDM template configured for IPv6
- ✓ Static IPv6 addresses assigned to PCs
- ✓ End-to-end connectivity verified

---

**Lab:** 12.9.2 - Configure IPv6 Addresses on Network Devices  
**Course:** Cisco CCNA  
**Completed:** January 26, 2026

---

## License

This lab documentation is for educational purposes only. Cisco and CCNA are trademarks of Cisco Systems, Inc.
