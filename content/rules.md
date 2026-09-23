---
title: "Rules"
summary: "Policy as code — versioned, diffable, reviewable, and unit-testable before it ever gates a real PR."
---

Because the policy is a file in your repo, it is versioned, diffable, reviewable, and
unit-testable with `.tln.test` before it ever gates a real PR. That is a claim no
LLM-based reviewer can make.

## A ruleset is `define` + `rule`

`define` blocks are named predicates. `rule` blocks match records against facts and
predicates, then declare a verdict (`allow` / `block` / `requires`) and the actions
to take (`do …`).

```talon
define "small_change" {
  attr "pr.lines_changed" < 50
  attr "pr.files_changed" < 5
}

define "critical_path" {
  attr "pr.changed_files" contains "internal/auth/"
    or attr "pr.changed_files" contains "billing/"
}

rule "Auto-approve safe changes" {
  for records where type == "pr"
    and is "small_change"
    and attr "pr.tests_passing" == true
    and attr "pr.has_description" == true
    and not is "critical_path"
  allow "merge"
  do approve "pr"
}

rule "Require human review for critical paths" {
  for records where type == "pr"
    and is "critical_path"
  requires "review.senior_engineer"
  do require "review.senior_engineer"
  do assign "pr" attr "user.owner"
  do comment "pr" "Touches critical code owned by {attr.user.owner} — human approval required"
}
```

## Facts a rule can match on

Rules match against facts extracted per PR. A few of the families:

- **`pr.*`** — `lines_changed`, `files_changed`, `changed_files`, `tests_passing`,
  `lint_passing`, `has_description`, `draft`, `is_fork`, `labels`, `diff`, and more.
- **`user.*`** — `owner` (CODEOWNERS, else last toucher), `owners`, `last_toucher`.
  Ownership is a *fact*, so "the author owns the code they changed — get an
  independent reviewer" becomes trivially expressible.
- **`review.*`** — folded from the PR's full review history: `human.approved`,
  `changes_requested`, `<team>.approved`, `<team>.stale`.
- **`module.*`** — the primary touched module (by changed lines), its owner and docs.
- **`llm.*` / `qa.*`** — verdicts a rule asked for, re-entering the engine as facts.
  See [How it works](/how-it-works/) and [Automated QA](/automated-qa/).

Unset facts fail closed: a rule gated on a fact that wasn't asserted simply doesn't
fire — the safe direction.

## Tested before it gates anything

A ruleset ships with a `.tln.test` file. You assert the verdict a set of facts should
produce, and CI fails if the policy drifts — the same discipline you'd apply to any
other code.

```bash
talooner rules validate .github/talooner/
talooner rules test     .github/talooner/
```
