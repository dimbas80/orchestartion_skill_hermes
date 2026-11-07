---
name: verification-before-completion
description: "Use when about to claim work is complete, fixed, or passing, before committing or creating PRs - requires running verification commands and confirming output before making any success claims; evidence before assertions always. Триггеры: «проверь что работает», «готово?», перед любым «done»/коммитом/PR. Iron Law: ни одного заявления о готовности без свежего доказательства."
---

# Verification Before Completion

## Overview

**Core principle:** Evidence before claims, always.

**Violating the letter of this rule is violating the spirit of this rule.**

## The Iron Law

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

If you haven't run the verification command in this message, you cannot claim it passes.

## The Gate Function

```
BEFORE claiming any status or expressing satisfaction:

1. IDENTIFY: What command proves this claim?
2. RUN: Execute the FULL command (fresh, complete)
3. READ: Full output, check exit code, count failures
4. VERIFY: Does output confirm the claim?
   - If NO: State actual status with evidence
   - If YES: State claim WITH evidence
5. ONLY THEN: Make the claim

Skip any step = lying, not verifying
```

## Common Failures

| Claim | Requires | Not Sufficient |
|-------|----------|----------------|
| Tests pass | Test command output: 0 failures | Previous run, "should pass" |
| Linter clean | Linter output: 0 errors | Partial check, extrapolation |
| Build succeeds | Build command: exit 0 | Linter passing, logs look good |
| Bug fixed | Test original symptom: passes | Code changed, assumed fixed |
| Regression test works | Red-green cycle verified | Test passes once |
| Agent completed | VCS diff shows changes | Agent reports "success" |
| Requirements met | Line-by-line checklist | Tests passing |
| Guard described as "executing" X | Cite the code path that runs X at the moment of verification | A test that only matches substrings of the probed source — describe it by its actual strength, not the strength you wish it had |

## Red Flags - STOP

- Using "should", "probably", "seems to"
- Expressing satisfaction before verification ("Great!", "Perfect!", "Done!", etc.)
- About to commit/push/PR without verification
- Trusting agent success reports
- Relying on partial verification
- Thinking "just this once"
- Tired and wanting work over
- **ANY wording implying success without having run verification**

## Rationalization Prevention

| Excuse | Reality |
|--------|---------|
| "Should work now" | RUN the verification |
| "I'm confident" | Confidence ≠ evidence |
| "Just this once" | No exceptions |
| "Linter passed" | Linter ≠ compiler |
| "Agent said success" | Verify independently |
| "Other files must be unchanged" | Diff them and name the drift |
| "Acceptance only greps file X" | The requirement is about behaviour; other paths enforce it too — grep them and report leftovers |
| "I'm tired" | Exhaustion ≠ excuse |
| "Partial check is enough" | Partial proves nothing |
| "Different words so rule doesn't apply" | Spirit over letter |

## Key Patterns

**Tests:**
```
✅ [Run test command] [See: 34/34 pass] "All tests pass"
❌ "Should pass now" / "Looks correct"
```

**Regression tests (TDD Red-Green):**
```
✅ Write → Run (pass) → Revert fix → Run (MUST FAIL) → Restore → Run (pass)
❌ "I've written a regression test" (without red-green verification)
```

**Build:**
```
✅ [Run build] [See: exit 0] "Build passes"
❌ "Linter passed" (linter doesn't check compilation)
```

**Requirements:**
```
✅ Re-read plan → Create checklist → Verify each → Report gaps or completion
❌ "Tests pass, phase complete"
```

**Agent delegation:**
```
✅ Agent reports success → Check VCS diff → Verify changes → Report actual state
❌ Trust agent report
```

## Static checkers (grep-based criteria)

A checker that "finds a trace of X in the code" can certify itself. Two rules:

1. **Search live code only.** Strip comments and any embedded snapshot/doc blocks
   (quoted requirements, generated examples, `original` sections) before matching.
   A mention inside an explanatory comment is not an implementation — and such a
   comment is exactly what a developer writes when the code is NOT there.
2. **One component, one check.** When a requirement has several parts
   (several values, files, branches), verify each part separately and name the
   missing ones in the output. A single match must not mark the whole row done.
   Dedupe first when the source format lists the same part twice (mirrored
   blocks), otherwise a correct implementation looks incomplete.

Prove the checker itself: a case that must be flagged (partial input → ✗) and a
case where the only trace sits in a comment or snapshot block (→ ✗). A checker
never shown to fail is not evidence that the work passes.

**Check what the checker measures, not what it is named.** A checker proves only
its own predicate: a bounds/lint/schema check that returns OK says nothing about
the property the requirement is written in ("values fit the screen" ≠ "values do
not overlap", "line exists" ≠ "line is correct"). Before trusting a green
result, state the predicate in one sentence and compare it with the requirement;
if they differ, the green run is not evidence — build the metric the requirement
actually names (parse the real units from the source of truth, not an estimate
quoted in the task) and report both numbers.

## Generated artifacts and filesystem-reading criteria

A generated artifact (map, index, inventory, report table) is only evidence if
it is stable. Before claiming a regenerated artifact was "reproduced":

1. **Run the generator twice and compare hashes of every produced file.** The
   second run must change nothing. A byte-identical second run proves your
   change, not the tree's churn.
2. **Split the diff into "rows I changed" and "rows that changed by".** Rows
   nobody touched in your work are external drift: the criterion read the
   filesystem (file presence, directory contents, timestamps) and some other
   process wrote into it. Label them with the input that caused them, in the
   report and on the shared board — do not claim them as your work and do not
   silently revert them.
3. **Never leave drift unexplained.** "Other sections unchanged" is a claim that
   needs a check, not an assumption: name every file whose diff is not yours and
   why.

## Same rule, other entry points

A limit/validation/formatting rule usually has more than one place that enforces
it: the on-device editor, the web/config form, the API import, the second
screen. Fixing one of them is not "the rule changed" — it is "one path changed".

1. After changing a rule, grep the whole tree for the field/constant it guards
   and list every place that clamps it.
2. Turn the list into a per-path check: each path fixed / each path still old.
3. A path you are not allowed to touch is a **finding with a file and line**, not
   the silence of a scoped acceptance criterion. Say plainly: the decision is
   implemented on path A and still contradicted on path B by `<file:line>`, so
   the owner or orchestrator must schedule it. Never report the change as
   complete for the device while another live path keeps the old cap.

## When To Apply

**ALWAYS before:**
- ANY variation of success/completion claims
- ANY expression of satisfaction
- ANY positive statement about work state
- Committing, PR creation, task completion
- Moving to next task
- Delegating to agents

**Rule applies to:**
- Exact phrases
- Paraphrases and synonyms
- Implications of success
- ANY communication suggesting completion/correctness
