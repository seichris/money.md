# Foundry cast cheatsheet for payments

This is a minimal cheatsheet for the money-evm-payments skill.

Setup
- Set RPC_URL to the chain you intend to use.
- For safety, always verify chain_id.

Identity and balances
```bash
cast chain-id --rpc-url RPC_URL
cast balance ADDRESS --rpc-url RPC_URL
```

ERC-20 inspection
```bash
cast erc20-token decimals TOKEN --rpc-url RPC_URL
cast erc20-token symbol TOKEN --rpc-url RPC_URL
cast erc20-token balance TOKEN ADDRESS --rpc-url RPC_URL
```

Unit conversions
```bash
cast to-wei AMOUNT ether
cast parse-units AMOUNT_UI DECIMALS
cast format-units AMOUNT_BASE DECIMALS
```

Send ETH
```bash
cast send RECIPIENT_ADDRESS --value AMOUNT_ETHether --rpc-url RPC_URL SIGNER_ARGS
```

Send ERC-20
```bash
cast erc20-token transfer TOKEN RECIPIENT_ADDRESS AMOUNT_BASE --rpc-url RPC_URL SIGNER_ARGS
```

Receipt and tx info
```bash
cast receipt TX_HASH --rpc-url RPC_URL
cast tx TX_HASH --rpc-url RPC_URL
```
