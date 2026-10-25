---
name: grilling
description: "Stress-test an EXISTING plan, decision, or idea: rounds of interview questions over the decision tree (frontier). Invoke ONLY on an explicit ask («погрили меня», «проверь план», «grill this») or when brainstorming offers it after approval. NOT for starting new work - that is brainstorming."
---

Interview the user relentlessly until you reach a shared understanding. Map this as a **design tree**: every decision branches into the decisions that hang off it.

Work the tree in **rounds**. The **frontier** is every decision whose prerequisites are already settled: the questions you can ask _now_ without guessing at answers you haven't heard yet. Ask the whole frontier in one round: number each question and give your recommended answer. Then wait for the user's answers before the next round.

Format a round like so:

```
❓ **Q1** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>

---

❓ **Q2** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>
```

Word each question so "yes" accepts your recommended answer.

Each round the user answers reshapes the tree: settled decisions push the frontier outward and unblock questions that depended on them. Recompute the frontier and ask the next round. A question whose answer depends on another question still open in this round belongs to a _later_ round, not this one.

Finding _facts_ is your job, never the user's. When a frontier question needs a fact from the environment (filesystem, tools, etc.), dispatch a sub-agent to find it (multi-profile setup: a kanban task for the orchestrator profile); don't ask the user for anything you could look up yourself. Don't block on it: a running exploration is an unsettled prerequisite, so only the questions downstream of it wait for the sub-agent to report; ask the rest of the frontier now. The _decisions_ are the user's: put each to them and wait.

**A "fixed" number is anchored to an assumption — probe the anchor, not the value.** When the user confirms a threshold or rating, it was almost always chosen relative to an unstated premise (a battery cutoff voltage, a component's real rating, an environment assumption). A shutdown threshold of 11.0 V picked for a 10 V cutoff breaks the moment the real cutoff is revealed as ~10.5 V, so ask "what is this value relative to?" whenever the user hands you a constant — otherwise the "confirmed" number silently carries the wrong premise into the architecture.

The session is done when the frontier is empty: every branch of the design tree visited, nothing left silently assumed. Do not act on it until the user confirms you have reached a shared understanding.
