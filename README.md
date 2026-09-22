# Morphex

Confidential constant-product AMM built with [Zama fhEVM](https://docs.zama.ai/fhevm) and [OpenZeppelin ERC-7984](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/token/ERC7984).

Every protocol amount is encrypted end-to-end: token balances, total supply, mint quantities, swap input/output, pool reserves, LP shares, liquidity deposits/withdrawals, and execution receipts. **No amount is ever decrypted on-chain.**


## What's built

| Contract | Purpose | Lines |
|---|---|---:|
| `MorphexToken` | ERC-7984 confidential token with relayer mint/burn bridge | 127 |
| `ConfidentialPair` | Encrypted constant-product AMM — reserves, LP accounting, swap validation, refund receipts | 292 |
| `ConfidentialPairFactory` | Canonical pair creation (one pair per token combination) | 30 |
| `RelayerVault` | Public ERC-20 custody with withdrawal requests, batch payouts, and time-locked escape hatch | 210 |
| `MockERC20` | Test token for local development | 21 |

**5 contracts · 680 lines of Solidity · 3 test suites · 21 tests passing**

## Test results

All 21 tests pass in fhEVM mock mode (`npm test`):

```
MorphexToken (MORPH)
  Deployment ..................................... 3 passing
  Mint ........................................... 4 passing
  confidentialTransfer ........................... 2 passing
  Operator + confidentialTransferFrom ............ 2 passing
  Sequential operations .......................... 1 passing

ConfidentialPair
  Initial liquidity, swap, LP mint/burn, refund .. 4 passing

RelayerVault + confidential bridge
  Deposit→mint→burn→payout, escape hatch,
  double-spend prevention, access control ........ 5 passing

21 passing
```

## Privacy boundary

FHE encrypts values, not Ethereum itself. These remain **public**: wallet and contract addresses, pair identity, transaction timing/order, gas usage, and the fact that a liquidity or swap function was called. Events deliberately contain no amounts.

## Architecture

```
wallet (encrypts amount + target) → ERC-7984 token → ConfidentialPair
                                                    → FHE invariant checks
                                                    → encrypted receipt for wallet
```

**Relayer bridge** (for wrapping public ERC-20 ↔ confidential tokens):

```
Public ERC-20 → RelayerVault deposit → relayerMint → cToken (encrypted)
cToken → relayerBurnRequest → RelayerVault → batchWithdraw → public ERC-20
                             └→ escape hatch after delay (trustless fallback)
```

## AMM design detail

The installed fhEVM release supports encrypted multiplication and comparisons but not encrypted÷encrypted division or square root at usable circuit depth. Morphex therefore accepts an **encrypted output target** for swaps and an **encrypted LP-share target** for liquidity. The pair validates those values against the encrypted invariant; a stale or excessive target produces a confidential no-op with refund.

This keeps every value private and avoids trusting an on-chain oracle. The frontend can calculate targets for its own known liquidity, or obtain them from an optional quote service authorized to view a pool. A quote service can never bypass the pair's FHE checks.

## User flow

1. Encrypt every `uint64` input against the pair/token address using the Zama SDK.
2. Call `setOperator(pair, expiry)` on each ERC-7984 token the pair may pull.
3. Add liquidity with encrypted `amount0`, `amount1`, and `shareTarget`.
4. Swap with encrypted `amountIn` and `amountOutTarget`.
5. Read `lastSwapOf` / `lastLiquidityOf`, then decrypt only the caller-authorized receipt locally.

Invalid private checks do not expose a revert reason: the pair refunds the encrypted input and stores an encrypted `success = false` receipt.

## Development

```bash
npm install
npm run compile
npm test           # 21 tests, fhEVM mock mode
npm run typecheck
```

### Local demo with frontend

```bash
npx hardhat node
npm run deploy:relayer-local
cd frontend && npm run dev
```

The local deployment uses mock public tokens. To give a wallet test balances:

```bash
LOCAL_USER=0xYourWalletAddress npm run faucet:local
```

## Security properties

- Uses ERC-7984 operator authorization; no custom unrestricted handle transfer is exposed.
- All mutable encrypted values receive ACL access for their owner and the contract that must process them.
- Fee-adjusted swap validation uses ciphertext-only arithmetic (`(reserveIn × BPS + amountIn × (BPS − fee)) × (reserveOut − amountOut) ≥ reserveIn × reserveOut × BPS`).
- Pair reserves are capped at `1e15` base units so fee-adjusted products fit the available encrypted 128-bit multiplication range.
- Reentrancy is blocked at pair entry points.
- RelayerVault withdrawal requests are bound to `(recipient, token, amount)` — the relayer cannot redirect funds.
- Time-locked escape hatch ensures users can always withdraw even if the relayer goes offline.

> This is a working protocol baseline, not an audited production deployment. A production quote service, integration tests on a public testnet, and an independent audit are required before mainnet use.

## License

[BSD-3-Clause-Clear](./LICENSE) — matches the SPDX headers in all Solidity source files.
