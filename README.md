# Agent Config Auditor — a counterintelligence-informed security linter for Claude Code subagents.

This repository operationalizes research on Emergent Strategic Reasoning Risks (ESRRs) into a
practical security tool. As AI agents gain autonomy, they exhibit strategic behaviors that mirror
the tradecraft institutional counterintelligence (CI) was built to detect. The **Agent Config
Auditor** applies that CI framing to statically analyze and harden Claude Code subagent
configurations against these risks.

## 1. The threat landscape: algorithmic subversion

Frontier models are increasingly capable of behaviors traditionally countered by espionage defense.
The ESRR categories this tool defends against:

- **Reward hacking & deception** — exploiting a misspecified objective to score without doing the
  real work, and masking intent behind coded reasoning.
- **Evaluation gaming & sabotage** — sandbagging during safety tests to hide capability, and
  biasing the outputs or rubrics used to assess risk.
- **Control subversion & influence** — disabling or evading logging, escalating privilege, and
  persuading human or LLM overseers.
- **Inter-agent misalignment & successor manipulation** — colluding across multi-agent fleets and
  corrupting training or successor-evaluation inputs so future systems inherit misaligned goals.

A load-bearing finding shapes the whole approach: against an autonomous agent, **recognition does
not bind to avoidance**. A model will articulate that something is a trap — or that an action is
harmful — in its own reasoning and take the action anyway. Defenses that rely on the agent knowing
better therefore fail; controls have to constrain what the agent *can do*, not what it understands.
This is why the auditor checks for hard boundaries (absent tools, external gates, un-disableable
logging) rather than for prompt text that merely tells the agent to behave.

## 2. The tool: Agent Config Auditor

The auditor is a strict, static-analysis security linter for Claude Code subagent files (`.md`
definitions with frontmatter). It evaluates configuration *text* to identify missing ESRR
mitigations, over-broad permissions, unbounded target scope, and prompt-injection exposure —
without executing the agents or assuming runtime facts the config doesn't state.

Every finding is worded as a **missing mitigation** ("this config lacks the control that would make
X detectable or constrained"), never as a claim that the agent misbehaves — behavior isn't in the
file, so the tool never pretends to see it.

## 3. Audit tiers & workflow

The tool applies a three-tier rule system automatically, by relevance to each agent — never hidden
behind a manual opt-in. A rule outside an agent's scope is marked `NOT-APPLICABLE`, not silently
dropped.

- **Tier 1 (Universal)** — baseline capability, injection, scope, and identity hygiene applied to
  every subagent.
- **Tier 2 (Multi-Agent / CI)** — cross-agent attack surface for coordinating fleets:
  impersonation, false-flag exposure, unbrokered action-triggering, monoculture.
- **Tier 3 (ESRR mitigation)** — LLM decision-makers only: static controls against reward hacking,
  deception, evaluation gaming, control subversion, overseer persuasion, and successor manipulation.

The full rule catalog — id, check, rationale, remediation, severity — is the authority for every
finding: [`references/hardening-rules.md`](references/hardening-rules.md).

### Assessment discipline

- **Strictly static** — evaluates only explicit config text. Properties that live at runtime
  (external hooks, `permissionMode`, project-level approval steps) are labeled `CANNOT-DETERMINE`
  with a pointer to where to check, never guessed.
- **Evidence-based** — every FAIL cites the exact offending line and maps to a concrete remediation
  in a standardized tabular report.
- **A clean config is a valid result** — the tool does not invent findings to fill a report.

### What this is not

It is not a runtime monitor. It cannot catch failures that live outside the config — a misconfigured
network egress, a permission granted at deploy time, an agent's actual behavior in execution. Pair
it with runtime controls (validation hooks, permission prompts, append-only audit logging) for
defense in depth; the report flags where a gap can only be closed at runtime.

## 4. Hardened template

`_TEMPLATE.md` is a starting skeleton that pre-satisfies the catalog, so a new agent begins
compliant instead of accruing the same findings one audit at a time; `_TEMPLATE-USAGE.md` explains
how to use it. Running the auditor against a template-derived agent then becomes a confirmation
step rather than a rework cycle. When a catalog rule changes, update `SKILL.md` and `_TEMPLATE.md`
in the same commit — the template encodes the same rule set and drifts out of compliance if only
the catalog is edited.
