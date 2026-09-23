+++
date = '2026-09-23'
draft = false
title = 'Introducing Talooner: rules decide, not the model'
summary = "Why we built a PR reviewer where the verdict comes from your repo's rules, and a model is consulted only where a rule asks for it."
tags = ['talooner', 'code-review', 'tln']
+++

Ask an LLM to review a diff and you get something that reads well and means little.
It doesn't know your team's rules, it's confident when it shouldn't be, and it gives
you a different answer on the same diff tomorrow. Real review is company-specific, and
a bare prompt can't encode that.

Talooner takes the opposite stance: **make the policy explicit, and keep the decision
out of the model's hands.**

## Policy as code

A repository declares its review policy as [`tln`](https://tln-lang.org) rules — a
file in `.github/talooner/`. Talooner extracts the PR's facts, runs them through the
inference engine, and executes whatever actions the rules produce. Approvals, blocks,
comments, review requests, assignees — all decided by rules you can read, diff, and
unit-test.

```talon
rule "Auto-approve safe changes" {
  for records where type == "pr"
    and is "small_change"
    and attr "pr.tests_passing" == true
    and not is "critical_path"
  allow "merge"
  do approve "pr"
}
```

No model decided that. The engine did, deterministically. Same commit, same rules,
same verdict — every time.

## Where a model *does* fit

A model is consulted only where a rule says so — `do llm_review` — and its answer
comes back **typed**, not as prose. Paired with a model like
[TypeSafe / Jev](https://typesafe.ai) that returns a structured decision plus
calibrated confidence, a rule can act on a confident verdict and escalate an unsure
one to a human. The model informs the engine; it never overrides it.

That same shape is what makes [Automated QA](/automated-qa/) possible: rules decide a
PR is worth deploying, a model generates the test scenarios, the scenarios run, and
the pass/fail results re-enter the engine as facts. The model proposes what to test —
the rules decide what the results mean.

## Self-hosted, permanently

You run the cluster. Your model credentials live there and never touch the CI runner,
so every token a rule spends is billed to whoever ran the rule. There's no hosted
tier and no plan for one.

## Where it stands

Early v1. The walking skeleton runs end to end — facts in, rules evaluated, review
actions executed on the PR. The `llm_review` path and Automated QA are the design
we're building toward. If a deterministic reviewer sounds like the right shape for
your team, the [code is on GitHub](https://github.com/opentalon/talooner).
