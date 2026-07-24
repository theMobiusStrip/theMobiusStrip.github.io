---
layout: post
title: "A Spending Guardrail for Agentic Payments — Hands-On x402"
date: 2026-07-11
description: "Let Claude pay an x402 testnet paywall without holding the wallet key, then test payment limits, rolling budgets, and duplicate protection."
tags: [ai-agents, agentic-payments, x402, tutorial, security, base]
image: /assets/img/x402-social-card.png
permalink: /x402-payment-proxy/
---

## TL;DR

I wanted to test one deployment question: can Claude open an x402 paywall without ever seeing the wallet key? The setup I ended up with is small. Claude gets one tool, `paid_fetch`; a local proxy owns the key; [`agentpay-guard`](https://www.npmjs.com/package/@themobiusstrip/agentpay-guard) gets the last word before anything is signed.

This post pays the public testnet demo once, then tries three failures: one payment that is too large, two smaller payments that cross the budget, and the same purchase twice after a restart. The last run hands the tool to Claude over MCP. About 15 minutes, no real money.

*Commands use `@themobiusstrip/agentpay-proxy@0.0.4`, `agentpay-guard@0.0.5`, and the public x402 Base Sepolia demo.*

[x402](https://x402.org) handles the payment handshake. It does not decide whether a merchant's request is sensible. That matters when the client is a coding agent that can be steered by a page, a tool response, or an old comment in its context. A prompt-injected agent wallet [lost roughly $150k–200k](https://www.giskard.ai/knowledge/how-grok-got-prompt-injected-an-x-user-drained-150-000-from-an-ai-wallet) in May 2026. The signer needs its own rules.

`agentpay-guard` is the policy plugin. [`agentpay-proxy`](https://www.npmjs.com/package/@themobiusstrip/agentpay-proxy) is the runnable CLI/MCP wrapper I use here to keep the signer outside the agent process.

![Architecture: Claude asks agentpay-proxy to fetch a URL and holds no key. The proxy deploys agentpay-guard before its signer. The guard enforces the tested payment envelope, per-payment maximum, rolling budget, authorization lifetime, and duplicate-authorization protection. A pass reaches the local EIP-3009 signer; a block returns to Claude. SQLite keeps policy state across restarts. The proxy completes the x402 handshake with the testnet site, which settles on Base Sepolia.](/assets/img/x402-proxy-architecture-signer.svg)

Claude receives `paid_fetch`, not the key. A paid request reaches the signer only after the guard accepts it.

## What you'll need

- Node.js **22.13.0 or newer** (`node:sqlite` is required)
- Two terminal windows, one for the proxy and one for you
- [Claude Code](https://claude.com/claude-code), but only for step 5; steps 1–4 are agent-free
- No real money. The wallet is generated for you and funded from a free testnet faucet.

## Setup: start the proxy, fund its wallet

There's nothing to clone. I use a scratch directory so the wallet and SQLite files do not land in another project:

```bash
mkdir agentpay-guard-lab
cd agentpay-guard-lab
mkdir state

STATE_DB=state/step1.sqlite \
ALLOWED_HOSTS=x402.org \
npx --yes @themobiusstrip/agentpay-proxy@0.0.4
```

The first run creates a **testnet** wallet, opens `state/step1.sqlite`, and starts listening on `127.0.0.1:4020`.

- The key lives in `.agentpay-proxy-wallet.json` with permissions `600`. Keep it out of git.
- It starts empty. The proxy prints the wallet's address: copy it, then grab free Base Sepolia USDC for it at [faucet.circle.com](https://faucet.circle.com) (network: **Base Sepolia**, token: **USDC**). When I funded mine, the USDC showed up in under a minute.
- It never needs ETH. The payer only signs; the merchant's facilitator submits the settlement transaction and pays its own gas.

Leave the proxy running in this terminal. It is the only process here that touches the key.

Each experiment below gets its own database so the results are independent. Restart with the same path and the budget and duplicate state remain. In production, keep one durable database and run one proxy process against it.

*(If you'd rather not deal with the wallet at all: the refusal in step 2 happens before anything is signed, so that demo works on an unfunded wallet. Successful payments in steps 1, 3, 4, and 5 need faucet funds.)*

## Step 1: pay the public testnet demo

Open the second terminal. For now, you are the agent:

```bash
curl -s -X POST http://127.0.0.1:4020/paid-fetch \
  -H 'content-type: application/json' \
  -d '{"url":"https://x402.org/protected"}' \
  -w '\nHTTP %{http_code}\n'
```

The proxy log shows the whole trip:

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

The payment worked. Now try to make the signer do something it should refuse.

## Where the guard sits

An x402 page first returns `HTTP 402` with a price: payee, amount, asset, and network. The client signs an EIP-3009 `transferWithAuthorization` and retries. The merchant's facilitator settles it and returns the content.

The useful gap is between receiving that price and signing it. The guard runs there. A refusal means no authorization exists for the facilitator to settle.

For each test, stop the proxy with Ctrl-C and restart it with the shown settings. A new database makes a clean lab. Restarting the same database does **not** reset policy.

## Step 2: try one payment above its maximum

Start with the easiest failure. The page costs `10000` base units, so restart the proxy with `MAX_PAYMENT=5000`:

```bash
MAX_PAYMENT=5000 \
STATE_DB=state/step2.sqlite \
ALLOWED_HOSTS=x402.org \
npx --yes @themobiusstrip/agentpay-proxy@0.0.4
```

Pay again, same curl as step 1:

```bash
curl -s -X POST http://127.0.0.1:4020/paid-fetch \
  -H 'content-type: application/json' \
  -d '{"url":"https://x402.org/protected"}' \
  -w '\nHTTP %{http_code}\n'
```

The actual response, wrapped for readability:

```text
HTTP 502
error:
  Failed to create payment payload: Payment creation aborted:
  payment_amount_exceeds: payment amount 10000 exceeds max 5000
intentId: …
```

It stops before the wallet signs. This check needs no mandate, payee address, or funded wallet.

It only limits one authorization. Several smaller payments can still add up, so the next test uses the rolling budget.

## Step 3: try to drain it in smaller payments

Set the five-minute cap to `15000`: enough for one payment, not two. The requests use different intent ids so they count as separate purchases:

```bash
CAP=15000 \
STATE_DB=state/step3.sqlite \
ALLOWED_HOSTS=x402.org \
npx --yes @themobiusstrip/agentpay-proxy@0.0.4
```

Run both requests:

```bash
curl -s -X POST http://127.0.0.1:4020/paid-fetch \
  -H 'content-type: application/json' \
  -d '{"url":"https://x402.org/protected","intentId":"budget-1"}' \
  -w '\nHTTP %{http_code}\n'

curl -s -X POST http://127.0.0.1:4020/paid-fetch \
  -H 'content-type: application/json' \
  -d '{"url":"https://x402.org/protected","intentId":"budget-2"}' \
  -w '\nHTTP %{http_code}\n'
```

The first returns `200`. The second would take the window total to `20000`:

```text
HTTP 502
error:
  Failed to create payment payload: Payment creation aborted:
  cap_exceeded: per-mandate cap 15000 would be exceeded
  (committed 10000 + 10000)
intentId: budget-2
```

First payment passed; second did not. `MAX_PAYMENT` bounds one signature. `CAP` bounds the total.

## Step 4: try to authorize twice

Next, reuse one purchase id. Start a clean database:

```bash
STATE_DB=state/step4.sqlite \
ALLOWED_HOSTS=x402.org \
npx --yes @themobiusstrip/agentpay-proxy@0.0.4
```

Then, in the second terminal:

```bash
curl -s -X POST http://127.0.0.1:4020/paid-fetch \
  -H 'content-type: application/json' \
  -d '{"url":"https://x402.org/protected","intentId":"order-42"}' \
  -w '\nHTTP %{http_code}\n'
```

The first run returns `200`. Run the same command again:

```text
HTTP 502
error:
  Failed to create payment payload: Payment creation aborted:
  duplicate_authorization: duplicate authorization for
  payer:0x…|intent:order-42|…
intentId: order-42
```

The stable id tells the guard this is a retry, not a new purchase. Omit it and the proxy creates a fresh UUID. The next step keeps the id and restarts the process.

This prevents a second client signature. Merchant-side replay is separate; [`@themobiusstrip/x402-idempotency-middleware`](https://www.npmjs.com/package/@themobiusstrip/x402-idempotency-middleware) handles that side using the signed EIP-3009 authorization.

## Step 5: hand it to Claude

So far, curl has played the agent. Now give the tool to Claude.

The package includes an MCP forwarder with no key of its own. It relays calls to the HTTP proxy at `PROXY_URL` (default `http://127.0.0.1:4020`). Start one last clean database:

```bash
STATE_DB=state/step5.sqlite \
ALLOWED_HOSTS=x402.org \
npx --yes @themobiusstrip/agentpay-proxy@0.0.4
```

Register the same package with [Claude Code](https://claude.com/claude-code):

```bash
claude mcp add paywall -- \
  npx --yes @themobiusstrip/agentpay-proxy@0.0.4 mcp
```

Run `/mcp` in Claude Code. `paywall` should expose one tool, `paid_fetch`. Ask:

> Fetch https://x402.org/protected with paid_fetch using intentId `claude-demo-1`, then summarize it.

Claude calls the tool and reports what came back. My run looked like this:

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
  - Intent ID: claude-demo-1

  Page is demo/reward page for x402 testnet — proves payment flow end-to-end,
  content itself is joke song.
```

So one cent buys a joke song. The useful part is the receipt: payer, transaction, network, and intent id came back with the page.

Stop the HTTP proxy and restart it with the **same** database:

```bash
STATE_DB=state/step5.sqlite \
ALLOWED_HOSTS=x402.org \
npx --yes @themobiusstrip/agentpay-proxy@0.0.4
```

Within the default five-minute authorization horizon, ask:

> Fetch https://x402.org/protected again with paid_fetch using the same intentId `claude-demo-1`.

This time the tool returns a refusal:

```text
error:
  Failed to create payment payload: Payment creation aborted:
  duplicate_authorization: duplicate authorization for
  payer:0x…|intent:claude-demo-1|…
intentId: claude-demo-1
```

The same intent was already authorized. SQLite remembered it across the restart. Claude still had the tool, but it could not turn the retry into another signature.

## Configuration notes

These are the relevant defaults. Amounts use USDC base units. `MAX_PAYMENT` is optional; malformed money values stop startup.

```bash
# optional one-payment bound; unset by default
# MAX_PAYMENT=5000      # $0.005 maximum for each authorization

# budget, reserved atomically before anything is signed
CAP=100000              # rolling-window budget, $0.10
AGG_CAP=200000          # principal-wide cap, $0.20; stops slow drain
WINDOW_MS=300000        # the rolling window itself, 5 minutes
MAX_ACCOUNTING_WINDOW_MS=300000
                        # durable upper bound, fixed when a database is created

# durable state and plumbing
STORE=sqlite            # default; memory is only for disposable tests
STATE_DB=.agentpay-proxy-state.sqlite
                        # restart-safe budget, lifecycle, and dedup state
CEILING_S=300           # longest authorization lifetime the guard will sign
ALLOWED_HOSTS=x402.org  # unset = any host; when set, redirects are refused too
HOST=127.0.0.1          # loopback only; this process holds a key
PORT=4020
```

`MAX_PAYMENT` needs no mandate. The exact maximum passes; anything larger gets `payment_amount_exceeds`. A value of `0` blocks every positive payment. Keep `CAP` and `AGG_CAP` for split payments and slow drain.

The demo asks for a 300-second authorization. Set `CEILING_S=30` and it fails with `valid_before_too_far`. The effective limit is `min(CEILING_S, WINDOW_MS/1000)`.

`MAX_ACCOUNTING_WINDOW_MS` is fixed when a database is created. It prevents a later, longer window from relying on history that SQLite already pruned. If the policy may grow to 15 minutes, create the database with `MAX_ACCOUNTING_WINDOW_MS=900000`. A larger `WINDOW_MS` then works later; anything above the stored maximum fails.

SQLite recovers before the proxy listens. Corrupt, unreadable, locked, or newer-schema state stops startup instead of falling back to memory.

To embed the plugin instead of running the CLI, install [`@themobiusstrip/agentpay-guard`](https://www.npmjs.com/package/@themobiusstrip/agentpay-guard). Its `/sqlite` entry exports the store used here. The adapter supports one active process per database; multi-worker or multi-host deployments need a different store owner.

## What this does not solve

- This is not a sandbox. An agent with shell access to the proxy host can read the wallet file.
- Without `ALLOWED_HOSTS`, `paid_fetch` can reach anything visible to the proxy. Keep it on loopback unless the hop is authenticated.
- `MAX_PAYMENT` limits amount, not payee or quoted price. Those need a trusted mandate, which this tutorial does not create.
- This build is pinned to `exact` + Base Sepolia + USDC and `@x402/*@2.17.0`. Mainnet is a different deployment with real loss.
- Intent dedup stops a second client signature. It does not replace merchant replay handling.

## When it doesn't work

Most of these I hit myself while putting this together:

`{"status":402}` after `[guard] signed`
: payer unfunded, so the facilitator rejected it at verify. Fund the wallet from the faucet and retry.

`valid_before_too_far`
: merchant wants a longer authorization than your effective ceiling. Raise `CEILING_S` only within `WINDOW_MS`. A longer window also needs sufficient `MAX_ACCOUNTING_WINDOW_MS`.

`SQLite max accounting window is …; requested window ceiling …`
: this database was created with a lower durable maximum. Keep `WINDOW_MS` within it or provision a database with the intended ceiling.

`database is locked` or startup stops before listening
: another process may own the database, or state is unavailable. The proxy does not fall back to memory.

`duplicate_authorization … intent:<id>`
: same `intentId` after a signature already exists for it. Correct refusal; omit `intentId` if it's a new purchase.

`payment_amount_exceeds`
: one payment is above `MAX_PAYMENT`.

`cap_exceeded` on honest traffic
: window spent. Wait out `WINDOW_MS` or raise `CAP`. Pending holds count against the cap until they resolve; that's intentional.

`envelope_network` on a discovered endpoint
: that endpoint is on Base **mainnet**, outside the testnet envelope on purpose.

## What I took away

Claude can ask for a payment, but it never gets to decide what the wallet signs. That is the useful boundary.

[`agentpay-guard`](https://www.npmjs.com/package/@themobiusstrip/agentpay-guard) is the policy plugin and SQLite entry. [`agentpay-proxy`](https://www.npmjs.com/package/@themobiusstrip/agentpay-proxy) is the wrapper used in this post. [`x402-idempotency-middleware`](https://www.npmjs.com/package/@themobiusstrip/x402-idempotency-middleware) covers merchant-side replay. Source and threat model are in [theMobiusStrip/agentpay-guard](https://github.com/theMobiusStrip/agentpay-guard).
