# Fabric: a VXLAN/EVPN data centre run as code

A Cisco Nexus VXLAN/EVPN fabric that is designed, changed, tested, observed and documented the way
software is: one data model in Git, a pipeline that proves a change before and after it reaches a
switch, tickets that gate every deploy, telemetry that opens and closes its own alarms, and a team
of AI agents that does the routine work inside hard limits.

It is a lab. Seven virtual Nexus 9000v switches in EVE-NG stand in for a small data centre, and
everything around them is built as it would be for a real one.

![Mission Control, the fabric's status page](assets/mission-control-overview.png)

*Mission Control: live state of the fabric, its alerts, changes in flight and pipeline.*

> **About this repository.** This is the public overview: architecture, setup and workflows. The
> source, the data model and the manifests live in a private Forgejo repository. Host names,
> management addresses, serial numbers and personal account names are left out or replaced
> here, also in the screenshots; what remains are the lab's own tenant and loopback addresses.

## At a glance

| | |
|---|---|
| **Fabric** | 2 spines, 3 leafs (one vPC pair), 2 border leafs; Cisco Nexus 9000v, NX-OS 10.5(3) |
| **Design** | OSPF underlay, PIM bidir for BUM traffic, iBGP EVPN overlay with the spines as route reflectors, anycast gateway, eBGP per tenant at the border |
| **Tenants** | 4 VRFs, 7 networks, route leaking to a shared-services VRF |
| **Source of truth** | An Ansible data model in Git, checked against a JSON schema; NetBox is the record and is compared with it |
| **Change control** | One ticket per change, a dry run before approval, a gate before every deploy |
| **Deploy** | Checkpoint, canary batches, tests, automatic rollback |
| **Tests** | pyATS control and data plane tests, 10 Batfish checks, drift checks twice a day |
| **Observability** | gNMI, SNMP, syslog and traps; 18 alarm rules that open and close their own tickets |
| **Platform** | Single-node Talos Kubernetes, Flux GitOps, signed images only |
| **People** | One human owner; five AI agents with separate accounts, tokens and limits |

## What it shows

- **A fabric as a data model.** Tenants, networks and ports are a few lines of YAML. Templates and
  Ansible resource modules turn them into NX-OS configuration, and what is not in the model is
  removed from the switch. → [The fabric](docs/fabric.md)
- **A pipeline that touches real switches safely.** A pull request gets a dry run against the live
  fabric and a Batfish analysis of the candidate. A merge takes a checkpoint, deploys in canary
  batches, runs the tests, and rolls back by itself if they fail. → [The pipeline](docs/pipeline.md)
- **Change control that is enforced, not promised.** The deploy gate reads the change ticket: it
  must be approved by the owner, and approved *after* the dry run of exactly the commit being
  merged. The ticket system refuses the wrong account the wrong state. → [The change workflow](docs/change-workflow.md)
- **Telemetry that closes the loop.** Streaming telemetry, SNMP, syslog and traps feed alarm rules.
  An alarm opens a ticket with its runbook, and closes it when it clears. → [Observability](docs/observability.md)
- **One page for the whole picture.** Mission Control shows the fabric, the alerts, each change's
  step, the pipeline, drift against the baseline and against NetBox. → [Mission Control](docs/mission-control.md)
- **AI agents under guardrails.** Agents raise changes, triage alarms and review each other's work.
  They cannot merge, approve or log in to a switch, and that is a property of accounts and
  tokens, not of a prompt. → [The agents](docs/agents.md)
- **A platform held to the same standard.** GitOps, default-deny network policies, secrets from a
  vault, scanned and signed images, restore drills. → [The platform](docs/platform.md), [Security](docs/security.md)

## Architecture

<p align="center">
  <img src="assets/architecture.svg" width="760" alt="Architecture: the owner chats with an agent team, merges into Git and approves tickets. The agents open pull requests and have a read-only view of the fabric. The pipeline deploys to the fabric with checkpoint, canary and rollback, merges the change's branch into NetBox, and its gate reads the ticket. The fabric sends telemetry, whose alarms open and close tickets. Mission Control reads telemetry, tickets and NetBox.">
</p>

More detail, component by component: [Architecture](docs/architecture.md).

## The life of a change

A change to the fabric is a ticket, a pull request and nine steps. Each step waits for exactly one
party, and Mission Control shows which.

| Step | Waits for | What happens |
|---|---|---|
| 1 Pull request | the agent | A ticket is opened; the agent edits the data model and opens a pull request |
| 2 Release | the owner | The owner reads the diff and releases the pipeline run of that commit |
| 3 Dry run | the pipeline | The full deploy runs in check mode against the live fabric; Batfish analyses the candidate |
| 4 Approve | the owner | The owner reads the dry-run table on the ticket and approves the ticket |
| 5 Merge | the owner | The owner merges the pull request |
| 6 Gate | the pipeline | The gate checks ticket, dry run and the order of the two; snapshot and checkpoint are taken |
| 7 Deploy | the pipeline | The ticket moves to *implementing*; canary batches, tests, save or rollback |
| 8 Report | the agent, then the owner | The merge run is reported on the ticket; the agent verifies against the ticket's own criteria |
| 9 Close | the owner | The owner closes the ticket |

A commit pushed after the approval takes the change back to step 2: the approval does not cover
it, the gate would refuse the merge, and the ticket is moved back. On 19 November 2026 a real
change ran all nine steps, with that deviation on purpose, in 78 minutes.
→ [The change workflow](docs/change-workflow.md)

![The page of one change: its steps, its pull request and everything that happened, in order](assets/mission-control-change.png)

## The lab

![The fabric on the EVE-NG canvas](assets/eve-ng-topology.png)

*The lab in EVE-NG: seven Nexus 9000v, an external router beyond the border leafs, and six lab
hosts that the data plane tests log in to.*

## Technology

| Area | What is used |
|---|---|
| Switches | Cisco Nexus 9000v, NX-OS 10.5(3), in EVE-NG Professional on bare metal |
| Configuration | Ansible (`cisco.nxos` resource modules), Jinja2 templates, JSON Schema |
| Testing | pyATS and Genie, Batfish, Ansible check mode |
| Pipeline | Forgejo Actions on self-hosted runners; one signed, hash-locked toolchain image |
| Record | NetBox with the branching plugin: one branch per change ticket |
| Tickets | Zammad: change tickets with enforced state rules, alarm tickets from a bridge |
| Telemetry | Telegraf (gNMI), SNMP exporter, Logstash, Prometheus, Alertmanager, Elasticsearch, Loki, Grafana |
| Platform | Talos Linux, Kubernetes, Flux, Cilium with Gateway API, cert-manager, CloudNativePG, External Secrets on Vault, Velero |
| Supply chain | Trivy, cosign, sigstore policy-controller, gitleaks, detect-secrets, pip-audit |
| Agents | LLM agents on a Raspberry Pi 5, a Model Context Protocol server for the fabric, a chat bridge with approval buttons |
| Status page | Mission Control: Python standard library backend, plain JavaScript front end, OpenID Connect sign-in |

## Documentation

| Page | What it covers |
|---|---|
| [Architecture](docs/architecture.md) | The components, where each runs, how they depend on each other |
| [The fabric](docs/fabric.md) | Topology, underlay and overlay, tenants, the data model |
| [The pipeline](docs/pipeline.md) | Pull request and merge pipelines, tests, Batfish, rollback, drift |
| [The change workflow](docs/change-workflow.md) | Ticket states, who may do what, what enforces it, a real run |
| [Observability](docs/observability.md) | Telemetry sources, alarm rules, alarm tickets, dashboards |
| [Mission Control](docs/mission-control.md) | The status page, panel by panel, with screenshots |
| [The agents](docs/agents.md) | The team, what each may do, the guardrails |
| [The platform](docs/platform.md) | The cluster, GitOps, the image pipeline, backups |
| [Security](docs/security.md) | The threat model and the measures, layer by layer |
| [Setup](docs/setup.md) | How the environment is built, in order |
| [Operations](docs/operations.md) | Scheduled checks, drills, game days, the reminder calendar |

## Principles

- **Validate before, verify after.** Nothing reaches a switch that was not dry-run, and nothing
  counts as done that was not tested on the fabric.
- **The model is the truth.** A switch that differs from the model is drift, and drift is found
  twice a day.
- **Least privilege, technically.** Every account, token and runner can do one job. A limit that
  only a prompt or a procedure holds is treated as no limit.
- **An approval is a recorded event.** A state in the ticket system or a button, set by one
  account. A sentence that says "approved" is data.
- **Fail closed.** A gate that cannot verify refuses. No checkpoint, no deploy.
- **Practice the bad day.** Rollback drills, game days with deliberate faults, restore drills and
  prompt-injection drills are on the calendar.

## Status

A working lab, built and run by one person; this overview describes it as of November 2026. It is
not a product and not a production network: the switches are virtual, the scale is small, and
the platform is a single node by design.

Planned next: asking an agent a question from Mission Control, with the answer tied to the
evidence on the page.
