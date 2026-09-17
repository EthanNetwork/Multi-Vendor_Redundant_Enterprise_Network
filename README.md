 Multi-Vendor Redundant Office Network Lab

A enterprise network built to demonstrate high-availability routing, switching, and firewalling across three vendors: **Cisco**, **Aruba**, and **Fortinet**. The lab models a dual-uplink office with VLAN segmentation, redundant default gateways, and dual-path dynamic routing between the core routers.

## Topology

![Topology diagram](topology.png)

```
                          Internet (WAN1 / WAN2)
                                   |
                          [ FortiGate-VM64-KVM ]
                          port3  port4  port1  port2
                                          |      |
                                        Gi0/1  Gi0/1
                                      [ASBR1] [ASBR2]   (EIGRP + OSPF, HSRP)
                                       Gi0/0   Gi0/0
                                        |        |
                                      Gi1/0    Gi1/0
                                    [Switch9]-Po1-[Switch10]  (L2 core, LACP)
                                    Gi0/0 Gi0/1  Gi0/0 Gi0/1
                                     |     |      |     |
                                 ArubaCX3 ArubaCX4 ArubaCX0 ArubaCX5
                                     |     |      |     |
                                  VPC11  VPC12  VPC13  VPC14
```

| Device | Role | Platform |
|---|---|---|
| FortiGate-VM64-KVM | Edge firewall / dual-WAN router | FortiOS (FortiGate-VM64) |
| ASBR1, ASBR2 | Distribution/border routers | Cisco IOSv, 15.9 |
| Switch9, Switch10 | L2 core switches | Cisco IOSv (L2), 15.2 |
| ArubaCX0, CX3, CX4, CX5 | Access switches | ArubaOS-CX Virtual 10.04 |
| VPC11–VPC14 | End hosts | Generic VPCS/PC nodes |

## Design Goals

- **Vendor interoperability** — standards-based protocols (802.1Q, LACP, HSRPv2, OSPF, EIGRP) tying Cisco, Aruba, and Fortinet gear together in one Layer 2/3 fabric.
- **Redundant edge** — FortiGate has two WAN uplinks (`port3`/`port4`) with a primary route and a floating static backup route (`distance 20`).
- **Redundant distribution** — two border routers (ASBR1/ASBR2), each with its own path to the FortiGate, load-share gateway duties for the office VLANs via HSRP.
- **Redundant core** — two core switches interconnected by a 2-link LACP port-channel, each uplinked to its own ASBR, so either core switch or ASBR can fail without an outage.
- **Segmented access layer** — four VLANs carried end-to-end to Aruba access switches, each providing a locked-down, port-secured edge port to a host.

## Addressing

| VLAN | Subnet | HSRP VIP | Active on |
|---|---|---|---|
| 10 | 172.168.10.0/24 | 172.168.10.3 | ASBR1 (priority 255) |
| 20 | 172.168.20.0/24 | 172.168.20.3 | ASBR1 (priority 255) |
| 30 | 172.168.30.0/24 | 172.168.30.3 | ASBR2 (priority 255) |
| 40 | 172.168.40.0/24 | 172.168.40.3 | ASBR2 (priority 255) |
| 99 | 172.168.99.0/24 | — | Native/management VLAN (EIGRP transit) |

| Link | Subnet |
|---|---|
| ASBR1 Gi0/1 ↔ FortiGate port1 | 172.168.48.0/30 |
| ASBR2 Gi0/1 ↔ FortiGate port2 | 172.168.48.4/30 |
| FortiGate port3 (WAN1) | 203.20.1.0/30 |
| FortiGate port4 (WAN2, backup) | 203.20.1.4/30 |

Router IDs / loopbacks: ASBR1 `10.1.1.2/32`, ASBR2 `10.1.1.1/32`.

## Routing Design

Both ASBRs run **two routing protocols simultaneously**, redistributed into each other:

- **EIGRP (AS 100, "OFFICE")** runs over all four VLAN sub-interfaces plus the native VLAN and each router's loopback, directly between ASBR1 and ASBR10 across the switched core. All EIGRP interfaces use **HMAC-SHA-256 authentication**.
- **OSPF (process 1, area 0)** runs only on the WAN-facing links to the FortiGate, plus VLAN10/20 on ASBR1 and VLAN30/40 on ASBR2, with mutual redistribution between EIGRP and OSPF (seed metric `10000 100 255 1 1500` into OSPF).

This gives each ASBR a path to the internet edge via its own OSPF adjacency with the FortiGate, and a second, independent path to every VLAN via EIGRP across the core switches — so the office subnets stay reachable even if one ASBR-to-FortiGate link or one ASBR itself goes down.

**First-hop redundancy (HSRPv2)** splits gateway duties evenly: ASBR1 is primary for VLAN10/20, ASBR2 is primary for VLAN30/40, with `preempt` enabled on both so the primary reclaims the role after recovery. Spanning-tree priorities on Switch9/Switch10 are aligned with this split (Switch9 is STP root for VLAN10/20, Switch10 is STP root for VLAN30/40), keeping the STP-forwarding topology and the L3 gateway on the same physical side.

## Switching Design

- **Switch9 ↔ Switch10**: Gi0/2 + Gi0/3 bundled into `Port-channel1` (`channel-group 1 mode on` — static LACP), trunking VLANs 10/20/30/40 with VLAN 99 as native.
- **Switch9/Switch10 → ASBR**: Gi1/0 trunk uplink to the router's dot1Q sub-interfaces.
- **Switch9/Switch10 → Aruba access switches**: Gi0/0 and Gi0/1 trunk downlinks.
- **Aruba access switches → hosts**: `1/1/1` is the uplink trunk (native 99, tagged 10/20/30/40); `1/1/2` is an access port hardened with `bpdu-guard`, `admin-edge` (PortFast equivalent), and port-security (`client-limit 10`, shutdown on violation).

| Access switch | Uplink (core) | Access port VLAN | Host |
|---|---|---|---|
| ArubaCX3 | Switch9 Gi0/0 | 10 | VPC11 |
| ArubaCX4 | Switch9 Gi0/1 | 20 | VPC12 |
| ArubaCX0 | Switch10 Gi0/0 | 20 | VPC13 |
| ArubaCX5 | Switch10 Gi0/1 | 40 | VPC14 |

## Firewall / Edge Design (FortiGate)

- `port1` / `port2`: LAN-side interfaces to ASBR1 / ASBR2 (`172.168.48.2/30`, `172.168.48.6/30`).
- `port3` / `port4`: WAN-side interfaces (`203.20.1.2/30`, `203.20.1.6/30`) for two upstream ISPs.
- Static routing: default route via `port3` (gateway `203.20.1.1`); a floating backup default route via `port4` (gateway `203.20.1.5`, `distance 20`) for WAN failover.
- Firewall policies:
  - `LAN_to_WAN` — allows and NATs all traffic from `port1`/`port2` out to `port3`/`port4`.
  - `ASBR1_to_ASBR2` / `ASBR2_to_ASBR1` — allows direct transit traffic between the two border routers through the firewall (supports the OSPF path between ASBRs across the FortiGate).

