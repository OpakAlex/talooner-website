---
title: "How it works"
summary: "The end-to-end flow — a deterministic engine with a single, optional probabilistic hop."
---

Talooner splits into two halves that never share secrets: an **ephemeral CI runner**
that talks to your forge, and a **self-hosted cluster** that holds the rules and the
model credentials. The runner extracts facts and executes actions; the cluster
does all the reasoning. Same commit, same rules, same verdict — every time.

{{< diagram src="img/how-it-works.svg" alt="Talooner data flow: Forge to CI runner to OpenTalon cluster, with the model consulted only inside the cluster." >}}

## The flow, step by step

1. **Trigger.** A PR/MR event fires, or a maintainer comments `!talooner /review`.
2. **Extract facts.** The ephemeral runner reads the forge API with its scoped
   token: `pr.*`, `user.*`, `review.*`, the diff, changed files, CI status.
3. **Evaluate.** Facts cross to the cluster over gRPC (`evaluate_pr`). The `tln`
   engine loads the repo's ruleset and resolves the verdict. Most PRs never touch a
   model.
4. **Consult a model — only if a rule asks.** Where a rule says `do llm_review`, the
   executor calls a model with the code-unit's *state* plus a *typed question*.
   With [TypeSafe / Jev](https://typesafe.ai) that answer comes back as a **typed
   decision with calibrated confidence** — a `Choice`, a `Score`, or a `Noul`
   (true/false, 0–1) — which re-enters the engine as an ordinary fact
   (`llm.risk`, `llm.risk_confidence`, …). Not prose.
5. **Act.** The engine returns a list of actions; the runner executes them on the
   forge — a check run / commit status, one sticky review comment, an advisory
   approve/block, assignees and review requests — then exits.

## Why this shape

- **The credential boundary is the whole point.** The runner holds only a scoped,
  short-lived forge token. The model credentials live in the cluster you run, so
  every token a rule spends is billed to whoever ran the rule. Nobody's review load
  lands on someone else's API limits.
- **Typed output makes confidence a first-class gate.** A rule can *act* on a
  high-confidence verdict and *escalate to a human* on a low one — something you
  can't express cleanly against free-text LLM output.
- **Determinism where it counts.** The engine is deterministic. The single
  probabilistic hop is isolated behind `do llm_review` and cached per code-unit, so
  its variance is contained rather than driving the verdict.

> **Not live yet.** The `do llm_review` path is not reachable end-to-end today —
> `code_units` aren't sent on `evaluate_pr` yet. Steps 4 above describe the intended
> shape. Everything else — fact extraction, rule evaluation, and action execution —
> runs today.
