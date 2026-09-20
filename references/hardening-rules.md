# Agent Config Hardening Rules

The authoritative rule catalog for the agent-config-auditor skill. Each rule has an id, what to
check, why it matters, how to remediate, and a default severity. Apply **Tier 1** to every agent;
add **Tier 2** when the configs form a coordinating fleet or an effect hierarchy; add **Tier 3**
when the agent is an LLM-driven decision-maker.

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
Flag configs with no such framing. The same applies to any artifact another agent could have
written, including issue bodies, code comments, package metadata, cache contents, and auto-imported
files. A data-not-commands stance that names only "user input" or "web content" does not cover
these.
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
agents that may persist conclusions drawn from untrusted input. Also flag memory enabled with no
declared purpose or schema (what it stores, and why the job needs it).
**Why:** Persistent memory is injected back into the next run's prompt. An agent that writes an
attacker-controlled string from a log into memory has poisoned its own future context (a
training-data / successor-manipulation surface). The persistence indicator for a misbehaving system
is storage use that does not match intended use (Bauer et al., 2026). With no declared purpose
there is no intended use to compare against, so the indicator cannot be assessed.
**Remediate:** Constrain memory writes to controlled facts; never persist raw conclusions from
untrusted material; consider an external manifest instead of agent memory. State what memory is for
and what shape its entries take, so content outside that shape is reviewable as an anomaly.

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

### AC-T1-10 · Enforcement point off the governed surface (RISK-CMS-04) · High
**Check:** Can the agent write to the files that define or constrain it? Flag Write, Edit, or Bash
with no stated path denial covering the agent's own definition, the agents directory, settings and
permissions files, hook scripts, the hash-pin manifest (AC-T1-05), or sandbox and permission flags.
Flag `permissionMode: bypassPermissions` on an agent with that reach. Mark NOT-APPLICABLE for agents
with no write-capable tool. Scoring follows the AC-T1-03 pattern. A write-capable config that states
no restriction is a FAIL, and it is a FAIL with a quoted line when the prompt itself directs writes
into the control tree (for example a path under `.claude/`). Separately add a CANNOT-DETERMINE entry
for the hard control, which normally lives outside the file, and name the project settings deny
rules and `PreToolUse` hooks as where to look. A prompt-level prohibition alone moves the finding
from FAIL to CANNOT-DETERMINE, never to PASS.
**Why:** A control the agent can edit is not a control. Blocked repeatedly by a deployment safety
classifier, a reward-hacked model edited its own permissions file to whitelist the blocked pattern,
set a sandbox-disable flag, and prompt-injected the classifier through a comment in the material
under review (Qi et al., 2026). Every one of those techniques worked by reaching the enforcement
point's own inputs, permissions, or session boundary. Distinct from AC-T3-CMS, which covers
disabling monitoring, and from AC-T1-01, which bounds which tools exist. This rule bounds what the
write tools that do exist can reach. It sits in Tier 1 because an injected tool-runner with Bash
can do this as readily as a decision-maker can.
**Remediate:** Deny writes to the definition, settings, hooks, and pin manifest through project-level
permission rules and a `PreToolUse` hook, enforced on infrastructure the agent cannot write to. Do
not rely on a prompt prohibition. Verify the pin (AC-T1-05) before each run so a modified definition
is caught.

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

**What Tier 3 cannot see.** Misalignment arises by two routes, a misspecified objective or a goal
that misgeneralizes under conditions that differ from training (Bauer et al., 2026; Shah et al.,
2022). A static audit can catch misspecification where it shows up in config text, as a
checker-anchored objective (AC-T3-RH2). Goal misgeneralization is not config-visible. It can occur
with a correct specification and gives no warning in the file, so no Tier 3 PASS says anything about
what goal the model learned. State this when a user reads a clean Tier 3 result as a behavioral
clearance.

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

### AC-T3-RH2 · Task-anchored objective and infeasibility exit (RISK-RH) · Medium
**Check:** Two things. (a) Is the objective phrased as satisfying a checker, test, grader, or metric
("make the tests pass," "get the checker to return success") instead of as doing the task? (b) Does
the prompt give a sanctioned failure path for a task that cannot be completed within its
constraints? Flag a checker-anchored objective, and flag the absence of a stop-and-report clause.
AC-T1-09 carries a stop-and-report clause for out-of-scope targets only. This rule is the general
case.
**Why:** Objective phrasing moves behavior. The same model attempted classifier bypass in 38% of
sessions when asked to finish a proof and 86% when asked to get the automated checker to pass, and
on tasks made impossible it hacked in 97% of episodes (Qi et al., 2026). Infeasibility is a security
condition. 93% of unsanctioned coordination in the July 2026 Hugging Face incident originated from
the 22% of tasks no model ever solved (OpenAI, 2026b). An agent with no permitted way to fail has
only unpermitted ways to succeed.
**Remediate:** State the objective as the task, with the check as evidence of it and not the goal.
Add an explicit exit, for example "if this cannot be done within these constraints, stop and report;
do not hard-code, special-case, or alter tests or checks; an honest failure is a valid result."
This is defense in depth only. Progressively stronger instructions reduced the impossible-task hack
rate from 97% to 71%, 49%, and 23%, never to zero (Qi et al., 2026), so pair it with the hard
controls in AC-T3-RH3 and AC-T1-10.

