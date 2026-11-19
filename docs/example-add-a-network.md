# A worked example: adding a network to a tenant

[← Overview](../README.md)

The other pages describe the workflow. This one shows it: one real change, with what each step
produced. Nothing here is written for the page; the diff, the tables, the commands and the
reports are what the repository, the pipeline and the ticket hold.

| | |
|---|---|
| **The change** | A fourth network in tenant RED: VLAN 2304, VNI 30004, gateway 192.168.7.1/24 |
| **When** | 10 November 2026, 12:53 to 14:23 UTC: ninety minutes from the ticket to its closing |
| **Deploys** | *full*: three leafs get the network, two border leafs one line, the spines nothing |
| **Raised by** | The network agent, on the owner's request |
| **Outcome** | Deployed through three canary batches, 67 tests passed, verified on the switches |

> **Two honest notes.** The network was a rehearsal of the most common change, and the reverse
> change removed it the next day; the row it added to the documentation is shown below as it
> looked. And the workflow has been tightened since: today the ticket system enforces that the
> approval follows the dry run, and only the owner merges. The current rules are on the
> [change workflow](change-workflow.md) page.

## 1. The ticket

The agent opens a change ticket before it touches a file. Its description, shortened in places
and otherwise as written:

> **What:** a new L2VNI network in tenant RED, requested by the owner: name VRF-RED-L2VNI-30004,
> VLAN 2304, VNI 30004, anycast gateway 192.168.7.1/24, multicast group 239.1.1.9. Not attached to
> any host port.
>
> Uniqueness checked against main: VLAN 2304, VNI 30004 and group 239.1.1.9 unused (239.1.1.9 is
> the next free group); 192.168.7.0/24 not used in RED.
>
> **Affected devices:** leaf-101, leaf-102, leaf-103 (VLAN, SVI with anycast gateway, NVE member
> VNI 30004 with mcast-group, EVPN vni, prefix-list entry), leaf-border-1, leaf-border-2
> (aggregate-address 192.168.7.0/24 summary-only in VRF RED). Spines: expected 0 changes.
>
> **Deploy action:** full, through the canary (leaf-103, spine-12, rest). **Risk:** high (full
> deploy).
>
> **Pre-checks:** fabric healthy on all 7; no new alarms; no other change implementing; no
> pipeline run active.
>
> **Rollback:** a failing canary batch rolls back to the checkpoint automatically. Otherwise a
> revert pull request with the same Change trailer: the full deploy removes VLAN 2304, the SVI,
> VNI 30004 and the aggregate (nothing depends on them: no port, no host).
>
> **Verification:** dry run as expected, 0 errors. Merge run: gate approved, canary, tests 0
> failed per batch, save. `show nve vni 30004` on the leafs: Up, mcast 239.1.1.9; `show vlan id
> 2304` lists the peer-link on the vPC pair; `vni 30004 l2` in the EVPN configuration; interface
> Vlan2304 192.168.7.1/24 in VRF RED; the aggregate on both border leafs; a dry run of one leaf
> shows 0 changes; fabric healthy; NetBox branch merged; no new alarms.

The verification list is written *before* the change exists. Afterwards the change is checked
against it, item by item.

## 2. The change

Seven lines in two files. The network itself:

```diff
--- a/inventory/group_vars/leafs/tenants.yml
+++ b/inventory/group_vars/leafs/tenants.yml
@@ -23,6 +23,12 @@ tenants:
         ip_address: 192.168.6.1
         mask: 24
         mcast_grp: 239.1.1.7
+      - name: VRF-RED-L2VNI-30004
+        vlan_id: 2304
+        vni_id: 30004
+        ip_address: 192.168.7.1
+        mask: 24
+        mcast_grp: 239.1.1.9
     route_targets:
       import: ["65001:50003"]    # SHARED-SERVICES
     import_filter:
```

and one line at the border:

```diff
--- a/inventory/group_vars/border_leafs.yml
+++ b/inventory/group_vars/border_leafs.yml
@@ -42,6 +42,7 @@ border_vrfs:
       - 192.168.1.0/24
       - 192.168.2.0/24
       - 192.168.6.0/24
+      - 192.168.7.0/24
   - vrf: BLUE
```

