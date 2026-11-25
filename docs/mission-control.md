# Mission Control

[← Overview](../README.md)

One page that answers "how is the fabric, and does anything wait for me?". It reads the systems
that already hold the answers and changes nothing: no button on it does anything to the fabric.

![Mission Control, the whole page](../assets/mission-control-overview-full.png)

> **About the pictures.** The screenshots on this page are from 19 November 2026, when the fabric
> was one site of seven switches. The page has since been adapted to the fabric of two sites: the
> Fabric panel and the NetBox comparison look different now, as described under
> [The topology](#the-topology) below. The other panels are as shown.

## What is on it

| Panel | Shows | Read from |
|---|---|---|
| **Verdict** | One line: operational, operational with warnings, or incident; and the three service levels | Alarm state |
| **Key figures** | What waits for the owner, switches healthy, alarms firing, open tickets, changes in flight, the pipeline | The panels below |
| **Fabric** | The topology by site, with each switch's and each link's state; per switch: BGP, OSPF and VTEP peers against the expected number, interfaces, CPU, memory, uptime, age of its telemetry; the tenants; switches that are joining by POAP | The data model on `main`, Prometheus |
| **Alerts** | What fires now, and the last five that resolved | Prometheus |
| **Changes** | Each open change: its step of nine, who it waits for, and in words what is awaited | The ticket system and the forge |
| **Fabric pipeline** | What runs now with its usual and its longest duration, the last deploy, the last scheduled test, the last 24 jobs | The forge |
| **Tickets** | Open and pending, in progress, the last closed | The ticket system |
| **Drift and records** | Is `main` deployed, does the fabric match its baseline, does NetBox agree with the fabric, when was the last rollback drill | The forge, NetBox |
| **Coming up** | The next operations tasks: rotations, updates, drills | The reminder calendar in Git |
| **The monitoring itself** | Is the collector up, how many telemetry streams and SNMP polls work | Prometheus |
| **Latest activity** | Alarms, pipeline jobs, changes and closed tickets in one timeline | All of the above |

Every source is polled on its own. A source that does not answer keeps its last good answer on the
page, marked with its age, and the footer says which source is behind. The page updates through a
stream, without reloading.

## The topology

Which switches there are, in which site, and how they are linked is not kept by hand in the page
and not asked of the monitoring. It is worked out from the fabric's data model, the inventory on
`main`, read through the forge's API every ten minutes.

| What the page draws | Worked out from |
|---|---|
| The switches and their roles | The inventory's groups: spines, leafs, border leafs, inter-site routers |
| The sites, each with its AS | The groups that declare themselves a site: DC-1, DC-2, the inter-site network |
| A link | Two routed interfaces of two switches in the same /31; an inter-site link when one end is an inter-site router |
| A vPC peer link | The peer-link members of a vPC pair |
| A border gateway | A spine with a link to an inter-site router |
| The tenants | The tenants file |
| Switches joining by POAP | The file of switches that are declared but not yet part of the fabric |

The Fabric panel draws one frame per site with its name and AS, the inter-site routers above
them, and the table of switches in one part per site. On 25 November 2026 that was 14 switches and
23 links, all of them answered for by the monitoring.

- **A switch that joins shows up by itself.** No new image and no edit of the page: the merge that
  puts a switch into the inventory is what puts it on the page.
- **The model says what should be there.** A switch of the model that the monitoring knows nothing
  of shows as down. A switch that only the monitoring knows is not on the page.
- **Half an answer is no answer.** If one file of the inventory cannot be read, the page keeps the
  last complete topology, and says under the table where its topology is from and why. Until
  `main` has answered for the first time, it starts from a copy that ships with the image.
- **One gap is shown as a gap.** The telemetry does not yet say which switch owns a border
  gateway's VTEP address, so the spines show no figure under *VTEP peers* and list the other
  gateway's VTEP as unknown.

## A change in flight

![Changes in flight and the running pipeline](../assets/mission-control-changes-demo.png)

*Recorded demo data (the panels are marked "demo"): three changes at different steps, and a
deploy in progress.*

Where a change stands is not read from the ticket's state alone. The page also asks the forge
about the change's pull request: is it open, merged or closed; which commit is its head; is the
run of that commit held or released; is there a dry run of it; is there a merge run, and how did
it end. So it can say things the ticket cannot:

- "Release the run of `834426fd`; do not approve the ticket yet"
- "The approval does not cover `834426fd`: the ticket goes back to assess"
- "Approved after the dry run of `834426fd`: merge the pull request"
- "The deploy failed, the checkpoint rolled back"

Only the dry run of the *head* commit counts, and the order of dry run and approval is read from
the ticket's history. The page applies the same rule as the deploy gate, so it shows a stale
approval before the gate would refuse it. The rules are on the page about
[the change workflow](change-workflow.md).

## Detail pages

### A change

![One change](../assets/mission-control-change.png)

Open or closed: the nine steps, the pull request and its checks, the ticket's description with
rollback and verification, and everything that happened in order: state changes, the follow-up's
steps, every note, and who did each.

### A switch

![One switch](../assets/mission-control-switch.png)

State and uptime, CPU and memory over 24 hours, every alarm of the last seven days on a time
line, every enabled interface with speed, traffic, utilisation and errors, the BGP neighbours,
OSPF adjacencies and VTEP peers one by one, the links to other switches, and the newest tickets
that name the switch. The picture is leaf-101 before the fourth leaf joined; it has since gained
leaf-104 as a VTEP peer.

### NetBox against the fabric

![NetBox compared with the fabric](../assets/mission-control-netbox.png)

NetBox is the record of the fabric, and a record is only worth something if it is right. The page
compares it three ways:

| NetBox against | What is compared |
|---|---|
| The inventory's sites and links | Every switch is a device and its site holds no other; every site of the fabric has its devices; every link has its cable, and every cable is a link |
| What the switches report | Interfaces with admin state and MTU, the software version, the serial number |
| The data model | VRFs, VLANs, VNIs, the addresses on interfaces and gateways |

It also lists NetBox branches of changes that were deployed but never merged.

The comparison covers every site of the fabric. A site of which NetBox records nothing at all is
one difference for the whole site, not one per switch, link and address. On 25 November 2026
there was one such difference: the inter-site network, whose two routers and five links were not
recorded in NetBox yet. The picture above shows the first site alone, in agreement.

## How it is built

| | |
|---|---|
| Backend | Python, standard library only: no framework, no dependency to patch |
| Front end | Plain HTML, CSS and JavaScript modules: no build step |
| Data | One snapshot of everything, a stream of it, and detail endpoints for a switch, an alarm, a change |
| Sources | A source per system, each with its own interval and a read-only account of its own; the topology from the data model on `main` |
| Sign-in | OpenID Connect against the lab's identity provider, with an allow-list |
| Deployment | A scanned and signed image on the cluster, behind the gateway |
| Tests | 136 backend tests without the network, against recorded answers of the real systems |
| Demo mode | Every source can be replaced by a recording, so the page can be shown without the lab |

Two rules shaped it. **Read-only**: every account it uses can read and nothing else, and every
name that goes into a question to a system is checked first. **Say what you do not know**: a
figure that could not be measured is shown as not measured, never as zero or as fine.

## Next

- [The change workflow](change-workflow.md): the rules behind the nine steps
- [Observability](observability.md): where the fabric's data comes from
- [Multi-Site](multisite.md): the two sites the Fabric panel draws
