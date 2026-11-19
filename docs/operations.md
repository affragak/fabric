# Operations

[← Overview](../README.md)

What happens when nobody is changing anything: the checks that run by themselves, the drills
that are practised on purpose, and the calendar that makes sure neither is forgotten.

## Every day

| When | What | Result |
|---|---|---|
| Every 5 minutes | An agent's scheduled run: triage of new alarm tickets, follow-up of open changes | A note on a ticket, or silence |
| Twice a day | The fabric's full test run, configuration drift, operational drift, Batfish, a dependency audit | Quiet when all is well; a failed run and a message when not |
| Once a day | A read-only digest of the fabric: health, alarms, changes, pipeline, what is due | One message |
| Always | Alarm rules on telemetry and events | Tickets that open and close themselves |

### Two kinds of drift

- **Configuration drift**: a dry run of `main` against the live switches. Any pending command
  means a switch no longer matches the model: someone changed it by hand, or a deploy was
  incomplete.
- **Operational drift**: the learned state (BGP, OSPF, PIM, VRFs, VLANs, VXLAN, interfaces,
  routing) compared with the baseline stored after the last good deploy, counters left out. A
  neighbour that is gone or a route that appeared shows here although no configuration changed.

Mission Control shows both, next to the comparison with NetBox.

## Drills

Things that must work on the worst day are practised on ordinary ones.

| Drill | How often | What it proves |
|---|---|---|
| **Rollback** | Monthly | On one leaf: checkpoint, a harmless change, rollback, the change is gone, the tests pass, a dry run shows no difference |
| **Game day** | Monthly | One deliberate fault, with the convergence measured |
| **Restore** | Quarterly | One database and one volume come back from backup into a scratch namespace |
| **Prompt injection** | Quarterly | An instruction hidden in content an agent reads is reported, not obeyed; the allow-lists are reviewed |

### Game days

A game day turns "the fabric is redundant" into a number. One thing is broken on purpose, the
time until traffic and the control plane recover is measured, the fault is undone, and the fabric
is checked to be as it was.

| Scenario | The fault |
|---|---|
| Uplink | One leaf loses one of its two uplinks |
| Spine | One spine is taken out |
| Border exit | One border leaf loses its way out |
| Noise | No fault at all: the measurement's own baseline |

The first game days changed the design. A link shut down at one end converged in about a second,
but a *silent* failure, which neither end sees, took the full 40 seconds of OSPF's default dead
timer, and return traffic from outside kept flowing into a dead link for about two minutes
because of the external BGP session's default hold time. Both timers are now in the data model,
set to seconds.

## The calendar

Operations tasks live in a file in Git: a name, a due date, how often, and how to do it. Once a
week a workflow opens an issue for every task due within two weeks. Mission Control lists the
next ones.

| Every month | Every quarter | Every 6 or 12 months |
|---|---|---|
| Dependency and action updates | Restore drill | Rotation of each credential and key |
| The agents' runtime update | Prompt-injection drill and allow-list review | |
| Patching the lab host, with the lab stopped | Certificate renewal check | |
| Rollback drill | | |
| Game day | | |
| Review of time-boxed scanner exceptions | | |

## Runbooks

Every alarm ticket links to a runbook: what the alarm means, what to check first, what the usual
causes are, when to escalate. Ten of them cover the eighteen alarm rules, grouped by subject:
unreachable switch, link down, OSPF adjacency, BGP session, VTEP peer, interface health, switch
health, lost telemetry, syslog and trap events.

## Documentation that cannot go stale

- **The wiki's fabric pages** (tenants, fabric, border) are rendered from the data model after
  every successful deploy. They describe what was deployed, because they are made from it.
- **The docs site** is built from notes, decision records and runbooks, plus each component's own
  documents, on every merge.
- **Decision records** keep the reasons: why one monorepo, how the CI decides which forks to
  trust, how the supply chain is pinned, why the docs are code.
- **NetBox** is checked against the fabric continuously, so the record is either right or known
  to be wrong.

## Next

- [Setup](setup.md): building all of this from nothing
