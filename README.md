# Secure Multi-Site SME Network

A segmented, secured network for a small business with a headquarters (HQ) and two branch offices, designed and built in **Cisco Packet Tracer**.

The project covers VLAN segmentation, inter-VLAN access control, NAT, DHCP, OSPF, a site-to-site VPN and Layer 2 hardening, and documents the design rationale, addressing scheme, configuration baseline and test evidence.

> **Author:** `<your name>` | **Tool:** Cisco Packet Tracer `<version>` | **Status:** work in progress (see below)

---

## Project status

- [x] **Phase 1:** Topology, base device hardening, VLANs, trunks
- [ ] **Phase 2:** Router sub-interfaces, loopbacks, DHCP
- [ ] **Phase 3:** ISP router, WAN links, default routes, NAT
- [ ] **Phase 4:** GRE tunnels and OSPF between sites
- [ ] **Phase 5:** IPsec encryption of inter-site traffic
- [ ] **Phase 6:** Inter-VLAN ACLs (firewall rule matrix)
- [ ] **Phase 7:** Layer 2 hardening (DHCP snooping, DAI, port security)
- [ ] **Phase 8:** Evidence, final report, release `v1.0`

---

## Objectives

1. Build a three-site network with separate VLANs for **staff, servers, guests and management**.
2. Enforce least-privilege traffic between VLANs with **ACLs**, and provide Internet access through **NAT**.
3. Provide addresses automatically with **DHCP**, and connect the sites with **OSPF** over an **encrypted site-to-site VPN**.
4. Harden Layer 2 with **DHCP snooping, Dynamic ARP Inspection (DAI) and port security**.
5. Produce an **IP plan, topology diagram and firewall rule matrix**, and prove the design with **connectivity and blocked-traffic tests**.

---

## Topology

![Topology](docs/diagrams/topology-v0.png)

Hub-and-spoke design: HQ has a tunnel to each branch, and the ISP router simulates the Internet between them.

```
                         [ ISP ]  (simulated Internet, 8.8.8.8)
                        /   |   \
         203.0.113.0/30  198.51.100.0/30  192.0.2.0/30
                      /     |      \
                [HQ-R1]  [BR1-R1]  [BR2-R1]     edge routers: NAT, ACLs, OSPF, VPN
                   |         |         |
               [HQ-SW1]  [BR1-SW1]  [BR2-SW1]   access switches: VLANs + L2 hardening
```

| Role | Device | Model (Packet Tracer) |
|---|---|---|
| Simulated Internet | `ISP` | Cisco 2911 |
| Edge routers | `HQ-R1`, `BR1-R1`, `BR2-R1` | Cisco 2901 |
| Access switches | `HQ-SW1`, `BR1-SW1`, `BR2-SW1` | Cisco 2960-24TT |

---

## Design

### VLANs

The same four VLAN IDs are used at every site, which keeps configurations consistent and easy to audit.

| VLAN | Name | Purpose |
|---|---|---|
| 10 | STAFF | Employee workstations |
| 20 | SERVERS | Internal servers (HQ) |
| 30 | GUESTS | Visitor devices, Internet access only |
| 99 | MGMT | Administration of network devices |
| 999 | BLACKHOLE | Native VLAN and parking VLAN for unused ports |

### Addressing scheme

Pattern: `10.<site>.<VLAN>.0/24`, with site = 10 (HQ), 20 (Branch 1), 30 (Branch 2). The site and VLAN can be read straight from any address.

| VLAN | HQ | Branch 1 | Branch 2 |
|---|---|---|---|
| 10 Staff | 10.10.10.0/24 | 10.20.10.0/24 | 10.30.10.0/24 |
| 20 Servers | 10.10.20.0/24 | 10.20.20.0/24 | 10.30.20.0/24 |
| 30 Guests | 10.10.30.0/24 | 10.20.30.0/24 | 10.30.30.0/24 |
| 99 Management | 10.10.99.0/24 | 10.20.99.0/24 | 10.30.99.0/24 |

Conventions: gateway is `.1` in every VLAN, switch management address is `.2` in VLAN 99, DHCP pool is `.100` to `.200` for Staff and Guests, and servers and management hosts use static addresses.

| Link | Network | Addresses |
|---|---|---|
| HQ to ISP | 203.0.113.0/30 | ISP .1, HQ .2 |
| Branch 1 to ISP | 198.51.100.0/30 | ISP .1, BR1 .2 |
| Branch 2 to ISP | 192.0.2.0/30 | ISP .1, BR2 .2 |
| Tunnel HQ to Branch 1 | 172.16.0.0/30 | HQ .1, BR1 .2 |
| Tunnel HQ to Branch 2 | 172.16.0.4/30 | HQ .5, BR2 .6 |
| Router loopbacks | 10.255.0.1 to .3 /32 | HQ, BR1, BR2 |

The WAN ranges are RFC 5737 documentation addresses, used here as stand-in public IPs.

### Switch port map

| Port | Connected device | VLAN |
|---|---|---|
| Gi0/1 | Trunk to the site router (allowed VLANs 10, 20, 30, 99; native 999) | trunk |
| Fa0/1 | Staff PC | 10 |
| Fa0/2 | Server (HQ only) | 20 |
| Fa0/3 | Guest PC | 30 |
| Fa0/4 | Management host | 99 |
| Fa0/5 | Reserved for the rogue-DHCP test | 30 (shut down) |
| Other ports | Unused, shut down | 999 |

