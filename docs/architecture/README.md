# Architecture

How the Shade Protocol contracts, factories, and shared components fit together.

- Workspace and crate layout (`shade`, `account`, `escrow`, `escrow_factory`, `subscription`, `ticketing`, `ticketing_factory`, `crowdfund`, `crowdfund_factory`) — *planned*.
- The `shade` contract's component structure (`contracts/shade/src/components/`) — *planned*.
- [Storage model](storage-model.md) — every `DataKey` variant, storage tier usage, ID-counter and collection patterns, and TTL/rent behavior in the `shade` contract.
- [Upgradeability, WASM Hash Management, and Versioning](upgradeability.md) — the admin and governance upgrade paths, storage compatibility rules for persisted types, migration patterns, and how a deployed build is identified.

← [Back to documentation home](../README.md)
