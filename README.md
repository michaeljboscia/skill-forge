# 🔨 Skill Forge

**A repeatable system for building AI coding-skill packages that actually change how an agent writes code.**

Not a library of skills. The *factory* that produces them.

---

## The problem this solves

You give your AI coding agent a "skill" — a big markdown file full of best practices — and it nods politely, then writes the exact code you told it not to.

That's not the agent being dumb. It's the skill being built wrong. After **months of failed attempts**, one package (Rust, 8 skills, zero-to-deployed in a single session) finally worked. Skill Forge is that process reverse-engineered from the one success, so it's repeatable instead of accidental.

## Why the previous attempts failed (so yours don't have to)

Each of these is a heresy. Name it, purge it, move on.

| The mistake | What actually happened |
|---|---|
| **One giant SKILL.md** | 2,000+ lines. The model got "lost in the middle" and ignored the rules buried in the center. |
| **No research phase** | Rules written from general knowledge — generic, uncited, blind to ecosystem-specific footguns. |
| **No anti-rationalization rules** | The skill said what *to* do but not what the model would be *tempted* to do wrong. The model found the loopholes. Every time. |
| **No difficulty layering** | Beginner and expert patterns mixed together. The model couldn't tell "always do this" from "maybe, in advanced cases." |
| **No provenance** | Rules without citations are opinions. Cited rules are verifiable. Models trust verifiable. |

## The 7-phase pipeline

The whole framework lives in **[`SKILLS-FACTORY-FRAMEWORK.md`](./SKILLS-FACTORY-FRAMEWORK.md)**. The short version:

| Phase | Time | What happens |
|---|---|---|
| **0 · Scope** | 30 min | Break the language/platform into 6–10 *modes of work*, each with a distinct AI failure mode. If two modes fail the same way, merge them. |
| **1 · Discovery** | 15 min | 3–5 searches for existing rule sets so you're not reinventing wheels (or repeating their mistakes). |
| **2 · Quicksearch saturation** | 2–3 hrs | 40–100 targeted searches. Yes, that many. This is where ecosystem-specific knowledge comes from. |
| **3 · Deep research** | 30–60 min | 7–10 deep-research prompts, run in parallel. |
| **4 · Writing** | 1–2 hrs | Synthesize into layered skills: *Level 1 always-works* → *Level 2 intermediate* → *Level 3 advanced*, plus Performance, Observability, and the secret sauce — **Enforcement: Anti-Rationalization Rules**. |
| **5 · Promotion** | 5 min | Copy the finished drafts into `~/.claude/skills/`. |
| **6 · Pressure test** | next session | A *fresh* agent tries to weasel out of the rules. Whatever loophole it finds becomes a new enforcement rule. |
| **7 · Reference population** | optional | Backfill citations and reference docs. |

## The one idea to steal even if you read nothing else

**Anti-rationalization rules.** Don't just tell the model the right thing to do. Name the *wrong* thing it's about to rationalize, and forbid that specifically:

> ❌ "Use proper error handling."
> ✅ "You will be tempted to `.unwrap()` because it compiles and the demo works. Do not. Every `.unwrap()` in non-test code is a panic waiting for production. Use `?` or handle the error."

The model can't argue with a rule that already predicted its excuse. Competence is the only credential — and competence, here, means closing the loopholes before the model finds them.

## What's in this repo

Pure system, no library:

- **[`SKILLS-FACTORY-FRAMEWORK.md`](./SKILLS-FACTORY-FRAMEWORK.md)** — the full 7-phase pipeline, the skill-file template, and the anti-pattern catalog
- **[`SKILL-FEEDBACK-LOOP.md`](./SKILL-FEEDBACK-LOOP.md)** — how skills keep improving after they ship
- **[`ROADMAP.md`](./ROADMAP.md)** — which language/platform packages are next
- **[`STRATEGIC-GAPS.md`](./STRATEGIC-GAPS.md)** — the honest list of what's still missing

## Who this is for

Anyone maintaining a fleet of AI coding agents who's tired of writing skills that get ignored. Model-agnostic in spirit; the templates lean toward Claude Code and the `mx-{lang}-{mode}` naming convention, but the method ports anywhere skills are markdown.

---

*A factory that builds the tools that build the code. It's recursion all the way down — and this is the base case.*