### Firewall rule matrix (policy intent)

Rows are the source, columns are the destination.

| From \ To | Staff | Servers | Guests | Mgmt | Internet | Other site (VPN) |
|---|---|---|---|---|---|---|
| **Staff** | n/a | HTTP, HTTPS, DNS, SMB | Deny | Deny | HTTP, HTTPS, DNS | Servers VLAN only |
| **Servers** | Replies only | n/a | Deny | Deny | Updates only | Replies only |
| **Guests** | Deny | Deny | n/a | Deny | HTTP, HTTPS, DNS | **Deny** |
| **Mgmt** | SSH, ICMP | SSH, HTTPS, ICMP | Deny | n/a | Deny | SSH to remote Mgmt |

Principles: default deny, least privilege, guests isolated from all internal networks, and management reachable only from the management VLAN. One identical ACL set is used at every site, using wildcard masks such as `10.0.20.0 0.255.0.255` ("the Servers VLAN at any site").

### Threat model

| Threat | Control | Where | Test |
|---|---|---|---|
| Guest reaches internal data | VLAN isolation and guest ACL | Edge routers | T06, T08 |
| Unauthorised device administration | Management ACL, VTY access-class, SSH only | Routers and switches | T07 |
| Rogue DHCP server | DHCP snooping | Switches | T13 |
| ARP spoofing / man-in-the-middle | Dynamic ARP Inspection | Switches | T14 |
| Unknown device on a port / MAC flooding | Port security | Switches | T15 |
| Eavesdropping between sites | IPsec over GRE | Edge routers | T11, T12 |
| Bogus routing updates | Passive interfaces by default | Edge routers | T03 |
| VLAN hopping, abuse of unused ports | DTP off, native VLAN 999, unused ports shut | Switches | T02 |

---

## Repository layout

```
.
├── README.md
├── configs/            sanitised running-config of every device
├── lab/                Packet Tracer project files (.pkt)
└── docs/
    └── diagrams/       topology screenshots
```

Planned additions as the build progresses: `evidence/` (screenshots and command outputs per test) and further documents under `docs/` (IP plan, firewall matrix, threat model, build log, final report).

### Config files

| File | Device |
|---|---|
| `configs/isp.cfg` | ISP |
| `configs/hq-r1.cfg`, `br1-r1.cfg`, `br2-r1.cfg` | Edge routers |
| `configs/hq-sw1.cfg`, `br1-sw1.cfg`, `br2-sw1.cfg` | Access switches |

> **Secrets:** passwords, hashes and VPN keys are replaced with `<REDACTED>` placeholders. No real credentials are stored in this repository.

---

## How to reproduce

1. Install Cisco Packet Tracer (free through Cisco Networking Academy) and open the latest file in `lab/`.
2. Or rebuild from scratch: place the devices as in the topology, then paste each device's file from `configs/` into its CLI, replacing the `<REDACTED>` secrets with your own lab-only values.
3. Build in this order so each step can be tested before the next: base hardening, VLANs and trunks, router sub-interfaces and DHCP, ISP and NAT, GRE and OSPF, IPsec, ACLs, Layer 2 hardening.

---

## Verification tests

Each test will have a screenshot and command output stored in `evidence/`. For blocked-traffic tests, both sides are shown: the failed action on the host and the counter or log on the device that blocked it.

| # | Test | Expected result | Status |
|---|---|---|---|
| T01 | DHCP at every site | Correct subnet and gateway | Pending |
| T02 | VLANs and trunks | Correct ports, allowed VLANs, native VLAN 999 | Pass (Phase 1) |
| T03 | OSPF neighbours | Two FULL neighbours at HQ | Pending |
| T04 | Routes learned | Remote subnets present | Pending |
| T05 | Staff to HQ server (allowed) | Page loads | Pending |
| T06 | Guest to Staff (blocked) | Fails, ACL counter increments | Pending |
| T07 | Staff to Management (blocked) | Fails | Pending |
| T08 | Guest across VPN (blocked) | Fails | Pending |
| T09 | Guest to Internet (allowed) | Replies | Pending |
| T10 | NAT translations | Private addresses translated | Pending |
| T11 | VPN established | Active SA, counters rising | Pending |
| T12 | VPN traffic encrypted | ESP only in packet capture | Pending |
| T13 | Rogue DHCP server | Offer dropped by snooping | Pending |
| T14 | DAI | Unbound ARP dropped | Pending |
| T15 | Port security | Violation recorded, port shut down | Pending |
| T16 | Tunnel failover (bonus) | Routes withdraw and recover | Pending |

---

## Known limitations

- The Packet Tracer VPN uses **IKEv1 with older algorithms** and a crypto map protecting GRE. A production network would use IKEv2 with stronger parameters.
- **DAI covers only the DHCP-assigned VLANs** (Staff, Guests). Static-address VLANs would need ARP ACLs.
- ACLs are **not fully stateful**: return traffic for UDP and ICMP is permitted explicitly. A zone-based firewall would improve this.
- Packet Tracer cannot run real attack tools, so ARP spoofing protection is demonstrated with a substitute test (an unbound static host being dropped).
- Feature support varies between Packet Tracer versions, so results may differ on yours.

## Roadmap

- Zone-based firewall for stateful inspection
- OSPF authentication
- Centralised logging (syslog)
- Rebuild in GNS3 or Containerlab with automated configuration (Ansible, Netmiko)

---

*Educational lab project. All addresses are private or reserved documentation ranges.*
