# Solana token account gotchas for payments

Key concept
- Wallet public keys identify owners.
- SPL tokens are stored in token accounts.
- The associated token account is the standard token account derived for a wallet and a mint.

Why this matters
- SOL transfers should go to a wallet pubkey, not a token account address.
- SPL token transfers require the correct token account, typically the associated token account.

Operational rules
1. For SOL payments, request and use the wallet pubkey.
2. For USDC payments, request the wallet pubkey and the mint, then let your tooling derive or create the associated token account.
3. If you are paying a recipient for the first time, prefer funding the recipient associated token account for better UX, but account for the SOL cost.

Common failure modes
- Sending SOL to a token account address, stranding funds.
- Sending USDC without funding the recipient associated token account when it does not exist.
- Confusing devnet and mainnet-beta mints.

Recommended CLI flags
- Use `--fund-recipient` for SPL token transfers when appropriate.
- Use `--with-memo` with a payment_id for reconciliation.
