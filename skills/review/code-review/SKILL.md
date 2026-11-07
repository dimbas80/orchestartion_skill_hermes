---
name: code-review
description: "Ревью диффа от фиксированной точки (commit/branch/tag/merge-base) по ДВУМ ОСЯМ: Standards (коду соответствует стандартам репо + база Фаулер-смеллов?) и Spec (код верно реализует спеку?). Оси гоняются параллельными субагентами, отчёты — рядом, без смешивания. Роль: reviewer. Триггеры: «ревью ветки», «review since X», «проверь дифф»."
---

Two-axis review of the diff between `HEAD` and a fixed point the user supplies:

- **Standards**: does the code conform to this repo's documented coding standards?
- **Spec**: does the code faithfully implement the originating issue / spec?

Both axes run as **parallel sub-agents** so they don't pollute each other's context, then this skill aggregates their findings.

The issue tracker and triage labels come from the project's tracker doc — `docs/agents/issue-tracker.md` (Local Markdown in `.scratch/`) and `docs/agents/triage-labels.md`. If the project has no tracker doc, ask the user where specs live before reviewing.

## Process

### 1. Pin the fixed point

Whatever the user said is the fixed point (a commit SHA, branch name, tag, `main`, `HEAD~5`, etc.). If they didn't specify one, ask for it.

Capture the diff command once: `git diff <fixed-point>...HEAD` (three-dot, so the comparison is against the merge-base). Also note the list of commits via `git log <fixed-point>..HEAD --oneline`.

Before going further, confirm the fixed point resolves (`git rev-parse <fixed-point>`) and the diff is non-empty. A bad ref or empty diff should fail here, not inside two parallel sub-agents.

### 2. Identify the spec source

Look for the originating spec, in this order:

1. Issue references in the commit messages (`#123`, `Closes #45`, GitLab `!67`, etc.), fetched via the workflow in the tracker doc.
2. A path the user passed as an argument.
3. A spec file under `docs/`, `specs/`, or `.scratch/` matching the branch name or feature.
4. If nothing is found, ask the user where the spec is. If they say there isn't one, the **Spec** sub-agent will skip and report "no spec available".

### 3. Identify the standards sources

Search the repo for every file that documents how code should be written. When `CODING_STANDARDS.md` or `CONTRIBUTING.md` exists, it must be on the list.

On top of whatever the repo documents, the Standards axis always carries the **smell baseline** below: a fixed set of Fowler code smells (_Refactoring_, ch.3) that applies even when a repo documents nothing. Two rules bind it:

- **The repo overrides.** A documented repo standard always wins; where it endorses something the baseline would flag, suppress the smell.
- **Always a judgement call.** Each smell is a labelled heuristic ("possible Feature Envy"), never a hard violation. Like any standard here, skip anything tooling already enforces.

Each smell reads *what it is* → *how to fix*; match it against the diff:

- **Mysterious Name**: a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code**: the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Feature Envy**: a method that reaches into another object's data more than its own. → move the method onto the data it envies.
- **Data Clumps**: the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession**: a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches**: the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery**: one logical change forces scattered edits across many files in the diff. → gather what changes together into one module.
- **Divergent Change**: one file or module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Speculative Generality**: abstraction, parameters, or hooks added for needs the spec doesn't have. → delete it; inline back until a real need shows.
- **Message Chains**: long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man**: a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest**: a subclass or implementer that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

### 4. Run both axes (Kanban mode primary, delegate_task fallback)

**Kanban mode (multi-profile setup):** the review arrives from the Orchestrator
as a kanban task for the reviewer profile. Run the two axes as two strictly
separate passes in this session (Standards first, then Spec); for a large diff,
as the Orchestrator for one review task per axis. Aggregate as in step 5. Never
merge the lenses mid-pass.

**Solo-session fallback:** one `delegate_task` call with both tasks runs them in
parallel; each child gets its prompt as `context`. If dispatch is unavailable,
run the two reviews yourself as two strictly separate passes.

**Standards sub-agent prompt** should include:

- The full diff command and commit list.
- The list of standards-source files you found in step 3, **plus the smell baseline from step 3** pasted in full (the sub-agent has no other access to it).
- The brief: "Report, per file/hunk where relevant, (a) every place the diff violates a documented standard: cite the standard (file + the rule); and (b) any baseline smell you spot: name it and quote the hunk. Distinguish hard violations from judgement calls: documented-standard breaches can be hard, but baseline smells are always judgement calls, and a documented repo standard overrides the baseline. Skip anything tooling enforces. Under 400 words."

**Spec sub-agent prompt** should include:

