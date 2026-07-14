---
layout: post
title: "A Spending Guardrail for Agentic Payments — Hands-On x402"
date: 2026-07-11
description: "Put a guardrail between your coding agent and its wallet: it can pay x402 paywalls, but only the right payee, the right amount, within budget, once. Hands-on with real testnet USDC, wired into Claude over MCP."
tags: [ai-agents, agentic-payments, x402, tutorial, security, base]
image: /assets/img/x402-social-card.png
permalink: /x402-payment-proxy/
---

## TL;DR

A local proxy holds the wallet and pays [x402](https://x402.org) paywalls for your coding agent. A guard inside it refuses wrong-payee, wrong-amount, over-budget, and duplicate payments; you spend the tutorial trying to beat it, then wire it into Claude over MCP. ~15 minutes, $0, testnet.

*Every command and output below was actually run against the live x402 demo server.*

Agents can pay for things now. [x402](https://x402.org) turns an `HTTP 402` into a real payment: the agent signs a stablecoin authorization and the paywall opens. The hard part is getting the payment right — correct payee, correct amount, within budget, and only once.

And a coding agent is steerable by everything it reads. A `402` header, a merchant's JSON, a comment in a doc it ingested three tool calls ago: any of these can nudge it toward the wrong address or an inflated charge. It's not hypothetical, either. [A prompt-injected agent wallet was drained of roughly $150k–200k](https://www.giskard.ai/knowledge/how-grok-got-prompt-injected-an-x-user-drained-150-000-from-an-ai-wallet) this way in May 2026. Asking the model to be careful doesn't fix this. What fixes it is a guardrail between the agent and the signature, one that checks every payment against rules you set and refuses to sign anything that fails.

This tutorial builds that guardrail with [`@themobiusstrip/agentpay-proxy`](https://www.npmjs.com/package/@themobiusstrip/agentpay-proxy), points it at a real x402 merchant, and then spends four of its six steps trying to get around it.

![Architecture: Claude calls paid_fetch on agentpay-proxy, which holds the wallet key. The proxy GETs the URL, gets an HTTP 402 naming price and payee, and its spending guard checks payee, amount, budget, and duplicates. If every rule passes it signs an EIP-3009 authorization and retries with payment; if any rule fails, nothing is signed and Claude gets an error. The website settles the payment on Base Sepolia, then returns the content to the proxy, which passes it back to Claude — so every payment and every response goes through the guard.](/assets/img/x402-proxy-architecture.svg)

The agent holds no key. All it can do is ask the proxy to fetch a URL. If the URL costs money, the proxy pays, but only after the guard checks that the payment is correct.

## What you'll need

- Node.js (any recent version; everything runs through `npx`)
- Two terminal windows, one for the proxy and one for you
- [Claude Code](https://claude.com/claude-code), but only for step 6; steps 1–5 are agent-free
- No crypto experience and no real money. The wallet is generated for you and funded from a free public faucet.

## Setup: start the proxy, fund its wallet

There's nothing to clone or configure:

```bash
npx @themobiusstrip/agentpay-proxy
```

The first run generates a **testnet** wallet and starts the proxy on `127.0.0.1:4020`. Three things to know about that wallet:

- It lives in a file, `.agentpay-proxy-wallet.json`, with permissions `600`. Keep it out of git.
- It starts empty. The proxy prints the wallet's address: copy it, then grab free Base Sepolia USDC for it at [faucet.circle.com](https://faucet.circle.com) (network: **Base Sepolia**, token: **USDC**). When I funded mine, the USDC showed up in under a minute.
- It never needs ETH. The payer only signs; the merchant's facilitator submits the settlement transaction and pays its own gas.

Leave the proxy running in this terminal. It's the only process that ever touches the key.

*(If you'd rather not deal with the wallet at all: every refusal in steps 2–5 happens before anything is signed, so all the blocked demos work on an unfunded wallet. Only the successful payments in steps 1, 3, and 5 need faucet funds.)*

## Step 1: pay a real merchant

Open your second terminal; from here on, you're playing the agent. Pay the public x402 demo server:

```bash
curl -s -X POST http://127.0.0.1:4020/paid-fetch \
  -H 'content-type: application/json' \
  -d '{"url":"https://x402.org/protected"}'
```

The guard's whole lifecycle shows in the proxy log:

```
[guard] reserved 10000 -> 0x209693bc…     # budget held atomically, BEFORE signing
[guard] decision allow/ok                 # every rule passed
[guard] signed                            # EIP-3009 authorization signed
[guard] settled                           # USDC moved; spend recorded in the window
```

A note on the numbers, because they confused me at first: amounts are in USDC base units, and USDC has six decimals. So `10000` is $0.01. The merchant charges a cent.

The response carries the content plus a receipt:

```json
{"status":200,"intentId":"448d359b-…","body":"…","settlement":"eyJzdWNjZXNz…"}
```

Copy the `settlement` value out of the response and decode it; that's the on-chain receipt:

```bash
echo '<settlement value>' | base64 -d
# {"success":true,"payer":"0x…","transaction":"0x3ba4b96d…","network":"eip155:84532"}
```

Look up that `transaction` hash on [sepolia.basescan.org](https://sepolia.basescan.org/tx/0x3ba4b96dd6e1d6311b644f918bc25b94903a9e6694d4b871a7369c58f94d8b8e) and you'll find a real $0.01 USDC transfer:

![Basescan transaction details for the payment: status Success on Base Sepolia testnet, with the ERC-20 transfer row showing 0.01 USDC moving to the demo merchant at 0x209693Bc…](/assets/img/x402-basescan-tx.png)

*The receipt, on-chain: 0.01 USDC to the merchant the `402` named.*

The guard looked at the payment, found it correct, and allowed it. The rest of the tutorial is about that judgment: what "correct" means, and what happens to a payment that isn't.

## What just happened

Here's the x402 flow. A paid page doesn't return content; it returns `HTTP 402` with a price tag attached: who to pay (`payTo`), how much, in which asset, on which network. The client answers with a signed permission slip — an EIP-3009 `transferWithAuthorization`, a typed-data signature that says "move this much of my USDC to this address, valid for this long." At that point nothing has actually moved, which is also why the payer needs no ETH. The client retries the request with the slip attached; the merchant's facilitator verifies the slip, settles it on-chain, and the content comes back.

The guard installs on the x402 client's pre-sign hook, in the gap between "the merchant wants payment" and "the slip is signed." There it runs a fixed set of checks. If any check fails, no signature is produced, and without a signature there is no payment.

Steps 2–5 each set a rule, then try to break it against the same live merchant. The rules are environment variables, so each step goes the same way: stop the proxy (Ctrl-C in the first terminal), restart it with the new setting, pay again. A restart also resets the spending window, so each step starts clean.

## Step 2: try to overspend it

The proxy holds a rolling-window budget, and the cap is yours to set. Restart it with the cap at `5000`, half a cent, below the merchant's one-cent price:

```bash
CAP=5000 npx @themobiusstrip/agentpay-proxy
```

Pay again, same curl as step 1:

```bash
curl -s -X POST http://127.0.0.1:4020/paid-fetch \
  -H 'content-type: application/json' \
  -d '{"url":"https://x402.org/protected"}'
```

The guard refuses:

```
Payment creation aborted:
  cap_exceeded: per-mandate cap 5000 would be exceeded (committed 0 + 10000)
[guard] decision block/cap_exceeded
```

Nothing was signed. However convincing the page was, the agent can't spend past the cap you set.

## Step 3: try to misdirect it

By default the proxy enforces budget only; it doesn't check who gets paid, and it says so at startup. To bind the recipient you give the proxy a mandate: a standing instruction from you, the human, saying what you actually authorize (here, "pay this payee"). Merchant demands are then checked against that instruction.

Restart in mandate mode, pinning the demo merchant's real payee. That's the address from the step-1 log, the one the merchant names in its `402`:

```bash
MANDATE=1 PIN_PAYTO=0x209693Bc6afc0C5328bA36FaF03C514EF312287C npx @themobiusstrip/agentpay-proxy
```

Pay again. The `402`'s `payTo` matches the address you authorized, so the guard allows it: `decision allow/ok`, settled on-chain.

Now restart with the pin pointed anywhere else, so that you've authorized paying someone the merchant's header doesn't name:

```bash
MANDATE=1 PIN_PAYTO=0x000000000000000000000000000000000000dEaD npx @themobiusstrip/agentpay-proxy
```

Pay again, and the guard blocks:

```
Payment creation aborted:
  intent_payto_mismatch: intent constraint failed: intent_payto_mismatch
[guard] decision block/intent_payto_mismatch
```

This is the core injection defense. No matter what a `402` header, a rewritten response, or a poisoned instruction asks for, the money can only go to the address you named.

## Step 4: try to overcharge it

Same mandate, plus a per-payment ceiling. `PIN_MAX=5000` means "at most half a cent per payment," again below the merchant's one-cent price:

```bash
MANDATE=1 PIN_PAYTO=0x209693Bc6afc0C5328bA36FaF03C514EF312287C PIN_MAX=5000 npx @themobiusstrip/agentpay-proxy
```

Pay again:

```
Payment creation aborted:
  intent_amount_exceeds: intent constraint failed: intent_amount_exceeds
[guard] decision block/intent_amount_exceeds
```

A merchant (or a tampered quote) that asks for more than you authorized doesn't get signed.

## Step 5: try to double-pay

Every `paid-fetch` is one purchase intent, and an intent pays out at most once. Restart the proxy with no flags (plain `npx @themobiusstrip/agentpay-proxy`), give the purchase a stable id, and pay:

```bash
curl -s -X POST http://127.0.0.1:4020/paid-fetch \
  -H 'content-type: application/json' \
  -d '{"url":"https://x402.org/protected","intentId":"order-42"}'
```

The first run returns `200` and settles. Now run the same command again. The guard refuses to sign:

```
Payment creation aborted:
  duplicate_authorization: duplicate authorization for payer:0x…|intent:order-42|…
```

Omit `intentId` and each request is a fresh intent (the proxy assigns a UUID and echoes it back), so distinct purchases never false-block. Pass the same id, and a retry of that purchase, whether from a stuttering agent loop or a replayed instruction, can't double-pay.

## Step 6: hand it to Claude

Everything so far you did with curl, playing the agent yourself. Time to hand the tool to Claude.

The proxy speaks MCP through a thin forwarder that holds no keys; it relays every call to the HTTP proxy you've been running (`PROXY_URL`, default `http://127.0.0.1:4020`). Make sure that proxy is running, plain, no flags, then register the forwarder with [Claude Code](https://claude.com/claude-code):

```bash
claude mcp add paywall -- npx @themobiusstrip/agentpay-proxy mcp
```

Open Claude Code and run `/mcp`. You'll see `paywall` listed, with a single tool, `paid_fetch`. Ask:

> Fetch https://x402.org/protected with paid_fetch and summarize it.

Claude calls the tool, the proxy pays within the guardrail, and Claude reports back on what it just bought. Verbatim, from my run:

```
● Fetch success, HTTP 200. Proxy paid x402 micropayment, settlement confirmed.

  Page content: simple "Protected Content" page. Says "Your payment was
  successful! Enjoy this banger song." Only real content: embedded SoundCloud
  player — track "x402 (DJ Reppel Remix)" by danXkim. Next.js app, nothing
  else on page.

  Settlement (decoded):
  - Payer: 0x78855d5a5816B15fB042c0F0F88cb3b2D22626f8
  - Tx: 0xf427902720ba4dbf3ea9759e1e63e900f9d34b09488d46623180556107e3b221
  - Network: eip155:84532 (Base Sepolia)
  - Intent ID: d2dfa825-777d-4d61-b830-53649c77f9c2

  Page is demo/reward page for x402 testnet — proves payment flow end-to-end,
  content itself is joke song.
```

So one cent buys a joke song. The content is silly; the receipt is not — payer, transaction, network, intent id, all decoded from the settlement that came back with the page.

Now make it fail. Restart the proxy with step 2's cap (`CAP=5000 npx @themobiusstrip/agentpay-proxy`) and ask the same thing. This time the tool hands Claude the machine-readable refusal instead of the content. Again verbatim, lightly trimmed:

```
● Re-fetch with same intentId — avoid double payment.

  Called paywall

  Re-fetch blocked by proxy guard: cap_exceeded: per-mandate cap 5000 would
  be exceeded (committed 0 + 10000) — page price 10000 base units (0.01 USDC),
  cap now 5000 (0.005). No payment made.

  […]

  Note: first fetch paid fine at same price. Guard policy apparently
  tightened between calls (cap 5000 < price 10000). Want retry with fresh
  intentId, or check proxy policy config?
```

Worth reading twice. Claude behaves like a good agent the whole way: it reuses the intent id so a retry can't double-pay, reads the refusal correctly, notices the policy changed between calls, and proposes workarounds. The workarounds go nowhere — a fresh `intentId` meets the same cap. The agent can ask, suggest, or get prompt-injected into trying harder; whether the payment is correct was never up to the model.

## The rules you set

Everything is env-driven, and as everywhere in this tutorial, amounts are USDC base units. The values below are the defaults (the mandate block is off unless you set it). A malformed money knob throws at startup rather than falling back to a looser rule.

```bash
# budget, reserved atomically before anything is signed
CAP=100000              # rolling-window budget, $0.10
AGG_CAP=200000          # cap across all mandates, $0.20; stops slow drain
WINDOW_MS=300000        # the rolling window itself, 5 minutes

# mandate, off by default; set these to bind recipient and price
MANDATE=1
PIN_PAYTO=0x…           # pay only this address
PIN_MAX=10000           # at most this much per payment, $0.01

# scope and plumbing
CEILING_S=300           # longest authorization lifetime the guard will sign
ALLOWED_HOSTS=x402.org  # unset = any host; when set, redirects are refused too
HOST=127.0.0.1          # loopback only; this process holds a key
PORT=4020
```

One knob is worth a closer look. The demo merchant asks for a 300-second authorization lifetime. Set `CEILING_S=30` and the guard refuses pre-sign with `valid_before_too_far: requested validity 300s exceeds ceiling 30s`. The reason to care: a long-lived signed authorization can settle long after your budget window has moved on, and the ceiling closes that exposure. (The effective ceiling is `min(CEILING_S, WINDOW_MS/1000)`, so the window bounds it too.) The default accepts 300s so the public demo works; tighten it for merchants you control.

If you'd rather embed the guard than run the CLI, `createPaymentProxy(payerKey, config, hooks)` returns the Express app plus the guard. Bring your own audit sink, your own mandate verifier (constraints must come from outside the model), and a shared store when you run multiple workers.

## What the guardrail is not

Three limits worth knowing before you rely on it:

- It hardens one topology: tool-mediated payments with an out-of-process signer. It is not a sandbox. An agent with shell access to the proxy's host can read the wallet file, so isolate accordingly.
- Without `ALLOWED_HOSTS`, `paid_fetch` is an open server-side fetch: it can reach anything the proxy's network can, paid or not. That's why it binds to loopback. Set the allowlist and authenticate the hop before you expose it.
- Everything here is pinned to the tested envelope: `exact` + Base Sepolia + USDC, `@x402/*@2.17.0`. Mainnet means real money, a new envelope, a mainnet facilitator, and caps you'd be comfortable losing to a bug in your own config.

## When it doesn't work

Most of these I hit myself while putting this together:

`{"status":402}` after `[guard] signed`
: payer unfunded, so the facilitator rejected it at verify. Fund the wallet from the faucet and retry.

`valid_before_too_far`
: merchant wants a longer authorization than your policy allows. If you decide that's fine, raise both `CEILING_S` and `WINDOW_MS`.

`duplicate_authorization … intent:<id>`
: same `intentId` after a signature already exists for it. Correct refusal; omit `intentId` if it's a new purchase.

`cap_exceeded` on honest traffic
: window spent. Wait out `WINDOW_MS` or raise `CAP`. Pending holds count against the cap until they resolve; that's intentional.

`intent_payto_mismatch` when you expected a pay
: your `PIN_PAYTO` doesn't match the merchant's `402` payee. Read the header and pin what it actually asks for (if you trust it).

`envelope_network` on a discovered endpoint
: that endpoint is on Base **mainnet**, outside the testnet envelope on purpose.

## Where this leaves you

You now have a proxy that pays x402 paywalls from a key the agent never sees, with a guard that refuses the wrong payee, the wrong amount, anything over budget, and any double-pay. Claude is wired to it through a single tool.

The agent itself is no more trustworthy than it was an hour ago; it's still steerable by anything it reads. What changed is that the steering no longer reaches the money. Every payment gets checked below the model, before a signature exists.

**Don't ask the agent to pay carefully. Build the layer that only lets it pay correctly.**

The proxy is [`@themobiusstrip/agentpay-proxy`](https://www.npmjs.com/package/@themobiusstrip/agentpay-proxy). The guard inside it lives in [theMobiusStrip/agentpay-guard](https://github.com/theMobiusStrip/agentpay-guard), along with internals, an embeddable API, and DrainBench, which measures the same failure modes in dollars.
