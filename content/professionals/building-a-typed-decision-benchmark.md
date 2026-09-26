---
title: "Building a Paradigm-Agnostic Benchmark for Typed Decisions"
date: 2026-09-26
draft: false
summary: "Notes on evaluating Choice / Score / Noul decisions across LLMs, cross-encoders, embedders, and NLI — and the metric bugs that bite you."
tags: ["evaluation", "benchmarks", "calibration", "typed-decisions"]
---

> Starter post — swap in your own framing and results.

A *typed decision* is a model output constrained to a declared type: **Choice** (pick one option), **Score** (an ordinal level), or **Noul** (yes/no). Evaluating such models fairly across very different paradigms — generative LLMs, cross-encoders, embedders, entailment models — turns out to be subtle. A few lessons that generalized:

## 1. Match the metric to the answer space
Exact-match is right for `choice`/`noul`, but punishing an off-by-one on an ordinal `score` as harshly as a wild miss is wrong. Use **quadratic-weighted kappa (QWK)** for ordinal scores, and report **MAE** alongside — it's the more stable companion on small subsets.

## 2. Run each paradigm in its *fair* mode
Scoring the literal tokens "yes"/"no" is degenerate for an entailment or similarity model — a `noul` statement is a premise/hypothesis pair. Running true NLI (premise = input, hypothesis = the statement) instead of token-scoring can move a model from chance to real signal. A mode bug can make a whole paradigm look broken when it isn't.

## 3. Surface silent failures
A wrong model id, a blocked model, or a tokenization bug can yield all-empty predictions that masquerade as a real low score. Guard for it: if a large fraction of predictions are null, flag the run as unreliable rather than reporting a fake number.

## 4. Small screening sets lie
A 60-example screen moved some models by ±14 points versus the full set. Screen small to iterate, but publish only full-set numbers.

*More to come: calibration (temperature scaling + conformal prediction) for confidence-aware decisions.*
