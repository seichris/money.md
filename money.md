# money.md — Sending & receiving crypto (ETH/SOL + USDC) on Ethereum (L1/L2) and Solana

**SKILL GOAL:** Teach an AI agent how to safely **send** and **receive** crypto payments across:
- **Ethereum ecosystem**: Ethereum L1 + EVM L2s (ETH + USDC)
- **Solana**: mainnet-beta + devnet/testnet (SOL + USDC)

**Last updated:** 2026-02-23  
**Non-goals:** trading, DEX routing, yield/DeFi, MEV, token issuance, anything that bypasses compliance.

---

## Table of contents

1. Universal rules (non-negotiables)
2. Cross-chain decision tree (choose custody/tooling)
3. Universal “payment intent” format
4. Ethereum L1 & L2s: ETH + USDC
5. Solana: SOL + USDC (SPL)
6. Receiving & reconciliation patterns
7. Incident response
8. References

---

## 1) Universal rules (non‑negotiables)

### 1.1 Never guess
- Never guess the **network/chain/cluster**.
- Never guess **token contract/mint addresses** (USDC varies by network).
- Never guess **token decimals** (verify on-chain).

### 1.2 Always confirm (before sending)
- Network identity:
  - EVM: chain name + chain ID + RPC URL
  - Solana: cluster (`mainnet-beta/devnet/testnet`) + RPC URL
- Recipient:
  - EVM: checksummed `0x…` and (if possible) allowlisted
  - Solana: base58 pubkey; ensure it is a **wallet address**, not a token account
- Asset:
  - Native gas token (ETH or SOL)
  - Or USDC (and which USDC: native vs bridged on EVM; correct mint on Solana)
- Amount:
  - Always compute and store **base units** (integer math; avoid floats)

### 1.3 Always keep gas / fees
- EVM: every sender must have **ETH for gas** on that chain.
- Solana: sender must have **SOL for fees** and potentially for **ATA creation** (rent-exempt deposit).

### 1.4 Start small, then scale
- First payment to a new counterparty / new network / new token = **test transfer** (small), verify, then send the full amount.

### 1.5 Idempotency & logging (mandatory)
Every send must have a unique `payment_id` and a saved record of:
- network (chain_id or cluster), rpc_url
- from, to
- asset + token address/mint if USDC
- amount_base_units
- tx id:
  - EVM: `tx_hash`
  - Solana: `signature`
- timestamp, status transitions
- any nonces / blockhash used (as applicable)

### 1.6 Guardrails before autonomy
- Per-tx maximums (ETH/SOL and USDC)
- Allowlist recipient addresses (and optionally contracts/programs)
- Restrict allowed networks (e.g., Base-only)
- Require human approval for treasury moves

---

## 2) Cross-chain decision tree (choose custody/tooling)

This tree is intentionally tradeoff-driven. Pick a stack *per wallet role* (treasury vs ops vs user wallet).

```text
START
│
├─ Q0: Which network family?
│     ├─ EVM (Ethereum L1/L2s) → go to Q1-EVM
│     └─ Solana → go to Q1-SOL
│
├─ Q1-EVM / Q1-SOL: Must funds be recoverable via a vendor (password reset / KYC / fiat rails)?
│     ├─ YES → Stack D: Custodial (Coinbase or similar)
│     └─ NO  → Q2
│
├─ Q2: Do you need multi-party approvals (treasury governance, high-value)?
│     ├─ YES → Stack B:
│     │         - EVM: Safe Smart Account (multisig)
│     │         - Solana: Squads Multisig
│     └─ NO  → Q3
│
├─ Q3: Want self-custody but recoverable (auth/passkeys/social login) + policies?
│     ├─ YES → Stack C: Privy (embedded/server wallets + policy guardrails)
│     └─ NO  → Q4
│
└─ Q4: Pure self-custody single-sig?
      ├─ Human-operated → Stack A1:
      │    - EVM: hardware wallet + Foundry `cast`
      │    - Solana: hardware wallet + Solana CLI
      └─ Agent-operated → Stack A2:
           - Prefer remote signing/MPC (KMS/Turnkey/Privy) over raw keys
```

**Recommended architecture for most agent systems**
- **Treasury:** Stack B (Safe on EVM, Squads on Solana)
- **Ops wallet(s):** Stack C (Privy server wallets + policies) or Stack A2 (remote signer / MPC)
- **Flow:** Treasury funds ops wallet → ops wallet executes many small payments → ops wallet topped up periodically

---

## 3) Universal “payment intent” (agent storage schema)

Use one canonical object for “send” and “receive”, with network-specific fields.

