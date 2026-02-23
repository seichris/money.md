# money.md

A repo of **agent skills** for safely **sending and receiving crypto payments** and designing **custody / treasury** workflows.

Current scope:
- **EVM**: Ethereum L1 + common L2s (ETH + USDC)
- **Solana**: mainnet-beta + devnet/testnet (SOL + USDC)

Non-goals:
- Trading, DEX routing, DeFi, yield, MEV, price advice

---

## Quick start

If you are not 100% sure which network or custody model to use, start here:

- **Decision tree (entry point):** [money-decision-tree/SKILL.md](money-decision-tree/SKILL.md)

That router will send you to the right focused skill:
- Execute a transfer on EVM: [money-evm-payments/SKILL.md](money-evm-payments/SKILL.md)
- Execute a transfer on Solana: [money-solana-payments/SKILL.md](money-solana-payments/SKILL.md)
- Design custody on EVM: [money-evm-custody/SKILL.md](money-evm-custody/SKILL.md)
- Design custody on Solana: [money-solana-custody/SKILL.md](money-solana-custody/SKILL.md)
- Receiving and reconciliation: [money-receiving-reconciliation/SKILL.md](money-receiving-reconciliation/SKILL.md)

---

## Skill map

### Router
- [money-decision-tree/SKILL.md](money-decision-tree/SKILL.md)  
  High-level routing: EVM vs Solana, custody tradeoffs, and the next skill to load.

### Payment execution
- [money-evm-payments/SKILL.md](money-evm-payments/SKILL.md)  
  Send/receive **ETH + USDC** on Ethereum and EVM L2s using Foundry `cast`.

- [money-solana-payments/SKILL.md](money-solana-payments/SKILL.md)  
  Send/receive **SOL + USDC** on Solana using `solana` + `spl-token`, including ATA gotchas and memos.

### Custody and treasury architecture
- [money-evm-custody/SKILL.md](money-evm-custody/SKILL.md)  
  Treasury and ops architecture on EVM: Safe multisig, recoverable ops wallets, caps, allowlists, incident response.

- [money-solana-custody/SKILL.md](money-solana-custody/SKILL.md)  
  Treasury and ops architecture on Solana: Squads multisig, ops wallet policies, incident response.

### Receiving, verification, and reconciliation
- [money-receiving-reconciliation/SKILL.md](money-receiving-reconciliation/SKILL.md)  
  Invoice matching strategies, confirmation/finality policies, and idempotent crediting across EVM and Solana.

---

## Registries and cheatsheets

These are used by the payment skills to avoid guessing addresses/mints and to keep SKILL.md concise.

### EVM
- Native USDC addresses (by chain):  
  [money-evm-payments/references/usdc-addresses.md](money-evm-payments/references/usdc-addresses.md)

- Foundry `cast` cheatsheet:  
  [money-evm-payments/references/cast-cheatsheet.md](money-evm-payments/references/cast-cheatsheet.md)

### Solana
- USDC mints (by cluster):  
  [money-solana-payments/references/usdc-mints.md](money-solana-payments/references/usdc-mints.md)

- Token account gotchas (wallet vs token account, ATAs):  
  [money-solana-payments/references/token-account-gotchas.md](money-solana-payments/references/token-account-gotchas.md)

- Solana CLI cheatsheet:  
  [money-solana-payments/references/solana-cli-cheatsheet.md](money-solana-payments/references/solana-cli-cheatsheet.md)

---

## Repo conventions

- Each skill lives in its own **kebab-case** folder with a **SKILL.md** at the top level.
- Supporting details belong in a **references/** folder inside the skill folder.
- SKILL.md files use YAML frontmatter for `name` and `description` so skill triggers are predictable.
- All money-moving flows require:
  - explicit network identity (chain_id or cluster)
  - explicit asset identity (token contract or mint for USDC)
  - idempotency and logging (payment_id + tx id)
  - conservative defaults if the user does not specify risk tolerance

---

## Development notes

Common edits:
- Add a new EVM chain USDC address: update  
  [money-evm-payments/references/usdc-addresses.md](money-evm-payments/references/usdc-addresses.md)
- Add a new Solana cluster mint: update  
  [money-solana-payments/references/usdc-mints.md](money-solana-payments/references/usdc-mints.md)

When you update registries, also update the relevant skill version in YAML frontmatter.

---

## License and safety

This repo is operational guidance for agents. It is not financial advice.

Always test on devnet or small amounts first, and treat any automated wallet as potentially compromiseable.
