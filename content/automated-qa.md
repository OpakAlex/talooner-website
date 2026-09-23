---
title: "Automated QA"
summary: "Rules that decide when a PR is worth deploying, then verify it with LLM-generated test scenarios — and fold the result back into the verdict as a fact."
---

Static review tells you whether a diff *looks* right. It can't tell you whether the
change actually *works*. Automated QA closes that gap without handing the decision to
a model: **the rules decide when a PR earns a throwaway deployment, an LLM generates
the test scenarios, the scenarios run, and the results come back as facts the engine
gates on.**

The model proposes scenarios. It never decides the outcome — pass/fail is measured,
and the verdict stays with the rules.

## The idea

Once a PR is deemed deploy-worthy by policy — green CI, off a critical path, small
enough to be safe — Talooner can:

1. **Spin up an ephemeral preview deployment** of the PR's code.
2. **Ask a model to generate test scenarios** from the diff, the PR description, and
   the module's own docs — the behaviours a reviewer would want exercised.
3. **Run those scenarios** against the preview environment.
4. **Fold the results back into the engine as facts** — `qa.scenarios_total`,
   `qa.scenarios_passed`, `qa.failed_scenarios`, `qa.coverage_confidence`.
5. **Let the rules decide** what the outcome means: approve, comment the failures,
   request changes, escalate to a human, or tear the environment down.

{{< diagram src="img/automated-qa.svg" alt="Automated QA loop: rule gate, preview deploy, generate scenarios, run and measure, verdict." >}}

## What it looks like in a ruleset

```talon
// Only PRs that already pass the cheap gates earn a deployment.
define "deploy_candidate" {
  attr "pr.tests_passing" == true
  attr "pr.lint_passing"  == true
  attr "pr.draft"         == false
  not is "critical_path"
}

// Rules ask for the deployment and the generated scenarios — they aren't automatic.
rule "Run automated QA on deploy candidates" {
  for records where type == "pr" and is "deploy_candidate"
  do deploy_preview "pr"
  do qa_scenarios "pr"
}

// The engine gates on the measured result, not on the model's opinion.
rule "Block on failed QA scenarios" {
  for records where type == "pr"
    and attr "qa.scenarios_passed" < attr "qa.scenarios_total"
  block "merge"
  do block "pr.merge"
  do comment "pr" "Automated QA failed {attr.qa.failed_scenarios} — see the run before merging"
  reason "generated scenarios failed"
  priority HIGH
}

// Low coverage confidence is a human's call, not an auto-approve.
rule "Escalate thin QA coverage" {
  for records where type == "pr"
    and attr "qa.coverage_confidence" < 0.8
  do comment "pr" "QA coverage looks thin (confidence {attr.qa.coverage_confidence}) — a human should confirm the risky paths are exercised"
}
```

## Why route it through rules instead of a model

- **The decision to deploy is policy, not a guess.** A rule — not a model — decides
  which PRs are safe to stand up. Fork PRs, critical paths, and secret-touching
  changes simply never enter the loop.
- **Scenarios are generated; outcomes are measured.** The LLM's job ends at proposing
  what to test. Whether those tests pass is a fact, and facts are what the engine
  gates on. A confident, wrong model can't approve a broken PR.
- **Every run is auditable.** The generated scenarios, the environment, and the
  pass/fail results are all recorded as facts — you can `explain` exactly why a PR
  was blocked or approved.
- **Confidence is explicit.** `qa.coverage_confidence` lets a rule distinguish
  "verified working" from "we couldn't meaningfully exercise this" and route the
  second case to a human.

## Generated scenarios are real tests

The scenarios aren't a prose summary of "what to check" — they're executable tests
the runner can actually run against the preview. A functional scenario, generated
from the diff, the PR description, and the module's docs, comes out as a Jest test:

```js
// talooner-generated · scenario: "checkout rejects an expired coupon"
// derived from: diff app/services/checkout/*, PR body, docs/checkout.md
import { test, expect } from '@jest/globals';

const DEPLOY = process.env.PREVIEW_URL;

test('an expired coupon is rejected at checkout', async () => {
  const res = await fetch(`${DEPLOY}/api/checkout`, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ items: [{ sku: 'A1', qty: 1 }], coupon: 'EXPIRED2020' }),
  });

  expect(res.status).toBe(422);
  const body = await res.json();
  expect(body.error).toMatch(/coupon.*expired/i);
});
```

The model proposed the case; Jest measures the outcome. `qa.scenarios_passed <
qa.scenarios_total` is a fact, and the rules gate on it — a confident, wrong model
can't turn a red suite green.

## Design conformance: does the page match Figma?

When a PR touches UI, Talooner can go past "does it work" to "does it look like the
design". The `design_check` executor reads the referenced Figma nodes (reference image
+ design tokens), renders the changed components in the preview, and compares them —
**both directions**:

- **Drift from Figma** — a component whose pixels or tokens (color, spacing, type)
  don't match its Figma source.
- **Not in Figma** — components rendered on the page that have *no* matching Figma
  component. Undocumented UI is drift too, and easy to miss in review.

It folds the result into `design.*` facts:

| Fact | Meaning |
|---|---|
| `design.figma_match` | every changed component matched its Figma source |
| `design.pixel_drift` | worst-case visual difference, 0–1 |
| `design.token_mismatches` | design tokens that didn't match (`color/primary`, `spacing/md`, …) |
| `design.undocumented_count` | rendered components with **no** Figma source |
| `design.undocumented_components` | which ones |
| `design.match_confidence` | calibrated confidence in the verdict |

The generated design scenario is a Jest + Playwright visual test:

```js
// talooner-generated · design scenario: "PriceCard vs Figma node 1284:57"
import { test, expect } from '@jest/globals';
import { chromium } from 'playwright';
import { fetchFigmaNode, diffAgainstFigma, componentsOnPage } from '@talooner/figma-qa';

const DEPLOY = process.env.PREVIEW_URL;

test('PriceCard renders within 1% of its Figma component', async () => {
  const ref = await fetchFigmaNode('1284:57');            // reference image + tokens
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto(`${DEPLOY}/components/price-card`);

  const shot = await page.locator('[data-component="price-card"]').screenshot();
  const { pixelDrift, tokenMismatches } = diffAgainstFigma(shot, ref);

  expect(pixelDrift).toBeLessThan(0.01);
  expect(tokenMismatches).toEqual([]);                    // colors/spacing/type match tokens
  await browser.close();
});

test('every rendered component maps to a Figma component', async () => {
  const rendered   = await componentsOnPage(`${DEPLOY}/pricing`);       // [data-component] nodes
  const documented = await fetchFigmaNode('page:pricing').then(n => n.componentNames);
  const orphans    = rendered.filter(c => !documented.includes(c));
  expect(orphans).toEqual([]);   // UI with no Figma source is a finding, not a pass
});
```

And the rule that catches the not-in-Figma case:

```talon
rule "Flag UI that isn't in Figma" {
  for records where type == "pr"
    and is "pr.touches_design"
    and attr "design.undocumented_count" > 0
  do require "review.design"
  do comment "pr" "These rendered components have no Figma source: {attr.design.undocumented_components}. Add them to the design file, or remove them"
  reason "undocumented UI"
  priority MEDIUM
}
```

Pair it with the drift and confidence rules on the [Rules]({{< relref "/rules" >}}) page and a UI
change is gated three ways: it must match Figma, contain nothing that *isn't* in
Figma, and — when the check is unsure — go to a human.

