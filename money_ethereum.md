# money.md — Sending & receiving crypto (ETH + USDC) on Ethereum L1 & L2s

**SKILL GOAL:** Teach an AI agent how to safely **send** and **receive** crypto payments (focus: **ETH** and **USDC**) on Ethereum mainnet (L1) and common EVM L2s.

**Last updated:** 2026-02-23  
**Scope:** Ethereum L1 + EVM L2s (e.g., Arbitrum One, OP Mainnet, Base, zkSync Era, Linea).  
**Non-goals:** Trading, yield farming, “best routes”, MEV strategies, or anything that bypasses compliance.

---

## 0) Non‑negotiables (apply to every branch)

1. **Never guess**:
   - Never guess the **chain**.
   - Never guess a **token contract address**.
   - Never guess **token decimals**.
2. **Always confirm**:
   - Chain name + chain ID
   - Recipient address (checksum + allowlist if possible)
   - Asset (ETH vs USDC) and **which USDC** (native vs bridged)
   - Amount in **base units** (wei for ETH; token base units for ERC‑20s)
3. **Always keep gas**: every account that must transfer USDC must also hold **ETH for gas** on that same chain.
4. **Start small**: first payment to a new counterparty = **test transfer** (small amount), then the real transfer.
5. **Idempotency & logging**: every send must have a unique `payment_id` and a saved record of:
   - chain_id, from, to, asset, token_address (if ERC‑20), amount_base_units, tx_hash, nonce, timestamp
6. **Guardrails before autonomy**:
   - Limit max value per tx
   - Restrict allowed chains
   - Restrict allowed contracts / recipient addresses
   - Require human approval for treasury moves

---

## 1) Decision tree (pick the “stack” first)

Use this to choose the best custody + tooling setup.

```
START
│
├─ Q1: Must funds be recoverable via a vendor (password reset / KYC / compliance / fiat rails)?
│     │
│     ├─ YES → Stack D: Custodial (Coinbase)
│     │         Best for: regulated custody, fiat on/off-ramps, enterprise controls.
│     │
│     └─ NO  → Q2
│
├─ Q2: Do you need multi-party approvals (treasury, team funds, or high-value)?
│     │
│     ├─ YES → Stack B: Safe Smart Account (multisig)
│     │         Best for: treasury, separation of duties, auditable approvals.
│     │
│     └─ NO  → Q3
│
├─ Q3: Do you want “self-custody but recoverable” via app auth (passkeys/social login) for users/agents?
│     │
│     ├─ YES → Stack C: Privy (embedded/server wallets + policies)
│     │         Best for: agent wallets w/ policy guardrails + recovery UX.
│     │
│     └─ NO  → Q4
│
└─ Q4: Pure self-custody single-sig?
      │
      ├─ Human-operated → Stack A1: Hardware wallet + CLI (Foundry cast)
      │
      └─ Agent-operated (automated) → Stack A2: Remote signer (KMS/Turnkey) + CLI (Foundry cast)
```

**Recommended architecture for most agent systems:**
- **Treasury:** Stack B (Safe multisig)
- **Operations wallet(s):** Stack C (Privy server wallets with policies) *or* Stack A2 (KMS/Turnkey key)
- **Flow:** Treasury funds ops wallet → ops wallet executes many small payments → ops wallet is periodically topped up

---

## 2) Chain selection (L1 vs L2)

### Default rule
- If the counterparty **did not explicitly specify a chain**, **do not send**. Ask for: “Which chain: Ethereum, Arbitrum, Optimism, Base, …?”

### When to use L1 (Ethereum mainnet)
- Counterparty only supports L1 (exchanges, some institutions, legacy workflows)
- High value + you want maximum neutrality / broad compatibility
- You’re interacting with contracts that only exist on L1

### When to use L2s
- Most everyday transfers and app payments (lower fees)
- Micro-payments and higher frequency payments

### L2 gotcha: “USDC” is not always the same token
On some chains there are multiple “USDC-like” tokens:
- **Native USDC** (Circle-issued) vs **bridged** variants like `USDC.e` or `USDbC`.
- You must match what the receiver expects.

---



## 2b) Moving funds between chains (bridging)

**Rule:** Don’t bridge unless you explicitly need to. Prefer sending on the chain the recipient requests.

### Bridging decision tree (ETH vs USDC)

