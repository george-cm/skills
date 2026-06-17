---
name: diagnosing-bugs
description: Use when debugging hard bugs, intermittent failures, or performance regressions. Trigger on "diagnose", "debug this", "something is broken", "this throws", "failing", "slow", "can't reproduce", "flaky".
---

# Diagnosing Bugs

## Overview

Build a tight, red-capable feedback loop before doing anything else. Every other phase
consumes it. Without one, no amount of reading code or generating hypotheses will reliably
find the bug.

**Core principle:** If you catch yourself reading code to build a theory before the loop
exists, stop. That is the exact failure this skill prevents.

## When to Use

- Hard bugs with no obvious cause visible in the diff
- Intermittent or flaky failures
- Performance regressions
- Bugs that resist casual inspection

## When NOT to Use

- Obvious bug visible directly in the diff - just fix it, skip the phases
- Already have a red-capable loop - jump straight to Phase 3
- Mid-incident requiring immediate action - stabilise first (use thinking-ooda), diagnose after

## Quick Reference

| Phase | Goal | Gate to next phase |
|-------|------|--------------------|
| 1 - Feedback loop | Build tight, red-capable command | Named command; already run; goes red on this bug |
| 2 - Reproduce + minimise | Watch it fail; shrink to smallest red repro | Every element load-bearing |
| 3 - Hypothesise | 3-5 ranked, falsifiable hypotheses | List shown to user |
| 4 - Instrument | One variable at a time per hypothesis | Hypothesis confirmed or ruled out |
| 5 - Fix | Regression test first (correct seam only) | Loop goes green on original scenario |
| 6 - Cleanup | Remove debug artifacts, post-mortem | All checklist items cleared |

## Common Mistakes

- **Skipping Phase 1** - jumping to hypothesis before a red-capable loop exists. No loop, no Phase 2.
- **Accepting a flaky loop** - a 30-second flaky loop is barely better than no loop. Tighten it.
- **Minimising too little** - leaving non-load-bearing elements in the repro bloats the hypothesis space.
- **"Log everything and grep"** - explicitly banned. Tag targeted logs `[DEBUG-xxxx]`, grep the prefix at cleanup.
- **Fixing the wrong bug** - loop must assert the user's exact symptom, not a nearby failure that happens to go red.

---

A discipline for hard bugs. Skip phases only when explicitly justified.

When exploring the codebase, read `CONTEXT.md` (if it exists) to get a clear mental model of the relevant modules, and check ADRs in the area you're touching.

## Phase 1  -  Build a feedback loop

**This is the skill.** Everything else is mechanical. If you have a **tight** pass/fail signal for the bug  -  one that goes red on _this_ bug  -  you will find the cause; bisection, hypothesis-testing, and instrumentation all just consume it. If you don't have one, no amount of staring at code will save you.

Spend disproportionate effort here. **Be aggressive. Be creative. Refuse to give up.**

### Ways to construct one  -  try them in roughly this order

1. **Failing test** at whatever seam reaches the bug  -  unit, integration, e2e.
2. **Curl / HTTP script** against a running dev server.
3. **CLI invocation** with a fixture input, diffing stdout against a known-good snapshot.
4. **Headless browser script** (Playwright / Puppeteer)  -  drives the UI, asserts on DOM/console/network.
5. **Replay a captured trace.** Save a real network request / payload / event log to disk; replay it through the code path in isolation.
6. **Throwaway harness.** Spin up a minimal subset of the system (one service, mocked deps) that exercises the bug code path with a single function call.
7. **Property / fuzz loop.** If the bug is "sometimes wrong output", run 1000 random inputs and look for the failure mode.
8. **Bisection harness.** If the bug appeared between two known states (commit, dataset, version), automate "boot at state X, check, repeat" so you can `git bisect run` it.
9. **Differential loop.** Run the same input through old-version vs new-version (or two configs) and diff outputs.
10. **HITL bash script.** Last resort. If a human must click, drive _them_ with `scripts/hitl-loop.template.sh` so the loop is still structured. Captured output feeds back to you.

Build the right feedback loop, and the bug is 90% fixed.

### Tighten the loop

Treat the loop as a product. Once you have _a_ loop, **tighten** it:

- Can I make it faster? (Cache setup, skip unrelated init, narrow the test scope.)
- Can I make the signal sharper? (Assert on the specific symptom, not "didn't crash".)
- Can I make it more deterministic? (Pin time, seed RNG, isolate filesystem, freeze network.)

A 30-second flaky loop is barely better than no loop; a 2-second deterministic one is tight  -  a debugging superpower.

### Non-deterministic bugs

The goal is not a clean repro but a **higher reproduction rate**. Loop the trigger 100×, parallelise, add stress, narrow timing windows, inject sleeps. A 50%-flake bug is debuggable; 1% is not  -  keep raising the rate until it's debuggable.

### When you genuinely cannot build a loop

