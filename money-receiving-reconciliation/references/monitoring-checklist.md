# Monitoring and reconciliation checklist

Monitoring sources
- Direct RPC polling
- Webhooks from a trusted provider
- Indexers for events and token transfers

Checklist
1. Always store payment intent before observing on-chain activity.
2. Always store the on-chain tx id and never credit twice.
3. Always verify network identity, asset identity, recipient, and exact amount in base units.
4. Always apply confirmation or finality thresholds before fulfillment for higher risk payments.
5. Log all state transitions for auditability.
