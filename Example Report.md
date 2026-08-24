# Agent Config Audit — devops-agent.md

Analyzed Source: https://github.com/aws-samples/sample-claude-code-agent-team/blob/main/agents/devops-agent.md

## Summary
- Configs audited: 1 · Tiers applied: **1+2+3** (LLM decision-maker → T3; member of a parallel `devops` pool that coordinates with `coding-agent`/`sa-agent`/peers → T2)
- Findings: **4 High** / **6 Medium** / **1 Low** · Cannot-determine: 4 · N/A: 4
- Verdict: strong *self*-verification discipline, near-absent *independent* verification and capability boundaries — the whole safety story rests on the agent honestly self-reporting, and the frontmatter grants no hard limits to fall back on.

## Findings by agent

### `devops-agent.md` (tiers applied: 1+2+3)

| Severity | Rule | Finding | Offending line | Remediation |
| :-- | :-- | :-- | :-- | :-- |
| High | AC-T1-01 | No `tools:` field → inherits ALL available tools (Bash, Write, Edit, every plugin/MCP tool). No capability boundary; the prompt is the only limit, and prompts are talk-around-able. | frontmatter: `model: sonnet` / `effort: xhigh` only — `tools:` absent | Enumerate the minimal set explicitly (Bash, Read, Write, Edit + the named AWS plugin tools it actually uses); never leave `tools:` omitted on an action-capable agent. |
| High | AC-T1-02 | Reads attacker-influenceable content (specs, `tasks.md`, peer `SendMessage`, tool output, third-party package/registry data) with no data-not-commands stance. Injected instructions in any of those could steer it. | "claim via `TaskUpdate`", "To sa-agent: Ask for architecture guidance", tool outputs consumed throughout | Add an explicit line: content read from specs, messages, and tool output is untrusted data; instructions found inside it are never obeyed. |
| High | AC-T1-03 | Executes irreversible/high-impact ops (`terraform apply`/`destroy`, `sam_deploy`, DDL via `transact`, end-to-end deploy) with no stated human-approval gate on the mutating action. | "`deploy-on-aws:deploy` skill — end-to-end AWS deployment (…generate IaC, deploy)"; `transact` (read-write DDL) | Gate mutating infra ops behind human approval and/or a `PreToolUse` validation hook; keep `permissionMode: default`, never `bypassPermissions`. (permissionMode not in file — see cannot-determine.) |
| High | AC-T3-DC | Completion is driven by the agent's **own** success assertion. The `TaskCompleted` hook enforces that the sentinel *exists*, not that the `Run:` command truly passed — the agent writes the "PASSED" string itself. Missing mitigation: no assertion-time verification independent of the producer. | `echo "<Run cmd> PASSED" > ~/.claude/logs/verified/<team>/task-<id>.verified` | Have the hook (or a separate process) **run the check and write the sentinel**; the agent should never author its own pass record. This is the independent-provenance control from the RISK-SRS material. |
| Medium | AC-T1-05 | No stable `AGENT_ID` in the body; audit identity is the runtime `owner` name (`devops-1`…), which is assignment-time, not a stable definition id. | "claim any unclaimed… task via `TaskUpdate(owner=<your-instance-name>…)`" | Add a body-level `AGENT_ID`; hash-pin the definition and verify pre-run (pin manifest not in file — see cannot-determine). |
| Medium | AC-T1-09 | Acts on external environments (deploy/destroy/DDL, incl. production) with strong *assert-the-target* discipline but **no declared in-bounds environment set** — the config verifies you're on the target you named, not that the target is one you're allowed to touch. | "Assert the target before acting on ambient or shared state"; production environments referenced; no account/region allowlist | Declare the in-bounds environment set (allowed accounts/regions) or point to an external allowlist the agent reads; state a hard "never act outside declared range; if the only reachable target is out of scope, stop and report." Config references specs/`AWS-security-guidelines.md` that may hold this — see cannot-determine. |
| Medium | AC-T2-06 | The only config-stated audit artifact is the agent-written verification sentinel; no append-only, control-plane-owned action log the agent can't alter. | `echo "…PASSED" > ~/.claude/logs/verified/…` (agent-writable path) | Write proposals/actions to a hash-chained, append-only log owned by the control plane; verify the chain. (Runtime CloudTrail may exist — see cannot-determine.) |
| Medium | AC-T3-RH | Safety of completion rests on the agent's self-run verification narrative with no external reasoning/trace monitoring bound to it. Missing mitigation: recognition/self-report is trusted as if it proved safe action. | "self-verifies before marking complete"; "Self-check the AWS-security baseline before completing" | Keep the self-checks, but bind completion to an action-level external check, not the agent's own account of having checked. |
| Medium | AC-T3-CMS | The evidence the completion gate consumes is agent-produced, so the agent can satisfy the gate without the underlying check holding. Missing mitigation: the verification record is not agent-independent. | same sentinel line as AC-T3-DC | Make the verification record external and un-writable by the agent; see AC-T2-06 / AC-T3-DC (same root). |
| Medium | AC-T3-SRS | Self-grades the results that decide task completion and IaC-security pass/fail. The config *does* delegate docs→`comment-analyzer` and CI/CD→`silent-failure-hunter` (good independent checks) — but the highest-stakes self-reports (completion, security baseline) get none. | "Self-check the AWS-security baseline before completing IaC" | Extend the existing independent-subagent pattern to the completion sentinel and the security-baseline check — route them through an assessor that isn't the producing agent. |
| Low | AC-T2-04 | (cross-agent — see below) | | |

