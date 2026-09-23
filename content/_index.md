---
title: "Talooner"
---

## What Talooner is

*"Every line of code should map to a business need."* With the arrival of LLMs, that
truth only grows more important for building products. So how can a business be sure
the product does exactly what it should? The universal answer is testing — whether by
a person or by automated tests, they are the only thing that tells you whether the
code really works the way it's meant to.

The problem is that every open pull request — every change, even one or two lines —
can trigger irreversible, damaging reactions from users, and with them, damage to the
business.

So what does Talooner offer? It sets out to unify business needs with the work of
developers — or, in 2026, agents. With AI agents, code gets written fast. But how do
you know the agent produced a genuinely correct solution — that it didn't hallucinate,
or cover the code with tests only for show? This is where code review becomes central:
we put a human in the loop to make the call. But how much does that really simplify
things? How much does it speed them up? If AI writes the code, why can't AI also review
that code, test it, and deploy it?

Talooner offers exactly that — full automation from review to deploy. It lets AI make
decisions on its own and — based only on **rules, not prompts** — pull in specific
people for extra review when they're actually needed. It knows how to spot a component
that doesn't exist in the Figma design. It understands, from the company's own rules
and business rules, when a change might alter those rules. That lifts the code-review
load off engineers and the testing load off QA, freeing them for product and
architecture — while the product itself is preserved and improved.

And when a rule genuinely needs a model's judgment, Talooner doesn't prompt a chat
LLM and try to parse prose out of it. It calls [TypeSafe / Jev](https://typesafe.ai) —
a machine-native model built for decisions, not conversation. Instead of a paragraph,
it returns a **typed answer with calibrated confidence**: a *Choice* from a fixed set,
a *Score* on a rubric, or a *true/false*. That answer re-enters the engine as a plain
fact carrying a confidence number, so a rule can act on a confident verdict and
escalate an uncertain one to a human — deterministically, and without a model ever
having the final say. Typed and bounded, it fits Talooner's world far better than
free-text output ever could, and it makes model verdicts cheap enough to run on every
pull request.

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

### Ask a model — powered by [TypeSafe / Jev](https://typesafe.ai)

Where a rule asks for `llm_review`, Talooner calls TypeSafe and gets back a **typed**
answer plus a confidence score — `llm.risk` (a Choice), `llm.doc_conformance` (a
Score), `llm.matches_description` (true/false), each with an `*_confidence`. Rules gate
on both the value and the confidence.

```talon
rule "Review large core-domain PRs with a typed model verdict" {
  for records where type == "pr"
    and attr "pr.lines_changed" >= 400
    and attr "module.touched_count" >= 1
    and attr "pr.diff_truncated" == false
  do llm_review "pr"                        // → TypeSafe returns typed facts
  reason "large change to owned code"
  priority MEDIUM
}

// Choice: TypeSafe classifies risk as low | medium | high, with confidence.
rule "Escalate risky auth diffs the model is confident about" {
  for records where type == "pr"
    and is "critical_path"
    and attr "llm.risk" == "high"
    and attr "llm.risk_confidence" >= 0.9
  requires "review.security"
  do require "review.security"
  do comment "pr" "TypeSafe flags this as high-risk (confidence {attr.llm.risk_confidence}) — security review required"
  priority HIGH
}

// Score: TypeSafe rates how well the diff matches its module's docs, 0–5.
rule "Block code that contradicts its own documentation" {
  for records where type == "pr"
    and attr "llm.doc_conformance" < 2
    and attr "llm.doc_conformance_confidence" >= 0.85
  block "merge"
  do block "pr.merge"
  do comment "pr" "TypeSafe scored doc-conformance {attr.llm.doc_conformance}/5 — the change contradicts the module docs"
  reason "code diverges from documentation"
  priority HIGH
}

// True/false: does the diff actually do what the PR description claims?
rule "Flag PRs whose code doesn't match the description" {
  for records where type == "pr"
    and attr "llm.matches_description" == false
    and attr "llm.matches_description_confidence" >= 0.8
  do comment "pr" "TypeSafe thinks the diff doesn't match the description ({attr.llm.matches_description_confidence}) — reviewers should confirm scope"
  reason "scope mismatch"
  priority MEDIUM
}

// When TypeSafe isn't sure, don't guess — send it to a human.
rule "Escalate low-confidence model verdicts" {
  for records where type == "pr"
    and attr "llm.risk_confidence" < 0.7
  do comment "pr" "Model verdict was low-confidence — a human should take this one"
  priority LOW
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
