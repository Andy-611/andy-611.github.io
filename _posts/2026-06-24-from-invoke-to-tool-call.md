---
layout: post
title: "From INVOKE to tools/call: a thin bridge between agent protocols and MCP"
date: 2026-06-24 09:00:00
description: A systems note on separating protocol routing from concrete tool transports.
tags: [ai-agents, agent-workflows]
categories: [technical-blog]
---

Tool use is often described as a model capability: the model decides to call a function, the function returns a result, and the conversation continues. That framing is useful for demos, but it hides the harder engineering question: once agents become networked entities, what is the durable system boundary between an agent protocol and the tools it can execute?

The ALN / Foundation Protocol MCP bridge gave me a concrete answer. The protocol layer should route an addressable request. The application layer should translate that request into the tool runtime. The tool transport should remain replaceable.

## The boundary that matters

In this design, `INVOKE` is not just a string in an enum. It is the protocol-level verb for "please execute an action." The payload carries two essential pieces of information: `method`, which names the tool behavior, and `params`, which carries the arguments. The bridge reads those fields and forwards them to an MCP client as a `tools/call`.

The important part is what the bridge does not do. It does not own the full agent policy. It does not decide whether the user goal is wise. It does not reinterpret the tool schema into a new private contract. Its job is narrow: preserve the protocol intent and make it executable in the MCP world.

**Insight:** a good tool bridge is deliberately boring. If it becomes clever, debugging becomes a fight between the agent policy, the protocol router, and the tool runtime. Keeping the bridge thin makes each layer accountable.

## Why transport belongs behind the adapter

MCP tools may run over stdio during local development or HTTP in a deployed service. That difference matters operationally, but it should not leak into the protocol verb. The caller should not care whether the entity is backed by a local command, a service URL, or a future transport. The entity metadata can describe the MCP configuration; the adapter can choose the transport; the protocol message can stay stable.

This is the same reason database drivers, message queues, and payment providers are usually hidden behind adapters. The application needs an execution contract. Deployment needs a transport. Mixing the two creates avoidable coupling.

## Design consequences

This small bridge changes how I think about agent infrastructure. Agent tool use should be inspectable as a sequence of typed events, not as a side effect inside a chat transcript. That means a production-ready system needs:

- stable entity identities, so tool calls are routed to the intended executor;
- explicit verbs such as `INVOKE`, so execution is distinguishable from normal conversation;
- structured payloads, so validation, replay, and audit are possible;
- transport adapters, so local tools and remote tools share the same protocol surface;
- clear ownership, so failure, escalation, and permission checks have a place to live.

## What I would build next

The next step is a stronger contract layer around tool invocation. I would add typed schemas for method names and params, structured error classes for transport failure versus tool failure, and a trace format that records the original protocol message, resolved entity, MCP request, MCP response, and final protocol reply.

That trace would make the bridge useful for research as well as engineering. It would let us compare tool routing strategies, detect brittle tools, and evaluate whether a model fails because it chose the wrong tool, filled the arguments incorrectly, or called a correct tool through a broken transport.

## What this project demonstrates

The value of this project is not only that it connects ALN / Foundation Protocol to MCP. It demonstrates a systems habit that matters in agent work: reduce a vague AI behavior into a small protocol contract, then make the contract observable, testable, and replaceable.
