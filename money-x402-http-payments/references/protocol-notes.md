# Protocol notes (x402 / HTTP 402)

This repo intentionally treats "x402" as an orchestration pattern:
- HTTP layer: `402 Payment Required` challenge + proof + retry
- Chain layer: actual settlement on EVM/Solana
- Ledger layer: idempotent reconciliation and crediting

Implementation detail reminder:
- Follow a specific x402 spec/version for exact headers/body fields.
- Keep this skill focused on workflow, verification, and safety invariants.

Safety invariants (do not relax)
- Always bind verification to a specific network identity (EVM chain_id / Solana cluster).
- Always bind verification to a specific asset identity (token contract / mint).
- Credit exactly once (idempotent transitions).
- Expire challenges and reject stale/replayed proofs.

