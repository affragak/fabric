# The fabric

[← Overview](../README.md)

Two small leaf-and-spine data centre fabrics with VXLAN and BGP EVPN, joined by an inter-site
network into one VXLAN EVPN Multi-Site fabric. Multi-tenant, with a vPC pair and an external
connection per tenant. It is described once, in a data model, and everything else is derived from
that description.

## Topology

<p align="center">
  <img src="../assets/diagram-topology.svg" width="760" alt="Two sites joined by an inter-site network. DC-1, AS 65001: spine-11 and spine-12, each linked to leaf-101, leaf-102, leaf-103, leaf-104 and two border leafs; leaf-101 and leaf-102 are a vPC pair with a dual-homed host; the border leafs peer with an external router by eBGP, one session per tenant. DC-2, AS 65002: spine-21 and spine-22, each linked to leaf-201 and leaf-202. The spines of both sites are Multi-Site border gateways. The inter-site network, AS 65000: isn-1 peers by eBGP with DC-1's spines, isn-2 with DC-2's, and the two with each other. The border gateways of the two sites peer by eBGP EVPN across it.">
</p>

Fourteen switches in three groups. Every switch belongs to exactly one of them, and the group
holds what is per site: the AS number, the route reflectors, the rendezvous point.

| Site | AS | Switches |
|---|---|---|
| **DC-1** | 65001 | 2 spines, 4 leafs, 2 border leafs |
| **DC-2** | 65002 | 2 spines, 2 leafs |
| **Inter-site network** | 65000 | 2 routers |

| Switch | Site | Role | Notes |
|---|---|---|---|
| spine-11, spine-12 | DC-1 | Spine | OSPF, PIM rendezvous point, BGP EVPN route reflectors; Multi-Site border gateways of site 1 |
| leaf-101, leaf-102 | DC-1 | Leaf | A vPC pair: shared VTEP address, a dual-homed host on a vPC port channel |
| leaf-103 | DC-1 | Leaf | Standalone; the canary of every deploy |
| leaf-104 | DC-1 | Leaf | Standalone, no host yet; the first switch that was provisioned zero-touch |
| leaf-border-1, leaf-border-2 | DC-1 | Border leaf | Carry the tenants' L3VNIs and peer with the outside, one eBGP session per tenant |
| spine-21, spine-22 | DC-2 | Spine | As DC-1's spines; Multi-Site border gateways of site 2 |
| leaf-201, leaf-202 | DC-2 | Leaf | Single-homed, no vPC; no host yet |
| isn-1, isn-2 | Inter-site | Router | Route IPv4 between the sites' border gateways, nothing else: no OSPF, no PIM, no VXLAN |

All fourteen are Cisco Nexus 9000v (the 9300v image) on NX-OS 10.5(3). Inside a site every leaf
has one routed point-to-point link to each spine with jumbo MTU. Beyond DC-1's border leafs sits
an external router, configured by hand and not part of the model, which stands for the rest of
the network; DC-2 has no border leaf, and its way out is through DC-1. Six small lab hosts hang
off DC-1's leafs; the data plane tests log in to them and ping.

How the two sites are joined, and what crosses between them, is on the page about
[Multi-Site](multisite.md).

![The first site on the EVE-NG canvas](../assets/eve-ng-topology.png)

*The first site as drawn in EVE-NG on 19 November 2026, including the out-of-band management
switch on the left. Since then the canvas has gained leaf-104, DC-2, the two inter-site routers
and the POAP server.*

## Underlay

Each site has an underlay of its own; nothing of it crosses to the other site.

| | |
|---|---|
| Routing | OSPF, one area per site, point-to-point links, hello 1 s and dead 6 s |
| Loopbacks | One for routing and BGP, one for the VTEP; the vPC pair shares a secondary VTEP address. A border gateway has a third, with the address that the gateways of its site share |
| BUM traffic | Multicast inside a site: PIM sparse mode with an anycast rendezvous point on both spines, one group per L2VNI, the same groups in both sites. Between the sites: ingress replication |
| MTU | 9216 on the fabric links and on the links to the inter-site network |
| Inter-site network | BGP only: each router peers by eBGP with the border gateways of its site and by iBGP with the other router, and carries the gateways' loopbacks |

The fast OSPF timers are a result of a game day: with the defaults, a silent link failure kept
traffic going into a dead link for the full 40 seconds of the dead timer. BFD is configured in the
model as well, but the virtual 9000v never brings a BFD session up, so the timers do the work here.

The rendezvous point was PIM bidir with a phantom address until the second site came. Cisco does
not support bidir as the underlay of a Multi-Site fabric, so DC-2 was built with sparse mode and
an anycast rendezvous point, and DC-1 was moved to it in a change of its own before its spines
became border gateways.

## Overlay

| | |
|---|---|
| Control plane | Per site: iBGP in the site's AS, address family L2VPN EVPN, peering between loopbacks, both spines as route reflectors |
| Between the sites | The spines are anycast border gateways: eBGP L2VPN EVPN from every gateway of one site to every gateway of the other, with the route targets rewritten to the receiving site's AS |
| Data plane | VXLAN, with host reachability learned through BGP |
| Gateway | Distributed anycast gateway: the same gateway address and MAC on every leaf, in both sites |
| Tenancy | One VRF per tenant with an L3VNI for routing between its networks, one L2VNI per network; the same VNIs in both sites |
| Border | The border leafs hold the L3VNIs only: no tenant VLANs, no gateways |
| External | One eBGP session per tenant VRF on dot1q subinterfaces; tenant networks out, the default route in; keepalive 3 s, hold 9 s |

## Tenants

