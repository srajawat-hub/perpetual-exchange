# Perpetual Exchange Protocol

Smart contracts for a decentralised perpetual futures exchange: off-chain order signing with on-chain matching and settlement, isolated-margin position accounting, funding-rate accrual, and a staked relayer network.

Solidity `0.8.18`, Hardhat, OpenZeppelin upgradeable contracts.

---

## Overview

Traders sign orders off-chain (EIP-712). Relayers pair compatible orders and submit them on-chain, where the contracts verify both signatures, match the assets, move collateral, and update each party's position. Positions are margined against a vault balance, marked against an oracle, and accrue funding over time.

The design keeps order *distribution* off-chain (cheap, fast) while keeping *settlement and custody* on-chain (trustless). A trader never gives up custody to a relayer — a relayer can only submit an order the trader has already signed.

---

## Contract layout

`contracts/` — 100 Solidity files, 7,729 lines.

### `matching-engine/`
| Contract | Lines | Role |
|---|---:|---|
| `MatchingEngineCore.sol` | 241 | Order pairing, signature validation, fill bookkeeping |
| `MatchingEngine.sol` | 21 | Entry point / initialiser over the core |
| `AssetMatcher.sol` | 36 | Checks two orders reference compatible assets |
| `TransferManager.sol` | 49 | Computes who pays what, including fees |
| `TransferExecutor.sol` | 36 | Performs the resulting transfers |

Matching is deterministic and settles atomically: both sides of a pair execute in one transaction or the transaction reverts.

### `orderbook/`
| Contract | Lines | Role |
|---|---:|---|
| `Positioning.sol` | 703 | Position open/close, PnL realisation, liquidation |
| `AccountBalance.sol` | 510 | Per-account position and funding state |
| `VaultController.sol` | 217 | Routes deposits/withdrawals across vaults |
| `Vault.sol` | 206 | Collateral custody |
| `PositioningConfig.sol` | 200 | Margin ratios and protocol parameters |
| `OrderValidator.sol` | 50 | EIP-712 order hashing and signature recovery |

`Positioning.sol` is the core of the protocol — it holds the liquidation path and the accounting that everything else feeds.

### `oracles/`
`PerpetualOracle.sol` (485 lines) maintains mark and index prices used for margin checks, liquidation triggers, and funding.

### `funding-rate/`
`FundingRate.sol` accrues the periodic payment between longs and shorts based on the mark/index spread.

### `relayer-staking/`
`Staking.sol` and `Slashing.sol` — relayers post a stake that can be slashed, making order-submission misbehaviour costly.

### Supporting
`periphery/PerpPeriphery.sol` (282 lines) as the user-facing entry point; plus `libs/`, `interfaces/`, `storage/`, `transfer-proxies/`, `tokens/`, `view/`, and `mocks/` + `tests/` for test doubles.

---

## Architecture

```
 trader signs order (EIP-712, off-chain)
              │
              ▼
      relayer pairs orders ──── staked; slashable for misbehaviour
              │
              ▼
   MatchingEngineCore ── verifies both signatures
              │          AssetMatcher: assets compatible?
              ▼
     TransferManager ──► TransferExecutor   (collateral + fees move)
              │
              ▼
        Positioning ──► AccountBalance      (positions, PnL, funding)
              │
              ├──► Vault / VaultController  (custody)
              ├──► PerpetualOracle          (mark & index price)
              └──► FundingRate              (periodic accrual)
```

---

## Upgradeability

13 contracts inherit OpenZeppelin upgradeable bases (`Initializable`, `OwnableUpgradeable`, etc.) and deploy behind proxies. Proxy state is tracked in `.openzeppelin/` and `.upgradable/`.

This matters for a perpetuals venue: parameters and logic can be corrected without forcing every open position to migrate.

---

## Tests

514 `it()` cases across the `test/` tree:

```
test/
├── matching-engine/     order matching and fills
├── positioning/         open, close, liquidation, PnL
├── oracles/             mark & index price behaviour
├── relayer/             staking and slashing
├── periphery/           end-to-end user flows
├── libs/  tokens/       library and token unit tests
├── Global.test.ts       cross-cutting integration
└── EIP712.js, sign.js, order.js   signing helpers
```

Run:

```bash
yarn install
yarn hardhat compile
yarn hardhat test
```

---

## Stack

| | |
|---|---|
| Solidity | `0.8.18` (pinned `=0.8.18` in 93 of 100 files) |
| Framework | Hardhat `^2.13.1` |
| Libraries | OpenZeppelin Contracts + Contracts-Upgradeable `^4.6.0` |
| Oracles | Chainlink Contracts `^0.6.1` |
| Testing | Waffle `2.0.1`, ethers `5.7.2`, TypeScript `^4.9.4` |

---

## Documentation tooling

Generate UML and call graphs from the contracts:

```bash
# UML
npx sol2uml ./contracts

# Call graph (per folder or file)
npx surya graph contracts/matching-engine/*.sol | dot -Tpng > MatchingEngine.png

# Function calls and event emits
npx solgraph ./contracts/matching-engine/MatchingEngineCore.sol > MatchingEngine.dot
dot -Tpng MatchingEngine.dot -o MatchingEngine.png

# Natspec docs
npx hardhat docgen
```

Rendered diagrams live in `uml/`. Revert reason codes are listed in `errorCodes.md`.

---

## Status

Working protocol implementation with a substantial test suite. It has **not** been through a third-party security audit, and is not deployed to mainnet — treat it as unaudited code.

## License

GPL-3.0 (see `LICENSE`).