```
Need to move value from Chain A → Chain B?
│
├─ Asset = USDC?
│    │
│    ├─ YES → If both chains support Circle CCTP:
│    │           Prefer CCTP (burn on source, mint on destination; avoids wrapped/LP bridge risk).
│    │        Else:
│    │           Prefer the canonical bridge for the destination L2 (may produce bridged variants like USDC.e).
│    │
│    └─ NO (ETH) → Prefer canonical bridge or a reputable on-ramp/off-ramp that can deliver ETH on the destination chain.
│
└─ After bridging:
     - Verify you received funds on the destination chain
     - Verify the asset is the expected variant (native USDC vs bridged)
     - Keep enough ETH for gas on the destination chain before attempting USDC transfers
```

### USDC variant warning (example: Arbitrum)
Some L2s support both:
- **Native USDC (`USDC`)** and
- **Bridged USDC (`USDC.e`)** (a different token contract)

You must confirm which one the receiver expects before you send.


## 3) Asset rules (ETH vs USDC)

### ETH
- Needed for gas and can be sent as a native value transfer.
- Amount unit: **wei** (1 ETH = 1e18 wei).

### USDC (ERC‑20)
- USDC is an ERC‑20 contract on each EVM chain.
- Amount unit: **token base units** (often 6 decimals, but don’t assume).
- Transfers call `transfer(to, value)` on the token contract.

---

## 4) Standard “payment intent” format (what the agent should store)

Use a single canonical object for “send” and “receive”.

```json
{
  "payment_id": "uuid-or-hash",
  "direction": "send | receive",
  "chain": "ethereum | arbitrum | optimism | base | ...",
  "chain_id": 1,
  "asset": "ETH | USDC",
  "token_address": "0x... (required if asset is USDC)",
  "from": "0x...",
  "to": "0x...",
  "amount_human": "12.34",
  "amount_base_units": "12340000",
  "required_confirmations": 1,
  "created_at": "ISO-8601",
  "tx_hash": "0x... (after broadcast)",
  "status": "prepared | broadcast | confirmed | failed | canceled"
}
```

---

## 5) Stack A — Pure self-custody + CLI (Foundry `cast`)

### Choose Stack A when
- You want maximum simplicity and portability
- You are okay managing private keys yourself
- You want a CLI-first workflow (agents, servers, scripts)

### A1: Human-operated (hardware wallet)
**Best for:** occasional manual transfers, highest key safety.

### A2: Agent-operated (remote signer)
**Best for:** automation while keeping private keys out of disk/env vars.

**Preferred remote signing options:**
- Cloud KMS (AWS / GCP)
- Turnkey (purpose-built signing infra)

### Minimal `cast` environment
Set:
- `ETH_RPC_URL` for your chain RPC endpoint (per environment/chain)
- Optional: `CHAIN` / `--chain` if you want extra safety checks

### Common read commands (safe)
```bash
# Identify the chain (sanity check)
cast chain-id --rpc-url $ETH_RPC_URL

# ETH balance (wei)
cast balance 0xYourAddress --rpc-url $ETH_RPC_URL

# ERC-20: decimals and balance
cast erc20-token decimals 0xTokenAddress --rpc-url $ETH_RPC_URL
cast erc20-token balance  0xTokenAddress 0xYourAddress --rpc-url $ETH_RPC_URL

# Convert human amounts <-> base units
cast parse-units  12.34 6      # -> 12340000
cast format-units 12340000 6   # -> 12.34
```

### Send ETH (Stack A)
```bash
# Send 0.01 ETH (agent or human)
cast send 0xRecipient \
  --value 0.01ether \
  --rpc-url $ETH_RPC_URL \
  --private-key $SENDER_PRIVATE_KEY
```

### Send USDC (Stack A)
1) Resolve the **correct USDC token address** on the target chain  
2) Query decimals  
3) Convert amount → base units  
4) Transfer

```bash
USDC=0xYourUsdcContract
TO=0xRecipient
DECIMALS=$(cast erc20-token decimals $USDC --rpc-url $ETH_RPC_URL)
AMOUNT=$(cast parse-units 12.34 $DECIMALS)

cast erc20-token transfer $USDC $TO $AMOUNT \
  --rpc-url $ETH_RPC_URL \
  --private-key $SENDER_PRIVATE_KEY
```

### Stronger key handling for Stack A
Avoid raw `$SENDER_PRIVATE_KEY` when possible:
- Use an encrypted keystore file (`cast send --keystore ...`)
- Use hardware wallet (`--ledger` / `--trezor`)
- Use remote signer (`--aws` / `--gcp` / `--turnkey`)

