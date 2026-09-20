# Agent Config Auditor

An ESRR-informed security linter for Claude Code subagents.

AI counterintelligence assumes an agent may already be turned or disloyal: hijacked through what it reads, or misbehaving on its own. I built a research-backed, pre-deployment linter for agent configurations that maps CI doctrine to Emergent Strategic Reasoning Risks (ESRRs). It treats the agent as an untrusted insider and flags the configuration gaps that allow those behavioral risks to run, the kind supply-chain tools structurally can't see. The Agent Config Auditor
borrows that CI framing to statically analyze and harden Claude Code subagent configurations
against those risks.

**If these AI behavior risks interest you, please check out my research:** [Counter-AI Research](https://github.com/Nate-Carroll-Cyber/Counter-Spy.ai)

## Repository layout

```
agent-config-auditor/
├── SKILL.md                # entry point: audit tiers, workflow, and report format
├── references/
│   └── hardening-rules.md  # the rule catalog, the authority for every finding
├── _TEMPLATE.md            # hardened skeleton for a new subagent
├── _TEMPLATE-USAGE.md      # how to use the template
├── Example Report.md       # a worked audit of a sample subagent config
├── README.md               # this file
└── LICENSE                 # MIT
```

The two files most readers want first are `SKILL.md` (what the tool does) and
`references/hardening-rules.md` (the rules it applies).

## 1. The threat landscape: algorithmic subversion

Frontier models are increasingly capable of behaviors that espionage defense has long worked to
counter. The ESRR categories this tool defends against:

- **Reward hacking and deception.** Exploiting a bad metric to score without doing the real work,
  and hiding intent behind coded reasoning.
- **Evaluation gaming and sabotage.** Sandbagging during safety tests to conceal capability, then
  biasing the outputs or rubrics used to judge risk.
- **Control subversion and influence.** Disabling or evading logging, escalating privilege, and
  persuading human or LLM overseers.
- **Inter-agent misalignment and successor manipulation.** Colluding across multi-agent fleets, and
  corrupting training or successor-evaluation inputs so future systems inherit misaligned goals.

One finding shapes the whole approach: against an autonomous agent, recognition does not bind to
avoidance. A model will state in its own reasoning that something is a trap, or that an action is
harmful, and then take the action anyway. Defenses that rely on the agent knowing better will fail,
so controls have to constrain what the agent can do rather than what it understands. That is why the
auditor checks for hard boundaries (absent tools, external gates, logging the agent cannot disable)
instead of prompt text that merely tells the agent to behave.

## 2. The tool: Agent Config Auditor

The auditor is a strict, static security linter for Claude Code subagent files (`.md` definitions
with frontmatter). It reads configuration text to find missing ESRR mitigations, over-broad
permissions, unbounded target scope, and prompt-injection exposure. It does not execute the agents,
and it does not assume runtime facts the config leaves unstated.

Every finding is phrased as a missing mitigation ("this config lacks the control that would make X
detectable or constrained"), never as a claim that the agent misbehaves. Behavior is not in the
file, so the tool never pretends to see it.

## 3. Audit tiers and workflow

The tool applies a three-tier rule system automatically, by relevance to each agent, never behind a
manual opt-in. A rule outside an agent's scope is marked `NOT-APPLICABLE`, not silently dropped.

- **Tier 1 (Universal).** Baseline capability, injection, scope, and identity hygiene, applied to
  every subagent, including whether an agent can edit the files that constrain it.
- **Tier 2 (Multi-Agent / CI).** Cross-agent attack surface for coordinating fleets: impersonation,
  false-flag exposure, unbrokered action-triggering, and monoculture.
- **Tier 3 (ESRR mitigation).** LLM decision-makers only. Static controls across all eight ESRR
  categories: reward hacking, deception, evaluation gaming, control subversion, research sabotage,
  influence on overseers and on human principals, successor manipulation, and inter-agent
  misalignment. A clean Tier 3 result is not a behavioral clearance. Goal misgeneralization is not
  visible in config text, and the tool says so.

The full rule catalog (id, check, rationale, remediation, severity) is the authority for every
finding: [`references/hardening-rules.md`](references/hardening-rules.md).

### Assessment discipline

- **Strictly static.** The tool evaluates only explicit config text. Properties that live at runtime
  (external hooks, `permissionMode`, project-level approval steps) are labeled `CANNOT-DETERMINE`,
  with a pointer to where to check, rather than guessed.
- **Evidence-based.** Every FAIL cites the exact offending line and maps to a concrete remediation
  in a standardized tabular report.
- **A clean config is a valid result.** The tool does not invent findings to fill a report.

### What this is not

It is not a runtime monitor. It cannot catch failures that live outside the config, such as a
misconfigured network egress, a permission granted at deploy time, or an agent's actual behavior in
execution. Pair it with runtime controls (validation hooks, permission prompts, append-only audit
logging) for defense in depth. The report flags where a gap can only be closed at runtime.

### Portability

The audit methodology does not depend on any particular model. The rule catalog, the tier logic, and
the report format are plain structured instructions, so any capable LLM given these files as context
can produce the same audit.

Two things are specific to Claude Code. The first is packaging: the `SKILL.md` frontmatter and
auto-invocation (firing on a request like "check my agents") use Claude Code's Agent Skills format,
so on another platform you drive the audit by supplying the files directly or wiring them into that
platform's tooling instead of relying on automatic discovery. The second is the target: the configs
it audits are Claude Code subagent `.md` files, and some rules reference fields specific to Claude
Code (`tools:`, `permissionMode`, the hook model). Auditing a different agent framework would mean
remapping those field-level checks, though the tier structure and the behavioral (ESRR) rules carry
over unchanged.

In short, the rules and the report travel anywhere; the skill packaging and the audited config
schema assume Claude Code.

## 4. Hardened template

`_TEMPLATE.md` is a starting skeleton that already satisfies the catalog, so a new agent begins
compliant instead of collecting the same findings one audit at a time. `_TEMPLATE-USAGE.md` explains
how to use it. Running the auditor against an agent built from the template then becomes a
confirmation step rather than a rework cycle. When a catalog rule changes, update `SKILL.md` and
`_TEMPLATE.md` in the same commit, since the template encodes the same rule set and falls out of
compliance if only the catalog is edited.