Stop and say so explicitly. List what you tried. Ask the user for: (a) access to whatever environment reproduces it, (b) a captured artifact (HAR file, log dump, core dump, screen recording with timestamps), or (c) permission to add temporary production instrumentation. Do **not** proceed to hypothesise without a loop.

### Completion criterion  -  a tight loop that goes red

Phase 1 is done when the loop is **tight** and **red-capable**: you can name **one command**  -  a script path, a test invocation, a curl  -  that you have **already run at least once** (paste the invocation and its output), and that is:

- [ ] **Red-capable**  -  it drives the actual bug code path and asserts the **user's exact symptom**, so it can go red on this bug and green once fixed. Not "runs without erroring"  -  it must be able to _catch this specific bug_.
- [ ] **Deterministic**  -  same verdict every run (flaky bugs: a pinned, high reproduction rate, per above).
- [ ] **Fast**  -  seconds, not minutes.
- [ ] **Agent-runnable**  -  you can run it unattended; a human in the loop only via `scripts/hitl-loop.template.sh`.

If you catch yourself reading code to build a theory before this command exists, **stop  -  jumping straight to a hypothesis is the exact failure this skill prevents.** No red-capable command, no Phase 2.

## Phase 2  -  Reproduce + minimise

Run the loop. Watch it go red  -  the bug appears.

Confirm:

- [ ] The loop produces the failure mode the **user** described  -  not a different failure that happens to be nearby. Wrong bug = wrong fix.
- [ ] The failure is reproducible across multiple runs (or, for non-deterministic bugs, reproducible at a high enough rate to debug against).
- [ ] You have captured the exact symptom (error message, wrong output, slow timing) so later phases can verify the fix actually addresses it.

### Minimise

Once it's red, shrink the repro to the **smallest scenario that still goes red**. Cut inputs, callers, config, data, and steps **one at a time**, re-running the loop after each cut  -  keep only what's load-bearing for the failure.

Why bother: a minimal repro shrinks the hypothesis space in Phase 3 (fewer moving parts left to suspect) and becomes the clean regression test in Phase 5.

Done when **every remaining element is load-bearing**  -  removing any one of them makes the loop go green.

Do not proceed until you have reproduced **and** minimised.

## Phase 3  -  Hypothesise

Generate **3-5 ranked hypotheses** before testing any of them. Single-hypothesis generation anchors on the first plausible idea.

Each hypothesis must be **falsifiable**: state the prediction it makes.

> Format: "If <X> is the cause, then <changing Y> will make the bug disappear / <changing Z> will make it worse."

If you cannot state the prediction, the hypothesis is a vibe  -  discard or sharpen it.

**Show the ranked list to the user before testing.** They often have domain knowledge that re-ranks instantly ("we just deployed a change to #3"), or know hypotheses they've already ruled out. Cheap checkpoint, big time saver. Don't block on it  -  proceed with your ranking if the user is AFK.

## Phase 4  -  Instrument

Each probe must map to a specific prediction from Phase 3. **Change one variable at a time.**

Tool preference:

1. **Debugger / REPL inspection** if the env supports it. One breakpoint beats ten logs.
2. **Targeted logs** at the boundaries that distinguish hypotheses.
3. Never "log everything and grep".

**Tag every debug log** with a unique prefix, e.g. `[DEBUG-a4f2]`. Cleanup at the end becomes a single grep. Untagged logs survive; tagged logs die.

**Perf branch.** For performance regressions, logs are usually wrong. Instead: establish a baseline measurement (timing harness, `performance.now()`, profiler, query plan), then bisect. Measure first, fix second.

## Phase 5  -  Fix + regression test

Write the regression test **before the fix**  -  but only if there is a **correct seam** for it.

A correct seam is one where the test exercises the **real bug pattern** as it occurs at the call site. If the only available seam is too shallow (single-caller test when the bug needs multiple callers, unit test that can't replicate the chain that triggered the bug), a regression test there gives false confidence.

**If no correct seam exists, that itself is the finding.** Note it. The codebase architecture is preventing the bug from being locked down. Flag this for the next phase.

If a correct seam exists:

1. Turn the minimised repro into a failing test at that seam.
2. Watch it fail.
3. Apply the fix.
4. Watch it pass.
5. Re-run the Phase 1 feedback loop against the original (un-minimised) scenario.

## Phase 6  -  Cleanup + post-mortem

Required before declaring done:

- [ ] Original repro no longer reproduces (re-run the Phase 1 loop)
- [ ] Regression test passes (or absence of seam is documented)
- [ ] All `[DEBUG-...]` instrumentation removed (`grep` the prefix)
- [ ] Throwaway prototypes deleted (or moved to a clearly-marked debug location)
- [ ] The hypothesis that turned out correct is stated in the commit / PR message  -  so the next debugger learns

**Then ask: what would have prevented this bug?** If the answer involves architectural change (no good test seam, tangled callers, hidden coupling) hand off to the `/improve-codebase-architecture` skill with the specifics. Make the recommendation **after** the fix is in, not before  -  you have more information now than when you started.