### Cast signer options (quick reference)

Foundry `cast` supports multiple signing modes; for agent systems, prefer **remote signers** to avoid raw keys on disk/env vars.

- **Raw**: `--private-key` (last resort)
- **Keystore**: `--keystore <path>` or `--account <name>` (+ `--password-file`)
- **Hardware**: `--ledger` or `--trezor`
- **Remote**:
  - `--aws` (requires `AWS_KMS_KEY_ID`)
  - `--gcp` (requires `GCP_PROJECT_ID`, `GCP_LOCATION`, `GCP_KEY_RING`, `GCP_KEY_NAME`, `GCP_KEY_VERSION`)
  - `--turnkey` (requires `TURNKEY_API_PRIVATE_KEY`, `TURNKEY_ORGANIZATION_ID`, `TURNKEY_ADDRESS`)



---

## 6) Stack B — Safe Smart Account (multisig treasury)

### Choose Stack B when
- You need **multi-party approval**
- You need separation of duties (agent proposes, humans approve)
- You want an auditable, battle-tested treasury

### Core pattern
- Agent proposes a Safe tx (off-chain)
- Owners co-sign (off-chain signatures)
- Once threshold is met, someone executes the tx (on-chain)

### Operational guidance
- **Treasury funds never live in an agent hot wallet.**
- Keep an **ops wallet** funded for routine tasks; refill from Safe.

### Recommended controls
- Threshold: 2-of-3 or 3-of-5 (context-dependent)
- Owner keys: diversify devices + storage (avoid all keys on one laptop)
- Use spending limits/roles for recurring low-risk payments
- Prefer built-in simulation tooling before executing

### Programmatic integration
If your agent needs to propose transactions:
- Use the Safe Transaction Service API (or Safe{Core} SDK API Kit) to store proposals + signatures.
- Use the Safe{Core} Protocol Kit to create/propose/execute txs in TypeScript.

---

## 7) Stack C — Privy wallets (embedded/server) + policies

### Choose Stack C when
- You want **recovery UX** (auth-based)
- You want **policy guardrails** for an autonomous agent
- You want server-side wallets that can transact with restrictions

### What “good” looks like for autonomous agents
- Create a dedicated **agent wallet**
- Attach **policies**:
  - max ETH per tx
  - allow only specific chains (e.g., Base only)
  - allow only specific recipient addresses (allowlist)
  - allow only specific contracts

### Receiving & monitoring
- Use webhook events for wallet lifecycle and recovery events.
- Always verify webhook payload signatures.

### USDC sending reminder (Privy)
- USDC contract address differs per network.
- USDC decimals are usually 6 (but still query on-chain).

---

## 8) Stack D — Custodial (Coinbase)

### Choose Stack D when
- You need fiat on/off ramps, KYC/compliance posture, account recovery
- You want enterprise-grade controls and approvals
- You want to avoid managing private keys directly

### Best-practice controls
- Enable address allowlisting (“address book” / whitelisting)
- Require UI approvals for withdrawals (especially high-value)
- Use idempotency keys for API-initiated withdrawals
- Record the withdrawal transaction ID + onchain tx hash

### Notes
- Custodial flows differ across Coinbase products (retail app vs Exchange vs Prime vs Commerce).
- Treat custodial APIs as “payment rails” with their own states (pending approval → approved → broadcast → confirmed).

---

## 9) Receiving payments (onchain)

### Pick a receiving address type
- **Ops wallet**: receives routine payments; limited balance
- **Treasury (Safe)**: receives large/long-term funds
- **Custodial deposit address**: best for fiat settlement workflows (but adds vendor risk)

### Provide a complete “payment request”
A request should always include:
- Chain (name + chain ID)
- Asset (ETH or USDC)
- Recipient address
- For USDC: token contract address + decimals (or “native USDC” requirement)
- Amount + currency
- `payment_id` / memo (off-chain reference)

Example (human-readable):
```text
Pay: 12.34 USDC
Chain: Base (8453)
To: 0xAbC...
Token: USDC (Circle native)
Payment ID: inv_2026_02_23_001
```



### Reconciling payments to invoices (EVM has no native memo)

If you need to reliably match “a payment” to an invoice/order/user, you need a strategy beyond “look at the wallet balance”.

Preferred options (in order):

1. **Unique address per invoice**  
   - Generate a fresh receive address for each invoice or customer.
   - Easiest to reconcile; best for agents.

