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

<div style="overflow-x:auto; margin:1.5rem 0;">
<svg viewBox="0 0 1000 250" width="1000" style="max-width:100%; height:auto; font-family:ui-monospace,SFMono-Regular,Menlo,monospace; background:#0E1417; border-radius:14px;" role="img" aria-label="Automated QA loop: gate, deploy preview, generate scenarios, run, facts, verdict.">
  <defs>
    <marker id="q-det" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0 0 L10 5 L0 10 z" fill="#57C7D4"/>
    </marker>
    <marker id="q-prob" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0 0 L10 5 L0 10 z" fill="#E4A84C"/>
    </marker>
  </defs>
  <!-- 5 stage boxes -->
  <rect x="20"  y="90" width="160" height="72" rx="9" fill="#18242A" stroke="#2C4A50"/>
  <text x="100" y="120" fill="#DCE6E9" font-size="13" text-anchor="middle" font-weight="600">Rule gate</text>
  <text x="100" y="140" fill="#57C7D4" font-size="10" text-anchor="middle">deploy-worthy?</text>

  <rect x="220" y="90" width="160" height="72" rx="9" fill="#18242A" stroke="#2C4A50"/>
  <text x="300" y="120" fill="#DCE6E9" font-size="13" text-anchor="middle" font-weight="600">Preview deploy</text>
  <text x="300" y="140" fill="#8FA1A9" font-size="10" text-anchor="middle">ephemeral env</text>

  <rect x="420" y="90" width="160" height="72" rx="9" fill="#221B10" stroke="#4A3A1E"/>
  <text x="500" y="116" fill="#DCE6E9" font-size="13" text-anchor="middle" font-weight="600">Generate</text>
  <text x="500" y="134" fill="#E4A84C" font-size="10" text-anchor="middle">scenarios (LLM)</text>
  <text x="500" y="150" fill="#8FA1A9" font-size="9" text-anchor="middle">from diff + docs</text>

  <rect x="620" y="90" width="160" height="72" rx="9" fill="#18242A" stroke="#2C4A50"/>
  <text x="700" y="120" fill="#DCE6E9" font-size="13" text-anchor="middle" font-weight="600">Run + measure</text>
  <text x="700" y="140" fill="#57C7D4" font-size="10" text-anchor="middle">pass / fail</text>

  <rect x="820" y="90" width="160" height="72" rx="9" fill="#18242A" stroke="#2C4A50"/>
  <text x="900" y="116" fill="#DCE6E9" font-size="13" text-anchor="middle" font-weight="600">Verdict</text>
  <text x="900" y="134" fill="#57C7D4" font-size="10" text-anchor="middle">rules decide</text>
  <text x="900" y="150" fill="#8FA1A9" font-size="9" text-anchor="middle">approve / escalate</text>

  <line x1="180" y1="126" x2="216" y2="126" stroke="#57C7D4" stroke-width="2" marker-end="url(#q-det)"/>
  <line x1="380" y1="126" x2="416" y2="126" stroke="#57C7D4" stroke-width="2" marker-end="url(#q-det)"/>
  <line x1="580" y1="126" x2="616" y2="126" stroke="#E4A84C" stroke-width="2" stroke-dasharray="5 4" marker-end="url(#q-prob)"/>
  <line x1="780" y1="126" x2="816" y2="126" stroke="#57C7D4" stroke-width="2" marker-end="url(#q-det)"/>

  <text x="500" y="40" fill="#63767E" font-size="12" text-anchor="middle" letter-spacing="1.5">AUTOMATED QA LOOP</text>
  <text x="500" y="205" fill="#63767E" font-size="10.5" text-anchor="middle">solid = deterministic · dashed = the model proposes scenarios, it never scores them</text>
  <text x="500" y="224" fill="#63767E" font-size="10.5" text-anchor="middle">results re-enter the engine as qa.* facts → the ruleset gates the merge</text>
</svg>
</div>

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

> **Roadmap, not shipped.** Automated QA is the design we're building toward. It
> depends on the `do llm_review` path landing first, plus new executors
> (`deploy_preview`, `qa_scenarios`) and the `qa.*` fact family. Today Talooner
> extracts facts, evaluates rules, and executes review actions; the QA loop above is
> where that goes next.
