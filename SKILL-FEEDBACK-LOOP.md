# Skill Feedback Loop — Continuous Improvement Protocol

**Version:** 1.0
**Created:** 2026-04-04

Skills that don't learn from failures are opinions frozen in time. This protocol turns every compilation error, code review correction, and runtime failure into a permanent skill improvement.

---

## The Loop

```
Write Code → Compile → Fail? → Capture → Classify → Update Skill → Re-test
     ↑                                                        |
     +────────────────────────────────────────────────────────+
```

---

## 1. Failure Capture (Automatic)

Every time an AI agent writes Rust code and it fails to compile or gets corrected by the user, the failure is captured in a **failure log**.

### Location
```
~/.claude/skill-forge/rust/failures/
  YYYY-MM-DD-{short-description}.md
```

### Failure Report Format
```markdown
---
skill: mx-rust-async
severity: compilation-error | runtime-bug | user-correction | code-review
agent: claude | codex | gemini
date: 2026-04-05
---

## What Happened
[One paragraph describing the failure]

## Code That Failed
```rust
[The exact code that was wrong]
```

## Error Message
```
[Compiler output or user feedback]
```

## Root Cause
[Which skill rule was violated, missing, or insufficient?]

## Fix Applied
```rust
[The corrected code]
```

## Skill Gap
[What rule should be ADDED or MODIFIED in which SKILL.md?]

## Classification
- [ ] Missing rule (skill doesn't cover this case)
- [ ] Weak rule (skill covers it but AI rationalized around it)
- [ ] Wrong rule (skill gives bad advice)
- [ ] Missing anti-rationalization (AI found a loophole)
- [ ] Missing code example (rule is abstract, needs concrete BAD/GOOD pair)
```

### When to Capture
- **Compilation failure** that required >1 iteration to fix
- **User says "no, not like that"** — a correction means the skill failed
- **Code review rejection** — the pattern wasn't idiomatic
- **Runtime bug** traced back to a pattern the skill should have prevented
- **AI rationalized around a rule** — used .unwrap() "just this once"

---

## 2. Failure Classification

Failures fall into 5 categories, each with a different fix:

| Category | Symptom | Skill Fix |
|----------|---------|-----------|
| **Missing rule** | Skill doesn't mention this pattern at all | Add a new rule to the appropriate Level |
| **Weak rule** | Rule exists but AI ignored it or found a loophole | Strengthen with anti-rationalization rule |
| **Wrong rule** | Rule gives advice that produces bad code | Correct the rule, add regression test |
| **Missing example** | Rule is too abstract, AI doesn't know how to apply it | Add BAD/GOOD code pair |
| **Missing cross-reference** | AI used the wrong skill or didn't load the right one | Update "When to also load" section |

---

## 3. Skill Update Protocol

### When to Update
- **Immediately** if severity is `compilation-error` and it's a missing rule
- **Batch weekly** if severity is `user-correction` or `code-review`
- **Never update during active coding** — update between sessions

### How to Update

1. Edit the DRAFT in `~/.claude/skill-forge/rust/drafts/mx-rust-{skill}/SKILL.md`
2. Add/modify the rule
3. If adding an anti-rationalization rule, use the exact format:
   ```
   **You will be tempted to:** [exact rationalization]
   **Why that fails:** [concrete failure with evidence from the failure report]
   **The right way:** [pattern with code]
   ```
4. Promote to production: `cp drafts/mx-rust-{skill}/SKILL.md ~/.claude/skills/mx-rust-{skill}/SKILL.md`
5. Test in fresh session

### Version Tracking

Each SKILL.md gets a version comment at the bottom:

```markdown
<!-- Skill Version: 1.3 | Last Updated: 2026-04-10 | Failures Addressed: 7 -->
```

---

## 4. Failure Metrics Dashboard

Maintain a running scorecard at:
```
~/.claude/skill-forge/rust/SCORECARD.md
```

### Format
```markdown
# Rust Skills Scorecard

## Overall
| Metric | Value | Target |
|--------|-------|--------|
| Total tests | 0 | — |
| First-try compile rate | —% | >80% |
| Avg iterations to compile | — | <2 |
| Skills updated from failures | 0 | — |

## Per-Skill Failure Count
| Skill | Failures | Last Failure | Top Issue |
|-------|----------|-------------|-----------|
| mx-rust-core | 0 | — | — |
| mx-rust-async | 0 | — | — |
| mx-rust-systems | 0 | — | — |
| mx-rust-network | 0 | — | — |
| mx-rust-data | 0 | — | — |
| mx-rust-services | 0 | — | — |
| mx-rust-testing | 0 | — | — |
| mx-rust-project | 0 | — | — |

## Per-Agent Performance
| Agent | Tests | First-Try Rate | Common Failure |
|-------|-------|---------------|----------------|
| Claude | 0 | — | — |
| Codex | 0 | — | — |
| Gemini | 0 | — | — |
```

---

## 5. Integration with Existing Infrastructure

### With /crystallize
When a failure pattern repeats 3+ times across sessions, invoke `/crystallize` to extract it into a permanent enforcement rule. The crystallized rule goes into the skill's `enforcement.md` file.

### With /reality-check
After writing Rust code, `/reality-check` should verify against the mx-rust-* skills before declaring done. If it finds violations, those become failure reports.

### With hookify
Consider a PostToolUse hook that auto-checks: "Did the Rust code just written violate any mx-rust-* rules?" This is the automated version of the feedback loop.

**Example hookify rule concept:**
```
After any Write/Edit to a .rs file:
- Check for .unwrap() in non-test code → warn
- Check for unbounded channels → warn  
- Check for std::thread::sleep in async context → warn
```

### With inter-agent protocol
When Codex or Gemini produce Rust code that fails, the failure report should note which agent produced it. Different agents may need different skill emphasis.

---

## 6. The Quarterly Review

Every 3 months (or after 50+ failure reports), do a full review:

1. **Read all failure reports** sorted by frequency
2. **Identify systemic gaps** — are multiple failures pointing to the same missing concept?
3. **Consider splitting skills** — if one skill has 10x more failures than others, it might need to split
4. **Consider merging skills** — if two skills have near-zero failures, they might be over-segmented
5. **Re-run deep research** on the top 3 failure areas to see if the ecosystem has new solutions
6. **Update the Skills Factory Framework** with lessons learned

---

## 7. The Virtuous Cycle

```
Session 1: Write Rust → 60% first-try compile rate
           → Capture 5 failures → Update 3 skills

Session 5: Write Rust → 75% first-try compile rate  
           → Capture 2 failures → Update 1 skill

Session 20: Write Rust → 90% first-try compile rate
            → Capture 0 failures → Skills are mature

Session 50: New Rust ecosystem change (tokio 2.0?)
            → Spike in failures → Re-research → Update skills → Back to 90%
```

The skills are never "done." They're a living system that gets sharper with every use.

---

## Quick Reference: What To Do When...

| Situation | Action |
|-----------|--------|
| Code doesn't compile | Write failure report. Fix code. Identify skill gap. |
| User corrects AI approach | Write failure report as `user-correction`. Update rule. |
| Same error happens twice | Escalate: add anti-rationalization rule. |
| Same error happens 3+ times | `/crystallize` into enforcement rule. |
| New Rust crate becomes standard | Run quicksearches. Update relevant skill. |
| Major Rust version release | Re-run deep research on affected modes. |
| New AI agent added to team | Run pressure test with new agent. Capture agent-specific failures. |
