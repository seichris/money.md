---
name: money-solana-payments
description: Sends and receives SOL and USDC on Solana with safe checks for cluster selection, SPL token accounts, ATA creation, memos, and verification. Use when the user asks to send SOL, send USDC on Solana, transfer SPL tokens, create or fund an associated token account, confirm a Solana signature, or build a payment request on mainnet-beta or devnet. Do not use for trading, DeFi, or price advice.
compatibility: Requires access to a Solana RPC endpoint. For CLI execution, requires Solana CLI and spl-token CLI installed in the runtime.
metadata:
  author: money-skill-pack
  version: 0.1.0
  category: finance
  tags: [solana, sol, usdc, spl, payments, memo, solana-pay]
---

# Solana payments: SOL and USDC

## Instructions

Use this skill for payment execution and verification on Solana:
- Sending SOL
- Sending USDC as an SPL token
- Handling associated token accounts and recipient funding
- Confirming a signature and choosing commitment

For custody architecture, use `money-solana-custody`.

## Inputs you must have before sending

If any are missing, stop and ask.

1. Cluster and RPC
- cluster: mainnet-beta, devnet, or testnet
- rpc_url

2. Asset
- SOL or USDC
- if USDC: mint address for that cluster from `references/usdc-mints.md`

3. Parties
- sender wallet pubkey
- recipient wallet pubkey

4. Amount
- human amount
- base units will be computed and stored

5. Policy
- commitment target: confirmed or finalized
- whether the sender is allowed to fund the recipient associated token account

## Step 1: Pre-flight safety checks

1. Confirm cluster
- Do not guess. Confirm the CLI config or your SDK settings.

2. Confirm recipient is a wallet pubkey
- Solana has wallet pubkeys and token account addresses.
- Sending SOL to a token account address can strand funds.

3. Confirm sender SOL balance
- Sender must have SOL for fees.
- For USDC sends, sender may also need SOL to create the recipient associated token account.

## Step 2: Resolve the asset precisely

### SOL
- unit is lamports
- avoid floating math in code; compute lamports with integer arithmetic

### USDC on Solana
- USDC is an SPL token mint
- recipient typically needs an associated token account for that mint

Use references:
- `references/usdc-mints.md`
- `references/token-account-gotchas.md`
- `references/solana-cli-cheatsheet.md`

## Step 3: Create and store the payment intent

Store:
- payment_id
- cluster, rpc_url
- from, to
- asset
- mint for USDC
- amount_human and amount_base_units
- status prepared

## Step 4: Execute the transfer

### Send SOL with Solana CLI

Command template
```bash
solana transfer   --from SENDER_KEYPAIR_PATH   RECIPIENT_WALLET_PUBKEY   AMOUNT_SOL   --allow-unfunded-recipient   --with-memo PAYMENT_ID   --url RPC_URL   --fee-payer SENDER_KEYPAIR_PATH
```

After broadcast
- capture the returned signature
- store signature with payment_id
- status broadcast

Confirm
```bash
solana confirm -v SIGNATURE --url RPC_URL
```

### Send USDC with spl-token CLI

Recommended approach: fund recipient associated token account if needed.

Command template
```bash
USDC_MINT=BASE58_MINT
RECIPIENT_WALLET=BASE58_PUBKEY

spl-token transfer   --fund-recipient   --with-memo PAYMENT_ID   $USDC_MINT   AMOUNT_USDC   $RECIPIENT_WALLET   --url RPC_URL
```

Notes
- The sender pays for associated token account creation when using --fund-recipient.
- If you do not fund recipient, the transfer can fail if the recipient has no token account.

Confirm and record status
- Use `solana confirm` on the signature.
- For higher assurance, use finalized commitment in your verification.

## Step 5: Receiving a payment on Solana

To request payment, provide:
- cluster and rpc_url if needed
- asset and amount
- recipient wallet pubkey
- for USDC: mint address
- payment_id memo requirement

If you need stronger invoice matching, consider Solana Pay transfer requests with a unique reference key.

## Examples

Example 1: Send 0.01 SOL on mainnet-beta
- Confirm cluster mainnet-beta
- Transfer with memo
- Confirm signature

Example 2: Send 12.34 USDC on mainnet-beta
- Use the mainnet-beta USDC mint from references
- Transfer with --fund-recipient
- Confirm signature and verify token balance

## Troubleshooting

Insufficient SOL for fees or ATA creation
- Ensure sender has enough SOL before sending USDC.

Recipient has no token account
- Use --fund-recipient or instruct recipient to create an associated token account.

Blockhash expired or intermittent RPC failures
- Do not resend blindly.
- First check whether the prior signature landed.
- If it failed due to blockhash expiry, rebuild and resubmit carefully with idempotency rules.
