# Multi-Vendor Enterprise Security Lab

Multi-vendor enterprise network lab in EVE-NG: Palo Alto Active/Passive HA, FortiGate and Juniper vSRX edge firewalls, Huawei CE12800 switching, route-based IPsec VPN with a BGP overlay.



---

## Table of contents

- [Overview](#overview)
- [Lab objectives](#lab-objectives)
- [Devices and images](#devices-and-images)
- [Network architecture](#network-architecture)
- [IP addressing](#ip-addressing)
- [Interface map](#interface-map)
- [Palo Alto high availability](#palo-alto-high-availability)
- [IPsec VPN and BGP overlay](#ipsec-vpn-and-bgp-overlay)
- [Layer 2 switching](#layer-2-switching)
- [Verification](#verification)
- [HA failover test](#ha-failover-test)
- [Repository structure](#repository-structure)
- [How to reproduce](#how-to-reproduce)
- [Key takeaways](#key-takeaways)
- [Author](#author)

---

## Overview

This lab simulates an enterprise network in which a data centre firewall pair provides secure
connectivity to two remote branches built on different vendors. It was built and fully verified in
EVE-NG.

The design covers four areas that come up constantly in real enterprise networks:

- **Resilience** — an Active/Passive firewall cluster with sub-second to low-second failover.
- **Multi-vendor interoperability** — Palo Alto, Fortinet and Juniper terminating IPsec with each other.
- **Dynamic overlay routing** — BGP peering across route-based VPN tunnels instead of static routes.
- **Transit switching** — Huawei CE12800 switches tuned for firewall-facing ports.

| Site | Role | Firewall | LAN |
|---|---|---|---|
| DC | Data centre core | Palo Alto HA pair (PA-FW-01 / PA-FW-02) | 192.168.10.0/24 |
| Branch B | Remote branch | FortiGate | 192.168.20.0/24 |
| Branch A | Remote branch | Juniper vSRX-NG | 192.168.30.0/24 |

---

## Lab objectives

1. Deploy a Palo Alto Active/Passive HA cluster with dedicated HA1 control and HA2 data links.
2. Terminate route-based IPsec VPN tunnels between three different firewall vendors.
3. Run BGP over the tunnels so every site learns remote prefixes dynamically.
4. Provide Layer 2 transit and LAN distribution with Huawei CE12800 switches.
5. Prove end-to-end reachability between all three sites and validate failover with zero or minimal packet loss.

---

## Devices and images

| Device | Vendor / platform | Role |
|---|---|---|
| PA-FW-01 | Palo Alto Networks VM-Series | HA primary (data centre) |
| PA-FW-02 | Palo Alto Networks VM-Series | HA secondary (data centre) |
| Fortinet | FortiGate | Branch B edge firewall |
| vSRX-NG | Juniper vSRX | Branch A edge firewall |
| Core-SW-1 | Huawei CE12800 (VRP v8) | Layer 2 transit switch |
| Core-SW-2 | Huawei CE12800 (VRP v8) | LAN distribution and transit switch |
| PC1 / PC2 / PC3 | Linux host | End-user test hosts |
| mgmt cloud | EVE-NG cloud | Out-of-band management network |

> Software versions used in this build are listed in [`docs/`](docs/).

---

## Network architecture

**Data centre**

The two Palo Alto firewalls run as an Active/Passive cluster. Each unit connects to Core-SW-1 for
transit towards Branch E and to Core-SW-2 for the DC LAN and the path to Branch A. Because the
cluster is Active/Passive, only the active unit forwards traffic; the passive unit's data ports are
held down so the transit switches never learn duplicate MAC addresses.

**Branch B**

A FortiGate firewall terminates the branch LAN on port3 and reaches the data centre through
Core-SW-1 over the 10.10.10.0/30 transit link.

**Branch A**

A Juniper vSRX terminates the branch LAN on ge-0/0/1 and reaches the data centre through Core-SW-2
over the 10.10.20.0/30 transit link.

**Overlay**

Route-based IPsec tunnels are built on top of the transit links, and BGP peers across those tunnels
carry the LAN prefixes between all three sites.

**Management**

Every firewall has a dedicated management interface in the 10.254.4.0/24 out-of-band network,
reachable through the EVE-NG management cloud.

---

## IP addressing

### LAN segments

| Segment | Subnet | Gateway | Host |
|---|---|---|---|
| Branch B LAN | 192.168.20.0/24 | 192.168.20.1 (FortiGate port3) | PC1 — 192.168.20.10 |
| DC LAN | 192.168.10.0/24 | 192.168.10.1 (Palo Alto eth1/1) | PC2 — 192.168.10.10 |
| Branch A LAN | 192.168.30.0/24 | 192.168.30.1 (vSRX ge-0/0/1) | PC3 — 192.168.30.10 |

### Transit links

| Link | Subnet | Endpoints |
|---|---|---|
| FortiGate ↔ Palo Alto (via Core-SW-1) | 10.10.10.0/30 | Palo Alto .1, FortiGate .2 |
| Palo Alto ↔ vSRX (via Core-SW-2) | 10.10.20.0/30 | Palo Alto .1, vSRX .2 |

### Management

| Device | Management IP |
|---|---|
| PA-FW-01 | 10.254.4.161/24 |
| PA-FW-02 | 10.254.4.153/24 |
| FortiGate | 10.254.4.37/24 |

---

## Interface map

### FortiGate

| Interface | Address | Connected to |
|---|---|---|
| port1 | 10.254.4.37/24 | Management cloud |
| port2 | 10.10.10.2/30 | Core-SW-1 GE1/0/0 |
| port3 | 192.168.20.1/24 | PC1 (Branch B LAN) |

### PA-FW-01 (primary)

| Interface | Address | Connected to |
|---|---|---|
| eth1/1 | 192.168.10.1/24 | Core-SW-2 GE1/0/1 (DC LAN gateway) |
| eth1/2 | 10.10.10.1/30 | Core-SW-1 GE1/0/1 |
| eth1/3 | 10.10.20.1/30 | Core-SW-2 GE1/0/4 |
| eth1/4 | HA1 control | PA-FW-02 eth1/4 (direct) |
| eth1/5 | HA2 data | PA-FW-02 eth1/5 (direct) |
| mgmt | 10.254.4.161/24 | Management cloud |

### PA-FW-02 (secondary)

| Interface | Connected to |
|---|---|
| eth1/1 | Core-SW-2 GE1/0/2 |
| eth1/2 | Core-SW-1 GE1/0/2 |
| eth1/3 | Core-SW-2 GE1/0/6 |
| eth1/4 | PA-FW-01 eth1/4 (HA1 control) |
| eth1/5 | PA-FW-01 eth1/5 (HA2 data) |
| mgmt | 10.254.4.153/24 |

Data-plane addressing is inherited from the active unit; the passive unit does not hold its own
data-plane IPs.

### Juniper vSRX-NG

| Interface | Address | Connected to |
|---|---|---|
| ge-0/0/0 | 10.10.20.2/30 | Core-SW-2 GE1/0/5 |
| ge-0/0/1 | 192.168.30.1/24 | PC3 (Branch A LAN) |

### Huawei CE12800 switches

**Core-SW-1 (transit)**

| Port | Connected to |
|---|---|
| GE1/0/0 | FortiGate port2 |
| GE1/0/1 | PA-FW-01 eth1/2 |
| GE1/0/2 | PA-FW-02 eth1/2 |

**Core-SW-2 (LAN and transit)**

| Port | Connected to |
|---|---|
| GE1/0/0 | PC2 |
| GE1/0/1 | PA-FW-01 eth1/1 |
| GE1/0/2 | PA-FW-02 eth1/1 |
| GE1/0/4 | PA-FW-01 eth1/3 |
| GE1/0/5 | vSRX ge-0/0/0 |
| GE1/0/6 | PA-FW-02 eth1/3 |

---

## Palo Alto high availability

| Setting | PA-FW-01 | PA-FW-02 |
|---|---|---|
| Mode | Active/Passive | Active/Passive |
| Device priority | 50 (preferred active) | 100 |
| Preemptive | Enabled | Enabled |
| Preemption hold time | 1 minute | 1 minute |
| HA1 control link | eth1/4 | eth1/4 |
| HA2 data link | eth1/5 | eth1/5 |
| Passive link state | Shutdown | Shutdown |

**Why these choices**

- **Lower priority wins.** In PAN-OS the firewall with the lower device priority value becomes
  active, so PA-FW-01 at 50 is the preferred active unit.
- **Passive link state = Shutdown.** Holding the passive unit's data ports down stops the transit
  switches from learning the same MAC addresses on two ports, which avoids MAC and ARP table
  thrashing on the CE12800s.
- **Preemption with a hold timer.** Preemption returns the cluster to the preferred unit after a
  failure, and the hold timer stops it flapping back before the recovered unit is ready to forward.
- **Dedicated HA links.** Separating HA1 control from HA2 data keeps heartbeat and hello traffic off
  the session-synchronisation path.



---

## IPsec VPN and BGP overlay

Route-based (tunnel interface) IPsec is used rather than policy-based, so that routing decides what
enters the tunnel and BGP can peer across it.

**Design**

- Tunnel interfaces on each firewall, with BGP peering over the tunnel addresses.
- Each site advertises its own LAN prefix; no static routes between sites.
- Recovery after a failure is handled by BGP reconvergence rather than manual intervention.

**Why BGP over the tunnels**

Static routing across three vendors is brittle: every new subnet means touching three devices. With
BGP, adding a prefix at one site propagates automatically, and path selection can be influenced with
standard attributes if a second path is added later.

Per-vendor configuration is in [`configs/`](configs/).



---

## Layer 2 switching

Both CE12800 switches run VRP v8 and provide pure Layer 2 transit. Key points:

- **STP disabled on firewall-facing ports** (`stp disable`). The firewalls do not participate in
  spanning tree, and leaving STP enabled adds unnecessary listening and learning delay to ports
  that can never form a loop through the firewall.
- **Separate VLANs per function on Core-SW-2**, keeping the DC LAN (192.168.10.0/24) and the vSRX
  transit segment (10.10.20.0/30) in different broadcast domains.
- **Jumbo frame support** up to 9216 bytes by default, which leaves headroom for the IPsec overhead
  added by tunnelling.

> Configs: `configs/huawei/`

---

## Verification

The lab is validated at four layers:

| Layer | Check |
|---|---|
| Physical / L2 | Interface and VLAN state, MAC address table on both switches |
| L3 transit | Ping across each /30 transit link |
| Overlay | IKE and IPsec SAs up on all three firewalls |
| Routing | BGP peers established, remote LAN prefixes present in every routing table |
| End to end | PC1 ↔ PC2 ↔ PC3 ping and traceroute |

Useful commands:

```text
# Palo Alto
show high-availability state
show vpn ike-sa
show vpn ipsec-sa
show routing protocol bgp summary
show routing route

# FortiGate
get vpn ipsec tunnel summary
get router info bgp summary
get router info routing-table all

# Juniper vSRX
show security ike security-associations
show security ipsec security-associations
show bgp summary
show route

# Huawei CE12800
display vlan
display interface brief
display mac-address
```



---

## HA failover test

| Step | Action | Expected result |
|---|---|---|
| 1 | Start a continuous ping from PC1 to PC3 | Replies with no loss |
| 2 | Confirm the cluster state | PA-FW-01 active, PA-FW-02 passive, all sync states green |
| 3 | Suspend or power off the active unit | PA-FW-02 transitions to active |
| 4 | Observe the ping | Only a small number of packets lost during the transition |
| 5 | Restore PA-FW-01 | It preempts back to active after the hold timer expires |



---

## Repository structure

```text
.
├── README.md
├── docs/                 # Full SOP / build document (PDF and Word)
├── topology/             # EVE-NG .unl export and topology diagram
├── configs/
│   ├── paloalto/
│   ├── fortigate/
│   ├── vsrx/
│   └── huawei/
└── screenshots/
    ├── ha/
    ├── vpn/
    ├── bgp/
    ├── switching/
    ├── verification/
    └── failover/
```

---

## How to reproduce

1. Install EVE-NG (Community or Pro) and upload the Palo Alto VM-Series, FortiGate, vSRX and Huawei
   CE12800 images to `/opt/unetlab/addons/qemu/`, then run
   `/opt/unetlab/wrappers/unl_wrapper -a fixpermissions`.
2. Import the lab file from [`topology/`](topology/) through the EVE-NG web UI.
3. Start the nodes and give each firewall a management IP in the 10.254.4.0/24 range.
4. Apply the switch configs from [`configs/huawei/`](configs/huawei/) first, so Layer 2 transit is up
   before the firewalls come online.
5. Build the Palo Alto HA pair, and confirm the cluster forms before adding any tunnels.
6. Apply the firewall interface, zone and policy configs, then verify the /30 transit links with ping.
7. Build the IPsec tunnels, then the BGP peerings on top of them.
8. Run the verification and failover steps above.

**Lab resources** — allow roughly 6.5 GB of RAM for the two Palo Alto VMs, plus memory for the
FortiGate, vSRX, switches and hosts. Check the vendor sizing guides for your specific images.

---

## Key takeaways

- In PAN-OS Active/Passive HA, the **lower** device priority value becomes active, and preemption
  needs a hold timer to prevent flapping.
- Setting the passive unit's link state to **Shutdown** is what keeps the upstream switches' MAC and
  ARP tables stable in a shared-switch HA design.
- **Route-based IPsec** is what makes dynamic routing over VPN possible; policy-based tunnels cannot
  carry a routing adjacency.
- Running **BGP over the overlay** removes the per-site static routing that makes multi-vendor
  designs fragile as they grow.
- Disabling STP on firewall-facing switch ports removes needless convergence delay on links that
  cannot form a loop.

---

## Author

**Abdur Rehman** — Network & Security Engineer

Built and verified in EVE-NG. Issues and suggestions are welcome.

---

*This lab is for education and testing. Configurations should be reviewed and hardened before being
adapted for production use.*
