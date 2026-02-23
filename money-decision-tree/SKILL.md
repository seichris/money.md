---
name: money-decision-tree
description: Chooses the safest network and custody approach for crypto payments (EVM vs Solana; multisig vs recoverable vs custodian). Use when the user asks how to send or receive ETH, SOL, or USDC, how to set up a treasury, which wallet to use, or how to design ops and treasury flows. Do not use for trading, DeFi, or price advice.
compatibility: Works anywhere for planning. For execution, requires the payment skills plus network RPC access and appropriate tooling.
metadata:
  author: money-skill-pack
  version: 0.1.0
  category: finance
  tags: [payments, custody, decision-tree, ethereum, solana, usdc]
---

# Money decision tree

## Instructions

Use this skill to route any money-related request to the right network and the right custody stack, then to the right execution skill.

This skill is intentionally high-level. When it reaches an execution step, it should point to the appropriate focused skill:
- EVM transfers: see `money-evm-payments`
- Solana transfers: see `money-solana-payments`
- EVM custody architecture: see `money-evm-custody`
- Solana custody architecture: see `money-solana-custody`
- Receiving and reconciliation: see `money-receiving-reconciliation`

## Step 1: Classify the request

Determine which category the user is asking for.

1. Payment execution
- The user wants to send or receive ETH, SOL, or USDC now.
- The user wants CLI commands, transaction steps, or verification.

2. Custody and architecture
- The user wants to choose wallets, key management, multisig, recovery, approvals, limits.

3. Receiving and reconciliation
- The user wants to detect payments, match invoices, confirm finality, handle partials, build a ledger.

If unclear, ask one short question: "Is this about executing a transfer now, or designing custody and payments architecture?"

## Step 2: Identify the network family

Do not guess. If the user did not specify the network, ask for it.

- EVM networks include Ethereum mainnet and L2s like Arbitrum, OP Mainnet, Base, zkSync, Linea.
- Solana networks are clusters: mainnet-beta, devnet, testnet.

If the user says "USDC" but not the chain or cluster, stop and ask for the chain or cluster.

## Step 3: Identify the risk profile

Collect these facts and log them in your plan:

- Value band: low, medium, high (define with org policy)
- Frequency: one-off, occasional, frequent
- Automation: human-operated, agent-operated, mixed
- Ownership: individual funds, team treasury, user funds
- Recovery requirement: required, preferred, not needed
- Compliance requirement: required, optional

If the user will not provide a value band, default to high-risk behavior: prefer multisig or custodial approvals.

## Step 4: Choose wallet roles

Recommend splitting funds by role.

1. Treasury wallet
- Holds most funds.
- Moves infrequently.
- Should require approvals.

2. Operations wallet
- Holds limited balance.
- Performs frequent payments.
- Refilled from treasury.

3. User wallets (optional)
- For end users or customers.
- Usually require recovery UX.

## Step 5: Choose the custody stack

Use this decision tree.

Q1: Must funds be recoverable via a vendor and supported by compliance controls or fiat rails
- Yes: choose a custodial or MPC platform (Stack D).
- No: continue.

Q2: Do you need multi-party approvals for treasury funds
- Yes: choose multisig smart accounts (Stack B).
  - EVM: Safe
  - Solana: Squads
- No: continue.

Q3: Do you want self-custody with recovery UX and policy guardrails for an always-online agent
- Yes: choose Privy-style embedded or server wallets with policies (Stack C).
- No: continue.

Q4: Pure self-custody single-sig
- Human-operated: hardware wallet (Stack A1).
- Agent-operated: remote signer or KMS-style signing (Stack A2). Avoid raw private keys on disk.

## Step 6: Route to the correct focused skill

- If network family is EVM and request is payment execution: use `money-evm-payments`.
- If network family is Solana and request is payment execution: use `money-solana-payments`.
- If request is custody and architecture:
  - EVM: `money-evm-custody`
  - Solana: `money-solana-custody`
- If request is receiving and reconciliation: `money-receiving-reconciliation`

## Output format

Return a short plan with:
1. Network family and specific chain or cluster
2. Selected custody stack and why
3. Wallet roles and flow (treasury to ops)
4. Next skill to load for execution details
5. Any missing inputs needed before money moves

## Examples

Example 1: Small recurring payouts on Base by an autonomous agent
- Network: EVM, Base
- Custody: Safe treasury plus Privy ops wallet with spend limits
- Next: `money-evm-custody` for architecture, then `money-evm-payments` for execution

Example 2: Team treasury on Solana with approvals
- Network: Solana, mainnet-beta
- Custody: Squads multisig treasury plus a low-balance ops wallet
- Next: `money-solana-custody`

Example 3: User says "Send 12.34 USDC to this address"
- Missing: chain
- Action: ask which chain and whether recipient expects native USDC or a bridged variant

## Troubleshooting

Skill triggers too often
- Add exclusions to the description, such as "not for trading or DeFi".
- Make triggers more specific to sending, receiving, custody, treasury, or invoicing.

User requests trading or investing advice
- Refuse and redirect to non-actionable educational content.
