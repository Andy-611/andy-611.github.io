---
layout: post
title: "Provider CLIs are not just backend workers"
date: 2026-06-28 10:30:00
description: A concrete design lesson from tracing an ALN message loop through queue handlers and provider CLIs.
tags: [ai-agents, systems-design]
categories: [technical-blog]
---

One of the most useful architecture lessons I got this month came from reading the current ALN message loop closely enough to explain it to a beginner. The original question was simple: why does the system feel awkward when it talks to tools like Codex or Claude through their CLIs?

The answer was not "the model is weak" or "the prompt is wrong." The answer was architectural. The runtime currently treats provider CLIs as backend workers. That choice is understandable, but it creates a mismatch once the desired experience becomes conversational, stateful, and visibly interactive.

## The concrete pipeline

The current flow is roughly:

`human/web/cli -> Host.route_mail() -> Entity.receive_mail() -> mailbox persistence -> AgentHandler.handle() -> CLIAdapter.run_turn() -> subprocess.run(provider CLI) -> parse stdout -> send reply back`

At first glance, this looks clean. Each layer has a narrow role. The protocol host routes mail. The entity stores it. The agent handler manages queue and session logic. The CLI adapter shells out to the provider command. The reply comes back as normal protocol traffic.

That design is good at one thing: turning a provider into a replaceable executor. If all you want is "take this request, run a command, return text," the boundary is easy to reason about.

## Where the discomfort starts

The trouble appears when the provider is not really a stateless worker. Tools like Codex and Claude are better understood as interactive environments. They have their own session model, UI assumptions, resume behavior, and human-facing rhythm. Treating them as one-shot subprocesses hides that nature behind a narrow batch interface.

Once I framed the problem that way, several symptoms made immediate sense:

- The runtime can route messages, but it does not naturally expose the provider's own live inbox and outbox.
- Session continuity becomes awkward because the true conversation state partly lives inside the provider environment, not only inside ALN mailboxes.
- Debugging becomes blurry because a failure may belong to the protocol layer, the queue handler, the subprocess boundary, or the provider's own interaction model.
- The web layer can stream messages over WebSocket, but the provider integration itself is still shaped like request-response batch work.

None of these are exotic bugs. They are signs of a boundary mismatch.

## The key design correction

The most important correction was conceptual: stop asking how to make the worker pipeline slightly smarter, and ask whether the provider should own the interactive surface in the first place.

That shift sounds abstract, but it changes the design conversation. If the provider is fundamentally an interactive agent environment, then ALN should not pretend it is merely a hidden function call with text output. Instead, ALN should decide which responsibilities belong to the protocol runtime and which belong to the provider's own session loop.

This creates two clearly different architectures:

Architecture A: provider CLI as backend worker.
ALN owns the main interaction loop, and the provider is invoked as a subprocess when needed.

Architecture B: provider environment as live surface.
The provider owns more of the session experience, while ALN becomes the routing, identity, storage, and coordination layer around it.

I do not think one of these is universally correct. But I do think teams lose time when they speak as if they are building Architecture B while the code is still firmly Architecture A.

## Why this matters for agent systems

This lesson generalizes beyond one codebase. A lot of agent infrastructure feels confusing because there are two very different kinds of abstractions hiding under the same vocabulary.

Sometimes an "agent" is really a backend capability: receive input, perform work, emit output.

Sometimes an "agent" is an environment with continuity: memory, interface expectations, interruption points, resumability, and a user-visible sense of being "in conversation."

The engineering mistake is to mix those two without naming the difference. Once that happens, every improvement request becomes ambiguous. "Make it real-time." "Let it resume." "Show the current state." "Make the messages feel natural." Those are not minor polish tasks. They are requests about where the true interaction surface lives.

## What I would change next

If I were evolving this design, I would make the boundary explicit before adding more behavior.

First, I would declare whether each provider integration is batch-oriented or session-oriented. That should be configuration, not folklore.

Second, I would record traces that separate protocol events from provider events. A good trace should show routed mail, queue decisions, subprocess calls, provider responses, and what the user actually saw.

Third, I would resist burying interactive assumptions inside adapter code. Adapters should be thin. If a provider needs persistent sessions, human checkpoints, or live event streaming, those should appear as first-class runtime concepts.

## What this debugging story taught me

The most valuable outcome was not a code patch. It was a better question. When an agent runtime feels clumsy, ask whether the system is forcing an interactive environment into a backend-worker abstraction.

That question is small, but it is powerful. It turns a vague complaint about "agent UX" into a concrete systems diagnosis: the wrong component may own the conversation.
