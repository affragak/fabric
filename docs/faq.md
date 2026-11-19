# Questions and answers

[← Overview](../README.md)

## About the project

**Is this a production network?**
No. It is a lab: seven virtual Nexus switches, four tenants, one person. Everything around the
switches is built as it would be for a real fabric, which is the point of it. What a lab cannot
show is listed under [limits](decisions.md#limits).

**Can I see the source?**
Not here. This repository is the overview; the data model, the roles, the pipelines and the
manifests are kept in their own repository. The pages quote the real artefacts where they help:
a diff, a dry-run table, a ticket, a report.

**What does it take to build something like it?**
A server that can run the virtual switches (here eight cores, bare metal), a small machine for
the cluster (here one node with 14 GB of memory), something small that stays up when the cluster
does not (here a Raspberry Pi 5), and storage for backups and secrets. In licences: the lab
software's professional edition and the vendor's virtual switch image. Everything else is open
source. The order to build it in is on the [setup](setup.md) page.

**Would it work on real hardware?**
The data model, the pipeline and the workflow do not care. Three things would differ: BFD would
run (the lab uses fast OSPF timers instead), traffic would spread over both spines (the virtual
switch uses one path), and counters would mean something. The [game day](game-days.md) numbers
would be better.

## About changes

**Why does a ticket gate a deploy? Is a reviewed pull request not enough?**
A merge says the code was accepted. It does not say that anyone looked at what the change does to
the running fabric. The ticket holds that: the risk, the rollback, how it will be verified, the
dry run's result, and an approval given *after* that result by the one account that may give it.
The pipeline checks all of it before it touches a switch, so "we always do a dry run first" is
not a habit but a condition.

**What if the dry run is wrong?**
It sometimes is. Check mode does not send commands, so it cannot know that a switch will refuse
one. That is why a deploy takes a checkpoint first, starts with one leaf, tests after every
batch, and rolls back by itself. It has been needed: a change with an out-of-range timer value
was stopped at the first leaf and rolled back, with the other six switches never touched.

**How long does a change take?**
The deploy itself: roughly twenty minutes for a tenant change, about half an hour for a full
deploy with canary batches, most of it testing. From request to closed ticket, the two changes
documented here took 78 and 90 minutes, of which the owner's part was reading two or three short
things and a few clicks.

**Is that not slow for one line of configuration?**
For one line, yes. The time buys three things: the line is proven before and after, the fabric
is known to match its model afterwards, and there is a record of who decided what and why. The
second change of the day costs the owner the same few minutes, whoever or whatever prepares it.

**What about an emergency?**
The same states and the same gate, in compressed order: the owner approves as soon as the dry
run is in. The bypasses that exist are the owner's alone: a manual pipeline run, or switching the
gate off, which is logged.

**What happens if someone changes a switch by hand?**
The next drift check finds it, within hours: a dry run of the model against the live switches
shows the pending commands. The next full deploy removes it, because the model is applied in
*overridden* state.

## About the agents

**Why can the agents not merge or approve?**
Because an agent that can, eventually will, and usually with good intentions. Their accounts
simply lack the right: the forge refuses the merge and the ticket system refuses the state.
Rules that were only instructions were the ones that got broken; rules in accounts and tokens
have held.

**Do the agents log in to the switches?**
No. They ask a read-only server that runs under a different user and holds the only read-only
key. It allows a fixed list of show commands, redacts secrets and logs every call. An agent's
own account has no key and no network route to a switch.

**What if a pull request or a ticket contains instructions for the agent?**
The agent treats it as data: it reports what it found and where, and carries on with what the
owner asked. Instructions come from the owner in chat; an approval is a button or a ticket state,
never a sentence. This is drilled every quarter, and the first drill found a real gap.

**Who reviews the agents' work?**
Another agent reviews every pull request, as advice. It is the same kind of model as the author,
so its mistakes can agree with the author's. The checks that do not share them are the pipeline
and the owner.

**What do the agents actually save?**
The preparation and the waiting: pre-checks, the ticket text, the NetBox objects, the pull
request, starting the dry run when the run is released, reading its output against the ticket,
reporting the deploy, verifying on the switches, and triaging every alarm within minutes. The
decisions stay with the owner.

## About the design

**Why is NetBox not the source of truth?**
Because the model in Git is what is reviewed, versioned and deployed. NetBox is the record, kept
in step by the pipeline (one branch per change, merged after the tests pass) and compared with
both the model and the switches. A difference is reported with all three values.

**Why tickets for alarms too?**
An alarm that is only a message is gone when it scrolls away. As a ticket it has a beginning, an
end, evidence, a runbook and a triage note, and it can be counted: how often, how long, how fast
resolved.

**Why one Kubernetes node?**
Because reproducibility was the goal, not availability. What must survive a rebuild of the
cluster runs elsewhere.

**Why is the status page read-only?**
So that it can be trusted with every account it uses and shown to anyone. Acting on the fabric
has exactly one path, and the page is not on it.

## Next

- [Decisions and limits](decisions.md)
- [A worked example](example-add-a-network.md)
