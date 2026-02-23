# money_solana.md — Sending & receiving crypto (SOL + USDC) on Solana

**SKILL GOAL:** Give an AI agent a safe, repeatable playbook to **send and receive SOL + USDC** on Solana (mainnet-beta + test clusters), with the right custody/tooling choice based on tradeoffs.

**Last updated:** 2026-02-23  
**Scope:** Solana **mainnet-beta**, **devnet**, **testnet**. Assets: **SOL** and **USDC (SPL token)**.  
**Non-goals:** trading/DEX routing, DeFi, bridging, token issuance, NFTs.

---

## 0) Non-negotiables (agent rules)

1) **Never guess the cluster.** Every action must specify:
- `cluster`: `mainnet-beta | devnet | testnet`
- RPC URL used (log it)
- commitment policy (`confirmed` or `finalized`)

2) **Never guess a token mint address.** For USDC:
- Use a canonical source (Circle / Solana docs) for the mint address.
- Still **verify on-chain** that the mint account:
  - is owned by the Token Program (usually `Tokenkeg...`) and
  - has expected decimals (USDC is typically **6**, but still verify).

3) **Differentiate wallet addresses vs token account addresses.**
- **SOL** transfers go to a **wallet address** (System Account / public key).
- **SPL tokens (USDC)** live in **token accounts** (usually an **Associated Token Account**, ATA).
- If you send SOL to a token account address, the SOL can be **stuck** (token accounts are owned by the token program, not the wallet).

4) **Always keep SOL for fees AND account creation.**
- SPL token transfers may need to create the recipient’s ATA (rent-exempt deposit).
- If you use `--fund-recipient`, **you pay** that cost; if you don’t, the transfer can fail.

5) **Start with a small test transfer.**  
New recipient? New mint? New cluster? First send the minimum viable amount, verify, then proceed.

6) **Idempotency is mandatory.**
- You must create a `payment_id`.
- Store: intent → signature → final status.
- Never “retry blindly” if you’re not sure whether a prior signature finalized.

7) **Limit blast radius.**
- Keep a low-balance **ops wallet** for routine sends.
- Keep long-term funds in a **multisig treasury** (Stack B).
- Enforce allowlists + per-tx caps for autonomous agents.

---

## 1) Decision tree (pick the right “stack”)

```text
START
│
├─ Q1: Do you need vendor-managed keys, recovery, compliance controls, or fiat rails?
│     ├─ YES → Stack D: Custodial / MPC platform (Coinbase)
│     └─ NO  → Q2
│
├─ Q2: Do you need multi-party approvals for funds (treasury governance)?
│     ├─ YES → Stack B: Squads Multisig (Solana-native smart accounts)
│     └─ NO  → Q3
│
├─ Q3: Do you want self-custody but with recoverability (auth-based) + policies?
│     ├─ YES → Stack C: Privy wallets (embedded/server) + policy guardrails
│     └─ NO  → Q4
│
└─ Q4: Pure self-custody single-sig?
      ├─ Human-operated → Stack A1: Solana CLI + hardware wallet
      └─ Agent-operated → Stack A2: Remote signer / MPC + SDK (minimize raw keys)
```

---

## 2) Cluster selection (Solana “networks”)

Solana has multiple clusters. Funds/tokens are **not interchangeable** across them.

**Common RPC endpoints (defaults):**
```text
mainnet-beta: https://api.mainnet-beta.solana.com
devnet:       https://api.devnet.solana.com
testnet:      https://api.testnet.solana.com
```

**CLI: inspect & set cluster**
```bash
solana config get
solana config set --url https://api.mainnet-beta.solana.com
# or: solana config set --url https://api.devnet.solana.com
```

**Agent guardrail:** any “production money move” must assert:
- cluster == `mainnet-beta`
- mint == canonical USDC mint for mainnet-beta
- recipient == allowlisted OR explicitly approved

---

## 3) Asset rules

### SOL
- Unit: **lamports**
- 1 SOL = **1,000,000,000 lamports**
- Used for transaction fees and rent-exempt deposits.

### USDC on Solana
- USDC is an **SPL token** (not native SOL).
- Typical decimals: **6** (verify on-chain anyway).
- Transfers may require an **Associated Token Account (ATA)** for the recipient.

---

## 4) Standard “payment intent” object (store before sending)

Agents should create and persist an intent *before* signing/broadcasting.

```json
{
  "payment_id": "inv_2026_02_23_001",
  "cluster": "mainnet-beta",
  "rpc_url": "https://api.mainnet-beta.solana.com",
  "asset": "SOL | USDC",
  "usdc_mint": "base58 mint (required if asset=USDC)",
  "to_wallet": "base58 pubkey",
  "amount_ui": "0.5 | 12.34",
  "amount_base_units": "lamports or token base units (string integer)",
  "commitment_required": "finalized",
  "created_at": "ISO-8601",
  "signature": "base58 (after broadcast)",
  "status": "prepared | broadcast | confirmed | finalized | failed | canceled"
}
```

**Notes**
- Store both `amount_ui` and `amount_base_units`.
- For USDC: `amount_base_units = amount_ui * (10^decimals)` computed using integer math (no floats).