2. **Smart-contract pay endpoint** (best when you control the app)  
   - User calls `pay(invoiceId, amount, token)` and the contract emits an event with `invoiceId`.
   - Most reliable at scale.

3. **Unique-amount encoding** (last resort)  
   - Add a tiny “invoice suffix” to the amount (e.g., 12.340017 USDC).
   - Works poorly with rounding/partial payments; can confuse humans.

4. **Off-chain payment_id only**  
   - Only works if the payer can provide the tx hash back out-of-band.


### Confirming receipt
- ETH: confirm the tx is mined and the value arrived
- USDC: confirm a `Transfer` event from the correct token contract and amount

**Always**:
- Wait for your chosen confirmation count
- Mark the payment as complete exactly once (idempotent)

---

## 10) USDC contract addresses (canonical: Circle-issued native USDC)

These are **native USDC** addresses published by Circle.

> You must still verify you are using the correct chain and not a spoofed token.

```text
Ethereum (1):     0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
Arbitrum (42161): 0xaf88d065e77c8cC2239327C5EDb3A432268e5831
OP Mainnet (10):  0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85
Base (8453):      0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
Linea (59144):    0x176211869cA2b568f2A7D4EE941E073a821EE1ff
zkSync (324):     0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4
```

### USDC variants warning
- If you see `USDC.e` or `USDbC`, treat it as a distinct asset.
- Never send bridged USDC variants to a system that expects “native USDC” without explicit confirmation.

---

## 11) Pre-send checklist (agent must pass all)

1. ✅ Have explicit chain + asset + recipient
2. ✅ Recipient address is checksummed and (if possible) allowlisted
3. ✅ For USDC: token address is from a canonical source; decimals read onchain
4. ✅ Amount base units computed and logged
5. ✅ Sender has enough ETH for gas on that chain
6. ✅ Simulate/preview if tooling supports it (especially contract calls)
7. ✅ For high-value: require human approval (Safe or custodial UI approval)
8. ✅ Broadcast, save tx hash immediately
9. ✅ Confirm receipt, then mark payment complete

---



## 11b) Nonce, concurrency, and stuck transactions (agent operations)

### Nonce locking (must-have for automated senders)
- Use a **single-writer** model per `from` address (or a DB/distributed lock).
- If multiple workers can send, implement a **nonce allocator** that hands out nonces atomically.
- Always record `nonce` with `payment_id` so retries are safe and idempotent.

### If a transaction is stuck (pending too long)
- Don’t spam new nonces blindly.
- Replace the pending tx **with the same nonce** and a higher fee (wallet/RPC must support replacement).
- If you need to “cancel”, send a **0 ETH** transfer to yourself with the same nonce and higher fee.


## 12) Incident response (if something goes wrong)

### If the agent is compromised
- Freeze funding: stop topping up ops wallets
- Rotate keys / disable policies / remove compromised owners
- Move remaining funds to Safe treasury (or to a known-secure wallet)
- Review logs for unauthorized txs

### If you sent on the wrong chain or wrong USDC variant
- Do not repeat the payment blindly
- Notify recipient with tx hash + chain
- Treat recovery as case-by-case (may require recipient cooperation)

---

## 13) Tooling alternatives (when not using `cast`)

Prefer `cast` by default. Alternatives when there’s a strong reason:
- `seth` (DappTools): UNIXy CLI composition style
- `ethdo`: wallet/account management + interactions with Ethereum consensus/remote signers
- `viem` / `ethers`: write a small script when you need richer programmatic control

---

## 14) Reference URLs (copy/paste)

```text
Foundry Cast docs:              https://getfoundry.sh/cast/
Foundry Cast CLI reference:     https://getfoundry.sh/cast/reference/
Safe docs (Smart Accounts):     https://docs.safe.global/
Circle USDC addresses:          https://developers.circle.com/stablecoins/usdc-contract-addresses
Circle CCTP (USDC bridging):    https://developers.circle.com/cctp
Arbitrum USDC vs USDC.e:        https://docs.arbitrum.io/arbitrum-bridge/usdc-arbitrum-one
Coinbase Prime withdrawals:     https://docs.cdp.coinbase.com/prime/concepts/transactions/withdrawals
Coinbase Prime API reference:   https://docs.cdp.coinbase.com/api-reference/prime-api/
Privy agentic wallets:          https://docs.privy.io/recipes/agent-integrations/agentic-wallets
Privy OpenClaw guide:           https://privy.io/blog/securely-equipping-openclaw-agents-with-privy-wallets
```

