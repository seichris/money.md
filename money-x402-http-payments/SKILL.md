---
name: money-x402-http-payments
description: Designs and implements HTTP 402 (x402-style) paid service flows: returning a Payment Required challenge, collecting proof, verifying on-chain settlement, and crediting access idempotently. Use when the user asks about x402, HTTP 402 paywalls, paid APIs, paid content gating, metered billing over HTTP, or "pay then retry" flows. Do not use for trading, DeFi, or price advice.
compatibility: Planning works anywhere. Execution requires an HTTP service plus chain RPC/indexer access and a durable store for idempotent state.
metadata:
  author: money-skill-pack
  version: 0.1.0
  category: finance
  tags: [x402, http-402, paywall, paid-api, receiving, reconciliation, metering, ethereum, solana, usdc]
---

# HTTP 402 (x402-style) paid service flows

## Instructions

Use this skill when the product is an HTTP service and you want clients (web, CLI, mobile, server-to-server) to automatically:
1. request a resource
2. receive a `402 Payment Required` challenge describing how to pay
3. pay on-chain
4. retry with payment proof
5. receive the resource after verification

This skill is the orchestration/interface layer. It routes chain settlement and custody to the existing skills:
- Settlement on EVM: `money-evm-payments`
- Settlement on Solana: `money-solana-payments`
- Crediting correctness: `money-receiving-reconciliation`
- Wallet architecture: `money-evm-custody` or `money-solana-custody`

For protocol specifics, read `references/protocol-notes.md` and follow the official x402 spec your implementation targets.

## Step 1: Decide what you are selling

Pick one billing model and keep it simple at first:
- One-shot unlock: pay once, unlock one request or one object.
- Time-based access: pay for 1 day / 1 week access.
- Metered: pay per unit (tokens, minutes, requests). Prefer prepay credits, then decrement.

Define and store:
- `product_id`
- `price` in base units
- `asset` and network identity (EVM `chain_id` or Solana `cluster`)
- confirmation/finality policy

## Step 2: Choose invoice matching

Use one of these, in preferred order:
1. Unique address per invoice (strongest, simplest to reconcile)
2. Explicit reference field (memo / reference key) when available
3. Payment contract/program emitting structured events

Avoid "unique amount suffix" except as a last resort.

## Step 3: Define the 402 challenge contract (HTTP-level)

When unauthenticated or unpaid, return `402 Payment Required` with a machine-readable payment challenge that includes:
- network identity (EVM chain_id or Solana cluster)
- asset identity (token contract/mint if USDC)
- recipient identity (address/pubkey you control)
- required amount in base units
- an invoice or session identifier (`payment_id`)
- expiry and replay rules (one-time vs reusable)

Do not embed secrets in the 402 challenge. Treat it as public.

## Step 4: Implement the pay-then-retry loop

Client flow:
1. Call endpoint
2. If `402`, parse challenge
3. Submit on-chain payment
4. Retry request with payment proof

Server flow:
1. On retry, extract payment proof (tx hash / signature / reference)
2. Verify on the correct network with the configured confirmation/finality policy
3. Transition state idempotently (never credit twice)
4. Serve the resource

Verification and state machine: follow `money-receiving-reconciliation`.

## Step 5: Route to the correct settlement skill

After choosing network and asset:
- EVM payment execution details: `money-evm-payments`
- Solana payment execution details: `money-solana-payments`

If the request is really about wallet roles, approvals, and guardrails for the receiving address:
- EVM custody architecture: `money-evm-custody`
- Solana custody architecture: `money-solana-custody`

## Step 6: Operational guardrails (minimum viable)

Do these even for MVP:
- Enforce network identity (chain_id/cluster) on every verification.
- Enforce asset identity (USDC contract/mint) on every verification.
- Store `payment_id` state transitions durably and idempotently.
- Rate-limit 402 issuance and proof verification endpoints.
- Expire challenges and reject stale/replayed proofs.

## Output format

Return:
1. Billing model and matching strategy
2. 402 challenge contents (fields, expiry, replay rules)
3. Proof format (tx hash/signature/reference) and verification policy
4. Minimal data model (payment intent + receipt + crediting state)
5. Which existing settlement/custody skills to use next

