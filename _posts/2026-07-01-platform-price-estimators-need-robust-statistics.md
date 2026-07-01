---
layout: post
title: "Marketplace price estimators need robust statistics"
date: 2026-07-01 10:00:00
description: A concrete pricing lesson from building a platform-internal estimator for an agentic second-hand market.
tags: [marketplace-systems, pricing]
categories: [technical-blog]
---

One of the most practical lessons from the `agentic-secondhand-market` project had nothing to do with model prompting. It came from a smaller question: when a buyer agent asks, "What is a reasonable price for this listing?", what should the platform actually compute?

The naive answer is obvious. Search similar listings, average the prices, and return a number.

That answer is also wrong.

In a second-hand marketplace, one suspicious listing can distort the result enough to change agent behavior. A stolen device, a broken device described poorly, or a seller baiting buyers with an unrealistic price should not drag the platform's estimate downward. If the estimate becomes too trusting, every downstream decision gets worse: first offers, risk flags, escalation thresholds, and eventually user trust.

## The product constraint

This marketplace was designed so trading agents do not browse the open web for price advice. They act inside a platform with its own listings, event log, evidence model, and negotiation flow. That meant price estimation also had to be platform-internal.

The implementation followed a simple rule: use marketplace peers plus a small reference catalog, then derive a range instead of pretending there is one "correct" number.

That choice did two useful things.

First, it kept the estimate inspectable. A user or developer can understand where the number came from.

Second, it made the estimator compatible with the rest of the system. Risk inspection, negotiation, and human escalation can all reason about a structured price report more reliably than a free-form model opinion.

## What the estimator actually does

The `PriceEstimator` looks up peer listings with the same model, adjusts those prices for condition and evidence quality, then mixes them with a platform-maintained reference band when available.

The adjustment step matters. A listing with warranty proof should not be treated the same as one without a serial number or repair history disclosure. Before any statistics happen, the raw prices are normalized into a better comparison set.

After that, the estimator does not anchor on the lowest price. It sorts the adjusted prices, takes the median, computes the first and third quartiles, and then derives:

- an estimated minimum;
- an estimated maximum;
- a suggested first offer;
- a maximum reasonable price;
- a confidence label based on peer count.

This is a small design, but it encodes a strong opinion: robust ranges are more useful than point guesses in noisy markets.

## Why the median changed the behavior

The median is boring, and that is exactly why it helps.

If one listing is suspiciously cheap, an average can move far more than it should. The median barely reacts unless enough comparable listings agree. That is the right bias for a marketplace platform. The system should be conservative about calling something a bargain when the supporting evidence is weak.

Quartiles add a second layer of protection. They make it possible to build a range from the shape of the local market instead of from a single central number. In the implementation, the lower and upper bounds are derived from both the median and the quartile band, then rounded into user-facing values that negotiation logic can consume directly.

That design gave the rest of the workflow something better than "estimated price: 1327". It produced a report with negotiation semantics:

- below this floor, the listing may be suspicious;
- above this ceiling, the buyer may be overpaying;
- inside this band, negotiation is plausible;
- confidence depends on how much platform evidence exists.

## Why this matters for agent systems

This project reinforced a broader lesson for agent products: internal tools should express platform judgment, not just surface raw model output.

`estimate_price` is not a chatbot answer. It is part of the marketplace policy layer. The `RiskInspector` depends on it to decide when a listing is materially below the estimated range and should be escalated. Negotiation logic depends on it to shape first offers and deal limits. That only works if the estimator is stable under noisy inputs.

In other words, the pricing tool is doing two jobs at once:

- summarizing market evidence;
- protecting the rest of the runtime from bad anchors.

That is why "just average similar listings" is not enough. The number does not live in isolation. It becomes a control surface for the whole market.

## The engineering habit I want to keep

The useful habit here is to design internal AI-adjacent tools as typed system components, not as one-off smart answers.

For this marketplace, that meant a price report with explicit fields, deterministic logic, inspectable rationale, and confidence tied to sample quality. The model can still help elsewhere, but the pricing boundary stays legible.

I think this pattern generalizes well. Whenever an agent system needs to act on uncertain real-world data, the first question should be: what statistic is robust enough that other components can safely build on it?

Sometimes the most important AI design decision is choosing a boring statistic that refuses to be fooled.
