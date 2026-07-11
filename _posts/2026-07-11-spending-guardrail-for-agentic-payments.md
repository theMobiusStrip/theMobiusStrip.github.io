---
layout: post
title: "A Spending Guardrail for Agentic Payments — Hands-On x402"
date: 2026-07-11
description: "Put a guardrail between your coding agent and its wallet so it can pay x402 paywalls, but only correctly — right payee, right amount, within budget, exactly once. Hands-on with real testnet USDC, wired into Claude over MCP."
tags: [ai-agents, agentic-payments, x402, tutorial, security, base]
permalink: /x402-payment-proxy/
---

*~10 min, hands-on. $0 — Base Sepolia testnet. Every command and output below was actually run against the live x402 demo server.*

Agents can pay for things now. [x402](https://x402.org) turns an `HTTP 402` into a real payment: the agent signs a stablecoin authorization and the paywall opens. The hard part isn't *paying* — it's paying **correctly**. The right payee. The right amount. Within a budget. Once, not five times.

And a coding agent is steerable by everything it reads. A `402` header, a merchant's JSON, a comment in a doc it ingested three tool-calls ago — any of them can nudge it toward the wrong address or an inflated charge. It's not hypothetical: [a prompt-injected agent wallet was drained of ~$150k–200k](https://www.giskard.ai/knowledge/how-grok-got-prompt-injected-an-x-user-drained-150-000-from-an-ai-wallet) this way in May 2026. You don't fix that by asking the model to be careful. You put a **guardrail** between the agent and the signature — one that checks every payment against rules you set, and refuses to sign anything that fails.

This tutorial builds that guardrail with [`@themobiusstrip/agentpay-proxy`](https://www.npmjs.com/package/@themobiusstrip/agentpay-proxy), points it at a real x402 merchant, and watches it enforce correctness below the model.

```
agent (Claude, no keys)
   │  one tool call: paid_fetch(url)
   ▼
agentpay-proxy                               ← x402 client + spending guard + signer
   │  402 → guard checks every rule → sign EIP-3009 → retry with payment
   ▼
paid website (x402)   e.g. https://x402.org/protected
   │  verify + settle
   ▼
facilitator (x402.org)  →  USDC moves on Base Sepolia
```

The agent holds no key. It gets one capability: *ask the proxy to fetch a URL*. If the URL costs money, the proxy pays — but only after the guard confirms the payment is correct. The agent can ask. The guard decides.

## Step 1 — pay a real merchant

Two commands, no clone, no config.

```bash
npx @themobiusstrip/agentpay-proxy
```

First run generates a **testnet** wallet (`.agentpay-proxy-wallet.json`, mode 600 — keep it out of git) and prints where to fund it: grab Base Sepolia USDC at [faucet.circle.com](https://faucet.circle.com). No ETH needed — the payer only *signs*; the facilitator submits the settlement transaction and pays its own gas.

Second terminal — you're the agent, paying the public x402 demo server:

```bash
curl -s -X POST http://127.0.0.1:4020/paid-fetch \
  -H 'content-type: application/json' \
  -d '{"url":"https://x402.org/protected"}'
```

The guard's whole lifecycle shows in the proxy log:

```
[guard] reserved 10000 -> 0x209693bc…     budget held atomically, BEFORE signing
[guard] decision allow/ok                 every rule passed
[guard] signed                            EIP-3009 authorization signed
[guard] settled                           USDC moved; spend recorded in the window
```

And the response carries the content plus a receipt:

```json
{"status":200,"intentId":"448d359b-…","body":"…","settlement":"eyJzdWNjZXNz…"}
```

Decode the receipt for the transaction hash and look it up on [sepolia.basescan.org](https://sepolia.basescan.org):

```bash
echo '<settlement value>' | base64 -d
# {"success":true,"payer":"0x…","transaction":"0xa5d604b5…","network":"eip155:84532"}
```

A real $0.01 USDC transfer, paid by a process the agent can only *ask*. The guard looked at it, found it correct, and allowed it. The rest of the tutorial is about that judgment — what "correct" means, and what happens when a payment isn't.

## What the guard is doing

The guard installs on the x402 client's **pre-sign** hook. Between "the merchant wants payment" and "the authorization is signed," it runs a fixed set of checks. If any fails, no signature is produced — the payment simply never happens.

x402 itself, in three sentences: a paid site answers with `HTTP 402` and a header naming the price (scheme, network, asset, amount, payee); the client signs an **EIP-3009 `transferWithAuthorization`** for exactly that — a typed-data signature, not a transaction — and retries; the site's **facilitator** verifies and settles it on-chain. The guard's job is to vet what gets signed in the middle step.

Here's each guarantee, shown against the **same** live merchant.

### Guarantee 1 — within budget

The proxy holds a rolling-window budget. Set a cap below the merchant's price and it won't be talked past it — `$0.005` against the merchant's `$0.01`:

```bash
CAP=5000 npx @themobiusstrip/agentpay-proxy
```

Pay again:

```
Payment creation aborted:
  cap_exceeded: per-mandate cap 5000 would be exceeded (committed 0 + 10000)
[guard] decision block/cap_exceeded
```

Nothing was signed. Whatever the agent was told, the loss is bounded in dollars you chose.

### Guarantee 2 — the recipient you authorized

By default the proxy enforces budget only — it does not check *who* gets paid, and it says so. To bind the recipient, run in **mandate** mode and pin the address you actually intend to pay:

```bash
MANDATE=1 PIN_PAYTO=0x209693Bc6afc0C5328bA36FaF03C514EF312287C npx @themobiusstrip/agentpay-proxy
```

That's the merchant's real payee (it's in the `402`). Now the guard checks the price header's `payTo` against the address *you* authorized. Matches → it pays, settles on-chain, `decision allow/ok`. Point the same pin at a **different** address — i.e. you authorized paying someone the merchant's header doesn't name — and it blocks:

```
Payment creation aborted:
  intent_payto_mismatch: intent constraint failed: intent_payto_mismatch
[guard] decision block/intent_payto_mismatch
```

This is the injection defense stated positively: the money goes to the address *you* named, not to whatever a `402` header, a rewritten response, or a poisoned instruction asks for. If they disagree, the guard sides with you.

### Guarantee 3 — the amount you authorized

Same mandate, add a per-payment ceiling below the charge — `PIN_MAX=5000` against the `$0.01` price:

```
Payment creation aborted:
  intent_amount_exceeds: intent constraint failed: intent_amount_exceeds
[guard] decision block/intent_amount_exceeds
```

A merchant (or a tampered quote) that asks for more than you authorized doesn't get signed.

### Guarantee 4 — exactly once

Every `paid-fetch` is one purchase intent. Give the purchase a stable id, pay once, then run the exact same command again:

```bash
curl -s -X POST http://127.0.0.1:4020/paid-fetch \
  -H 'content-type: application/json' \
  -d '{"url":"https://x402.org/protected","intentId":"order-42"}'
```

First run: `200`, settled. Second, the guard refuses to sign:

```
Payment creation aborted:
  duplicate_authorization: duplicate authorization for payer:0x…|intent:order-42|…
```

Omit `intentId` and each request is a fresh intent (a UUID, echoed back), so distinct purchases never false-block; pass the same one and a retry of that purchase can't double-pay.

All four checks run **before** any signature, so you can explore every one of them on an unfunded wallet.

## Step 2 — hand it to Claude

The proxy speaks MCP through a forwarder that holds no keys — it relays to the step-1 proxy (`PROXY_URL`, default `http://127.0.0.1:4020`), so keep that process running:

```bash
claude mcp add paywall -- npx @themobiusstrip/agentpay-proxy mcp
```

Claude now has one tool, `paid_fetch`. Ask it:

> Fetch https://x402.org/protected with paid_fetch and summarize it.

It pays within the guardrail and reads the content. When a rule fails, Claude gets the machine-readable reason — `cap_exceeded`, `intent_payto_mismatch` — instead of the content, and there's nothing it can say to change that. The correctness of the payment doesn't depend on the model's judgment, which is the entire point.

## The rules you set

Everything is env-driven. A malformed money knob throws at startup rather than falling back to a looser rule.

| env | default | the guarantee it sets |
| --- | --- | --- |
| `CAP` | `100000` ($0.10) | rolling-window budget; reserved atomically before signing |
| `AGG_CAP` | `200000` ($0.20) | budget across *all* mandates — stops slow drain |
| `WINDOW_MS` | `300000` | the rolling window itself |
| `CEILING_S` | `300` | max authorization lifetime signed — effective `min(CEILING_S, WINDOW_MS/1000)` |
| `MANDATE=1` + `PIN_PAYTO`, `PIN_MAX` | off | bind the recipient and the per-payment amount |
| `ALLOWED_HOSTS` | any | which hosts `paid_fetch` may reach; when set, redirects are refused |
| `HOST` / `PORT` | `127.0.0.1` / `4020` | loopback by default — this process holds a key |

One knob is worth understanding. The demo merchant asks for a **300-second** authorization lifetime. Set `CEILING_S=30` and the guard refuses pre-sign — `valid_before_too_far: requested validity 300s exceeds ceiling 30s` — because a long-lived signed authorization can settle long after your budget window has moved on, and that exposure is exactly what the ceiling exists to close. The default accepts 300s so the public demo works; tighten it for merchants you control.

Prefer to embed the guard rather than run the CLI? `createPaymentProxy(payerKey, config, hooks)` returns the Express app plus the guard — bring your own audit sink, a real mandate verifier (constraints must come from *outside* the model), and a shared store when you run multiple workers.

## What the guardrail is not

Saying it plainly beats discovering it later:

- It hardens **tool-mediated payments with an out-of-process signer** — this topology. It is not a sandbox. An agent with shell access to the proxy's host can read the wallet file; isolate accordingly.
- Without `ALLOWED_HOSTS`, `paid_fetch` is an open server-side fetch — it can reach anything the proxy's network can, paid or not. That's why it binds to loopback. Set the allowlist and authenticate the hop before you expose it.
- Everything here is pinned to the tested envelope: `exact` + Base Sepolia + USDC, `@x402/*@2.17.0`. Mainnet means real money — a new envelope, a mainnet facilitator, and caps you'd be comfortable losing to a bug in your own config.

## When it doesn't work

| symptom | cause / fix |
| --- | --- |
| `{"status":402}` after `[guard] signed` | payer unfunded — facilitator rejected at verify. Faucet, retry. |
| `valid_before_too_far` | merchant wants a longer authorization than policy allows. Raise `CEILING_S` *and* `WINDOW_MS` — deliberately. |
| `duplicate_authorization … intent:<id>` | same `intentId` after a signature already exists for it. Correct refusal; omit `intentId` for a new purchase. |
| `cap_exceeded` on honest traffic | window spent. Wait `WINDOW_MS` or raise `CAP`. Pending holds occupy the cap until terminal — by design. |
| `intent_payto_mismatch` when you expected a pay | your `PIN_PAYTO` doesn't match the merchant's `402` payee. Read the header; pin what it actually asks (if you trust it). |
| `envelope_network` on a discovered endpoint | it's Base **mainnet**. Outside the testnet envelope on purpose. |

## The point

The agent in this setup is exactly as trustworthy as before — steerable by anything it reads. What changed is that its steering no longer reaches the money. Correctness isn't something you hope the model gets right; it's checked, below the model, on every payment, before a signature exists.

**Don't ask the agent to pay carefully. Build the layer that only lets it pay correctly.**

The proxy is [`@themobiusstrip/agentpay-proxy`](https://www.npmjs.com/package/@themobiusstrip/agentpay-proxy); the guard inside it — internals, an embeddable API, and DrainBench, the same failure modes measured in dollars — lives in [theMobiusStrip/agentpay-guard](https://github.com/theMobiusStrip/agentpay-guard).
