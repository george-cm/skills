---
name: domain-modeling
description: Use when pinning down domain terminology, establishing a shared language, or recording architectural decisions. Trigger on "domain model", "CONTEXT.md", "what do we call this", "term conflicts", "glossary", "ADR", "ubiquitous language", "fuzzy language", "define this concept".
---

# Domain Modeling

## Overview

Maintain a living glossary of precise domain terms (`CONTEXT.md`) and a record of hard
architectural decisions (`docs/adr/`). Capture them inline as they crystallise - not in batch afterward.

**Core principle:** Merely reading `CONTEXT.md` for vocabulary is not this skill. This skill
is for when you are changing the model, not just consuming it.

## When to Use

- Starting a new project or feature area with unclear terminology
- A term is being used inconsistently or two people mean different things by it
- An architectural decision is being made that is hard to reverse
- Another skill calls for domain model maintenance

## When NOT to Use

- Just reading `CONTEXT.md` to pick up vocabulary - one-line habit, not this skill
- Documenting implementation details or technical specs - `CONTEXT.md` is a glossary only
- Recording every small decision - ADRs only for hard-to-reverse, surprising, real trade-offs

## Quick Reference

| Action | When | Key rule |
|--------|------|----------|
| Update `CONTEXT.md` | Term resolved in conversation | Inline, not batched |
| Challenge a term | Conflicts with existing glossary | Call it out immediately |
| Sharpen fuzzy language | Vague or overloaded term | Propose a single canonical term |
| Create ADR | Hard to reverse + surprising + real trade-off | All 3 required |
| Skip ADR | Easy to reverse, or obvious | Don't document the obvious |

## Common Mistakes

- **Treating `CONTEXT.md` as a spec or decision log** - it is a glossary and nothing else
- **ADRs for easy-to-reverse decisions** - if you can just change it, don't file it
- **Batching term updates** - capture the moment they crystallise, not at end of session
- **Not challenging conflicting terminology** - let it slide once, it compounds
- **General programming terms in `CONTEXT.md`** - only domain-specific concepts belong

---

Actively build and sharpen the project's domain model as you design. This is the *active* discipline - challenging terms, inventing edge-case scenarios, and writing the glossary and decisions down the moment they crystallise. (Merely *reading* `CONTEXT.md` for vocabulary is not this skill - that's a one-line habit any skill can do. This skill is for when you're changing the model, not just consuming it.)

## File structure

Most repos have a single context:

```
/
├── CONTEXT.md
├── docs/
│   └── adr/
│       ├── 0001-event-sourced-orders.md
│       └── 0002-postgres-for-write-model.md
└── src/
```

If a `CONTEXT-MAP.md` exists at the root, the repo has multiple contexts. The map points to where each one lives:

```
/
├── CONTEXT-MAP.md
├── docs/
│   └── adr/                          ← system-wide decisions
├── src/
│   ├── ordering/
│   │   ├── CONTEXT.md
│   │   └── docs/adr/                 ← context-specific decisions
│   └── billing/
│       ├── CONTEXT.md
│       └── docs/adr/
```

Create files lazily  -  only when you have something to write. If no `CONTEXT.md` exists, create one when the first term is resolved. If no `docs/adr/` exists, create it when the first ADR is needed.

## During the session

### Challenge against the glossary

When the user uses a term that conflicts with the existing language in `CONTEXT.md`, call it out immediately. "Your glossary defines 'cancellation' as X, but you seem to mean Y  -  which is it?"

### Sharpen fuzzy language

When the user uses vague or overloaded terms, propose a precise canonical term. "You're saying 'account'  -  do you mean the Customer or the User? Those are different things."

### Discuss concrete scenarios

When domain relationships are being discussed, stress-test them with specific scenarios. Invent scenarios that probe edge cases and force the user to be precise about the boundaries between concepts.

### Cross-reference with code

When the user states how something works, check whether the code agrees. If you find a contradiction, surface it: "Your code cancels entire Orders, but you just said partial cancellation is possible  -  which is right?"

### Update CONTEXT.md inline

When a term is resolved, update `CONTEXT.md` right there. Don't batch these up  -  capture them as they happen. Use the format in [CONTEXT-FORMAT.md](./CONTEXT-FORMAT.md).

`CONTEXT.md` should be totally devoid of implementation details. Do not treat `CONTEXT.md` as a spec, a scratch pad, or a repository for implementation decisions. It is a glossary and nothing else.

### Offer ADRs sparingly

Only offer to create an ADR when all three are true:

1. **Hard to reverse**  -  the cost of changing your mind later is meaningful
2. **Surprising without context**  -  a future reader will wonder "why did they do it this way?"
3. **The result of a real trade-off**  -  there were genuine alternatives and you picked one for specific reasons

If any of the three is missing, skip the ADR. Use the format in [ADR-FORMAT.md](./ADR-FORMAT.md).