```json
{
  "payment_id": "uuid-or-hash",
  "direction": "send | receive",

  "network_family": "evm | solana",
  "rpc_url": "https://...",

  "evm": {
    "chain": "ethereum | arbitrum | optimism | base | ...",
    "chain_id": 1
  },
  "solana": {
    "cluster": "mainnet-beta | devnet | testnet"
  },

  "asset": "ETH | SOL | USDC",
  "token_address_or_mint": "0x... (EVM USDC) or base58 mint (Solana USDC)",

  "from": "address/pubkey",
  "to": "address/pubkey",

  "amount_human": "12.34",
  "amount_base_units": "integer string",

  "confirmations_or_commitment": "EVM confirmations OR Solana confirmed/finalized",

  "created_at": "ISO-8601",
  "tx_id": "EVM tx_hash OR Solana signature",

  "status": "prepared | broadcast | confirmed | finalized | failed | canceled"
}
```

---

# 4) Ethereum L1 & L2s (EVM) — ETH + USDC

## 4.1 Chain selection (L1 vs L2)
- If the counterparty **did not explicitly specify a chain**, **do not send**.
- L1: broadest compatibility, higher fees.
- L2: lower fees; best for most routine payments.

### USDC gotcha on L2s
On some chains there are multiple “USDC-like” assets:
- **Native USDC** vs bridged variants like `USDC.e` or `USDbC`.
- You must match what the receiver expects.

## 4.2 Asset rules
### ETH
- Unit: **wei** (1 ETH = 1e18 wei).

### USDC (ERC‑20)
- USDC is an ERC‑20 contract on each chain.
- Decimals are commonly 6, but **verify on-chain**.

## 4.3 Stack A — Self-custody + CLI (Foundry `cast`)
### Read-only checks
```bash
cast chain-id --rpc-url $ETH_RPC_URL
cast balance 0xYourAddress --rpc-url $ETH_RPC_URL

cast erc20-token decimals 0xTokenAddress --rpc-url $ETH_RPC_URL
cast erc20-token balance  0xTokenAddress 0xYourAddress --rpc-url $ETH_RPC_URL

cast parse-units  12.34 6
cast format-units 12340000 6
```

### Send ETH
```bash
cast send 0xRecipient   --value 0.01ether   --rpc-url $ETH_RPC_URL   --private-key $SENDER_PRIVATE_KEY
```

### Send USDC
```bash
USDC=0xYourUsdcContract
TO=0xRecipient
DECIMALS=$(cast erc20-token decimals $USDC --rpc-url $ETH_RPC_URL)
AMOUNT=$(cast parse-units 12.34 $DECIMALS)

cast erc20-token transfer $USDC $TO $AMOUNT   --rpc-url $ETH_RPC_URL   --private-key $SENDER_PRIVATE_KEY
```

### Key handling (strongly preferred)
Avoid raw private keys when possible:
- keystores, hardware wallets, remote signing (KMS/Turnkey) where supported by your tooling.

## 4.4 Stack B — Safe Smart Account (multisig treasury)
- Agent proposes → owners sign → execute once threshold met.
- Keep treasury funds out of hot wallets; use an ops wallet for routine sends.

## 4.5 Stack C — Privy wallets + policies
- Dedicated agent wallet + policies (limits, allowlists, chain restrictions).
- Always treat USDC contract addresses as per-chain constants from canonical sources.

## 4.6 Stack D — Custodial (Coinbase)
- Use allowlisted addresses + approvals.
- Use idempotency keys for withdrawal APIs.
- Record platform withdrawal IDs + on-chain tx hash.

## 4.7 Canonical native USDC addresses (Circle-issued)
> Still verify you are not using a spoof token; do not “copy from random explorer pages”.

```text
Ethereum (1):     0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
Arbitrum (42161): 0xaf88d065e77c8cC2239327C5EDb3A432268e5831
OP Mainnet (10):  0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85
Base (8453):      0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
Linea (59144):    0x176211869cA2b568f2A7D4EE941E073a821EE1ff
zkSync (324):     0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4
```

---

# 5) Solana — SOL + USDC (SPL)

## 5.1 Cluster selection (Solana networks)
Solana clusters are distinct. Funds/tokens are **not interchangeable**.

```text
mainnet-beta: https://api.mainnet-beta.solana.com
devnet:       https://api.devnet.solana.com
testnet:      https://api.testnet.solana.com
```

CLI:
```bash
solana config get
solana config set --url https://api.mainnet-beta.solana.com
```

## 5.2 Core Solana gotcha: wallet vs token account
- SOL transfers go to a **wallet pubkey**.
- USDC is an **SPL token** stored in token accounts (usually the **Associated Token Account**, ATA).
- If you send SOL to a token account address, SOL can become **stuck**.

