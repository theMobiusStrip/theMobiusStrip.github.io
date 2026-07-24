---
layout: page
title: About
permalink: /about/
---

This is my learning blog — a public notebook of things I'm studying and building. Lately that means AI coding agents: what to trust them with, what not to, and the guardrails worth putting underneath them.

Everything I post here is something I actually ran or built; the posts include the commands and their real output.

## Things I've built

[perch](/perch/)
: A read-only, local-only, open-source watcher for coding agents. It sits in the macOS menu bar and shows you what your agents are doing — because an agent can't be the one telling you it's safe. The thinking behind it is in [the essay](/your-coding-agent-will-always-tell-you-its-safe/).

[agentpay-proxy](https://www.npmjs.com/package/@themobiusstrip/agentpay-proxy)
: A deployment wrapper for the main agentpay-guard artifact. It pays x402 paywalls while enforcing per-payment ceilings, restart-safe budgets, and duplicate protection below the agent. Guard internals live in [agentpay-guard](https://github.com/theMobiusStrip/agentpay-guard); the walkthrough is [the tutorial](/x402-payment-proxy/).

## Elsewhere

- GitHub: [@theMobiusStrip](https://github.com/theMobiusStrip)
- RSS: [feed.xml](/feed.xml)
- Posts are plain Markdown in [`_posts/`](https://github.com/theMobiusStrip/theMobiusStrip.github.io/tree/main/_posts)
