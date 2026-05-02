# Rocky-exchange DAML contracts

DAML smart contracts for [Rocky.exchange](https://rocky.exchange), a perpetual-futures DEX deployed on the [Canton Network](https://www.canton.network/).

This repository contains the on-chain logic for two related packages:

| Package | Purpose |
| --- | --- |
| [`exchange-app/`](./exchange-app) | **v2 perpetuals stack**: KYC/KYB, LockedAmulet-style margin, atomic batch settlement, FeaturedAppActivityMarker (V2-shaped) emission, insurance fund. The legacy v1 ROCKY-mining flow (TradeOrder + MiningReward + TokenHolding) is preserved for backward compatibility. |
| [`rocky/`](./rocky) | A standalone token implementing the Splice `HoldingV1` interface (`splice-api-token-holding-v1`). |

The two packages are independent. `rocky` is a separate experiment in implementing the Splice token API.

## Repository layout

```
daml-contracts/
├── exchange-app/
│   ├── daml.yaml
│   └── daml/
│       ├── Common.daml         -- shared types: Side, MarketSymbol, Fill, helpers
│       ├── Margin.daml         -- KYC/KYB + MarginAccount + LockedMargin
│       ├── Perpetual.daml      -- markets, OrderIntent, Position, BatchSettlement,
│       │                         FundingRound, Liquidation, FeaturedAppActivityMarker,
│       │                         InsuranceFund
│       └── ExchangeApp.daml    -- top-level operator (App Provider) + legacy v1 templates
├── rocky/
│   ├── daml.yaml
│   ├── daml/
│   │   └── MyToken.daml
│   └── dars/                   -- local-only: drop Splice DAR dependencies here
└── README.md
```

## Prerequisites

- DAML SDK `3.3.0-snapshot.20250415.13756.0.vafc5c867` (matches `sdk-version` in each `daml.yaml`)
- For `rocky`: the two Splice token-API DARs (see below)

## Building

### `exchange-app`

```bash
cd exchange-app
daml build
```

### `rocky`

`rocky` depends on two DARs from the [Splice](https://github.com/digital-asset/splice) project. They are **not** redistributed here. Drop them into `rocky/dars/` before building:

```
rocky/dars/splice-api-token-holding-v1-1.0.0.dar
rocky/dars/splice-api-token-metadata-v1-1.0.0.dar
```

If you have a local Splice checkout, the DARs typically live at `<splice-root>/daml/dars/`. Once the files are in place:

```bash
cd rocky
daml build
```

## Architecture (v2 — perpetuals)

Rocky Exchange is a hybrid CLOB/perpetuals DEX:

* An **off-chain Rust matching engine** sequences fills with sub-microsecond latency and submits one Daml transaction per batch (documented 8–15:1 batching ratio, sub-linear ledger pressure).
* The **on-chain Daml stack** in `exchange-app/daml/` handles compliance, custody, position lifecycle, settlement, funding, liquidation, and Canton-Coin reward emission via FeaturedAppActivityMarkers tied to **verified notional**.

### Trade flow

```
Trader app ──► OrderIntent (signatory operator + trader)
                                    │
                                    ▼
              Off-chain Rust matching engine batches matched fills
                                    │
                                    ▼
              Operator submits ONE Daml transaction:
                ExchangeApp.OpenBatch ──► BatchSettlement
                BatchSettlement.ProcessFill (× N, atomic)
                  ├─ fetchValidKyc (taker, maker)
                  ├─ wash-trade & cross-market guards
                  ├─ exercise OrderIntent.Match (taker, maker)
                  ├─ MarginAccount.LockForPosition + ApplyDelta(fees)
                  ├─ create Position (taker)
                  ├─ create Position (maker)
                  └─ create FeaturedAppActivityMarker × 2  (verified notional)
                BatchSettlement.MarkOperatorRevenue
                BatchSettlement.MarkInsuranceContribution
                BatchSettlement.CloseBatch
```

If any single `ProcessFill` fails (KYC expired, wash-trade, insufficient margin, market paused, etc.) the entire batch transaction is rolled back — Daml's transaction model delivers the documented all-or-nothing settlement.

### Position lifecycle

```
Position
   │
   ├── ApplyFunding   (per fundingIntervalSec, signed delta)
   │      │
   │      └── MarginAccount.ApplyDelta
   │
   ├── ReducePosition (full or partial close at markPrice)
   │      │
   │      ├── full   : LockedMargin.Release  → MarginAccount += freed
   │      │           MarginAccount.ApplyDelta(realised P&L)
   │      └── partial: LockedMargin.Split → release proportional slice,
   │                   keep residue, recreate Position with reduced size
   │
   └── Liquidate      (mark price crosses bankruptcy)
          │
          └── LockedMargin.Seize → operator routes seized + fee to InsuranceFund
              + emits an "INSURANCE" FeaturedAppActivityMarker
```

### Compliance

`ExchangeApp.kycRequired` is enforced at every position-opening choice via `Margin.fetchValidKyc`, which checks:

* `KycCertificate.trader` matches the OrderIntent's `trader`
* `expiresAt > now` (no stale certs)

Off-chain identity verification is mandatory before a `KycCertificate` is issued. Combined with the OrderIntent multi-party signature requirement, this gives the protocol three layers of wash-trade defence:

1. **Identity** — `KycCertificate` filters bots.
2. **Economic** — taker fee 2.5 bps + Canton traffic fees make round-trip self-trading negative-EV.
3. **Technical** — `BatchSettlement.ProcessFill` aborts if `taker == maker` and the off-chain Rust engine layers in self-match prevention and related-account heuristics before the batch even reaches the ledger.

Activity markers carry **verified notional** (sourced from the consumed `OrderIntent` quantities × matched price), not raw trade-count, so reward weighting cannot be inflated by spamming small fills.

## v1 ROCKY-mining flow (legacy, kept for backward compat)

```
ExchangeApp (operator)
        │
        │ RecordTrade (or direct CreateCommand)
        ▼
TradeOrder + MiningReward         ── on-chain record + claimable reward
        │
        │ ClaimReward (controller: trader)
        ▼
TokenHolding (issuer = operator, owner = trader)
        │
        │ ProposeTransfer
        ▼
TokenHolding (sender remainder) + PendingTokenHolding
                                          │
                                          │ Accept (controller: receiver)
                                          ▼
                                  TokenHolding (issuer, receiver)
```

`PendingTokenHolding` is the receiver-side inbox pattern: it is signed only by the issuer so the sender can lock funds without the receiver pre-authorising. Reject flips the direction so the original sender can Accept it back.

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

- **`ExchangeApp.RecordTrade` (v1) is still nonconsuming.** The reference integration script creates `TradeOrder` and `MiningReward` directly via `CreateCommand` rather than exercising `RecordTrade`. As a side effect the v1 supply-cap check is not enforced at runtime. The v2 perpetuals stack does not depend on this code path.
- **FeaturedAppActivityMarker is a local template, not the Splice interface.** The shape mirrors `splice-amulet`'s V2 marker; once the `splice-featured-app` DAR is added as a `data-dependency` in `exchange-app/daml.yaml`, the template can implement the interface without changing call-sites.
- **No DAML script tests yet.** Independent third-party audit of both the Daml package and the Rust backend is a hard pre-mainnet milestone (target: late May 2026).

## License

[MIT](./LICENSE)
