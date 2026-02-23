---
name: money-evm-custody
description: Designs and operates custody for ETH and USDC on Ethereum and EVM L2s using the right tradeoffs (Safe multisig treasury, recoverable wallets with policies, hardware wallets, remote signers, custodians). Use when the user asks about wallet choice, multisig setup, treasury and ops flows, key management, approvals, recovery, or incident response. Do not use for trading, DeFi, or price advice.
compatibility: Planning works anywhere. Execution depends on chosen stack and tooling such as Safe, Privy, and chain RPC access.
metadata:
  author: money-skill-pack
  version: 0.1.0
  category: finance
  tags: [ethereum, evm, custody, safe, privy, coinbase, security]
---

# EVM custody: wallet and treasury architecture

## Instructions

Use this skill to design a custody architecture for ETH and USDC on Ethereum and EVM L2s. The output should be a clear decision and an operating model.

This skill is about custody and governance, not about executing a specific transfer. For transfer steps, use `money-evm-payments`.

## Step 1: Gather constraints

Collect these inputs. If missing, make reasonable conservative defaults.

- Treasury governance required: yes or no
- Number of approvers and their availability
- Required recovery and compliance posture
- Automation requirement for an agent: none, low, high
- Chain scope: which EVM chains are supported
- Risk policy: max value per tx, daily limits, allowlists

## Step 2: Choose wallet roles

Recommend a multi-wallet model for most teams.

1. Treasury wallet
- Holds most funds
- Low frequency
- Always approvals

2. Ops wallet
- Limited balance
- High frequency payments
- Refills from treasury

3. Optional per-user wallets
- If building a product
- Usually needs recovery UX

## Step 3: Choose a custody stack per role

### Stack B: Safe multisig for treasury
Use Safe when you need team approvals, auditability, and separation of duties.

Recommended treasury settings
- Threshold: 2 of 3 or 3 of 5, depending on risk
- Owners: diverse devices and custody, not all on one laptop
- Policy: treasury never used as a hot wallet for routine sends

Agent integration pattern
- Agent prepares a transaction proposal
- Humans approve
- Execution occurs after threshold signatures

### Stack C: Recoverable wallets with policies for ops
Use a recoverable wallet platform with policies when you need:
- always-online automation
- spend limits and allowlists
- recovery UX for operational continuity

Ops wallet guardrails
- destination allowlist
- per-tx and per-day caps
- chain allowlist
- token allowlist

### Stack A1: Hardware wallet for single-sig human ops
Use hardware wallets for a single operator when automation is not required.

### Stack A2: Remote signer for agent-operated single-sig
Use remote signing infrastructure when automation is required but you cannot accept a hot key on disk.

Guidance
- Do not store raw private keys in environment variables.
- Prefer HSM or KMS-backed signing, or a purpose-built signing service.

### Stack D: Custodial or MPC platform
Use this when you need:
- compliance controls
- enterprise approvals and policy
- fiat rails and account recovery

## Step 4: Funding flows

Recommended flow
1. Treasury Safe funds ops wallet on a schedule or when balance is low.
2. Ops wallet executes frequent payments with caps and allowlists.
3. Ops wallet maintains enough ETH for gas on each active chain.

Avoid
- Keeping large balances in ops wallets.
- Allowing the agent to move treasury funds unilaterally.

## Step 5: Incident response plan

Minimum playbook
- Stop top-ups to ops wallets.
- Rotate compromised keys and revoke credentials.
- Remove compromised Safe owners.
- Sweep remaining funds to a known-secure treasury.
- Audit logs and produce an incident report.

## Output format

Produce:
1. Recommended stacks for treasury and ops
2. Roles, permissions, and spending caps
3. Key management model and recovery story
4. Approval and execution workflow
5. Incident response steps

## Examples

Example 1: Startup with Base payments and a small team
- Treasury: Safe 2 of 3
- Ops: recoverable wallet with allowlist and 1,000 USDC per-tx cap
- Chains: Base only for ops
- Refill: weekly from treasury

Example 2: Single founder with occasional payments
- Hardware wallet for single-sig
- Manual execution
- Use `money-evm-payments` for transfer steps

## Troubleshooting

Approvals are too slow
- Lower the threshold only if risk policy allows.
- Add an emergency owner with strict policy and strong custody.

Ops wallet drained
- Reduce caps.
- Enforce destination allowlists.
- Isolate per-merchant wallets if needed.
