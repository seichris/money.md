---
name: money-receiving-reconciliation
description: Designs safe receive, verification, and reconciliation flows for crypto payments across EVM and Solana. Use when the user asks how to confirm a payment arrived, match invoices, handle confirmations and finality, generate payment requests, build an internal ledger, or prevent double-counting. Do not use for trading, DeFi, or price advice.
compatibility: Planning works anywhere. Execution requires access to chain RPCs or an indexer, plus your storage for idempotent state.
metadata:
  author: money-skill-pack
  version: 0.1.0
  category: finance
  tags: [receiving, reconciliation, monitoring, invoices, idempotency, ethereum, solana]
---

# Receiving and reconciliation

## Instructions

Use this skill to design and implement reliable receiving flows for crypto payments. The key goal is correctness:
- do not mark paid when unpaid
- do not double-count
- do not accept the wrong asset
- do not accept funds on the wrong network

This skill complements the payment execution skills:
- EVM execution: `money-evm-payments`
- Solana execution: `money-solana-payments`

## Step 1: Define what counts as a valid payment

A valid payment is a tuple:
- network identity
  - EVM: chain_id
  - Solana: cluster
- asset identity
  - native token: ETH or SOL
  - USDC: token contract address on EVM, mint address on Solana
- recipient identity
  - the address or pubkey you control
- amount identity
  - exact base units
  - if you allow overpayment, define the rule explicitly
- confirmation policy
  - EVM confirmations threshold
  - Solana commitment: confirmed or finalized

## Step 2: Choose an invoice matching strategy

Preferred order

1. Unique address per invoice
- EVM: generate a new address per invoice, then monitor it.
- Solana: generate a new wallet per invoice, but remember USDC needs token accounts.

2. Explicit payment reference
- Solana: require memo and verify it.
- Solana Pay: use a unique reference key for deterministic lookup.
- EVM: references require contract-level support. Basic transfers do not carry a reliable memo.

3. Payment contract
- Accept payments via a contract or program that emits a structured event including an invoice id.

4. Unique amount suffix
- Last resort only. Fragile and fails with partial payments.

## Step 3: Implement idempotent state transitions

Use a small state machine per payment_id:
- prepared
- observed_onchain
- sufficiently_confirmed
- credited
- failed

Rules
- Never credit twice.
- Store the chain tx id:
  - EVM: tx_hash
  - Solana: signature
- Store observed amount in base units and compare to expected.

## Step 4: Verification approach by network

### EVM
For ETH payments
- Validate recipient and value on the correct chain_id.
- Wait for the confirmation threshold.

For USDC payments
- Validate Transfer logs emitted by the correct USDC contract.
- Match recipient address and amount.
- Wait for confirmations.

### Solana
For SOL payments
- Validate signature landed on the expected cluster.
- Validate recipient and amount.
- Prefer finalized for irreversible fulfillment.

For USDC payments
- Validate SPL token transfer for the correct mint.
- Validate recipient associated token account owner and amount.
- Validate memo if required.

## Step 5: Handle partials, refunds, and disputes

Define policy up front:
- Do you allow partial payments
- Do you allow overpayments
- How do you handle refunds, and on which network and asset

Never initiate a refund automatically unless your policy and allowlists allow it.

## Output format

Provide:
1. Matching strategy recommendation
2. Verification rules per network and asset
3. Confirmation and finality policy
4. Data model for storing intents and receipts
5. Failure modes and monitoring checklist

## Examples

Example 1: SaaS invoices paid in USDC
- Use unique address per invoice on a single preferred chain.
- Require exact base units.
- Credit after confirmations threshold.

Example 2: Consumer payments on Solana
- Use memo payment_id or Solana Pay transfer requests.
- Verify finalized before fulfillment for higher value.

## Troubleshooting

Payments arrive on the wrong network
- Your payment request did not specify the chain or cluster clearly.
- Always include chain_id or cluster in the request.

Token confusion
- Enforce token address or mint checks.
- Treat each variant as distinct.
