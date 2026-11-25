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
| A new switch starts locked | The configuration a switch gets at its first boot holds locked accounts and one public key, nothing confidential; the pipeline cannot reach the switch until its host key is pinned by a pull request |
| Telemetry certificates from the fabric's own CA | Every switch's gNMI server presents a certificate that the collector verifies; details below |

### Certificates for telemetry

Without a certificate of its own, a switch's gRPC server makes itself a self-signed one that is
valid for a day and is not renewed. Telemetry then only works with verification switched off,
which is how the lab ran at first. Now every one of the fourteen switches serves a certificate
from the fabric's own certificate authority, and the collector verifies it.

| Part | How it is built |
|---|---|
| Root CA | Offline; its key never touches the vault or a networked host. Ten years |
| Issuing CA | In the vault's PKI engine; its key was generated there and cannot be exported; its certificate is signed by the root. Five years |
| Switch keys | Made on the switch, not exportable; never in the vault or in Git |
| Switch certificates | One year, for the switch's name and its management address |
| Enrolment | One manual run of a workflow per switch: the switch makes its key and a signing request, the request is checked, the vault signs it, the certificate is imported and read back |
| Binding | A setting in the data model, deployed like any change: the role refuses to bind a trustpoint that holds no certificate |
| Proof | A test in every run makes a TLS handshake to each switch's gRPC port and requires a certificate from this CA, for this switch, accepted by a verifying client |

At first the owner signed each request by hand between two runs. With six new switches and a
renewal every year that became the slow part, so the vault now signs inside the run. What keeps
that from being a signing oracle:

| Measure | Effect |
|---|---|
| No stored credential | The signing job logs in to the vault with its own short-lived identity token from the forge. There is no secret to steal from the forge, the runner or the repository |
| Only `main` signs | The vault's role accepts only a token that names this repository, this workflow file, the `main` branch and a manual start. A workflow edited on a branch gets a token that cannot log in |
| What is signed comes from the model | The name and the address in the certificate are taken from the data model, never from the request or the run's inputs, and the address must be the switch's own management address |
| A runner of its own | The signing job runs on a runner that takes no other job: a fresh container per job, no checkout and no repository code, and a network that reaches the forge and the vault and nothing else, no switch included |
| Short-lived everything | The vault token lasts minutes and a few uses, and is revoked when the job ends; the vault records every certificate it signs |
| A fallback | The two-run path with the owner signing in between still works, and deleting one role in the vault switches the automatic path off |

What remains is said in the design as well: the runner's pod is the boundary, not a sandbox of
its own. Whoever controls that pod or its node could read a token while a job runs, and could
then have server certificates signed for a year. That would let someone pose as a switch to the
collector; it gives no login to a switch, no other path in the vault and no CA key.

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
- [Setup](setup.md#2-day-zero-on-each-switch): how a new switch gets its first configuration
- [Operations](operations.md): the drills that keep these measures honest
