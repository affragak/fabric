# Decisions and limits

[← Overview](../README.md)

Why the project is shaped the way it is, and what it deliberately does not do. The first part
summarises the four decision records kept with the source; the second lists decisions that show
in the design; the third is the honest list of limits.

## The decision records

### One monorepo

**Context.** The fabric automation, the cluster's GitOps, the agent team, the telemetry image and
the documentation lived in five repositories that referred to each other in about sixty places. A
change that crossed them needed several coordinated pull requests.

**Decision.** One repository, one folder per component, with each repository's history rewritten
into its folder and merged once, so `git log` and `git blame` reach back unbroken. Before the
import the full history of each was scanned for secrets. Only repositories and folders were
renamed: images, hosts, users and services kept their names, because renaming those changes
running systems for no gain in the code. The components were cut over one at a time, lowest risk
first and the fabric last, with the old repositories archived so each step could be undone.

**Consequences.** One pull request can change a role, its documentation and the alarm that
watches it. Every component's required check now sees every pull request, and skips its work when
its folder is untouched. Per-agent edit limits and code owners take the place of separate
repositories.

### Whom the CI trusts

**Context.** The pipelines run on runners inside the lab. A pull request's workflow runs the pull
request's own code, and on a runner next to the switches that is remote code execution, whatever
the job has or has not in its environment.

**Decision.**

1. Every job that a pull request can start checks the pull request's origin first: only the
   repository's own branches and a short list of trusted forks reach a runner.
2. Anyone else's pull request runs nothing; a maintainer reviews it and pushes the reviewed
   commits to a branch.
3. Even a trusted fork's code never runs with the switch key until the owner has released that
   exact commit.
4. A job that needs elevated context runs only the base branch's code and reads the pull request
   through the API.
5. Secrets reach single steps, never a pull request's jobs.

**Consequences.** Nothing that comes from the switches goes into logs, artefacts or pull-request
comments. The dry-run comment carries counts; the commands, test results, state diffs and
configurations go to a private package whose privacy is checked before every upload.

### Pinned actions, verified tools, signed images

**Context.** The CI downloads code on every run: actions, scanners, signers, base images. A moved
tag or a tampered download would run with the job's secrets.

**Decision.** Actions pinned by commit. The CI's own tools installed by one script that holds
each one's checksum. Base images by digest. Images scanned before they are pushed, signed, and
refused by the cluster when unsigned.

**Consequences.** An update is a deliberate pull request that changes a pin. The runner images are
part of the trust boundary and are built and signed the same way.

### Docs next to their code

**Context.** Documentation drifts from code when it lives apart from it, and a review of a change
should see its documentation in the same folder.

**Decision.** Each component keeps its documents. One site is built from the knowledge-base notes
plus what the build collects from every component, with links rewritten. The fabric's generated
pages stay in a wiki that only the pipeline writes.

**Consequences.** One search covers notes, designs and runbooks. A documentation change rebuilds
the site and deploys nothing else.

## Decisions that show in the design

| Decision | Instead of | Because |
|---|---|---|
| The data model is the truth, applied in *overridden* state | Merging changes into whatever a switch has | A switch then cannot hold configuration nobody can explain; the price is that a deleted line deletes configuration, which is what the dry run is for |
| A ticket gates every deploy | Trusting the merge | A merge says the code was accepted. An approved ticket says someone looked at what the change does to the fabric, after the dry run showed it |
| The approval must follow the dry run, checked from the ticket's history | A rule in a procedure | It was once approved first. The order is now something the gate verifies |
| Agents work from forks and cannot merge | Agents with write access | An agent with a token that can merge will one day use it |
| Limits in accounts and tokens before limits in prompts | Instructions | The rules that were only instructions were the ones that got broken |
| NetBox is a record, compared with the fabric | NetBox as the source of truth | The model in Git is reviewed, versioned and deployed; a second source would have to be kept in step by hand |
| One read-only status page | Buttons that act | The page can then be trusted with every account it uses, and shown to anyone |
| What must survive the cluster runs off it | Everything on Kubernetes | Identity, secrets, backups, the cluster's management and the agents are needed exactly when the cluster is down |
| Fast OSPF timers in the lab | Waiting for BFD | The virtual switch accepts BFD and never runs it; measured, the timers cut a silent failure from 39 seconds to 6 |

## Limits

What this lab is not, and what it does not claim.

### It is a lab

- **Virtual switches.** The data plane is software. It programs one path where hardware would use
  two, so a fault hits all flows or none. Some counters are synthetic: an interface-error alarm on
  this lab was once triaged to a counter that jumped by 2³¹ in one step.
- **No BFD.** It is in the model and never comes up. Convergence numbers here are evidence of a
  method, not of what hardware would do.
- **Small.** Seven switches, four tenants. Nothing here was tested for scale.
- **A noisy host.** The whole lab stalls for up to four seconds every few minutes, and every
  measurement has to live above that.

### It has one of several things

- **One cluster node.** Reproducible, not highly available. Losing the node stops the forge, the
  runners and the monitoring until it is rebuilt.
- **One person.** Every approval, release and merge is one owner's. Separation of duties is
  between a person and the systems, not between people.
- **One kind of reviewer.** The agents that review pull requests are the same kind of model as
  the agents that write them, so their mistakes can agree. The pipeline and the owner are the
  checks that do not share them.

### Things it deliberately does not do

- **The dry run is not a guarantee.** Check mode does not send commands, so it cannot learn that a
  switch will refuse one. The canary, the checkpoint and the tests exist for exactly that, and
  have been needed.
- **The external router is configured by hand.** It stands for someone else's network and is
  outside the model.
- **The platform's gate is lighter than the fabric's.** It checks that a change has an approved
  ticket, not the order of approval and dry run.
- **Agents do not act on the fabric.** Not in an emergency either: the fast path is a revert
  through the same pipeline, or an action an agent names for a person.
- **No change without a pull request.** There is no "quick fix on the switch". A change made by
  hand shows up as drift within hours and is removed by the next full deploy.

## Next

- [Security](security.md): the measures these decisions led to
- [Game days](game-days.md): the measurements behind the timers
- [FAQ](faq.md)