---

## 5) Stack A — Self-custody + CLI (Solana CLI + SPL Token CLI)

### Choose Stack A when
- You want maximum portability & minimal dependencies
- You can safely manage keys (or use a hardware wallet / remote signer)

### A1: Human-operated (recommended for high-value single-sig)
- Use a hardware wallet with Solana CLI.

### A2: Agent-operated (automation)
- Avoid raw private keys in env vars/files when possible.
- Prefer remote signing (Stack C or MPC platform) if the agent is always-online.

### Install tools (reference)
- Solana CLI (`solana`, `solana-keygen`)
- SPL Token CLI (`spl-token`) via `cargo install spl-token-cli` (common workflow)

---

### A0: Basic safety checks (read-only)

```bash
# Check current cluster
solana config get

# Check SOL balance
solana balance <WALLET_ADDRESS> --url https://api.mainnet-beta.solana.com

# Confirm a transaction signature (verbose)
solana confirm -v <SIGNATURE> --url https://api.mainnet-beta.solana.com
```

---

### A1: Send SOL (CLI)

**Rule:** if the recipient is a brand-new address with no account state yet, include `--allow-unfunded-recipient`.

```bash
solana transfer \
  --from /path/to/sender.json \
  <RECIPIENT_WALLET_ADDRESS> \
  0.01 \
  --allow-unfunded-recipient \
  --url https://api.mainnet-beta.solana.com \
  --fee-payer /path/to/sender.json
```

**Attach a memo (invoice/payment id)**
```bash
solana transfer \
  --from /path/to/sender.json \
  <RECIPIENT_WALLET_ADDRESS> \
  0.01 \
  --allow-unfunded-recipient \
  --with-memo "inv_2026_02_23_001" \
  --url https://api.mainnet-beta.solana.com \
  --fee-payer /path/to/sender.json
```

---

### A2: Receive SOL (CLI)

To receive SOL, you only need a wallet address (pubkey):
```bash
solana address             # prints your current CLI wallet address
solana balance <YOUR_ADDR> # check
```

---

### A3: Send USDC (SPL Token CLI)

#### Core gotcha: recipient needs a USDC token account (ATA)
If the recipient doesn’t already have an ATA for USDC, you have two options:

- **Sender funds it** (best UX): `--fund-recipient`
- **Recipient pre-creates it** (recipient must have SOL): `spl-token create-account <USDC_MINT>`

#### Recommended send (sender funds recipient ATA)
```bash
USDC_MINT=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
TO=<RECIPIENT_WALLET_ADDRESS>

spl-token transfer \
  --fund-recipient \
  $USDC_MINT \
  12.34 \
  $TO \
  --url https://api.mainnet-beta.solana.com
```

**Important:** the sender must have enough SOL for:
- transaction fee
- **plus** the rent-exempt deposit to create the ATA (if needed)

#### Optional: include a memo (when required / for reconciliation)
Some token accounts (or exchanges) may require a memo on transfers.

```bash
spl-token transfer \
  --with-memo "inv_2026_02_23_001" \
  --fund-recipient \
  $USDC_MINT \
  12.34 \
  $TO \
  --url https://api.mainnet-beta.solana.com
```

#### Verify recipient received USDC
1) Determine the recipient’s USDC token account (ATA).
2) Check balance / transaction status.

```bash
# Show your token accounts (including USDC) for the current keypair
spl-token accounts --url https://api.mainnet-beta.solana.com

# Check a specific token account balance
spl-token balance <TOKEN_ACCOUNT_ADDRESS> --url https://api.mainnet-beta.solana.com

# Confirm the transfer signature
solana confirm -v <SIGNATURE> --url https://api.mainnet-beta.solana.com
```

---

## 6) Stack B — Squads Multisig (treasury / multi-party approval)

### Choose Stack B when
- You need multi-party approvals for funds (team treasury)
- You want roles, spending limits, timelocks, and an auditable approval trail
- You want the agent to **propose**, but humans to **approve**

### Core pattern (recommended)
- Treasury assets live in Squads
- Agent holds a low-balance ops wallet
- Agent proposes transactions; owners approve; then execute

### Operational guidance
- Threshold: **2-of-3** or **3-of-5** depending on risk
- Owners: diversify devices + custody (avoid “all keys on one laptop”)
- Use spending limits/roles for recurring low-risk payouts

### Tooling
- Squads App UI for day-to-day treasury management
- Squads SDK / CLI for automation (agent prepares & proposes)

---

## 7) Stack C — Privy wallets (embedded/server) + policies

### Choose Stack C when
- You want self-custody *with* recovery UX (auth-based)
- You want policy guardrails for autonomous agents
- You want to sign programmatically without raw private keys in your infra

### What “good” looks like for autonomous agents
- Create a dedicated **agent wallet**
- Attach policies:
  - max SOL per tx
  - max USDC per tx
  - allowlist destination addresses
  - enforce cluster allowlist (`mainnet-beta` only)
  - optional contract/program allowlists for non-transfer calls

