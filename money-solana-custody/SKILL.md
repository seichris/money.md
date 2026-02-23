---
name: money-solana-custody
description: Designs and operates custody for SOL and USDC on Solana using the right tradeoffs (Squads multisig treasury, recoverable wallets with policies, hardware wallets, custodians). Use when the user asks about Solana treasury governance, Squads setup, ops wallet design, key management, approvals, recovery, or incident response. Do not use for trading, DeFi, or price advice.
compatibility: Planning works anywhere. Execution depends on chosen stack and tooling such as Squads, Privy, and Solana RPC access.
metadata:
  author: money-skill-pack
  version: 0.1.0
  category: finance
  tags: [solana, custody, squads, privy, security, treasury]
---

# Solana custody: wallet and treasury architecture

## Instructions

Use this skill to design custody for SOL and USDC on Solana.

This skill is about custody and governance, not about a specific transfer. For transfer steps, use `money-solana-payments`.

## Step 1: Gather constraints

Collect:
- Treasury governance requirement
- Number of approvers
- Automation requirement for an agent
- Recovery requirement for operators or users
- Cluster scope: mainnet-beta only, or devnet for testing
- Risk policy: caps, allowlists, and funding rules for associated token accounts

## Step 2: Choose wallet roles

Recommend:
1. Treasury
- Holds most funds
- Multisig approvals

2. Ops
- Low balance
- Frequent payments
- Refilled from treasury

3. Optional user wallets
- If building a product, emphasize recovery UX

## Step 3: Choose custody stacks

### Stack B: Squads multisig for treasury
Use Squads when you need multi-party approvals and an auditable workflow.

Guidance
- Pick a threshold appropriate to risk.
- Store owner keys on diverse devices.
- Treasury should not be the hot wallet used for routine payouts.

### Stack C: Recoverable wallets with policies for ops
Use recoverable wallets with spend limits and allowlists for an always-online agent.

Ops guardrails
- destination allowlist
- per-tx caps for SOL and USDC
- cluster allowlist, usually mainnet-beta only
- restrict who can fund recipient associated token accounts

### Stack A1: Hardware wallet for single operator
Use a hardware wallet when automation is not required.

### Stack D: Custodial or MPC platform
Use this when compliance, approvals, recovery, or enterprise controls are required.

## Step 4: Funding flows

Recommended flow
1. Squads treasury funds ops wallet as needed.
2. Ops wallet executes routine payouts.
3. Ops wallet always maintains enough SOL for fees and account creation.

Avoid
- Keeping large balances in ops wallets.
- Allowing an agent to move treasury funds unilaterally.

## Step 5: Incident response plan

Minimum playbook
- Stop top-ups to ops wallets.
- Rotate keys and revoke credentials.
- Update Squads membership if an owner is compromised.
- Sweep remaining funds to a known-secure treasury.
- Audit transaction history and produce an incident report.

## Output format

Return:
1. Recommended stacks for treasury and ops
2. Roles, caps, and allowlists
3. Recovery model
4. Approval and execution workflow
5. Incident response checklist

## Examples

Example 1: Team treasury on Solana
- Treasury: Squads multisig
- Ops: low-balance wallet with caps
- Receiving: publish a treasury address for large payments, ops address for routine

Example 2: Product with user wallets
- Users: recoverable wallets
- Treasury: multisig
- Ops: service wallet with strict policies
