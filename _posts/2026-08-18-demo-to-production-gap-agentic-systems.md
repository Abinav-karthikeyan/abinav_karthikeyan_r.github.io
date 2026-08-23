---
layout: post
title: "Notes on the demo-to-production gap for agentic systems"
date: 2026-08-18 09:00:00 +0000
categories: [agentic-ai, engineering]
---

A working demo of an agent is a spectacular thing. You give it a prompt, it reasons, it calls a couple of tools, it produces an answer that would have taken a person half an hour. It feels like the work is basically done.

Then you try to put it in front of real users, and the gap opens up.

Most of what I spend my time on these days sits inside that gap. Not the model, not the prompt — the scaffolding that turns a plausible response into a system a business can rely on. A few things I keep coming back to:

## Context is a stack, not a prompt

The naive view is that context is "whatever you stuff into the system prompt." That falls over the moment you have more than one kind of user, more than one workflow, or any notion of domain that changes region-to-region or client-to-client.

What actually works is treating context as layered: something like an organisation layer that never changes, a domain layer that captures the vocabulary and rules of what you're doing, a session layer that carries the user's intent through the conversation, and a task layer that scopes what the agent is doing right now. Each layer has its own lifecycle. Each has its own owner. The retrieval story is different at each level. Muddling them is where most "why is the agent hallucinating this policy" bugs come from.

## The tool surface *is* the product

Once an agent has tools, the tools are the API. If your tool descriptions are sloppy, the agent misuses them. If two tools overlap, the agent picks the wrong one under load. If a tool has a side effect that's not obvious from its signature, you will find out in production.

MCP has made this easier to reason about because it forces you to publish tools as a first-class contract. But the discipline is the same whether you're using MCP or not: treat each tool the way you'd treat a public endpoint. Version it. Document its failure modes. Assume the agent will call it with the worst plausible arguments.

## Evals are load-bearing, not optional

If you cannot measure whether a change made the agent better, you cannot ship changes safely. That's the whole game. The bar isn't "we ran it on ten examples and it looked good" — it's a suite that runs on every change, catches regressions, and lets you argue about improvements with numbers instead of vibes.

The hardest part is building the eval set. Real production traffic beats synthetic every time, but you need to be careful about PII, and you need the humans who know the domain to label. Which brings me to the last point.

## The bottleneck is almost never the model

It's requirements. It's the gap between what the sales team said the users need and what the users actually do. It's the fact that half the documents you're supposed to retrieve from are misfiled. It's the workflow where step three requires a human decision that nobody has written down.

The agent is the easy part. The system around it is the work.

---

*More on the context stack specifically in a follow-up.*
