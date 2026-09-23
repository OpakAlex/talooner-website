---
title: "Talooner"
---

## What Talooner is

Generic "ask an LLM to review this diff" doesn't solve the real problem: review is
company-specific, and a bare prompt can't encode that. Talooner makes the policy
explicit. A repository declares its review policy as [`tln`](https://tln-lang.org)
rules. Talooner ingests the PR's facts, runs the rules through the inference
engine, and executes the resulting actions against the GitHub or GitLab API.

**No model decides whether a PR is approved — rules do.** A model is consulted only
where a rule explicitly asks for it (`do llm_review …`), and its output re-enters
the engine as a fact.

- **[How it works →]({{< relref "/how-it-works" >}})** — the flow, end to end, and where a model fits.
- **[Automated QA →]({{< relref "/automated-qa" >}})** — rules that decide to deploy and run LLM-generated test scenarios.
- **[Rules →]({{< relref "/rules" >}})** — policy as code: versioned, diffable, unit-testable.
- **[Blog →]({{< relref "/blog" >}})** — notes from building a deterministic reviewer.

## Example rules

A taste of what a policy looks like. These are `tln` rules a repo drops into
`.github/talooner/rules.tln`. Every rule matches PR facts and declares a verdict
(`allow` / `block` / `requires`) plus the actions to take. See the full worked
ruleset and its test suite on the [Rules]({{< relref "/rules" >}}) page.

### Auto-approve the safe, boring stuff

```talon
define "small_change" {
  attr "pr.lines_changed" < 80
  attr "pr.files_changed" < 6
}

rule "Auto-approve small, tested, documented changes" {
  for records where type == "pr"
    and is "small_change"
    and attr "pr.draft" == false
    and attr "pr.has_description" == true
    and attr "pr.tests_passing" == true
    and attr "pr.lint_passing" == true
    and not is "critical_path"
  allow "merge"
  do approve "pr"
  do comment "pr" "Small, green, off the critical path — no objections from me"
  priority MEDIUM
}

rule "Auto-approve translation-only changes" {
  for records where type == "pr"
    and attr "pr.changed_files" contains "config/locales/"
    and not attr "pr.changed_files" contains "app/"
    and attr "pr.tests_passing" == true
  allow "merge"
  do approve "pr"
  priority LOW
}
```

### Require the right humans

```talon
define "critical_path" {
  attr "pr.changed_files" contains "internal/auth/"
    or attr "pr.changed_files" contains "billing/"
}

rule "Human review on the critical path" {
  for records where type == "pr" and is "critical_path"
  requires "review.senior_engineer"
  do require "review.senior_engineer"
  do assign "pr" attr "user.owner"
  do comment "pr" "Touches critical code owned by {attr.user.owner} — human approval required"
  reason "critical path"
  priority HIGH
}

rule "Don't let an author approve their own critical code" {
  for records where type == "pr"
    and is "critical_path"
    and attr "user.owner" == attr "pr.author"
  requires "review.senior_engineer"
  do require "review.senior_engineer"
  do comment "pr" "Author {attr.pr.author} owns this code — needs an independent reviewer"
  reason "author owns the code they changed"
  priority CRITICAL
}

rule "DBA sign-off on schema migrations" {
  for records where type == "pr"
    and attr "pr.changed_files" contains "db/migrate/"
  requires "review.dba"
  do require "review.dba"
  do comment "pr" "Schema change — confirm it's reversible and index-safe on large tables"
  reason "schema migration"
  priority HIGH
}
```

### Gate incomplete or oversized work

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
  for records where type == "pr" and attr "pr.lines_changed" >= 1500
  block "merge"
  do block "pr.merge"
  do comment "pr" "{attr.pr.lines_changed} lines across {attr.pr.files_changed} files — split it before review"
  reason "unreviewably large"
  priority HIGH
}

rule "Nudge PRs that span too many modules" {
  for records where type == "pr" and attr "module.touched_count" > 2
  do comment "pr" "Spans {attr.module.touched_count} modules — consider splitting so each owner reviews their part"
  priority LOW
}
```

### Security & supply chain

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

rule "Flag dependency upgrades for a look" {
  for records where type == "pr"
    and attr "pr.upgraded_dependencies" > 0
    and attr "pr.new_dependencies" == 0
  do comment "pr" "{attr.pr.upgraded_dependencies} dependency upgrade(s) — check the changelogs for breaking changes"
  priority LOW
}
```

### Design changes — check against Figma

```talon
define "pr.touches_design" {
  attr "pr.changed_files" contains "app/components/"
    or attr "pr.changed_files" ends_with ".scss"
    or attr "pr.changed_files" ends_with ".css"
}

rule "Design review + Figma check on UI changes" {
  for records where type == "pr" and is "pr.touches_design"
  do design_check "pr"
  do require "review.design"
  do comment "pr" "UI change — running Figma conformance and requesting a design review"
  priority MEDIUM
}

rule "Block UI that drifts from Figma" {
  for records where type == "pr"
    and is "pr.touches_design"
    and attr "design.figma_match" == false
    and attr "design.match_confidence" >= 0.9
  block "merge"
  do block "pr.merge"
  do comment "pr" "Rendered UI drifts from Figma: {attr.design.token_mismatches}. Align with the tokens, or update Figma first"
  reason "design drift"
  priority HIGH
}

rule "Flag UI that isn't in Figma at all" {
  for records where type == "pr"
    and is "pr.touches_design"
    and attr "design.undocumented_count" > 0
  do require "review.design"
  do comment "pr" "These rendered components have no Figma source: {attr.design.undocumented_components}"
  reason "undocumented UI"
  priority MEDIUM
}
```

### Ask a model — only where a rule says so

```talon
rule "LLM review large core-domain PRs" {
  for records where type == "pr"
    and attr "pr.lines_changed" >= 400
    and attr "module.touched_count" >= 1
    and attr "pr.diff_truncated" == false
  do llm_review "pr"
  reason "large change to owned code"
  priority MEDIUM
}

rule "Escalate risky auth diffs the model is confident about" {
  for records where type == "pr"
    and is "critical_path"
    and attr "llm.risk" == "high"
    and attr "llm.risk_confidence" >= 0.9
  requires "review.security"
  do require "review.security"
  do comment "pr" "Model flags this as high-risk (confidence {attr.llm.risk_confidence}) — security review required"
  priority HIGH
}
```

### Run the tests it generates

```talon
rule "Run automated QA on deploy candidates" {
  for records where type == "pr"
    and attr "pr.tests_passing" == true
    and attr "pr.lint_passing" == true
    and not is "critical_path"
  do deploy_preview "pr"
  do qa_scenarios "pr"
}

rule "Block on failed QA scenarios" {
  for records where type == "pr"
    and attr "qa.scenarios_passed" < attr "qa.scenarios_total"
  block "merge"
  do block "pr.merge"
  do comment "pr" "Automated QA failed {attr.qa.failed_scenarios} — see the run before merging"
  reason "generated scenarios failed"
  priority HIGH
}
```

Want more? The [Rules]({{< relref "/rules" >}}) page has the full ruleset grouped by
purpose, plus a `.tln.test` suite that proves each rule before it gates a real PR.
