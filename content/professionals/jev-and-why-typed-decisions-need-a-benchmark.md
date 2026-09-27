---
title: "Jev, and Why Typed-Decision Models Need Their Own Benchmark"
date: 2026-09-27
draft: false
summary: "A quick tour of TypeSafe AI's Jev, why free-form LLM benchmarks don't measure typed decisions well, and JevBench — a small, paradigm-agnostic benchmark for Choice / Score / Noul decisions."
tags: ["jev", "typed-decisions", "benchmarks", "evaluation", "calibration"]
cover:
  hidden: true
---

A week after TypeSafe AI released **Jev**, my GitHub search for reproductions returned 450+ repos. That's a lot of energy — and almost no common yardstick to compare any of it. This post is about what Jev is, why it needs a *different* kind of benchmark than the ones we point at chat models, and the small one I built to scratch that itch: **[JevBench](https://github.com/model-collapse/jev-bench)**.

## What Jev is

Most of the models we benchmark are **System Two**: autoregressive LLMs that *generate* an answer token by token, and that you then have to parse and hope the format held.

Jev is a **System One** *typed-decision* model. Instead of generating prose, it returns a value of a **declared type**:

- **Choice** — pick exactly one of N enumerated options (an `enum`).
- **Score** — one level on a fixed ordinal scale (a bounded ordinal, e.g. severity 0–3).
- **Noul** — yes / no (a `bool`).

Three properties make it interesting:

1. **Non-autoregressive.** It scores the candidates in essentially one forward pass rather than decoding a sentence — so it's fast and **order-invariant** (the answer doesn't drift when you shuffle the options).
2. **Calibrated confidence.** Every decision comes back with a probability, not just a label — which is exactly what you need to *gate* a decision ("act if confident, escalate to a human if not").
3. **Type-safety.** The output is always a valid member of a small, known set. No parsing, no "the model wrote a paragraph when I asked for a label."

If you've built agent pipelines, you already know why this matters: a huge fraction of an agent's "thinking" is really small, repeated, *typed* decisions — route this ticket, is this alert malicious, how risky was this action, does this invoice reconcile. Paying a full autoregressive generation for each of those is slow and fragile. A typed-decision model is the right tool for that job.

## Why a benchmark is necessary

Here's the problem: **the benchmarks we already have don't measure this well**, and the reproduction gold-rush made it worse. Concretely —

**1. Free-form LLM benchmarks score the wrong thing.** MMLU-style accuracy assumes a single exact answer. But a `Score` decision is *ordinal* — predicting severity `2` when the truth is `3` is a small miss; predicting `0` is a big one. Exact-match punishes both equally. Typed decisions need type-appropriate metrics (exact for Choice/Noul, a distance-aware metric for Score) — and a measure of **calibration**, which accuracy ignores entirely.

**2. "It does decisions" spans wildly different paradigms.** The reproductions I found weren't one architecture — they were generative LLMs, cross-encoders, bi-encoders, NLI classifiers, tiny frozen-backbone + head models. Each claims to do typed decisions; each is strong at a *different* subset. Without a **paradigm-agnostic** harness that runs each fairly, you can't compare them — and it's easy to make a paradigm look broken by running it in the wrong mode. (I learned this the hard way: an NLI model looked like it was at chance on yes/no questions, until I realized the harness was scoring the literal tokens "yes"/"no" instead of running actual entailment. Real signal was hiding behind a mode bug.)

**3. Hundreds of reproductions, no shared ground.** With 450+ repos and no common test set, every README reports its own numbers on its own data. A shared, reliably-labeled benchmark turns "trust me, it's good" into a comparable row in a table.

**4. Evaluation itself is full of traps.** Building this surfaced a metric bug (two disagreeing kappa formulas that flipped rankings), a train/test **contamination** case (a fine-tuned repro that had seen the benchmark's distribution), and the reminder that **small screening sets lie** (a 60-example screen moved some models by ±14 points versus the full set). A benchmark is only useful if it's rigorous about its own failure modes.

## JevBench, illustrated

JevBench is deliberately small and honest: **2,934 ground-truth examples**, all three primitives, across topics from trivial perception to compositional reasoning.

**Structure — two dimensions:**

- **Task type:** Choice / Score / Noul.
- **Task context (topic):** objective skill-probes (sentiment, arithmetic, logic, multi-hop lookup…), realistic *operational* decisions (customer-service triage, security-alert disposition, invoice processing, agent-trace observability), real human-labeled intent classification (banking77), and a relevance add-on.

**Metrics that fit the type:**

- `Choice` / `Noul` → **exact accuracy**.
- `Score` → **quadratic-weighted kappa (QWK)** plus **MAE** — so an off-by-one on an ordinal scale is a small penalty, a wild miss a large one. (Read QWK as tiers, not decimals: ~0.8+ = calibrated ordinal judgment; ≈0 = no signal; **negative = worse than chance**, i.e. the model ranks severity *backwards*.)

**One harness, every paradigm.** The evaluator reduces every question to "pick the best candidate," then adapts to the model: prompt-and-parse for LLMs, log-prob option-scoring for base LMs (the Jev/laya style), pairwise scoring for cross-encoders, cosine for embedders, entailment for NLI, and a native typed-decision API path. It auto-detects the model type, and it **flags a run as unreliable** if too many predictions come back empty — so a broken model id can't masquerade as a real low score.

**A snapshot of the board** (232-example gold subset, read the top cluster as a tie):

| model | paradigm | overall | choice | noul | score-QWK |
|---|---|--:|--:|--:|--:|
| GPT-5.5 / Opus 5 / GPT-6 | frontier LLM | ~90% | ~94 | ~94 | 0.6–0.8 |
| **Jev** | native typed-decision API | 88.8% | 94 | 92 | 0.96 |
| gpt-oss-20b | open LLM | 88.4% | 91 | 94 | 0.96 |
| laya (open, zero-shot) | encoder + [MASK] readout | 54% | 47 | 72 | 0.21 |
| cross-encoder / bi-encoder | similarity | 35–41% | — | — | ~0 |

Two things jump out. First, **the purpose-built specialist (Jev) sits right in the frontier cluster** while being far smaller and faster — and it has the *cleanest* ordinal calibration (score-QWK 0.96). Second, **the real separation is by paradigm, not by vendor**: reasoning-capable models cluster near the top; similarity models collapse on anything requiring verification or computation (they can do sentiment and routing, not arithmetic or logic).

And the reproductions? Benchmarked on the same set, the strongest open ones — a **frozen-backbone + LoRA + typed head + temperature-calibration** design — reach ~78–82%, just under the top cluster. That's the encouraging part: the open recipe for a small, calibrated typed-decision model is *close*, and now there's a yardstick to measure the gap.

## Try it

JevBench is open — data, the paradigm-agnostic harness, the leaderboard, and the task taxonomy are all in **[github.com/model-collapse/jev-bench](https://github.com/model-collapse/jev-bench)**. Point it at a model:

```bash
python bench_eval.py --model <your-model> --data data --tier gold
```

If you're building a typed-decision model, I'd love to see where it lands. In the next post I'll dig into the part accuracy hides: **calibration** — temperature scaling and conformal prediction for confidence you can actually gate on.
