---
title: "What Is a Machine-Learning Model, Really?"
date: 2026-09-26
draft: false
summary: "A model is just a function that learned its settings from examples. Here's the intuition, with a everyday analogy."
tags: ["basics", "intuition"]
---

> This is a starter post — replace it with your own writing.

A machine-learning **model** is, at heart, just a function: it takes an input and produces an output. What makes it "learned" is that its internal settings (its *parameters*) weren't hand-written — they were tuned automatically from examples.

## An everyday analogy

Think of learning to estimate the price of a used car. Nobody gives you a formula. Instead you see many cars and their prices, and over time your brain forms a rough rule: *newer + lower mileage → higher price*. A model does the same thing, but with math and lots of examples.

## The three pieces

1. **Data** — the examples (inputs paired with the right answers).
2. **The model** — a flexible function with adjustable knobs (parameters).
3. **Training** — the process that turns the knobs so the model's outputs match the examples.

That's it. Everything else — neural networks, transformers, LLMs — is a more powerful version of "a function whose knobs were tuned from data."

*Next up: what "training" actually does, and why more data usually helps.*