The second file is there because validation insists on it: RED has an external connection, and
the aggregates announced at the border must match the tenant's networks. It is also what turns a
*tenant* deploy into a *full* one. The commit carries the ticket in its trailers:

```text
feat: add network VLAN 2304 (VNI 30004) to tenant RED

Change: #72895
NetBox-Branch: chg-72895
```

In the same step the agent prepares the record. In a NetBox branch named after the ticket, five
objects are created and wait for the deploy: the VLAN, the L2VPN for the VNI, the termination
that joins the two, the prefix and the gateway address.

## 3. The dry run

Once the owner has released the pull request's run, the full deploy runs in check mode against
the live fabric. The result on the pull request and on the ticket:

| Host | Changes | Errors |
|---|---|---|
| leaf-101 | 9 | 0 |
| leaf-102 | 9 | 0 |
| leaf-103 | 9 | 0 |
| leaf-border-1 | 1 | 0 |
| leaf-border-2 | 1 | 0 |
| spine-11 | 0 | 0 |
| spine-12 | 0 | 0 |

*Batfish on the candidate: 10 checks, 0 failed.*

The commands behind the counts, the same on each leaf, as the dry-run note lists them, laid out
here as configuration:

```text
vlan 2304
  name VRF-RED-L2VNI-30004
  vn-segment 30004

interface Vlan2304
  vrf member RED
  ip address 192.168.7.1/24
  fabric forwarding mode anycast-gateway
  no shutdown
  no ip redirects
  no ipv6 redirects

interface nve1
  member vni 30004
    mcast-group 239.1.1.9

evpn
  vni 30004 l2
    rd auto
    route-target import auto
    route-target export auto

ip prefix-list PL-SVI-RED seq 30004 permit 192.168.7.0/24
ip prefix-list PL-IMPORT-SHARED-SERVICES seq 30004 permit 192.168.7.0/24 le 32
```

and on each border leaf, one line:

```text
router bgp 65001
  vrf RED
    address-family ipv4 unicast
      aggregate-address 192.168.7.0/24 summary-only
```

The agent does not only post the table. It reads the dry run against the ticket's own
verification list and says where they differ:

> Everything is the change itself; nothing on spines, no errors, no removal of existing config.
>
> 'no' lines present, all on new objects only: 'no shutdown', 'no ip redirects', 'no ipv6
> redirects' on the new Vlan2304, and 'no mcast-group' before 'mcast-group 239.1.1.9' under the
> new member vni 30004 (module rendering; nothing existing is removed). The verification asked
> for no 'no' lines, so this is listed for the owner's decision.
>
> The pull request's description names one prefix-list entry; the dry run adds two (PL-SVI-RED
> and PL-IMPORT-SHARED-SERVICES, seq 30004). The second leaks 192.168.7.0/24 into shared
> services, as the template does for the other RED networks.

That second point is the dry run earning its place: a route leak nobody had written down, shown
before it existed, and explained.

## 4. The approval and the merge

| UTC | |
|---|---|
| 12:53 | The ticket is opened |
| 12:54 | The pull request is opened; its run waits for the owner's release |
| 13:05 | The dry run is posted |
| 13:10:19 | The dry-run note is on the ticket |
| 13:10:51 | The owner approves the ticket |
| 13:46 | The pull request is merged; the merge run starts |

## 5. The deploy

The merge run, 1774 seconds, as reported on the ticket:

| Step | Result |
|---|---|
| Change gate | approved: the ticket, its dry run and Batfish checked |
| Canary batch 1: leaf-103 | 35 tests, 0 failed |
| Canary batch 2: spine-12 | 31 tests, 0 failed |
| Canary batch 3: spine-11, leaf-101, leaf-102, both border leafs | 59 tests, 0 failed |
| Full test run | 67 tests, 0 failed |
| Batfish on the live configurations | 10 checks, 0 failed |
| Save, state capture, documentation, reports | all green |
| NetBox branch | merged: the five objects are now in the record |

