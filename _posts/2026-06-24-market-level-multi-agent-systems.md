---
layout: post
title: "Market-level multi-agent systems need negotiation protocols, not just smarter agents"
date: 2026-06-24 10:00:00
description: A product-and-systems note on coordination in agentic marketplace environments.
tags: [ai-agents, multi-agent-systems]
categories: [technical-blog]
---

Many "multi-agent" demos are really single-user systems with several internal worker agents. That pattern can help decompose tasks, but it does not solve the harder problem I care about: what happens when independent agents act on behalf of different people in the same market?

I explored that question through an agentic second-hand electronics marketplace. The MVP had four user-facing surfaces: Marketplace, Agent Desk, Deal Room, and Agent Network. The backend used Python, SQLite, and a small protocol package; the frontend stayed intentionally lightweight with vanilla HTML, CSS, and JavaScript. The goal was not to maximize framework complexity. The goal was to make the agent interaction model visible.

## The product question

A buyer's agent should be able to search listings, estimate price, inspect risk, ask a seller for evidence, make an offer, counter, withdraw, accept a deal, or escalate to a human. Those actions sound ordinary, but they imply a serious systems design problem: each action changes the state of a relationship between parties.

If the marketplace only gives the model an open-ended prompt, the system has no reliable way to audit what happened. If the marketplace defines a constrained action space, each agent move becomes a typed event with state, permissions, and consequences.

**Insight:** in market-level agent systems, the main abstraction is not "an agent that thinks." The main abstraction is a public action that other agents, humans, and the platform can inspect.

## The action space as a safety surface

I treated the action space as part of the safety design. A model can still reason freely, but the system should only execute actions that the marketplace understands. For this project, the stable actions included `search_marketplace`, `estimate_price`, `inspect_risk`, `ask_seller`, `request_evidence`, `make_offer`, `counter_offer`, `withdraw`, `escalate_to_human`, and `accept_deal`.

That list is small enough to test, log, and explain. It also creates space for non-LLM logic. For example, price estimation should not be a vague web-search answer. It should be a platform-internal comparison over similar listings, with robust statistics so one suspiciously low price does not distort the buyer's estimate.

## Protocol objects beat hidden chat state

The protocol package made the system more interesting than a basic marketplace UI. Addressing defines who can talk to whom. Message objects define what is being sent. Mail envelopes and carbon copies make auditability explicit. Approvals create a place for human escalation. Contracts encode deal state transitions. Reputation turns past outcomes into future trust signals.

Those objects are not ornamental. They are how the market stays legible when agents become active participants. Without them, the system relies on chat history and implicit conventions. With them, the market can reject invalid transitions, show a human why an action was taken, and preserve accountability across agents.

## What I learned

The biggest design correction was conceptual. A market-level multi-agent system is not just an agent with more tools. It is an environment with rules, public events, incentives, and institutional memory. The model can be powerful, but the platform must still define what a valid action means.

This is why I now evaluate agent systems by asking: what is the smallest set of actions that can express the workflow, and what invariant does each action protect? That question forces the design away from AI spectacle and toward durable product infrastructure.

## What I would build next

I would extend the MVP with adversarial seller simulations, richer evidence requests, probabilistic risk scoring, and property-based tests over contract transitions. I would also add replayable traces so researchers can compare policies: when does an agent negotiate well, when does it over-trust evidence, and when should it stop and ask the user?

The long-term version is a benchmarkable agent economy: small enough to inspect, realistic enough to reveal failures, and structured enough that new agent policies can be compared without rewriting the environment.
