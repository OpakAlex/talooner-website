---
title: "How it works"
summary: "The end-to-end flow — a deterministic engine with a single, optional probabilistic hop."
---

Talooner splits into two halves that never share secrets: an **ephemeral CI runner**
that talks to your forge, and a **self-hosted cluster** that holds the rules and the
model credentials. The runner extracts facts and executes actions; the cluster
does all the reasoning. Same commit, same rules, same verdict — every time.

<div style="overflow-x:auto; margin:1.5rem 0;">
<svg viewBox="0 0 1000 470" width="1000" style="max-width:100%; height:auto; font-family:ui-monospace,SFMono-Regular,Menlo,monospace; background:#0E1417; border-radius:14px;" role="img" aria-label="Talooner data flow: forge to CI runner to OpenTalon cluster, with the model consulted only inside the cluster.">
  <defs>
    <marker id="a-det" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0 0 L10 5 L0 10 z" fill="#57C7D4"/>
    </marker>
    <marker id="a-prob" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0 0 L10 5 L0 10 z" fill="#E4A84C"/>
    </marker>
  </defs>

  <!-- zone labels -->
  <text x="24" y="34" fill="#63767E" font-size="12" letter-spacing="1.5">ZONE A · FORGE</text>
  <text x="274" y="34" fill="#63767E" font-size="12" letter-spacing="1.5">ZONE B · CI RUNNER</text>
  <text x="524" y="34" fill="#63767E" font-size="12" letter-spacing="1.5">ZONE C · OPENTALON CLUSTER (your VPS)</text>

  <!-- Zone A: Forge -->
  <rect x="24" y="90" width="212" height="150" rx="10" fill="#161F24" stroke="#2A373E"/>
  <text x="130" y="128" fill="#DCE6E9" font-size="17" text-anchor="middle" font-weight="600">Forge</text>
  <text x="130" y="152" fill="#8FA1A9" font-size="13" text-anchor="middle">GitHub / GitLab</text>
  <text x="130" y="188" fill="#9FB0B8" font-size="12" text-anchor="middle">PR / MR · diff</text>
  <text x="130" y="208" fill="#9FB0B8" font-size="12" text-anchor="middle">comments · checks</text>

  <!-- Zone B: Runner -->
  <rect x="274" y="90" width="212" height="150" rx="10" fill="#161F24" stroke="#2A373E"/>
  <text x="380" y="128" fill="#DCE6E9" font-size="17" text-anchor="middle" font-weight="600">CI runner</text>
  <text x="380" y="152" fill="#E4A84C" font-size="12" text-anchor="middle">ephemeral · forge token only</text>
  <text x="380" y="188" fill="#9FB0B8" font-size="12" text-anchor="middle">1 · extract facts</text>
  <text x="380" y="208" fill="#9FB0B8" font-size="12" text-anchor="middle">4 · execute actions</text>

  <!-- Zone C: Cluster -->
  <rect x="524" y="60" width="452" height="345" rx="10" fill="#141C21" stroke="#2A373E"/>
  <text x="750" y="88" fill="#57C7D4" font-size="12" text-anchor="middle">holds LLM / TypeSafe credentials</text>

  <!-- engine box -->
  <rect x="548" y="108" width="196" height="92" rx="9" fill="#18242A" stroke="#2C4A50"/>
  <text x="646" y="140" fill="#DCE6E9" font-size="15" text-anchor="middle" font-weight="600">tln engine</text>
  <text x="646" y="162" fill="#57C7D4" font-size="11" text-anchor="middle">deterministic</text>
  <text x="646" y="182" fill="#8FA1A9" font-size="11" text-anchor="middle">2 · rules decide</text>

  <!-- typesafe box -->
  <rect x="780" y="108" width="172" height="92" rx="9" fill="#221B10" stroke="#4A3A1E"/>
  <text x="866" y="140" fill="#DCE6E9" font-size="15" text-anchor="middle" font-weight="600">TypeSafe / Jev</text>
  <text x="866" y="162" fill="#E4A84C" font-size="11" text-anchor="middle">typed decision</text>
  <text x="866" y="182" fill="#8FA1A9" font-size="11" text-anchor="middle">3 · only if a rule asks</text>

  <!-- db box -->
  <rect x="548" y="300" width="404" height="66" rx="9" fill="#18242A" stroke="#2C4A50"/>
  <text x="750" y="330" fill="#DCE6E9" font-size="14" text-anchor="middle" font-weight="600">fact + decision store (tln-db)</text>
  <text x="750" y="350" fill="#8FA1A9" font-size="11" text-anchor="middle">facts outlive the 30-second run · per-unit llm_review cache</text>

  <!-- arrows: A -> B (facts) -->
  <line x1="236" y1="165" x2="270" y2="165" stroke="#57C7D4" stroke-width="2" marker-end="url(#a-det)"/>
  <text x="253" y="156" fill="#8FA1A9" font-size="10" text-anchor="middle">facts</text>

  <!-- B -> C (gRPC) -->
  <line x1="486" y1="165" x2="544" y2="154" stroke="#57C7D4" stroke-width="2" marker-end="url(#a-det)"/>
  <text x="515" y="140" fill="#8FA1A9" font-size="10" text-anchor="middle">gRPC · evaluate_pr</text>

  <!-- engine -> typesafe (dashed, do llm_review) -->
  <line x1="744" y1="140" x2="778" y2="140" stroke="#E4A84C" stroke-width="2" stroke-dasharray="5 4" marker-end="url(#a-prob)"/>
  <text x="762" y="125" fill="#E4A84C" font-size="9.5" text-anchor="middle">do llm_review</text>
  <!-- typesafe -> engine (typed fact) -->
  <line x1="778" y1="172" x2="746" y2="172" stroke="#E4A84C" stroke-width="2" stroke-dasharray="5 4" marker-end="url(#a-prob)"/>
  <text x="762" y="194" fill="#8FA1A9" font-size="9" text-anchor="middle">typed fact + confidence</text>

  <!-- engine <-> db -->
  <line x1="646" y1="200" x2="646" y2="298" stroke="#57C7D4" stroke-width="2" marker-end="url(#a-det)"/>

  <!-- C -> B return (actions) -->
  <path d="M548 340 C 360 400, 360 260, 340 232" fill="none" stroke="#57C7D4" stroke-width="2" marker-end="url(#a-det)"/>
  <text x="430" y="392" fill="#8FA1A9" font-size="10" text-anchor="middle">5 · actions returned</text>

  <!-- B -> A return (execute) -->
  <path d="M300 232 C 200 300, 120 300, 90 232" fill="none" stroke="#57C7D4" stroke-width="2" marker-end="url(#a-det)"/>
  <text x="150" y="300" fill="#8FA1A9" font-size="10" text-anchor="middle">check · comment · approve</text>

  <!-- trust boundaries -->
  <line x1="255" y1="250" x2="255" y2="420" stroke="#63767E" stroke-width="1" stroke-dasharray="2 4"/>
  <line x1="505" y1="250" x2="505" y2="420" stroke="#63767E" stroke-width="1" stroke-dasharray="2 4"/>
  <text x="380" y="438" fill="#63767E" font-size="10" text-anchor="middle">forge token stays here</text>
  <text x="740" y="438" fill="#63767E" font-size="10" text-anchor="middle">LLM creds never leave the cluster</text>
</svg>
</div>

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
