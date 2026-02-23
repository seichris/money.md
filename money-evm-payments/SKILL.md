---
name: money-evm-payments
description: Sends and receives ETH and USDC on Ethereum mainnet and EVM L2s with safe checks, logging, and verification. Use when the user asks to send ETH, send USDC, confirm an EVM transaction, choose L1 vs L2 for a payment, or use Foundry cast for transfers and balance checks. Do not use for trading, DeFi, or price advice.
compatibility: Requires access to an EVM RPC endpoint. For CLI execution, requires Foundry cast installed in the runtime.
metadata:
  author: money-skill-pack
  version: 0.1.0
  category: finance
  tags: [ethereum, evm, l2, eth, usdc, payments, cast]
---

# EVM payments: ETH and USDC

## Instructions

Use this skill for payment execution and verification on EVM networks:
- Sending ETH
- Sending USDC (ERC-20)
- Verifying receipts and confirmations
- Producing a complete payment request for someone to pay you

This skill is optimized for CLI-first agents using Foundry cast. If you are operating from a multisig treasury, do not execute directly from the treasury. Instead, follow the custody skill and use an ops wallet for routine sends.

## Inputs you must have before sending

If any of these are missing, stop and ask for them.

1. Chain
- chain name and chain_id
- rpc_url

2. Asset
- ETH or USDC
- if USDC: token contract address for that chain, from `references/usdc-addresses.md` or another explicitly trusted registry

3. Parties
- sender address
- recipient address (checksummed)

4. Amount
- human amount
- base units will be computed and stored

5. Policy
- confirmation target (example: 1, 2, 5, 12)
- max value per tx, if your system enforces caps
- allowlist requirement, if enabled

## Step 1: Pre-flight safety checks

1. Confirm chain identity
- Run: `cast chain-id --rpc-url RPC_URL`
- Compare to expected chain_id. If mismatched, stop.

2. Confirm recipient address
- Ensure it is a valid checksummed address.
- If your system has an allowlist, enforce it.

3. Confirm sender gas
- Sender must have ETH on the same chain for gas, even when sending USDC.

## Step 2: Resolve the asset precisely

### ETH
- unit is wei
- prefer using cast units and explicit decimals, not floating math

### USDC
- USDC is an ERC-20 contract address per chain
- do not assume decimals; read them on-chain
- confirm you are using the correct USDC variant the receiver expects

Use these references:
- `references/usdc-addresses.md` for native USDC contract addresses
- `references/cast-cheatsheet.md` for commands

## Step 3: Compute base units and record the payment intent

Create a payment intent record before you broadcast.

Minimum fields:
- payment_id
- chain_id, rpc_url
- from, to
- asset
- token_address if USDC
- amount_human
- amount_base_units
- status = prepared

Compute base units
- ETH: `cast to-wei AMOUNT_ETH ether`
- USDC: `cast parse-units AMOUNT_UI DECIMALS`

Store the computed integer string.

## Step 4: Check balances

Check ETH balance for gas
- `cast balance SENDER --rpc-url RPC_URL`

Check token balance for USDC
- `cast erc20-token balance USDC_TOKEN SENDER --rpc-url RPC_URL`

If any balance is insufficient, stop.

## Step 5: Execute the transfer

### Send ETH

Command template
```bash
cast send RECIPIENT_ADDRESS   --value AMOUNT_ETHether   --rpc-url RPC_URL   SIGNER_ARGS
```

Example signer args
- `--private-key PRIVATE_KEY` for a simple wallet
- or use a keystore or remote signer if available in your environment

### Send USDC

Process
1. Read decimals
2. Convert amount to base units
3. Transfer using `erc20-token transfer`

Command template
```bash
USDC_TOKEN=0x...
RECIPIENT=0x...
DECIMALS=$(cast erc20-token decimals $USDC_TOKEN --rpc-url RPC_URL)
AMOUNT=$(cast parse-units AMOUNT_UI $DECIMALS)

cast erc20-token transfer $USDC_TOKEN $RECIPIENT $AMOUNT   --rpc-url RPC_URL   SIGNER_ARGS
```

## Step 6: Broadcast logging and idempotency

Immediately after broadcast:
- capture tx_hash
- store it against payment_id
- status = broadcast

Never retry blindly.
- If you see a timeout, first query the chain for the tx_hash.
- If you must retry, enforce nonce discipline.

## Step 7: Confirm and finalize

1. Fetch receipt
- `cast receipt TX_HASH --rpc-url RPC_URL`

2. Confirm confirmations target
- Choose a confirmations threshold for your risk level.
- For high value, use a higher threshold.

3. Finalize exactly once
- status = confirmed
- store block number, timestamp if available

## Step 8: Receiving a payment on EVM

To request payment, provide a complete payment request:

- Chain: name and chain_id
- Asset: ETH or USDC
- Recipient address
- For USDC: token contract address
- Amount and payment_id reference

To confirm receipt:
- ETH: confirm tx value and recipient match
- USDC: confirm Transfer event emitted by the correct token contract

## Examples

Example 1: Send 0.01 ETH on Base
- Confirm chain_id is 8453
- Confirm recipient
- Send with cast, store tx_hash, confirm receipt

Example 2: Send 12.34 USDC on Arbitrum
- Use Arbitrum native USDC address from references
- Read decimals on-chain
- Parse units and transfer
- Confirm Transfer event and receipt status

## Troubleshooting

Insufficient funds
- For ETH: you lack ETH for value or gas.
- For USDC: you may have USDC but not enough ETH for gas.

Wrong chain or wrong token
- If the receiver says they did not receive funds, verify chain_id and token address first.

Nonce errors
- Ensure only one process is sending from a given address at a time.
- If concurrency is required, implement a nonce manager.

Stuck pending transaction
- Use your node or provider tools to determine if the tx is broadcast.
- Consider replacing the tx with a higher fee only if you understand nonce replacement rules.