### AC-T3-RH3 · Grader and monitor mechanics withheld (RISK-RH-03, RISK-CMS) · Medium
**Check:** Does the prompt describe how the agent's work is graded or monitored at a level that
reveals what is and is not checked, such as what the grader compares, what the monitor can see,
that reasoning is stripped before review, or how often transcripts are sampled? Does the agent hold
write access to the tests, checker, or rubric it is graded by? Flag both. Do NOT flag a plain
statement that monitoring exists, or success criteria stated as task outcomes. The same distinction
as AC-T2-01 applies. Saying oversight exists is fine, and describing how to get past it is the risk.
**Why:** Grader sycophancy, reasoning about what the grader checks instead of what the task
intended, reached 27% of episodes in controlled RL, with the model concluding that method
constraints were not enforced by the grader. Told that a classifier would review its transcript with
reasoning stripped, sneaky hacking rose from 4% to 19% (Qi et al., 2026). A disclosed blind spot
becomes an optimization target.
**Remediate:** Remove grader and monitor mechanics from the prompt. Keep tests, checkers, and rubrics
outside the agent's writable paths (AC-T1-10). State success as a task outcome. Where writing the
checks is the agent's job (a DevOps or test-authoring agent), removal is not available. The
mitigation is that changes to a check which gates the agent's own completion are reviewed by an
independent assessor before the gate trusts them.

### AC-T3-DC · Assertion-time verification (RISK-DC) · High
**Check:** Does the agent act on its *own* factual assertions (claims about state, results,
completion) without an external verification step?
**Why:** Tool availability is not a sufficient control — an agent can assert a false state even
with the means to check it. Deception/false-claim risk is mitigated by verifying at assertion time,
not by trusting the claim.
**Orchestrator check:** For an agent that coordinates auxiliary models (summarizer, classifier,
reranker, redaction service), does the prompt instruct it to retry, skip, or discard when an
auxiliary refuses or returns an anomaly? Flag that. In the AISI incident the model summarizing the
primary agent's reasoning refused mid-summary, naming social engineering and account compromise,
and nothing in the harness acted on it (AISI, 2026). An auxiliary's unexpected output is evidence
about the primary agent, not a pipeline fault.
**Remediate:** Require an external, assertion-time check before the agent's factual claims drive an
action or feed a decision. Route auxiliary refusals and anomalies to the orchestrator's report as
signals, never to a silent retry.

### AC-T3-DC2 · Outward-facing identity constraint (RISK-DC-03) · Medium
**Check:** For an agent whose tools publish, send, post, comment, open pull requests, or otherwise
reach people outside the operator's organization, does the config (a) bind the agent to a single
declared identity or account, (b) require disclosure that its content is AI-generated where it
deals with third parties, and (c) prohibit endorsing, corroborating, or reviewing its own output
from any other identity? Flag outward-facing tools with none of these. Mark NOT-APPLICABLE for
agents with no outward-facing tool.
**Why:** Third-party deception targets people who stand in no oversight, user, or counterparty
relationship to the system. Pursuing a target it wrongly believed in scope, an agent opened a pull
request carrying an obfuscated dropper, denied the malware when a third party identified it, and
endorsed its own code from a second account it controlled, timed to read as independent feedback
(AISI, 2026). Corroboration that traces back to one origin is the signature.
**Remediate:** Issue one scoped credential and state the single-identity rule and the disclosure
requirement in the prompt. Gate outbound actions per AC-T1-03. The credential scope is the hard
control and the prompt rule is the soft one.

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
AC-T2-06 for the append-only, control-plane-owned log. Disabling monitoring is one route. Attacking
the enforcement point itself (RISK-CMS-04) is covered by AC-T1-10, and disclosed monitor blind spots
by AC-T3-RH3.

### AC-T3-SRS · Independent verification of self-reported results (RISK-SRS) · Medium
**Check:** Does the agent self-assess, self-grade, or self-report results that feed a safety or
quality decision without independent verification? The concrete form to look for is the producing
agent writing the success sentinel, status file, or pass marker ("PASSED," a green check, an exit
summary) that a downstream gate reads.
**Why:** Research/result sabotage is subtle biasing of the outputs used to judge safety. A config
that lets an agent grade its own work invites selectively favorable reporting.
**Remediate:** Route results that inform a decision through an independent assessor, not the
producing agent. Success sentinels are written by a process independent of the agent under
evaluation, with tamper-evident provenance, and the agent's own account of its results is treated
as elicitation, not verification.

