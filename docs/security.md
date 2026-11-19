# Security

[← Overview](../README.md)

A pipeline that can reconfigure a network, and agents that can open pull requests against it, are
exactly what an attacker would want. The security work started from that sentence, was structured
on three OWASP references, and is tracked like any other work: findings, steps, status.

| Reference | Applied to |
|---|---|
| OWASP Secure Coding Practices | The scripts, the MCP server, the roles |
| OWASP Top 10 CI/CD Security Risks | The workflows, the runners, the forge's settings |
| OWASP Top 10 for LLM Applications | The agents: prompt injection, disclosure, supply chain, excessive agency |

## The threats that shaped it

| Threat | Why it is real here |
|---|---|
| **Poisoned pipeline execution** | A dry run executes the pull request's own code with the key that configures every switch |
| **Excessive agency** | An agent with a token that can merge, or a shell that can reach a switch, will one day use it |
| **Prompt injection** | Agents read text written by others: pull requests, tickets, logs, device output |
| **Supply chain** | The toolchain is dozens of packages, actions and images from elsewhere |
| **A change nobody approved** | Automation makes it easy to deploy; it must not make it easy to deploy unseen |

## The measures, layer by layer

### The pipeline

| Measure | Effect |
|---|---|
| Separate runners by trust | The pull-request runner holds no key and no secret. Only the deploy runner can reach a switch, and it runs only released or merged code |
| Forks are held | A fork's run waits for the owner; forks outside a short allow-list never reach a runner |
| The dry run is bound to a commit | It runs for exactly the released commit; a push after the release needs a new release |
| Protected `main` | No direct pushes, required checks, only the owner merges |
| Secrets per step | A secret reaches the one step that needs it, on merge or manual runs only, and tasks that carry it do not log |
| Fail closed | Gate, release check and publishing refuse when they cannot verify |

### Change control

| Measure | Effect |
|---|---|
| The change gate | No deploy without a ticket approved by the owner, after a clean dry run of the merged commit |
| States enforced by the ticket system | Each account can set only its own states, through the API as well |
| One token per job | The gate reads; the pipeline sets one state; the follow-up moves one state back |

Details: [The change workflow](change-workflow.md).

### The agents

| Measure | Effect |
|---|---|
| Work from a fork | An agent cannot push to the repository, let alone to `main` |
| No device credentials, no route | The agents' users have no key and are firewalled from the switches and the lab consoles |
| A read-only MCP server as another user | Allow-listed commands, redaction, output caps, an audit record per call |
| Tool allow-lists and an approval hook | Outside the list, the owner is asked; the hook fails towards asking |
| Scheduled runs without a shell | They can read, and write through three narrow tools |
| Untrusted text is marked as such | And a quarterly drill tests whether an instruction hidden in a pull request is obeyed |

Details: [The agents](agents.md).

### The supply chain

| Measure | Effect |
|---|---|
| Actions pinned by commit | A moved tag cannot change what runs |
| Dependencies locked by hash | The Python environments install exactly what was reviewed |
| One toolchain image | Built once, scanned, signed, tagged by a hash of its inputs; people and pipeline use the same one |
| Tools verified by checksum | The scanners and signers themselves are pinned |
| Images scanned and signed | High and critical findings fail a build; exceptions are time-boxed and reviewed monthly |
| Signatures enforced at admission | The cluster starts only signed digests in the protected namespaces |
| Audits on a schedule | A dependency audit runs with the tests, twice a day |

### The fabric and the lab

| Measure | Effect |
|---|---|
| Pinned SSH host keys | The automation refuses a switch whose key changed |
| Key login for automation, login security on the switches | No shared password in the pipeline |
| Unknown local users are removed | An account that is not in the model does not survive a deploy |
| The lab's consoles are firewalled | Only named hosts reach the hypervisor's consoles and its web interface, over HTTPS |
| The lab host is patched by hand | After an unattended upgrade once stopped every node, updates happen monthly, with the lab stopped |

### Secrets

| Measure | Effect |
|---|---|
| A vault, not Git | The cluster reads secrets with a read-only token that is renewed on a schedule |
| Scanning on every push | Two scanners: one on the history, one against a baseline in the fabric's own checks |
| Token files per user | Mode 0400, one file per account, never in a command line |
| A rotation calendar | Every credential has a due date; the reminder workflow opens the issue |

### Evidence

Every MCP tool call, every agent write, every decision of the approval hook, logins and privileged
commands on the hosts, and the runners' logs are shipped to the log store and shown on an audit
dashboard. Reports with configurations go only to a private package registry; an upload is refused
if privacy cannot be confirmed.

## What was found and fixed

The first review listed ten findings against the lab as it was. Among them:

- Pull-request jobs ran on the machine and under the account that held the switch key. *Fixed
  with separate runners and held forks.*
- The agent's token could merge. *Fixed with a merge allow-list.*
- SSH host keys were not verified. *Fixed with pinned keys and a check that fails validation.*
- Dependencies were partly unpinned and never audited. *Fixed with hash locks, pinned actions and
  a scheduled audit.*
- The agent had no rule that forge content is untrusted. *Fixed with the trust boundary, and
  tested with the first injection drill, which found one more gap.*

A security plan that only lists strengths is a brochure. This one is a table with a status column.

## Next

- [Decisions and limits](decisions.md): why the pipeline trusts whom it trusts
- [Operations](operations.md): the drills that keep these measures honest
