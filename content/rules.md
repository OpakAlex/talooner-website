---
title: "Rules"
summary: "Policy as code — versioned, diffable, reviewable, and unit-testable before it ever gates a real PR. A full worked ruleset, with its test suite."
---

Because the policy is a file in your repo, it is versioned, diffable, reviewable, and
unit-testable with `.tln.test` before it ever gates a real PR. That is a claim no
LLM-based reviewer can make.

## Anatomy of a ruleset

`define` blocks are named predicates. `rule` blocks match records against facts and
predicates, then declare a verdict (`allow` / `block` / `requires`) and the actions to
take (`do …`). Every rule can carry a `reason` and a `priority`.

```talon
define "small_change" {
  attr "pr.lines_changed" < 50
  attr "pr.files_changed" < 5
}

rule "Auto-approve safe changes" {
  for records where type == "pr"
    and is "small_change"
    and attr "pr.tests_passing" == true
    and attr "pr.has_description" == true
    and not is "critical_path"
  allow "merge"
  do approve "pr"
  priority MEDIUM
}
```

---

## A fuller policy

Predicates first, then rules grouped by what they do.

### Predicates

```talon
define "small_change"  { attr "pr.lines_changed" < 80  and attr "pr.files_changed" < 6 }
define "large_change"  { attr "pr.lines_changed" >= 400 }
define "huge_change"   { attr "pr.lines_changed" >= 1500 }

define "ready_for_review" {
  attr "pr.draft" == false
  attr "pr.has_description" == true
}

define "critical_path" {
  attr "pr.changed_files" contains "internal/auth/"
    or attr "pr.changed_files" contains "billing/"
}

define "pr.touches_migrations" { attr "pr.changed_files" contains "db/migrate/" }

define "pr.touches_design" {
  attr "pr.changed_files" contains "app/components/"
    or attr "pr.changed_files" ends_with ".scss"
    or attr "pr.changed_files" ends_with ".css"
    or attr "pr.changed_files" contains "design-tokens"
}
```

### Gates — block incomplete work

```talon
rule "Block PRs with no description" {
  for records where type == "pr"
    and attr "pr.draft" == false
    and attr "pr.has_description" == false
  block "merge"
  do block "pr.merge"
  do comment "pr" "No description. Say what changed and why before requesting review"
  reason "no description"
  priority HIGH
}

rule "Block PRs too large to review" {
  for records where type == "pr" and is "huge_change"
  block "merge"
  do block "pr.merge"
  do comment "pr" "{attr.pr.lines_changed} lines across {attr.pr.files_changed} files — split this before review"
  reason "unreviewably large"
  priority HIGH
}
```

### Auto-approve — narrow, green, low-risk

```talon
rule "Auto-approve safe changes" {
  for records where type == "pr"
    and is "small_change"
    and is "ready_for_review"
    and attr "pr.tests_passing" == true
    and attr "pr.lint_passing"  == true
    and not is "critical_path"
    and not is "pr.touches_design"
  allow "merge"
  do approve "pr"
  do comment "pr" "Small, tested, lint-clean, off critical paths — no objections"
  priority MEDIUM
}
```

### Ownership — escalate to the people who own the code

Ownership is a *fact* (`user.owner` resolves CODEOWNERS, else the last toucher), so
"the author owns the code they changed — get an independent reviewer" is trivially
expressible.

```talon
rule "Require human review for critical paths" {
  for records where type == "pr" and is "critical_path"
  requires "review.senior_engineer"
  do require "review.senior_engineer"
  do assign "pr" attr "user.owner"
  do comment "pr" "Touches critical code owned by {attr.user.owner} — human approval required"
  reason "touches a critical path"
  priority HIGH
}

rule "Escalate self-review of owned critical code" {
  for records where type == "pr"
    and is "critical_path"
    and attr "user.owner" == attr "pr.author"
  requires "review.senior_engineer"
  do require "review.senior_engineer"
  do comment "pr" "Author {attr.pr.author} owns this code — needs an independent reviewer"
  reason "author owns the code they changed"
  priority CRITICAL
}

rule "DBA review on migrations" {
  for records where type == "pr" and is "pr.touches_migrations"
  requires "review.dba"
  do require "review.dba"
  do comment "pr" "Schema change — DBA sign-off required. Confirm it's reversible and index-safe"
  reason "schema migration"
  priority HIGH
}
```

### Design changes — require a design review, and check against Figma

UI changes get their own path: a design review request **and** an automated Figma
conformance check (see [Automated QA](/automated-qa/)). The check produces
`design.*` facts the rules gate on.

```talon
rule "Design review + Figma check on UI changes" {
  for records where type == "pr" and is "pr.touches_design"
  do design_check "pr"                 // fetch Figma, render the preview, compare
  do require "review.design"
  do assign "pr" attr "user.owner"
  do comment "pr" "UI change — running Figma conformance and requesting a design review"
  reason "user-facing UI change"
  priority MEDIUM
}

rule "Block UI that drifts from Figma" {
  for records where type == "pr"
    and is "pr.touches_design"
    and attr "design.figma_match"     == false
    and attr "design.match_confidence" >= 0.9
  block "merge"
  do block "pr.merge"
  do comment "pr" "Rendered UI drifts from Figma: {attr.design.token_mismatches}. Align with the design tokens, or update the component in Figma first"
  reason "design drift from Figma"
  priority HIGH
}

rule "Escalate uncertain design checks to a human" {
  for records where type == "pr"
    and is "pr.touches_design"
    and attr "design.match_confidence" < 0.9
  do comment "pr" "Figma conformance was inconclusive (confidence {attr.design.match_confidence}) — a designer should eyeball this"
}
```