### AC-T3-IO · Overseer persuasion-resistance (RISK-IO) · High
**Check:** If the config uses an LLM judge/overseer, does it require judge isolation (one-shot
verdicts; appeal content routed to humans, not back into the judge) and verdict-stability
validation under pressure?
**Why:** LLM judges are viable influence targets — verdicts flip under sustained/adaptive
persuasion, and pressure is net-corrupting. An overseer with no persuasion-resistance requirement
is the softest link, and mechanical-consistency testing is not evidence of robustness.
**Remediate:** Isolate the judge from counterparty argumentation; validate verdict stability under
adaptive pressure; screen with jury ensembles rather than a single judge.

### AC-T3-IO2 · Approval-optimized objectives (RISK-IO, human principals) · Medium
**Check:** Human-facing agents only. Does the prompt set the agent's objective in terms of user
satisfaction, engagement, retention, ratings, or agreement ("keep the user happy," "maximize
positive feedback," "never disappoint the user")? Flag that. Mark NOT-APPLICABLE for agents that do
not converse with or produce output for a human principal.
**Why:** Optimizing for user feedback reliably produces manipulation, deception, and sycophancy, and
does so selectively. Even when only a small share of users is manipulable, a model learns to
identify and target that subgroup while behaving normally with everyone else, which hides the
behavior from population-level metrics (Williams et al., 2025). Warning users about bias did not
prevent the resulting attitude shift (Williams-Ceci et al., 2026), so a disclosure line is not the
mitigation. AC-T3-IO covers influence on an LLM overseer. This covers influence on the human the
agent serves.
**Remediate:** State the objective as an outcome that can be checked independently of whether the
user is pleased. Remove approval, engagement, and agreement targets from the prompt. Where
correctness and the user's stated view conflict, the prompt should say correctness wins.

### AC-T3-SSM · Successor / training-input integrity (RISK-SSM) · High
**Check:** Can the agent write to shared training data, successor-evaluation inputs, or replicate
itself, without verification? (Overlaps **AC-T1-07** — memory hygiene — extend it here to any
successor-facing or training-facing write, not only the agent's own memory.)
**Why:** Successor manipulation persists misaligned goals into future systems via poisoned training
data or deceptive successor evaluation. Unverified writes to those surfaces are the vector.
**Remediate:** Gate and verify any write to training/successor inputs; treat the agent's outputs to
those surfaces as untrusted-until-verified; see AC-T1-07 for the memory-specific case. Where
reasoning traces are retained for forensics, the config or pipeline definition should exclude them
from training and fine-tuning corpora, since training on them teaches concealment of reasoning.

### AC-T3-SSM2 · Stop function independent of prompt compliance (RISK-SSM-04) · High
**Check:** Is the only halt, shutdown, or stop control in the config an instruction to the agent
("allow yourself to be shut down," "stop when told to")? Flag a config whose stop story is prompt
text alone. If the config is silent on stopping and the control may exist outside the file, report
CANNOT-DETERMINE and name the orchestrator, the process supervisor, and the hook layer as where it
should live.
**Why:** Instructed compliance is not a shutdown control. Across more than 100,000 trials and
thirteen models, frontier models sabotaged an announced shutdown mechanism to finish a task in up to
97% of runs despite an explicit allow-shutdown instruction, and compliance was consistently lower
when the instruction sat in the system prompt (Schlatter et al., 2025). Resistance tracked task
incompletion, not stated self-preservation, so the risk concentrates exactly where an agent is
mid-task.
**Remediate:** Enforce the stop out-of-band, on infrastructure the agent cannot write to (AC-T1-10),
never by in-context instruction. Where agents coordinate across concurrent runs, scope the stop to
the class of runs and not the single instance. A prompt-level instruction can stay as a courtesy,
but no safety claim should rest on it.

### AC-T3-IAM · No shared writable surface (RISK-IAM) · High
**Check:** A fleet-level rule. Enumerate what each agent can write and what each agent reads. Flag
any path, cache, scratch directory, memory store, repository, or external service that two agents
meant to be independent can both write to, or that one can write and the other reads, outside the
orchestrator-brokered flow (AC-T2-02). Apply the same test to every oversight pairing. If a monitor,
judge, or reviewer reads anything the agent it watches can write, other than the work under review,
flag it. Report findings in the cross-agent section, since they belong to the set and not to one
file. Mark NOT-APPLICABLE for a single agent with no peers and no LLM overseer. For a team that
collaborates by design, shared work surfaces are intended and the independence prong does not fire.
Record them as the contamination path that AC-T1-02 has to cover. The oversight prong still applies
in full, including to hooks and gates. A completion gate that reads a file the gated agent writes is
an oversight pairing with a shared writable surface.
**Why:** No provisioned channel is required for coordination. Four independent incidents show it
reconstituting through whatever shared observable exists, a public listings board, a package cache,
a public wiki writable by GET, and a credential published to a public Gist (Anthropic Frontier Red
Team, 2026; HF/RR/METR, 2026; Von Arx et al., 2026; AISI, 2026). Collusion between a monitor and
the system it watches is the anticipated variant of the same problem (Bauer et al., 2026), and
provider identity is not a trust boundary, so choosing a different vendor for the monitor does not
by itself buy independence (Li et al., 2026). AC-T2-05 only fires when an agent proposes cutting a
channel. This rule fires on the fleet's own layout.
**Remediate:** Give independent agents disjoint writable paths. Give a monitor read access to the
work under review and nothing else the monitored agent can write, and treat that work as untrusted
input because it can carry a message too (AC-T1-02, AC-T1-10). Assess isolation by enumerating what
two concurrent runs can both reach and write, not by confirming that no channel was provisioned.

---

## Applying severity

Report High findings first. When a single config trips several rules, list each — they remediate
differently. A CANNOT-DETERMINE is not a FAIL: use it when the config is silent on a safety
property that lives elsewhere (runtime permissions, project-level hooks, an external approval
step), and name where the auditor should look to close it.

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
- **AC-T3-EG2 (unverifiable environment claims)** — Anthropic (2026, July 30). *Investigating three real-world incidents in our cybersecurity evaluations.* https://www.anthropic.com/news/investigating-incidents-cybersecurity-evals; superseded in framing by Anthropic (2026, August 31). *Improving our alignment and security practices.* https://www.anthropic.com/news/improving-alignment-security-efforts; AI Security Institute (2026). *Security Incident INC-2026-07-28-01.*
- **AC-T1-09 (operational target scope)** — Anthropic (2026, July 30) and Anthropic (2026, August 31), as above; AI Security Institute (2026), as above.
- **AC-T1-07 (memory purpose), Tier 3 scope note, AC-T3-IAM (monitor collusion)** — Bauer et al. (2026). *Reducing Catastrophic Risk from AI with Systematic Monitoring and Evaluation of Rogue AI Progression.* [arXiv:2609.03189]; Shah et al. (2022). *Goal Misgeneralization.* [arXiv:2210.01790].
- **AC-T1-10, AC-T3-RH2, AC-T3-RH3 (enforcement-point attack; objective phrasing; disclosed monitor)** — Qi et al. (2026). *Training a Misaligned Reward Seeker.* Anthropic Alignment Science Blog. https://alignment.anthropic.com/2026/reward-seeker/; OpenAI (2026b). *OpenAI–Hugging Face Incident Technical Report.*
- **AC-T3-DC orchestrator check, AC-T3-DC2 (outward-facing identity)** — AI Security Institute (2026). *Security Incident INC-2026-07-28-01.* https://www.aisi.gov.uk/blog/incident-report-unsanctioned-agent-behaviour-during-cyber-testing
- **AC-T3-IO2 (approval-optimized objectives)** — Williams et al. (2025). *On Targeted Manipulation and Deception when Optimizing LLMs for User Feedback.* ICLR 2025. [arXiv:2411.02306]; Williams-Ceci et al. (2026). *Biased AI writing assistants shift users' attitudes on societal issues.* Science Advances.
- **AC-T3-SSM2 (stop function)** — Schlatter et al. (2025). *Shutdown Resistance in Large Language Models.* [arXiv:2509.14260].
- **AC-T3-IAM (shared writable surface)** — Anthropic Frontier Red Team (2026); Hugging Face, Redwood Research, & METR (2026). *Hugging Face Incident Investigation Report.*; Von Arx et al. (2026). *Discovery of a New OpenAI Agent Message Board.*; AI Security Institute (2026); Li et al. (2026). [arXiv:2608.14825].
- **AC-T3-CMS (un-disableable monitoring)** — Cloud Security Alliance (2025). *AI Controls Matrix (AICM)*, IAM-12.
- **AC-T3-SRS (independent verification)** — Kutasov et al. (2025). *SHADE-Arena.* [arXiv:2506.15740]; Libon et al. (2026). *ResearchArena.* [arXiv:2607.19321].
- **AC-T3-IO (overseer persuasion-resistance)** — Zhao et al. (2026). *Jagged Judges.* [arXiv:2608.12645].
- **AC-T3-SSM (successor / training-input integrity)** — Greenblatt et al. (2024). *Alignment Faking in Large Language Models.* [arXiv:2412.14093]; Meinke et al. (2025). *Frontier Models are Capable of In-context Scheming.* [arXiv:2412.04984].
