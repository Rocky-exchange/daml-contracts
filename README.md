# Rocky-exchange DAML contracts

DAML smart contracts for Rocky.exchange, deployed on the [Canton Network](https://www.canton.network/).

This repository contains the on-chain logic for two related packages:

| Package | Purpose |
| --- | --- |
| [`exchange-app/`](./exchange-app) | Records filled trades on chain and mints a per-trade Mining Reward (`ROCKY`) that the trader can later claim into a `TokenHolding`. |
| [`my-token/`](./my-token) | A standalone token implementing the Splice `HoldingV1` interface (`splice-api-token-holding-v1`). |

The two packages are independent. `exchange-app` defines its own `TokenHolding` template (not Splice-interface compatible) for the mining-reward flow. `my-token` is a separate experiment in implementing the Splice token API. See **Architecture notes** below.

## Repository layout

```
daml-contracts/
├── exchange-app/
│   ├── daml.yaml
│   └── daml/
│       └── ExchangeApp.daml
├── my-token/
│   ├── daml.yaml
│   ├── daml/
│   │   └── MyToken.daml
│   └── dars/                 # local-only: drop Splice DAR dependencies here
└── README.md
```

## Prerequisites

- DAML SDK `3.3.0-snapshot.20250415.13756.0.vafc5c867` (matches `sdk-version` in each `daml.yaml`)
- For `my-token`: the two Splice token-API DARs

## Building

### `exchange-app`

```bash
cd exchange-app
daml build
```

### `my-token`

`my-token` depends on two DARs from the [Splice](https://github.com/digital-asset/splice) project. They are **not** redistributed here. Drop them into `my-token/dars/` before building:

```
my-token/dars/splice-api-token-holding-v1-1.0.0.dar
my-token/dars/splice-api-token-metadata-v1-1.0.0.dar
```

If you have a local Splice checkout, the DARs typically live at `<splice-root>/daml/dars/`. Once the files are in place:

```bash
cd my-token
daml build
```

## ExchangeApp flow

```
ExchangeApp (appProvider)
        │
        │ direct create  (or via RecordTrade choice)
        ▼
TradeOrder + MiningReward         ── on-chain record + claimable reward
        │
        │ ClaimReward (controller: trader)
        ▼
TokenHolding (issuer = appProvider, owner = trader)
        │
        │ ProposeTransfer
        ▼
TokenHolding (sender remainder) + PendingTokenHolding
                                          │
                                          │ Accept (controller: receiver)
                                          ▼
                                  TokenHolding (issuer, receiver)
```

`PendingTokenHolding` is the receiver-side inbox pattern: it is signed only by the issuer so the sender can lock funds without the receiver pre-authorizing. Reject flips the direction so the original sender can Accept it back.

## MyToken flow

```
MyToken (owner)
   │
   │ ProposeTransfer  (consuming - locks the full balance)
   ▼
TransferProposal (signatory: sender, issuer)
   │
   ├── AcceptTransfer  (controller: receiver)  → (Optional sender-remainder, receiver-holding)
   ├── RejectTransfer  (controller: receiver)  → all funds back to sender
   └── CancelTransfer  (controller: sender)    → all funds back to sender
```

The proposal is consuming on purpose: this prevents proposing the same balance twice.

## Known issues / future work

- **`ExchangeApp.RecordTrade` is currently bypassed.** The reference integration script (not included here) creates `TradeOrder` and `MiningReward` directly via `CreateCommand` rather than exercising `RecordTrade`. As a side effect the supply-cap check inside `RecordTrade` is not enforced. If you reintroduce `RecordTrade` into the live flow, change it to a consuming choice that recreates `ExchangeApp` with an updated `totalRewardsMinted`.
- **Two parallel token implementations.** `exchange-app` issues `TokenHolding` (custom, not Splice-interface compatible). `my-token` defines `MyToken` with the `HoldingV1` interface. Mining rewards are minted as `TokenHolding`, not `MyToken`, so they are not interoperable with Splice token-aware tooling. Unify before mainnet.
- **No DAML script tests.** Add `daml/Test.daml` scripts covering at least: trade record + claim, full-balance transfer, partial transfer with remainder, reject, cancel.

## License

[MIT](./LICENSE)
