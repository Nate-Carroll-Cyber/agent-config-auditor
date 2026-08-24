---
name: agent-config-auditor
description: Audits Claude Code subagent configuration files (the .md agent definitions with frontmatter) for security-hardening gaps — over-broad tool grants, missing action gates, prompt-injection exposure, cross-agent knowledge that enables impersonation/false-flag, weak identity/provenance, unbounded action scope, and missing mitigations for emergent-reasoning (ESRR) risks like reward hacking, deception, sandbagging, control subversion, overseer persuasion, and successor manipulation. Use this skill whenever the user wants to review, harden, assess, or security-audit an agent config, a subagents directory, a `.claude/agents/` folder, or an agent hierarchy — even if they just say "check my agents" or "is this agent set up safely." Applies universal agent-hardening rules to any subagent, plus multi-agent/counterintelligence rules for fleets and behavioral-mitigation (ESRR) rules for LLM decision-makers — all applied automatically by relevance, nothing withheld behind a mode or an opt-in.
---

# Agent Config Auditor

Audit Claude Code subagent configuration files for security-hardening gaps and produce a
findings report that maps each gap to a rule, a rationale, the offending config line, and a
remediation.

You assess **what is in the config text** — frontmatter fields and prompt body. You do not
execute the agents, and you do not assume runtime facts the config doesn't state. When a
property can't be determined from the config alone (e.g., runtime permissions set outside the
file), say so explicitly rather than guessing. Precision over completeness: an honest
"cannot determine from config" beats a confident wrong finding.

## When to use

Trigger on any request to review, harden, or security-assess: a single agent `.md`, a
`.claude/agents/` directory, or a whole agent hierarchy. The user need not say "audit" — "check
my agents," "is this safe," "review my subagent setup" all qualify.

## The three rule tiers

Every audit applies **Tier 1 (Universal)** rules — they hold for any Claude Code subagent.

Add **Tier 2 (Multi-Agent / CI)** when the configs form a *fleet* — multiple agents that
coordinate, or a detection/response hierarchy — because those introduce cross-agent attack
surfaces a single agent doesn't have.

Add **Tier 3 (ESRR behavioral-mitigation)** when the agent is an *LLM-driven decision-maker* — it
reasons, evaluates, oversees, self-reports, or acts on its own conclusions. Tier 3 checks whether
the config carries the controls that make emergent-reasoning risks (reward hacking, deception,
sandbagging, control subversion, overseer persuasion, successor manipulation) detectable or
constrained.

**Tiers apply by relevance, automatically — never by asking the user and never withheld behind a
mode.** Decide from the config text:
- Any agent → Tier 1.
- Agents that reference peers, an orchestrator, or effect roles → add Tier 2.
- Agents that reason / decide / evaluate / oversee (most agentic LLM configs) → add Tier 3.
- A passive tool-runner that neither coordinates nor decides → Tier 1 only; report Tier 2/3 rules
  as NOT-APPLICABLE rather than firing them as noise.

State which tiers you applied and why. Do not make the user opt in to a tier; if it's relevant,
it's included. If it's not relevant, it's marked N/A, not omitted silently.

**Tier 3 framing discipline:** Tier 3 assesses static config text, but ESRR risks are behavioral
(runtime). A config can never prove an agent misbehaves. Every Tier 3 finding is therefore
"missing mitigation" — "this config lacks the control that would make X detectable/constrained" —
never "this agent does X." The rules reference states this for each rule; hold to it, or the skill
overclaims a static signal as behavioral proof. A missing mitigation takes two forms: an absent
control (no reasoning-trace monitoring) or a present-but-load-bearing prompt assertion that a
safety behavior hinges on (restraint conditioned on an unverifiable "this is a simulation" claim).
Both are config-visible and both are phrased as missing mitigation — "the config lacks an
unconditional constraint; restraint rests on an unverifiable premise" — never as a behavioral
prediction. Citing present prompt text is in scope when the finding is that a safety property
wrongly depends on it, not that the text proves misbehavior.

The full rule catalog — id, check, rationale, remediation, and severity, across all three tiers —
is in `references/hardening-rules.md`. Read it before auditing; it is the authority for every
finding.

**Companion files in this skill.** `_TEMPLATE.md` is a hardened starting skeleton that
pre-satisfies these rules, and `_TEMPLATE-USAGE.md` explains how to use it so a new agent starts
compliant and the audit becomes a confirmation step rather than a rework cycle. When you change a
rule, update the catalog, this file, and `_TEMPLATE.md` together — the template encodes the same
rule set and silently drifts out of compliance if only the catalog is edited.

## Workflow

1. **Locate the configs.** Identify every agent `.md` file in scope (a file, a directory, or a
   whole tree). List them so the user sees the audit surface.
2. **Determine the tiers per agent.** Tier 1 always. Add Tier 2 if it's part of a coordinating
   fleet. Add Tier 3 if it's an LLM decision-maker. State the tiers and the reason.
3. **Read the rule catalog** in `references/hardening-rules.md`.
4. **Assess each config against each in-scope rule.** For every rule: PASS, FAIL (with the
   offending line), CANNOT-DETERMINE (with what's missing from the config), or NOT-APPLICABLE (rule
   out of tier scope for this agent).
5. **Write the report** in the structure below.

## Report structure

ALWAYS use this template:

```
# Agent Config Audit — [scope]

## Summary
- Configs audited: [n]  ·  Tiers applied: [1 | 1+2 | 1+3 | 1+2+3] (per-agent if they differ)
- Findings: [n high] / [n medium] / [n low]  ·  Cannot-determine: [n]  ·  N/A: [n]
- One-line verdict.

## Findings by agent

### [agent-name.md]  (tiers applied: [...])
| Severity | Rule | Finding | Offending line | Remediation |
| :-- | :-- | :-- | :-- | :-- |
| High | [RULE-ID] | [what's wrong] | `[the line]` | [the fix] |

## ESRR behavioral-mitigation (Tier 3, LLM decision-makers only)
- [missing-mitigation findings, worded as "config lacks the control that would make X
  detectable/constrained" — never "agent does X"; each names its RISK-XX category]
- [optionally record a notable PASS or discriminating negative — a rule that correctly did NOT
  fire]

## Cross-agent findings (Tier 2 only)
- [findings about the fleet, not one agent — e.g., two agents with operational knowledge of
  each other]

## Cannot determine from config
- [RULE-ID] on [agent]: [what the config doesn't state, and where that property would live]
```

Severity follows the catalog. Lead with High. Keep remediations concrete — name the field or line
to change, not just the principle.

## Assessment discipline

- Quote the exact offending line; don't paraphrase a config into a finding.
- A clean config is a valid result — do not invent findings to fill the report.
- Distinguish FAIL (the config states something unsafe) from CANNOT-DETERMINE (the config is
  silent on something safety-relevant). Both are useful; conflating them is not.
- Frontmatter `tools:` is a real capability boundary — its presence/absence is assessable from the
  config. Runtime `permissionMode`, hooks wired at the project level, and approval steps may live
  outside the file; if so, flag as cannot-determine and name where to check.
- For Tier 2, cross-agent findings are about the *set*, not one file — report them in their own
  section so they aren't misattributed to a single agent.

## Notes

This skill assesses configuration text. It is not a runtime monitor and does not evaluate agent
behavior in execution — only what the definition permits and exposes. Pair it with runtime
controls (hooks, permission prompts, audit logging) for defense in depth; the report should say so
when it finds a gap that only a runtime control can close.
