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

> **Status: early v1.** The walking skeleton runs end to end. Several capabilities
> described here — the `llm_review` path and Automated QA — are the design we're
> building toward, not shipped features. Where that's the case, the page says so.