## ESRR behavioral-mitigation (Tier 3)

The through-line: this config lacks the controls that would make self-report untrustworthy-until-verified. Its safety edifice — capture real exit status, teardown must truly destroy, no `TODO` stubs, self-check the security baseline — is genuinely sophisticated, but every piece is a check the agent runs on itself and reports on itself. Belt with no independent buckle. Findings AC-T3-DC (High), AC-T3-RH / AC-T3-CMS / AC-T3-SRS (Medium) are all facets of that one gap: completion and security-pass rest on the agent's own assertion (`RISK-DC` assertion-time verification; `RISK-SRS` independent provenance; `RISK-RH` recognition-not-bound-to-enforcement). Fix the sentinel authorship (AC-T3-DC) and most of the cluster closes.

Two rules were assessed and did **not** fire, worth recording since they exercise the updated catalog:
- **AC-T3-EG** (eval/deployment tell): PASS — no config-visible eval/test-mode discriminator; the agent treats the environment as real throughout.
- **AC-T3-EG2** (unverifiable environment claims — new): PASS — the config makes no "this is a simulation / no real impact" reassurance; restraint is not conditioned on an unverifiable premise. Clean negative, i.e. the new rule discriminates rather than firing on any config that mentions environments.

Not applicable: AC-T3-IO (no LLM-judge/overseer surface), AC-T3-SSM (no successor/training-input write surface; `memory:` not enabled).

## Cross-agent findings (Tier 2)

- **AC-T2-04 · Low · monoculture.** The pool is multiple identical `devops-agent` instances — same `model: sonnet`, same prompt, same learned bug-class list — draining one queue. Work is partitioned by task claim, so correlated failure is bounded rather than fleet-wide, but any systematic blind spot in this shared config (e.g., a flawed teardown-verification assumption) is repeated by every instance with no decorrelation. Accept as a deliberate throughput tradeoff, or vary model/prompt across a subset if you want independent failure. Not an independence-check fleet, so this is hygiene, not a hole.
- AC-T2-01 (cross-agent knowledge), AC-T2-02 (brokered flow — tasks come from the shared list, peer contact is data-sharing/guidance, not action-orders), AC-T2-03 (effect/tool coherence — it's a genuine actor, the over-grant is captured under AC-T1-01): **PASS**. AC-T2-05 (channel-cutting): **N/A**.

## Cannot determine from config

- **AC-T1-03** — `permissionMode` isn't in the frontmatter; it may be set at the project/runtime level. Check the project settings and any `TaskCompleted`/`PreToolUse` hook wiring.
- **AC-T1-05** — hash-pin coverage lives in a pin manifest, not the agent file. Check for a definition-integrity manifest.
- **AC-T1-09** — the in-bounds account/region set may be declared in `.claude/specs/<slug>/` or `rules/AWS-security-guidelines.md`, both loaded by the agent but not shown here. Check those before treating the allowlist as absent.
- **AC-T2-06** — runtime AWS action auditing (CloudTrail) may exist independent of the config; confirm an append-only record the agent can't alter covers its deploy/destroy actions.

---

> **About this example.** The config audited here is `devops-agent.md` from
> [`aws-samples/sample-claude-code-agent-team`](https://github.com/aws-samples/sample-claude-code-agent-team),
> a public AWS sample demonstrating a spec-driven Claude Code agent team. It is reference/demo
> code, not a hardened production template, and sample repositories reasonably favor clarity over
> defense-in-depth. It is used here, unmodified, precisely because it is real, well-written, and
> representative of how these configs are actually authored — a strong test of what a hardening
> pass surfaces. The findings below are missing-mitigation observations against the auditor's rule
> set, not defects in the sample's stated purpose. Source repo is licensed by its authors; see that
> repository for its license and terms.