| Tenant | Networks | External | Reaches |
|---|---|---|---|
| RED | 3 | yes | itself, SHARED-SERVICES, outside |
| BLUE | 2 | yes | itself, SHARED-SERVICES, outside |
| GREEN | 1 | yes | itself, SHARED-SERVICES, outside |
| SHARED-SERVICES | 1 | no | every tenant, nothing outside |

The tenants are the same in both sites: one file describes them, and every leaf of either site
gets them from it. All four L3VNIs and all seven L2VNIs are to be stretched between the sites;
[how far that is](multisite.md#where-it-stands) is on the Multi-Site page.

Tenants are isolated from one another. Each one leaks routes to and from SHARED-SERVICES through
extra route targets and an import filter that admits the other side's gateway subnets and nothing
else. The tests check both directions: pairs that should reach each other do, and pairs that
should not, do not.

Numbering follows one scheme, so a number says what it is. With *x* the tenant's index:

| Object | Scheme | Example (BLUE, x = 1) |
|---|---|---|
| L3VNI | `5000x` on VLAN `200x` | VNI 50001 on VLAN 2001 |
| L2VNI | `30x0y` on VLAN `2(3+x)0y` | VNI 30101 on VLAN 2401 |
| Multicast group | the next free one of a fixed range | |

## The data model

The model is Ansible inventory. A few kinds of file, and each kind of change deploys differently:

| File | Describes | A merge deploys |
|---|---|---|
| `group_vars/leafs/tenants.yml` | Tenants: VRFs, networks, VNIs, gateways, route leaking | **tenant**: BGP and overlay roles on the leafs and border leafs. While a border gateway carries the tenants, **full**: the gateways are spines, and a tenant deploy does not reach them |
| `host_vars/<switch>.yml` | One switch: links, loopbacks, host ports, vPC, external connections | **full**: every role on every switch |
| `group_vars/<site>.yml` | One site: its AS, route reflectors and their clients, rendezvous point, OSPF timers, Multi-Site settings | **full** |
| `group_vars/*.yml` | Settings per role and fabric-wide: features, NTP, BGP, login security, telemetry | **full** |
| `hosts.yml` | The switches, their roles and their sites | **full** |
| `provisioning.yml` | Switches declared but not yet part of the fabric | **nothing**: the pipeline does not see them until they join |
| Documentation, tests, scripts, CI | | **nothing** |

A script decides the deploy action from the list of changed files; the same script answers the
question before a pull request is opened.

A network is a few lines:

```yaml
tenants:
  - name: BLUE
    l3vni:
      vlan_id: 2001
      vni_id: 50001
    networks:
      - name: VRF-BLUE-L2VNI-30101
        vlan_id: 2401
        vni_id: 30101
        ip_address: 192.168.3.1
        mask: 24
        mcast_grp: 239.1.1.3
    route_targets:
      import: ["{{ bgp_asn }}:50003"]  # SHARED-SERVICES, in the site's own AS
    import_filter:
      networks_of: [SHARED-SERVICES]
    advertise_networks: true
```

From this the leafs of both sites get the VLAN, the VNI, the SVI with the anycast gateway, the
EVPN instance and the multicast group; the border leafs get the route advertisement; the tests get the expectation
that the network is up on every leaf and reachable from the outside; the documentation gets a row.

### Validation

Before anything else, the model is checked without touching a device:

- **Syntax and shape**: yamllint, then a JSON schema for the tenants.
- **Consistency**: VLAN ids, VNI ids and names unique across all tenants; no overlapping subnets
  between tenants that leak routes to each other; leak rules that name existing tenants;
  aggregates at the border that match the tenants' networks; timers and port numbers in range.
- **Sites**: every switch in exactly one site; every leaf peers with every spine of its own site
  and with nothing else; OSPF links stay inside a site; both ends of every link declared; a
  border gateway has its loopbacks, its links to the inter-site network and the other site's
  gateways as peers, each in the right AS; no address used twice.
- **Safety**: a pinned SSH host key for every switch; login security present.

### Rules that bite

- **What is not in the model is removed.** Most resources are applied in *overridden* or
  *replaced* state. A deleted line in the model is a deleted line of configuration.
- **The feature list is authoritative.** NX-OS features that are not listed are disabled.
- **Unused ports are shut down.** A port exists in the model or it is off.

These rules are why the dry run matters: it shows every command a deploy would send, including
the removals nobody intended.

## From model to switch

| | |
|---|---|
| Engine | Ansible with the `cisco.nxos` resource modules over SSH; one automation account with key login; host keys pinned |
| Templates | 30 Jinja2 templates render the modules' payloads from the model |
| Roles | common, underlay, bgp, overlay, border, border gateway, inter-site router, SNMP, telemetry, login security, removal of unknown users |
| New switches | A POAP server gives a new switch a minimal first configuration rendered from the same model; the ordinary deploy does the rest ([setup](setup.md#2-day-zero-on-each-switch)) |
| Dry run | The same playbook in check mode; the commands each switch would receive are recorded per host |
| Toolchain | One container image with Ansible, pyATS, Batfish's client and the linters, every dependency locked by hash; the image is scanned and signed, and its tag is a hash of its inputs |

A person and the pipeline use the same image through the same wrapper, so "it worked on my
machine" and "it worked in CI" are the same statement.

## Next

- [Multi-Site](multisite.md): the second site and what joins the two
- [A worked example](example-add-a-network.md): six lines of this model becoming a network
- [The pipeline](pipeline.md): how a change to the model reaches the switches
- [Game days](game-days.md): how the timers above were arrived at
- [Setup](setup.md): how the lab is built