- The diff command and commit list.
- The path or fetched contents of the spec.
- The brief: "Report: (a) requirements the spec asked for that are missing or partial; (b) behaviour in the diff that wasn't asked for (scope creep); (c) requirements that look implemented but where the implementation looks wrong. Quote the spec line for each finding. Under 400 words."

If the spec is missing, skip the Spec sub-agent and note this in the final report.

### 5. Aggregate

Present the two reports under `## Standards` and `## Spec` headings, verbatim or lightly cleaned. Do **not** merge or rerank findings, because the two axes are deliberately separate (see _Why two axes_).

End with a one-line summary: total findings per axis, and the worst issue _within each axis_ (if any). Don't pick a single winner across axes: that's the reranking the separation exists to prevent.

## Why two axes

### Verifying detectors, criteria and coverage checks

When the deliverable is itself a check — a map/report generator criterion, a
linter rule, a coverage predicate, a "gap detector" — reading its code is not a
review. Verify it by **mutation on real inputs**: take the passing artifact,
remove or comment out the very line the criterion is supposed to see, and
require the criterion to flip to failing. Then re-add the trace in the two
places that must *not* count (a comment, a quoted/reference block) and require
it to stay failing. A criterion that only ever returns ✓ is indistinguishable
from one that always returns ✓; the ✗ case must be demonstrated, not assumed.

Two classes of defect this catches that reading never does:

- **the trace and the claim live at different granularity** — the predicate
  matches a value anywhere in the file, so a delta declared in one branch is
  "satisfied" by a delta in another branch;
- **the source of truth is narrower than the prose around it** — prose can
  promise an effect the underlying data/code never applies; a criterion built
  on the data will be truthfully silent about it, and that is a documentation
  finding, not an implementation bug.

### Verify artifact and visual invariants mechanically

- **Patch artifacts can be EOL-normalized.** When a per-batch diff will not apply against the working tree, do not fail the review on that alone: check whether it was produced in text mode (no `\r` in the patch while the target files are CRLF). Reconstruct the pre-change tree independently, take your own binary-safe diff, and compare content after EOL normalization. If the content matches, the code is verified and the artifact generation is a separate (usually LOW) finding.
- **"Layout / format unchanged" is a multiset assertion, not a code read.** Extract (anchor or column expression, string literal) pairs from the drawing/printing calls of the pre- and post-change versions, strip comments and trailing whitespace, and diff the multisets. Remaining differences are the real layout changes; strings that only padded a field before a new clear-and-redraw mechanism appeared will show up and are expected.
- **"No new warnings" is a subset test.** Compare warning identities (source file + message, with line:col stripped) of before and after; counts alone hide both fixed and newly introduced warnings. The same shape proves that unit/label literals survived a display refactor.

### Red-probe a new regression against the pre-fix baseline

A regression test that ships with the fix was written against the fixed tree; it
proves nothing until it has been shown to fail on the broken baseline. Run the
new/updated test file against a pristine copy of the pre-change files (export
the commit's versions of just the touched files into a scratch copy of the
tree — never mutate the repo) and require the failure set to be non-empty and
semantically on point. Record which tests stay green there: those are toothless
and need a sharper assertion. Two recurring shapes:

- the test asserts the *shape* the old code already had (a container is
  centred while its lines are not) and so passes on the bug it claims to guard;
- the test compares against a hard-coded constant taken from the fixed render
  instead of the measured geometry it should defend.

### Guards that scan text can be broken by prose

When a project guard greps/lints a *region* of source (a config block, a
section, a whole file) for forbidden literals, new explanatory comments inside
that region can trip it — the guard sees digits in prose as numbers. A full
suite that was green before the change and is red after is a blocking finding
even when the cause is only a comment: verify the causality (baseline green,
reworded prose green, as-is red) by mutation in a scratch copy, then require the
comment to be reworded or moved outside the scanned region.

Also sweep every row the criterion covers, not just the sample: recompute the
verdict independently and diff it against the published artifact, so the count
of ✓ and the count of "✓ with a missing component" are both known numbers.
When recomputing, mirror the artifact's *display rule* — which rows get a
verdict at all, and which side of a multi-verdict tuple (e.g. grant vs spend
per row) the published column shows. A harness that compares the wrong tuple
side, or expects verdicts where the criterion deliberately shows none, floods
the sweep with false mismatches and buries the one real finding: debug the
harness's own unpacking/display bugs before blaming the criterion.

A change can pass one axis and fail the other:

- Code that follows every standard but implements the wrong thing → **Standards pass, Spec fail.**
- Code that does exactly what the issue asked but breaks the project's conventions → **Spec pass, Standards fail.**

Reporting them separately stops one axis from masking the other.