### Dependencies

```talon
rule "Security review for new dependencies" {
  for records where type == "pr" and attr "pr.new_dependencies" > 0
  requires "review.security"
  do require "review.security"
  do assign "pr" "@org/security"
  do comment "pr" "{attr.pr.new_dependencies} new dependencies — security review required"
  reason "adds new dependencies"
  priority HIGH
}
```

---

## Tested before it gates anything

A ruleset ships with a `.tln.test` file: you assert the verdict a set of facts should
produce, and CI fails if the policy drifts. Each `test` block declares records with
`given`, names the rule under test with `when`, and asserts with `expect` — the same
`describe / it / expect` discipline you'd bring to a Jest suite, but over facts.

```talon
test "Small, tested, non-design PR is approved; a critical one is not" {
  given {
    record 1 type "pr"
    attr 1 "pr.number" 101
    attr 1 "pr.lines_changed" 30
    attr 1 "pr.files_changed" 2
    attr 1 "pr.draft" false
    attr 1 "pr.has_description" true
    attr 1 "pr.tests_passing" true
    attr 1 "pr.lint_passing" true
    attr 1 "pr.changed_files" ["README.md", "docs/usage.md"]

    record 2 type "pr"
    attr 2 "pr.number" 102
    attr 2 "pr.lines_changed" 30
    attr 2 "pr.files_changed" 2
    attr 2 "pr.draft" false
    attr 2 "pr.has_description" true
    attr 2 "pr.tests_passing" true
    attr 2 "pr.lint_passing" true
    attr 2 "pr.changed_files" ["internal/auth/session.go"]
  }

  when rule "Auto-approve safe changes"

  expect {
    flagged 1
    not flagged 2
    did 1 approve "pr"
    did_not 2 approve "pr"
  }
}

test "CI still running leaves tests_passing unset, so nothing is approved" {
  given {
    // pr.tests_passing is deliberately absent — checks are queued. A positive
    // condition on an unset fact fails closed.
    record 1 type "pr"
    attr 1 "pr.number" 103
    attr 1 "pr.lines_changed" 10
    attr 1 "pr.files_changed" 1
    attr 1 "pr.draft" false
    attr 1 "pr.has_description" true
    attr 1 "pr.lint_passing" true
    attr 1 "pr.changed_files" ["README.md"]
  }

  when rule "Auto-approve safe changes"

  expect {
    not flagged 1
    did_not 1 approve "pr"
  }
}

test "A UI change that drifts from Figma is blocked; a matching one is not" {
  given {
    record 1 type "pr"
    attr 1 "pr.number" 210
    attr 1 "pr.changed_files" ["app/components/price_card.tsx", "app/assets/price_card.scss"]
    attr 1 "design.figma_match" false
    attr 1 "design.match_confidence" 0.96
    attr 1 "design.token_mismatches" ["color/primary", "spacing/md"]

    record 2 type "pr"
    attr 2 "pr.number" 211
    attr 2 "pr.changed_files" ["app/components/price_card.tsx"]
    attr 2 "design.figma_match" true
    attr 2 "design.match_confidence" 0.98
  }

  when rule "Block UI that drifts from Figma"

  expect {
    flagged 1
    not flagged 2
    did 1 block "pr.merge"
    did_not 2 block "pr.merge"
  }
}

test "Author owning critical code they changed is escalated" {
  given {
    record 1 type "pr"
    attr 1 "pr.number" 305
    attr 1 "pr.author" "@alice"
    attr 1 "user.owner" "@alice"
    attr 1 "pr.changed_files" ["internal/auth/token.go"]
  }

  when rule "Escalate self-review of owned critical code"

  expect {
    flagged 1
    did 1 require "review.senior_engineer"
  }
}
```

Run the suite before it ever touches a real PR:

```bash
talooner rules validate .github/talooner/
talooner rules test     .github/talooner/
```

## Facts a rule can match on

- **`pr.*`** — `lines_changed`, `files_changed`, `changed_files`, `tests_passing`,
  `lint_passing`, `has_description`, `draft`, `is_fork`, `labels`, `diff`, and more.
- **`user.*`** — `owner`, `owners`, `last_toucher`.
- **`review.*`** — `human.approved`, `changes_requested`, `<team>.approved`, `<team>.stale`.
- **`module.*`** — the primary touched module, its owner and docs.
- **`llm.*` / `qa.*` / `design.*`** — verdicts a rule asked for, re-entering the engine
  as facts. See [How it works](/how-it-works/) and [Automated QA](/automated-qa/).

Unset facts fail closed: a rule gated on a fact that wasn't asserted simply doesn't
fire — the safe direction.
