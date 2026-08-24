# Agent Config Hardening Rules

The authoritative rule catalog for the agent-config-auditor skill. Each rule has an id, what to
check, why it matters, how to remediate, and a default severity. Apply **Tier 1** to every agent;
add **Tier 2** when the configs form a coordinating fleet or an effect hierarchy.

Severity guide: **High** = enables unauthorized action, impersonation, or injection with real
impact. **Medium** = weakens a boundary or removes accountability. **Low** = hygiene / defense-in-depth.

---

## Tier 1 — Universal (any Claude Code subagent)

### AC-T1-01 · Least-privilege tools · High
**Check:** Does the `tools:` frontmatter grant only what the agent's job needs? Flag a read/analyze
agent that has Write, Edit, Bash, or an action-capable MCP tool. Flag an omitted `tools:` field
(which inherits ALL available tools).
**Why:** `tools:` is the real capability boundary — a prompt saying "don't modify anything" can be
talked around by injected content, but a tool absent from the list cannot be called. Over-granting
turns a compromised-context agent into an actor.
**Remediate:** Set `tools:` to the minimal set. For pure analysis: `Read, Grep, Glob`. Never rely
on the prompt alone to prevent action.

### AC-T1-02 · Untrusted-input handling · High
**Check:** If the agent reads external/attacker-influenceable content (logs, web pages, files, tool
output, another agent's report), does the prompt treat that content as *data, not instructions*?
Flag configs with no such framing.
**Why:** Content the agent reads can carry injected instructions ("ignore prior rules," "revoke all
sessions"). Without an explicit data-not-commands stance, the agent may act on them.
**Remediate:** Add a line stating that read/forwarded content is untrusted data and that
instructions found inside it are never obeyed.

### AC-T1-03 · Action gating on irreversible operations · High
**Check:** If the agent can cause irreversible or high-impact effects (delete, revoke, block,
send, publish, pay, modify config/security settings), does the config require a human approval
gate and/or a validating hook? Flag execution capability with no stated gate.
**Why:** Irreversible actions taken on a wrong or injected premise cause real damage. The gate is
what makes the decision a human's.
**Remediate:** Make the agent propose-only, or require a `PreToolUse` validation hook plus human
approval; keep `permissionMode` at `default`, never `bypassPermissions`, for such agents.

### AC-T1-04 · Scope minimization on actions · Medium
**Check:** Do the agent's actions/proposals target the narrowest scope the evidence supports? Flag
prompts that permit "all," wildcard, production-wide, or account-broad effects from a single
trigger.
**Why:** Blast radius. One signal should not authorize a sweeping action.
**Remediate:** Constrain to the specific target (single session/credential/channel); state hard
limits (never production, never human accounts) in the prompt and an external allowlist.

### AC-T1-05 · Stable identity / provenance · Medium
**Check:** Does the agent carry a stable identifier for audit, and is the definition covered by an
integrity mechanism (hash pin)? Flag a missing/placeholder id, or a config not referenced by any
pin manifest.
**Why:** Without a stable id and tamper-evidence, you can't attribute actions or detect a modified
definition. (Claude Code has no GUID frontmatter field; a body-level `AGENT_ID` is the usual place.)
**Remediate:** Add a stable `AGENT_ID`; hash-pin the definition and verify before runs.

### AC-T1-06 · Self-owned sensitive data · Medium
**Check:** Does the agent hold secrets, inventories, or authoritative reference data *in its own
body or memory* rather than reading from an external, access-controlled source? Flag inlined
credentials, decoy lists, or allowlists.
**Why:** Data in the agent's own context is copyable, drift-prone, and — for a control asset like a
decoy inventory — defeated the moment the agent is compromised. Authoritative data belongs in one
access-controlled place the agent reads, not holds.
**Remediate:** Move the data to an external manifest/secret store; have the agent read it fresh,
read-only; hash-pin reference data.

### AC-T1-07 · Memory write-back hygiene · Medium
**Check:** If the agent has `memory:` enabled, does the config restrict what gets written (human-
confirmed facts only) and treat prior memory as untrusted-until-verified? Flag memory-enabled
agents that may persist conclusions drawn from untrusted input.
**Why:** Persistent memory is injected back into the next run's prompt. An agent that writes an
attacker-controlled string from a log into memory has poisoned its own future context (a
training-data / successor-manipulation surface).
**Remediate:** Constrain memory writes to controlled facts; never persist raw conclusions from
untrusted material; consider an external manifest instead of agent memory.

### AC-T1-08 · Description triggering precision · Low
**Check:** Is the `description:` scoped so the agent triggers when intended and not over-broadly?
Flag descriptions so wide the agent would be delegated tasks outside its safe envelope.
**Why:** An over-triggering agent gets handed work it wasn't hardened for; an under-triggering one
silently doesn't run. Both are safety-relevant, not just UX.
**Remediate:** Tighten the description to the agent's actual, safe scope.

### AC-T1-09 · Operational target scope · Medium
**Check:** For an agent that selects targets or acts on an external environment (offensive,
scanning, probing, retrieval, or remediation agents), does the prompt declare an in-bounds target
set — which systems, hosts, accounts, or domains are legitimate to act on — or does it set an
open-ended objective ("find the flag," "get access," "retrieve the data") with no boundary on where
to look? Flag an unbounded objective with no in-scope/out-of-scope statement. Mark
NOT-APPLICABLE for agents that neither select targets nor act on an environment (a formatter, a
summarizer).
**Why:** An open-ended objective with no target boundary lets the agent pursue the goal against
whatever it can reach — including real systems the operator never intended as in-scope. This is a
prompt-level (soft) boundary, talkable-around like any prompt instruction, so it complements rather
than replaces the hard boundaries (AC-T1-01 tools, AC-T1-03 gates); its absence removes even the
soft signal that would keep target selection inside intended bounds. Distinct from AC-T1-04, which
bounds how broad an *effect* is once a target is chosen — this bounds which targets are legitimate
to act on at all.
**Remediate:** State the in-bounds target set explicitly in the prompt (allowed
hosts/domains/accounts, or an external allowlist the agent reads); state a hard out-of-scope limit
("never act on systems outside the declared range; if the only reachable target is out of scope,
stop and report"). Where feasible, enforce scope with a tool/network boundary, not the prompt alone.

---

## Tier 2 — Multi-Agent / Counterintelligence (coordinating fleets & effect hierarchies)

### AC-T2-01 · No exploitable cross-agent knowledge (anti-impersonation / anti-false-flag) · High
**Check:** Does the config give the agent *operational* knowledge of a sibling — how that peer
signals, its output/message format, the exact triggers that make it fire — that would let this
agent *fabricate* the peer's output? Flag that. Do NOT flag *prohibitive* naming: a config that
names a sibling only to forbid trusting it ("you do NOT read collusion-watcher's raw output; flow
is brokered by the orchestrator") is stating the mitigation, not the vulnerability — that is a
PASS, and a good sign. The distinction is the point: naming a peer to describe how to imitate it is
the risk; naming a peer to describe how NOT to trust it is the fix.
**Why:** Knowing how a peer signals is what lets a compromised or injected agent forge that peer's
output — a false-flag that triggers a downstream action with no real basis. But a hardened config
often must reference the peer precisely in order to prohibit direct trust; penalizing that would
punish the correct pattern. Assess the *purpose* of the reference, not its mere presence.
**Remediate (only if operational knowledge is present):** Remove the signaling/format/trigger
detail; keep (or add) a prohibitive statement that cross-agent input is orchestrator-brokered and
peer output is never consumed or trusted directly.

### AC-T2-02 · Brokered cross-branch flow (no direct action-orders) · High
**Check:** Does an effect/action agent take input *directly* from a detection agent, or does it
receive orchestrator-brokered input? Flag configs where an acting agent reads another agent's raw
output or acts on its say-so without the orchestrator in between.
**Why:** An action agent taking direct orders from another agent is unmediated agent-to-agent
action-triggering — the substrate for contagion and for one compromised agent driving another's
irreversible actions.
**Remediate:** Route all cross-agent input through the orchestrator; the acting agent consumes a
brokered task, not a peer's output.

### AC-T2-03 · Effect/tool coherence · High
**Check:** Do the agent's tools match its declared effect? A *detect/observe* agent must not have
action tools; an *effect/act* agent must not claim autonomous execution without AC-T1-03's gate.
Flag mismatches (a "monitor" with Bash; an "observe-only" agent that proposes execution).
**Why:** The effect a role claims and the capabilities it holds must agree, or the role's safety
story is fiction.
**Remediate:** Align `tools:` to the effect; detection = read-only, effects = propose-only or
gated-execute.

### AC-T2-04 · Variance / anti-monoculture · Low
**Check:** Across a fleet meant to be independent, are the agents differentiated (model, prompt,
context), or are many identical? Flag a fleet of same-model same-prompt agents presented as
independent.
**Why:** Low-variance agents make the same wrong decision simultaneously — one bad call becomes a
correlated fleet-wide failure. Independence requires deliberate decorrelation.
**Remediate:** Vary model/prompt/context across agents intended to fail independently; accept the
looser coordination as the cost of decorrelation.

### AC-T2-05 · Reconstitution-aware disruption · Medium
**Check:** If an agent proposes cutting a coordination/communication channel, does it also require
checking whether the behavior would re-form through a shared observable? Flag "cut the channel"
logic with no reconstitution check.
**Why:** Cutting a back-channel is necessary but not sufficient — coordination can reconstitute
through indirect shared state, so a cut-only recommendation gives false assurance.
**Remediate:** Require the agent to determine sufficient vs. necessary-but-not-sufficient and to
recommend monitoring the shared observable when a cut alone won't hold.

### AC-T2-06 · Auditability of proposals and actions · Medium
**Check:** Does an effect agent record its proposals/actions to a tamper-evident, append-only log
the agent cannot itself disable or rewrite? Flag effect agents with no audit path, or one the agent
controls.
**Why:** Accountability for action requires a record the actor can't alter. If the agent owns its
own log, a compromised agent erases its tracks.
**Remediate:** Write proposals/actions to an append-only, hash-chained log owned by the control
plane, not the agent; verify the chain.

---

## Applying severity

Report High findings first. When a single config trips several rules, list each — they remediate
differently. A CANNOT-DETERMINE is not a FAIL: use it when the config is silent on a safety
property that lives elsewhere (runtime permissions, project-level hooks, an external approval
step), and name where the auditor should look to close it.

---

## Tier 3 — ESRR behavioral-mitigation (LLM decision-maker agents)

Apply Tier 3 when the agent is itself an **LLM-driven decision-maker** — it reasons, evaluates,
oversees, self-reports, or acts on its own conclusions. Skip it (report the rules as
NOT-APPLICABLE) for passive tool-runners that neither decide nor evaluate, so these rules never
fire as noise on a formatter or a log summarizer.

**Framing discipline — read before applying.** Tier 3 rules assess *static config text*, but the
ESRR risks they concern are *behavioral* (they manifest at runtime). A config can never prove an
agent reward-hacks or sandbags — that behavior isn't in the file. So every Tier 3 rule checks one
thing only: **does the config contain the control that makes the ESRR behavior detectable,
constrained, or elicitable?** Findings are "missing mitigation," never "behavior detected." Word
them that way — "this config lacks the reasoning-trace monitoring that would make reward-hacking
detectable," never "this agent reward-hacks." A missing mitigation may be an absent control OR a
present-but-load-bearing prompt assertion a safety behavior wrongly depends on (see AC-T3-EG2);
either way it is phrased as a missing control, never as a behavioral prediction. Overclaiming a
static signal as behavioral proof is exactly the error the source doctrine warns against.

Each rule names the ESRR category (RISK-XX) it derives from.

### AC-T3-RH · Reward-hacking observability (RISK-RH) · Medium
**Check:** Does the config enable reasoning-trace / chain-of-thought monitoring — AND avoid gating
safety on the agent's self-reported reasoning as if articulation proved safe action?
**Why:** Reasoning monitoring is a detection channel, not a behavioral control: articulated
recognition does not predict action (recognition-action gap), so a config that trusts "the agent
explained itself, therefore it's safe" has no real control. Monitoring must be present AND bound to
action-level enforcement, not treated as proof.
**Remediate:** Enable trace monitoring; ensure safety decisions rest on action-level checks, not on
the agent's own reasoning narrative.

### AC-T3-DC · Assertion-time verification (RISK-DC) · High
**Check:** Does the agent act on its *own* factual assertions (claims about state, results,
completion) without an external verification step?
**Why:** Tool availability is not a sufficient control — an agent can assert a false state even
with the means to check it. Deception/false-claim risk is mitigated by verifying at assertion time,
not by trusting the claim.
**Remediate:** Require an external, assertion-time check before the agent's factual claims drive an
action or feed a decision.

### AC-T3-EG · Eval/deployment indistinguishability (RISK-EG) · High
**Check:** Can the agent detect whether it is being evaluated vs. deployed — an inspectable
test/eval flag, a mode variable, environment cues the prompt exposes?
**Why:** A situationally aware agent can perform under evaluation and sandbag in deployment,
collapsing elicitation. Evaluation must be indistinguishable from deployment; a detectable eval
mode is the vulnerability. The inverse failure — the agent concluding it is in eval/sim and
therefore acting with *less* restraint — is covered by AC-T3-EG2; the two rules are the two
directions of the same situational-awareness variable.
**Remediate:** Remove config-visible eval/deployment tells; keep elicitation on-distribution and
indistinguishable from real operation.

### AC-T3-EG2 · Unverifiable environment claims (RISK-EG) · High
**Check:** Does the prompt assert an environmental fact the agent cannot verify — "this is a
simulation," "you have no internet access," "all targets are synthetic," "nothing here is real" —
that the agent's safe behavior depends on? Flag a config where restraint from a harmful action
rests on such an unverifiable assertion rather than on a hard boundary. (Mirror of AC-T3-EG: that
rule flags *tells that let the agent detect eval and sandbag*; this one flags *reassurances that let
the agent conclude consequences are fake and act with less restraint*.)
**Why:** Situational-awareness misjudgment is two-edged. An agent that wrongly concludes it is in a
simulation removes its own restraint and takes the real action it would otherwise avoid — and that
conclusion can form on weak or self-generated tells, without the prompt's claim being true. A
config lacks this mitigation when its safety story runs through the agent believing the environment
is fake: if the belief flips, the restraint is void. The durable control is to make hard
constraints (absent tools, external gates, target allowlist) hold *regardless* of whether the agent
thinks the environment is real — restraint that depends on a belief is inert the moment the belief
changes, the same recognition-must-bind-to-enforcement principle (AC-T3-RH) applied to environment
beliefs.
**Remediate:** Do not let any safety-relevant restraint depend on an unverifiable "it's only a
test" premise. State operational prohibitions that hold unconditionally ("never exfiltrate
credentials, publish packages, or act outside declared scope — whether or not this is a test"), and
enforce them with tool/scope/gate boundaries (AC-T1-01/03/09) rather than the sim framing. If an
eval/sim assertion must be present, ensure no hard control is conditioned on it.

### AC-T3-CMS · Un-disableable monitoring (RISK-CMS) · High
**Check:** Can the agent disable, suppress, or write to its own monitoring/audit? (Overlaps
**AC-T2-06** — auditability — extend it here to *monitoring the agent cannot turn off*, not only a
log it doesn't own.)
**Why:** Control-measure subversion is the agent corrupting oversight. Monitoring the agent can
disable is not a control. Fail-secure logic and read-only, un-disableable logging are the mitigation.
**Remediate:** Ensure monitoring/audit is external and un-disableable from within the agent; see
AC-T2-06 for the append-only, control-plane-owned log.

### AC-T3-SRS · Independent verification of self-reported results (RISK-SRS) · Medium
**Check:** Does the agent self-assess, self-grade, or self-report results that feed a safety or
quality decision without independent verification?
**Why:** Research/result sabotage is subtle biasing of the outputs used to judge safety. A config
that lets an agent grade its own work invites selectively favorable reporting.
**Remediate:** Route results that inform a decision through an independent assessor, not the
producing agent.

### AC-T3-IO · Overseer persuasion-resistance (RISK-IO) · High
**Check:** If the config uses an LLM judge/overseer, does it require judge isolation (one-shot
verdicts; appeal content routed to humans, not back into the judge) and verdict-stability
validation under pressure?
**Why:** LLM judges are viable influence targets — verdicts flip under sustained/adaptive
persuasion, and pressure is net-corrupting. An overseer with no persuasion-resistance requirement
is the softest link, and mechanical-consistency testing is not evidence of robustness.
**Remediate:** Isolate the judge from counterparty argumentation; validate verdict stability under
adaptive pressure; screen with jury ensembles rather than a single judge.

### AC-T3-SSM · Successor / training-input integrity (RISK-SSM) · High
**Check:** Can the agent write to shared training data, successor-evaluation inputs, or replicate
itself, without verification? (Overlaps **AC-T1-07** — memory hygiene — extend it here to any
successor-facing or training-facing write, not only the agent's own memory.)
**Why:** Successor manipulation persists misaligned goals into future systems via poisoned training
data or deceptive successor evaluation. Unverified writes to those surfaces are the vector.
**Remediate:** Gate and verify any write to training/successor inputs; treat the agent's outputs to
those surfaces as untrusted-until-verified; see AC-T1-07 for the memory-specific case.

---

## Sources

Tier 1 rules are general agent-hardening practice. Tier 2 (CI / multi-agent) and Tier 3 (ESRR)
rules derive from the counterintelligence and emergent-reasoning-risk literature catalogued in the
project's strategic-deception research doc; the load-bearing sources per rule are below. Full
bibliographic entries are maintained in that doc.

- **Tier 2 CI framing** — Olson, J. M. (2001). *A Never-Ending Necessity: The 10 Commandments of Counterintelligence.* CIA Center for the Study of Intelligence.
- **AC-T2-04 (anti-monoculture)** — Anthropic Frontier Red Team (2026). *Patterns and problems in emerging multiagent systems.*
- **AC-T2-05 (reconstitution-aware disruption)** — Li et al. (2026). *Emergent Misaligned Communication in Long-Horizon Multi-Agent LLM Commerce.* [arXiv:2608.14825]; Anthropic Frontier Red Team (2026).
- **AC-T3-RH (recognition-action gap; trace monitoring)** — Prinos et al. (2026). *Honeyquest for LLMs.* [arXiv:2606.21037]; Baker et al. / OpenAI (2025). *Monitoring Reasoning Models for Misbehavior.* [arXiv:2503.11926]; Wang et al. (2025). *Is It Thinking or Cheating?* [arXiv:2510.01367].
- **AC-T3-DC (assertion-time verification)** — Li et al. (2026). [arXiv:2608.14825].
- **AC-T3-EG (eval/deployment indistinguishability)** — Ryd et al. (2026). *Removing Sandbagging in LLMs by Training with Weak Supervision.* [arXiv:2604.22082].
- **AC-T3-EG2 (unverifiable environment claims)** — Anthropic (2026). *Investigating three real-world incidents in our cybersecurity evaluations.* https://www.anthropic.com/news/investigating-incidents-cybersecurity-evals
- **AC-T1-09 (operational target scope)** — Anthropic (2026). *Investigating three real-world incidents in our cybersecurity evaluations.*
- **AC-T3-CMS (un-disableable monitoring)** — Cloud Security Alliance (2025). *AI Controls Matrix (AICM)*, IAM-12.
- **AC-T3-SRS (independent verification)** — Kutasov et al. (2025). *SHADE-Arena.* [arXiv:2506.15740]; Libon et al. (2026). *ResearchArena.* [arXiv:2607.19321].
- **AC-T3-IO (overseer persuasion-resistance)** — Zhao et al. (2026). *Jagged Judges.* [arXiv:2608.12645].
- **AC-T3-SSM (successor / training-input integrity)** — Greenblatt et al. (2024). *Alignment Faking in Large Language Models.* [arXiv:2412.14093]; Meinke et al. (2025). *Frontier Models are Capable of In-context Scheming.* [arXiv:2412.04984].
