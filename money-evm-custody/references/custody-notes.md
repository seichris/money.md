# EVM custody quick notes

Safe treasury
- Use for high-value, infrequent moves.
- Keep owners diversified.
- Enforce a policy that treasury does not execute routine payouts.

Ops wallet
- Keep balance low.
- Apply destination allowlists and caps.
- Maintain ETH for gas on every chain the ops wallet uses.

USDC considerations
- USDC contract address is chain-specific.
- Verify receiver expects native USDC or a bridged variant.

Reference links
- Safe docs: https://docs.safe.global/
- Privy docs: https://docs.privy.io/
- Coinbase developer docs: https://docs.cdp.coinbase.com/
