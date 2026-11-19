# Setup

[← Overview](../README.md)

How the environment is put together. The source is private, so this is not a tutorial to copy and
paste; it is the build order and the reasons, enough for someone who wants to build the same kind
of thing. Each layer needs the ones below it: the diagram reads from the bottom up, and the sections
follow in the same order.

<p align="center">
  <img src="../assets/diagram-build-order.svg" width="760" alt="Eleven layers, each needing the ones below it: 1 The lab; 2 Day zero; 3 Toolchain and data model; 4 First deploy; 5 The platform; 6 The pipeline; 7 Telemetry and alarms; 8 Tickets and the record; 9 Change control; 10 The agents; 11 Mission Control.">
</p>

## 1. The lab

| | |
|---|---|
| Hypervisor | EVE-NG Professional on a bare-metal server (8 cores); the switches need nested virtualisation and several gigabytes each |
| Nodes | Seven Nexus 9000v as KVM nodes; an external router, an out-of-band management switch and six hosts as lightweight IOS nodes: sixteen in all |
| Cabling | Every leaf to both spines; the vPC pair's peer link; the border leafs to the external router; every `mgmt0` to the management switch and from there to the lab network |
| Access | The web interface over HTTPS with a real certificate; consoles firewalled to named hosts |
| Care | Backed up before changes; patched by hand once a month with the lab stopped |

![The lab in EVE-NG](../assets/eve-ng-topology.png)

## 2. Day zero on each switch

The one thing done by hand, because automation needs something to log in to. A short snippet per
switch:

- the management VRF and the `mgmt0` address;
- an automation account with an SSH public key;
- NTP, so that logs and certificates make sense.

From here on the switches are changed only through the data model. Two playbooks are run once by
hand before the first full deploy, because they must exist before the fabric can form: the vPC
peer link and the vPC access ports.

## 3. The toolchain and the data model

| Step | Result |
|---|---|
| Build the toolchain image | Ansible, pyATS, the Batfish client and the linters in one image, every dependency locked by hash; a wrapper runs each command in a fresh rootless container |
| Pin the switches' host keys | A `known_hosts` file in the repository; validation fails without a key for every switch |
| Write the inventory | The switches and their groups; one file per switch for links, loopbacks and ports |
| Write the tenants | VRFs, networks, VNIs, gateways, route leaking; checked by a JSON schema |
| Validate | Lint, schema and consistency rules, with no device access |

## 4. First deploy, tests, baseline

1. **Dry run** the whole fabric and read every command.
2. **Deploy** the full playbook: common settings and the underlay, SNMP and telemetry, login
   security, the border, then BGP and the overlay.
3. **Test** with pyATS: the control plane against expectations rendered from the model, then the
   data plane from the lab hosts.
4. **Store the baseline**: the learned operational state that later drift checks compare with.

At this point the fabric works and can be changed from a laptop. Everything after this exists so
that it does not have to be.

## 5. The platform

| Step | Result |
|---|---|
| Install the cluster | One Talos node, managed from outside the cluster |
| Bootstrap Flux | The cluster applies the `platform` directory of `main`: controllers first, then their configuration, then the forge, then the applications |
| Secrets | A vault off the cluster; the cluster reads it with a read-only token |
| The forge and its registry | Repositories, packages, container images |
| Runners | Three, separated by trust: pull requests, deploys, image builds |
| Backups | Database archiving and volume backups to another machine, and a first restore drill |

## 6. The pipeline

1. `validate` as the required check on `main`, on the runner without secrets.
2. The dry run and Batfish on pull requests, on the deploy runner.
3. The merge pipeline: deploy decision, snapshot, checkpoint, canary, tests, save or rollback.
4. Branch protection: no direct pushes, only the owner merges, forks held until released.
5. The scheduled workflows: tests and drift twice a day, the drills, the reminders.

## 7. Telemetry and alarms

1. Enable gNMI on the switches, each with a certificate from a small private CA.
2. Deploy the collector, the SNMP exporter, Prometheus and Grafana.
3. Send syslog and traps to Logstash and on to Elasticsearch.
4. Write the alarm rules, one runbook per subject.
5. Add the dead-man's switch for the monitoring itself.

## 8. Tickets and the record

| Step | Result |
|---|---|
| Zammad | A group for alarms and a group for changes; the change ticket's fields (kind, risk, deploy action, devices, pull request, rollback, verification) |
| The alarm bridge | Alarms open, annotate and close their tickets |
| NetBox | Devices, interfaces and cables; then tenants, VLANs, VNIs and addresses loaded from the data model through the first change ticket |
| NetBox branching | A change's NetBox edits live in a branch named after its ticket, merged by the pipeline after the tests pass |

## 9. Change control

1. The ticket states and the workflow rules that restrict each role, tested through the API.
2. Separate accounts and tokens: the gate (read-only), the pipeline (one state), the follow-up
   (one state back).
3. The change gate as a step of the merge pipeline, failing closed.
4. A first real change through the whole flow, with a deliberate deviation, before trusting it.

## 10. The agents

1. A host of their own, one operating-system user per agent, one forge and one ticket account
   each, with the narrowest role that works.
2. The read-only MCP server under a separate user, the only one with a route to the switches.
3. The chat bridge with approval buttons, and the approval hook.
4. The skills: the procedures as versioned text.
5. The timers: alarm triage, change follow-up, the daily digest.
6. An injection drill before giving them real work.

## 11. Mission Control

Built last, because it only reads what the other layers already record: one source per system,
each with a read-only account, tested against recorded answers, deployed like any other signed
image behind the identity provider.

## What it takes

| | |
|---|---|
| Hardware | One server for the lab, one small machine for the cluster, a Raspberry Pi, a storage box |
| Licences | The hypervisor's professional edition and the vendor's virtual switch image; everything else is open source |
| Effort | The fabric and its first deploy are the small part. The pipeline, the gate and the discipline around them are the real work |

## Next

- [Architecture](architecture.md): the same components, as they run
- [Operations](operations.md): what keeps running after the build
