# Native USDC addresses for EVM networks

Use this file as a registry for native USDC contract addresses on common EVM chains.

Rules
- Do not guess token addresses.
- If a chain is not listed here, fetch the address from an explicitly trusted source and add it to this registry.
- Always verify decimals on-chain.

Note on variants
- Some chains have bridged USDC variants like USDC.e or USDbC.
- Treat each variant as a distinct asset. Only use it if the receiver explicitly requests it.

Registry

Ethereum mainnet
- chain_id: 1
- USDC: 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48

Arbitrum One
- chain_id: 42161
- USDC: 0xaf88d065e77c8cC2239327C5EDb3A432268e5831

OP Mainnet
- chain_id: 10
- USDC: 0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85

Base
- chain_id: 8453
- USDC: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913

Linea
- chain_id: 59144
- USDC: 0x176211869cA2b568f2A7D4EE941E073a821EE1ff

zkSync Era
- chain_id: 324
- USDC: 0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4

Source
- Circle USDC contract addresses: https://developers.circle.com/stablecoins/usdc-contract-addresses
