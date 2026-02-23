# Solana CLI cheatsheet for payments

Cluster config
```bash
solana config get
solana config set --url https://api.mainnet-beta.solana.com
```

Balances and confirmation
```bash
solana balance WALLET_PUBKEY --url RPC_URL
solana confirm -v SIGNATURE --url RPC_URL
```

Send SOL
```bash
solana transfer --from SENDER_KEYPAIR_PATH RECIPIENT_WALLET_PUBKEY AMOUNT_SOL --allow-unfunded-recipient --with-memo PAYMENT_ID --url RPC_URL --fee-payer SENDER_KEYPAIR_PATH
```

SPL token accounts and balances
```bash
spl-token accounts --url RPC_URL
spl-token balance TOKEN_ACCOUNT_ADDRESS --url RPC_URL
```

Send SPL token, funding recipient associated token account if needed
```bash
spl-token transfer --fund-recipient --with-memo PAYMENT_ID MINT AMOUNT_UI RECIPIENT_WALLET_PUBKEY --url RPC_URL
```