### Solana token gotcha
Your sending code must handle:
- recipient ATA existence
- decimals (USDC = 6 on mainnet; still verify)

---

## 8) Stack D — Custodial / MPC (Coinbase)

### Choose Stack D when
- You want managed key security + recovery
- You need compliance posture, approvals, withdrawal controls
- You want to avoid handling private keys entirely

### Best-practice controls
- Address allowlisting for withdrawals
- Human approvals for large withdrawals
- Store platform withdrawal IDs + onchain signatures

---

## 9) Receiving payments (reconciliation & verification)

Solana supports **memos** and also has the **Solana Pay** “transfer request” standard.

### Preferred strategies (in order)

1) **Solana Pay transfer request (best for invoice reconciliation)**
- Generate a unique `reference` per invoice/payment
- Payer sends SOL/USDC and includes the reference key
- Your server verifies using on-chain lookup (match reference + recipient + amount + mint)

2) **Memo program (simple human-friendly reconciliation)**
- Instruct payer to include `memo = payment_id`
- You scan confirmed/finalized transactions for that memo and validate:
  - recipient
  - amount
  - mint (for USDC)
  - signature status

3) **Unique address per invoice (works, but more overhead for SPL tokens)**
- If you generate a new wallet per invoice, you may need to handle ATA creation for USDC.

### Confirmation policy
- For low-risk UX, you might accept `confirmed`.
- For irreversible fulfillment, prefer `finalized`.

---

## 10) Canonical addresses (USDC + core programs)

### USDC mint addresses
```text
USDC mint (mainnet-beta): EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
USDC mint (devnet):       4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU
```

### Core program IDs (useful for verification)
```text
SPL Token Program:        TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
Token-2022 Program:       TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb
Associated Token Program: ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL
```

---

## 11) Pre-send checklist (agent must pass all)

1. ✅ `cluster` + `rpc_url` explicitly set (and logged)
2. ✅ Recipient address validated (base58 pubkey) + allowlist checked if enabled
3. ✅ Asset is explicit (`SOL` or `USDC`)
4. ✅ If USDC:
   - ✅ mint address is canonical for that cluster
   - ✅ decimals verified on-chain (don’t trust memory)
   - ✅ plan for recipient ATA creation (`--fund-recipient` or recipient-precreated)
5. ✅ Sender has enough SOL for:
   - fees
   - + ATA rent if needed
6. ✅ Small test transfer done if new recipient/mint/cluster
7. ✅ For high-value: require human approval (Stack B or Stack D)
8. ✅ Broadcast: store signature immediately
9. ✅ Verify status at `confirmed`/`finalized` (per policy) exactly once (idempotent completion)

---

## 12) Blockhash / retries / concurrency (Solana-specific)

Solana transactions include a **recent blockhash** for “liveness”. This changes how retries must work.

### Rules for automated senders
- Always store the signature you broadcast.
- If you broadcast asynchronously and can’t confirm status:
  - **Do not immediately resend a “fresh” transaction** (risk double payment).
  - Wait until the original blockhash is expired, then decide whether to retry.

### Typical failure modes
- `Blockhash not found` / expired: rebuild with a new blockhash and resubmit.
- Account-in-use / parallelism conflicts: retry after a short delay (don’t spam).
- RPC inconsistencies: use a reliable RPC and/or query multiple sources for verification.

### When you need long-lived signing
- Use **durable nonces** / offline signing patterns (advanced; only if required).

---

## 13) Incident response

### If an agent wallet is compromised
- Stop funding ops wallets immediately
- Rotate credentials / revoke policies / disable API keys
- Move funds to multisig treasury (Stack B) or secure custody (Stack D)
- Audit logs: recipients, amounts, signatures

### If you sent USDC to the wrong mint / wrong cluster
- Do not repeat payment blindly
- Provide the signature + cluster to the counterparty
- Recovery is case-by-case and may require recipient cooperation

---

## 14) Tooling alternatives (when CLI isn’t enough)

- TypeScript: `@solana/web3.js` + `@solana/spl-token`
- Python: `solders` + `solana-py` (+ `spl-token` Python bindings)

Use code when you need:
- memos + reference keys in one transaction (custom instruction set)
- robust monitoring / indexing
- complex policy logic

---

## 15) Reference URLs (copy/paste)

```text
Solana CLI “Send & Receive Tokens”: https://docs.anza.xyz/cli/examples/transfer-tokens
Solana exchange integration guide (blockhash, finality, spl-token tips): https://solana.com/developers/guides/advanced/exchange
SPL Token docs (ATA + transfers): https://spl.solana.com/token
Payment verification guidance: https://solana.com/docs/advanced/payment-verification
Solana Pay transfer requests: https://docs.solanapay.com/core/transfer-request
USDC on Solana (Circle): https://www.circle.com/multi-chain-usdc/solana
Circle stablecoin docs (addresses & guides): https://developers.circle.com/stablecoins
Privy (Solana SPL token sending recipe): https://docs.privy.io/recipes/solana/send-spl-tokens
Squads Multisig docs: https://docs.squads.so/main
Coinbase CDP docs: https://docs.cdp.coinbase.com/
```
