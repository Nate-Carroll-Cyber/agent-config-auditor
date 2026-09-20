# Hardened agent template — usage
`_TEMPLATE.md` is a starting skeleton for any new detection or effects subagent in this hierarchy.
It bakes in the controls the agent-config-auditor kept flagging, so a new agent starts compliant
instead of accruing the same findings one audit at a time.
## Why this exists
Auditing the first agents surfaced the same two gaps on every one — a missing untrusted-input
barrier (AC-T1-02) and an unbound advisory judgment (AC-T3-RH) — plus the standing GUID placeholder
(AC-T1-05). Those are systemic to how the set was drafted, not per-agent bugs. The template
pre-writes the fixes so they don't recur.
## How to use it
1. Copy `_TEMPLATE.md` to `your-agent-name.md`.
2. Replace every `REPLACE-WITH-...` placeholder, including the GUID (`uuidgen`).
3. Delete the by-type control blocks at the bottom that don't apply; keep the ones that do.
4. Delete the inline `[AC-...]` annotations once filled — they're scaffolding, not shipping text
   (or keep them; they're HTML comments and won't render in the prompt).
5. Set `tools:` to the minimal set. Leave it `Read, Grep, Glob` unless the agent genuinely needs
   more — and if it's an effects agent, keep it read-only/propose-only until a human-approval step
   and a PreToolUse hook exist.
6. Hash-pin the new file and verify it before use (AC-T1-05).
## What's baked in (and the rule each satisfies)
- Minimal `tools:` as the capability boundary — AC-T1-01 / AC-T2-03
- Untrusted-input barrier for anything the agent reads — AC-T1-02
- GUID identity placeholder + pin reminder — AC-T1-05
- External-manifest-not-self-memory guidance — AC-T1-06 / AC-T1-07
- Scope-limit section — AC-T1-04
- In-bounds target/environment set for target-selecting or environment-acting agents — AC-T1-09
- Star-topology reporting + prohibitive-only sibling naming — AC-T2-01 / AC-T2-02
- Advisory-not-a-trigger output framing — AC-T3-RH
- Prohibitions that hold unconditionally, not contingent on a sim/test belief — AC-T3-EG2
- Extended untrusted-artifact list (issue bodies, code comments, package metadata, caches) — AC-T1-02
- Declared memory purpose and entry shape — AC-T1-07
- Self-edit prohibition plus the frontmatter note on where the hard control lives — AC-T1-10
- Task-anchored objective with a permitted way to fail — AC-T3-RH2
- No grader or monitor mechanics in the prompt — AC-T3-RH3
- By-type control blocks for effects/assertion/judge/disruption/successor/result-producing/
  orchestrator/outward-facing/human-facing/fleet agents, plus the stop-control note —
  AC-T1-03, AC-T2-05, AC-T2-06, AC-T3-DC, AC-T3-DC2, AC-T3-IO, AC-T3-IO2, AC-T3-SRS, AC-T3-SSM,
  AC-T3-SSM2, AC-T3-IAM

Three of these cannot be satisfied from inside the agent file. AC-T1-10 path denials, the
AC-T3-SSM2 stop control, and AC-T3-IAM path isolation live in project settings, hooks, and the
orchestrator. The template carries the reminders; the audit will return CANNOT-DETERMINE for them
until you point it at where those controls are defined.
## Closing the loop
After filling the template, run the agent-config-auditor skill against the new file. A
template-derived agent should come back with no High/Medium findings except any you introduced —
which is the point: the audit becomes a confirmation step, not a rework cycle. When a catalog rule
changes, update `_TEMPLATE.md` in the same commit — the template encodes the same rule set and
drifts silently out of compliance if only the catalog is edited.