## 5.3 Asset rules
### SOL
- Unit: **lamports**
- 1 SOL = **1,000,000,000 lamports**

### USDC on Solana
- USDC is an SPL token mint.
- Decimals are typically 6, but **verify on-chain**.

## 5.4 Stack A — Self-custody + CLI (Solana CLI + SPL Token CLI)
Read-only:
```bash
solana balance <WALLET_ADDRESS> --url https://api.mainnet-beta.solana.com
solana confirm -v <SIGNATURE> --url https://api.mainnet-beta.solana.com
```

Send SOL:
```bash
solana transfer   --from /path/to/sender.json   <RECIPIENT_WALLET_ADDRESS>   0.01   --allow-unfunded-recipient   --with-memo "inv_2026_02_23_001"   --url https://api.mainnet-beta.solana.com   --fee-payer /path/to/sender.json
```

Send USDC (recommended: fund recipient ATA if needed):
```bash
USDC_MINT=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
TO=<RECIPIENT_WALLET_ADDRESS>

spl-token transfer   --fund-recipient   --with-memo "inv_2026_02_23_001"   $USDC_MINT   12.34   $TO   --url https://api.mainnet-beta.solana.com
```

Verify:
```bash
spl-token accounts --url https://api.mainnet-beta.solana.com
solana confirm -v <SIGNATURE> --url https://api.mainnet-beta.solana.com
```

## 5.5 Stack B — Squads Multisig (treasury)
- Treasury lives in multisig; agent proposes; humans approve; execute.
- Use roles/spending limits where appropriate.

## 5.6 Stack C — Privy wallets + policies
- Embedded/server wallets with recovery UX and policy guardrails.
- Must handle recipient ATA existence and creation policy.

## 5.7 Stack D — Custodial / MPC (Coinbase)
- Managed key + approvals + allowlists.
- Store platform withdrawal IDs + on-chain signatures.

## 5.8 Canonical USDC mints + core program IDs
```text
USDC mint (mainnet-beta): EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
USDC mint (devnet):       4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU

SPL Token Program:        TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
Token-2022 Program:       TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb
Associated Token Program: ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL
```

## 5.9 Solana retry / liveness note (blockhash)
Transactions use a **recent blockhash**. For automation:
- Always store the signature you broadcast.
- Don’t immediately resubmit “fresh” transactions if you’re unsure of status (double-pay risk).
- Handle blockhash-expiry failures by rebuilding with a new blockhash *only after* you’ve checked whether the original finalized.

---

# 6) Receiving & reconciliation patterns (both ecosystems)

### 6.1 Always provide a complete “payment request”
Include:
- network (chain/cluster + id), rpc (optional)
- asset (ETH/SOL/USDC)
- recipient address/pubkey
- USDC token address/mint
- amount
- `payment_id` reference (off-chain)

### 6.2 Confirmation policy
- EVM: wait for your chosen confirmation count (risk-based).
- Solana: choose `confirmed` (faster) vs `finalized` (safer).

### 6.3 Reconciliation strategies (preferred order)
1. **Unique address per invoice/customer** (best general approach)
2. **Contract/program pay endpoint** that emits a structured event (best at scale)
3. **Memo/reference** (Solana supports memos well; EVM needs contract-level support)
4. **Unique amount suffix** (last resort; fragile)

---

# 7) Incident response

### If an agent wallet is compromised
- Freeze funding immediately (stop topping up ops wallets)
- Rotate keys / revoke API keys / remove compromised multisig owners
- Move remaining funds to a secure treasury multisig or secure custody
- Audit logs: recipients, amounts, tx ids

### If you sent to the wrong network / wrong USDC variant / wrong mint
- Do not repeat payment blindly
- Share the tx id + network with the recipient
- Treat recovery as case-by-case (may require recipient cooperation)

---

# 8) References (copy/paste)

```text
Foundry Cast docs:                 https://getfoundry.sh/cast/
Safe docs (Smart Accounts):        https://docs.safe.global/
Circle USDC addresses (EVM):       https://developers.circle.com/stablecoins/usdc-contract-addresses
Solana CLI transfer examples:      https://docs.anza.xyz/cli/examples/transfer-tokens
SPL Token docs:                    https://spl.solana.com/token
Solana Pay transfer requests:      https://docs.solanapay.com/core/transfer-request
Privy docs:                        https://docs.privy.io/
Squads Multisig docs:              https://docs.squads.so/main
Coinbase CDP docs:                 https://docs.cdp.coinbase.com/
```