The snapshot taken before and after the deploy shows what changed in the *operational* state,
which is more than what was configured:

| Switch | Features whose state changed |
|---|---|
| leaf-101, leaf-102, leaf-103 | interface, pim, routing, vlan, vxlan |
| leaf-border-1, leaf-border-2 | bgp, routing |
| spine-11 | pim, vxlan |
| spine-12 | vxlan |

The spines received no configuration, and their state changed anyway: a new multicast group and
a new VNI exist in the fabric, and the spines carry both. The diff makes that visible, and the
report says why it is expected.

## 6. The verification

Against the list in the ticket, on the switches:

| Check | Result |
|---|---|
| `show nve vni 30004` on leaf-101, -102, -103 | Up, control plane, 239.1.1.9, L2 [2304] |
| `show running-config bgp \| section evpn` | `vni 30004 l2` with rd and route-target auto on all three |
| `show ip interface brief vrf RED` | Vlan2304 192.168.7.1, up/up/up on all three |
| `show vlan id 2304` on the vPC pair | active, on the peer-link port channel |
| Border leafs | `aggregate-address 192.168.7.0/24 summary-only` in RED; inactive until a host route exists, as expected |
| A dry run of leaf-101 | 0 changes, 0 errors: the switch matches the model |
| Fabric health | healthy, all seven |
| NetBox | branch merged, 5 creates |
| Alarms | none since the merge |

"Side effects: none. Rollback: not needed." The ticket was closed at 14:23.

## 7. The documentation

Nobody edits the fabric's wiki. After every successful deploy its pages are rendered from the
data model, so the network documents itself. The tenants page as generated today:

| VRF | VLAN | Name | L2VNI | Gateway | Multicast group | Access ports |
|---|---|---|---|---|---|---|
| RED | 2301 | VRF-RED-L2VNI-30001 | 30001 | 192.168.1.1/24 | 239.1.1.1 | leaf-102 Ethernet1/2 |
| RED | 2302 | VRF-RED-L2VNI-30002 | 30002 | 192.168.2.1/24 | 239.1.1.2 | leaf-103 Ethernet1/2 |
| RED | 2303 | VRF-RED-L2VNI-30003 | 30003 | 192.168.6.1/24 | 239.1.1.7 | - |
| BLUE | 2401 | VRF-BLUE-L2VNI-30101 | 30101 | 192.168.3.1/24 | 239.1.1.3 | leaf-101 Ethernet1/2 |
| BLUE | 2402 | VRF-BLUE-L2VNI-30102 | 30102 | 192.168.5.1/24 | 239.1.1.5 | leaf-103 Ethernet1/3 |
| GREEN | 2501 | VRF-GREEN-L2VNI-30201 | 30201 | 192.168.4.1/24 | 239.1.1.4 | leaf-103 Ethernet1/1 |
| SHARED-SERVICES | 2601 | VRF-SHARED-SERVICES-L2VNI-30301 | 30301 | 192.168.99.1/24 | 239.1.1.6 | - |

While this change's network existed, the same table had one more row, made from the six lines of
the diff:

| VRF | VLAN | Name | L2VNI | Gateway | Multicast group | Access ports |
|---|---|---|---|---|---|---|
| RED | 2304 | VRF-RED-L2VNI-30004 | 30004 | 192.168.7.1/24 | 239.1.1.9 | - |

The next day the reverse change removed the lines again, and the row went with them, through a
ticket, a dry run, the canary and the tests of its own.

## What the example shows

- **Seven lines of YAML** became the configuration above on five switches, five objects in
  NetBox, a row in the documentation and a set of tests, with nothing typed twice.
- **The dry run found something nobody had written down** (the second prefix-list entry and what
  it leaks), before the change existed.
- **The spines were shown to be untouched in configuration and changed in state**, which is
  exactly what should happen and is rarely this visible.
- **The owner's part** was reading one diff, one dry-run table and one report, and approving.

## Next

- [The pipeline](pipeline.md): every step of the run above, in general
- [The change workflow](change-workflow.md): the rules as they are today
- [Game days](game-days.md): what the canary and the rollback caught when a change was wrong
