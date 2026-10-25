# LP share pricing and yield accounting

> **Testnet only.** ClearLedger runs on Stellar Testnet for evaluation. The
> contracts are unaudited, Testnet USDC has no promised redemption value, and
> nothing here is a Mainnet-readiness statement or a promise of financing or
> returns. See [production readiness gates](../PRODUCTION_READINESS.md).

This page explains how the LP portal ([`apps/web/app/lp/page.tsx`](../../apps/web/app/lp/page.tsx))
and the shared pool clients present LP shares, available liquidity, and yield,
and where the accounting is approximate. It complements the higher-level
[Liquidity Pool](../protocol/liquidity-pool.md) and
[Understanding Yield](understanding-yield.md) pages.

## Where the logic lives

The pool contract source is **not** in this repository. It lives in
[`Stellar-Clear/clearledger-contracts`](https://github.com/Stellar-Clear/clearledger-contracts)
(see its
[`docs/LIMITATIONS.md`](https://github.com/Stellar-Clear/clearledger-contracts/blob/main/docs/LIMITATIONS.md),
section "Integer Arithmetic and Rounding" and "Known Gaps"). The formulas below
are taken from that document; this checkout only calls and displays the
results.

| Concern | File in this repo |
| --- | --- |
| LP portal UI (stats, position, deposit/withdraw forms, estimates) | `apps/web/app/lp/page.tsx` |
| Liquidity-state explanation (`available` / `fully-committed` / `empty`) | `apps/web/lib/poolLiquidity.ts` (`explainPoolLiquidity`) |
| Web mutations and queries (`deposit`, `withdraw`, `poolStats`, `lpPosition`) | `apps/web/hooks/usePool.ts` |
| API payload parsing (`parseRawPoolStats`, `parseRawLPPosition`) | `apps/web/lib/api.ts` |
| Contract client (`deposit`, `withdraw`, `getStats`, `getLPPosition`, `getUtilizationRate`) | `packages/sdk/src/clients/pool.ts` |
| React wrappers over the SDK client | `packages/sdk-react/src/usePool.ts` |
| `GET /pool/stats`, `GET /pool/position/{address}` | `indexer/api/handlers_stats.go` (`HandleGetLPPosition`) |

## Share pricing

LP ownership is a number of shares, not a fixed USDC balance. Amounts are
integer USDC stroops (1 USDC = 10,000,000 stroops, `STROOPS_PER_UNIT` in the LP
page).

- **Mint (deposit).** First deposit into an empty pool mints 1:1. After that,
  `shares = amount * total_shares / total_deposits`, rounded **down**. A
  result of zero shares is rejected (`MinimumDeposit`).
- **Burn (withdraw).** Redemption value and position value are
  `shares * total_deposits / total_shares`, rounded **down**. A zero-valued
  withdrawal is rejected. Read-only position values can omit a fractional
  stroop.
- **Precision.** Everything is integer arithmetic; callers should expect at
  most a whole-stroop difference from exact arithmetic.
- **Share price** is not a field of `/pool/stats`; it is approximately
  `total_deposits / total_shares` ([ANALYTICS_METRICS.md](../ANALYTICS_METRICS.md#pool-metrics-get-poolstats-and-poolsnapshots)).

What the portal does with this today:

- `apps/web/app/lp/page.tsx` simulates `deposit` through
  `PoolClient.simulateTransaction` (400 ms debounce) and shows the contract's
  returned share count as "Expected LP shares from contract simulation". It is
  a simulation result, not a guarantee; the price can move before submission.
- The page can render a "Share Price" tile from `stats.sharePrice`, but neither
  `parseRawPoolStats` (`apps/web/lib/api.ts`) nor `/pool/stats` currently
  supplies that field, so the tile is **not shown** with the current code.
- The withdrawal estimate is computed client-side as
  `position.usdcValue * enteredShares / position.shares`, capped at available
  liquidity, from the latest indexed position and pool data. The confirmed
  amount may differ.
- Inputs are converted with `Math.floor(Number(input) * 10_000_000)`. This uses
  JavaScript floating point, so very long or large decimal inputs may not map
  to the exact stroop amount typed.
- Share balances are formatted by `formatShareValue`, which uses the same
  7-decimal format and the "USDC" suffix as USDC amounts. Shares are a separate
  unit; the suffix is a display shortcut.

## Fees

- **Protocol fee.** On repayment, the contract splits yield into an LP part and
  a protocol cut: `protocol_cut = yield * fee_bps / 10_000`, rounded down; the
  rounding remainder is credited to LPs. The indexer reads the repayment event
  as `(amount, lp_yield, protocol_cut)` (`indexer/listener/handlers.go`,
  `repayment_received`). `total_yield_distributed` counts LP yield only, after
  the protocol cut ([ANALYTICS_METRICS.md](../ANALYTICS_METRICS.md)). The
  `fee_bps` setting is a contract setting and is not exposed by the web app or
  `/pool/stats`.
- **No deposit/withdrawal fee in the web code.** The LP page, `usePool`, and
  `PoolClient` add no fee to deposits or withdrawals. The
  [Liquidity Pool](../protocol/liquidity-pool.md#withdrawal) page describes no
  lock-up period or early-withdrawal penalty; that is contract behavior this
  repo does not implement.
- **Network fees.** Each transaction also costs Soroban fees paid in XLM; the
  LP page's Testnet notice points to Friendbot for funding the account.

## Available liquidity

`available_liquidity = total_deposits - total_funded`
([ANALYTICS_METRICS.md](../ANALYTICS_METRICS.md)), where `total_funded` is
principal deployed in funded, unsettled invoices. The contract panics a
`withdraw` that exceeds available liquidity.

In the portal:

- `explainPoolLiquidity` (`apps/web/lib/poolLiquidity.ts`) clamps negatives to
  zero and returns `available` when `availableLiquidity > 0`,
  `fully-committed` when deposits exist but nothing is available, and `empty`
  otherwise.
- The Withdraw button is disabled when the requested shares exceed
  `position.shares`, when the estimated value exceeds available liquidity, or
  while the withdraw simulation is pending/failed.
- A status message ("The invoice index and pool totals do not match") appears when the indexed invoice snapshot (statuses `Funded`,
  `Active`, `Confirmed`, first 100 invoices) does not match the pool's
  `activeInvoiceCount` and `totalFunded`. Stats refetch every 45 seconds
  (`apps/web/hooks/usePool.ts`), so values can lag the chain.
- `utilization_rate_bps = floor(total_funded * 10_000 / total_deposits)`; new
  funding is capped by `max_utilization_bps` (8500 in the sample response in
  [QUICKSTART.md](../QUICKSTART.md)).

## Current limitation: yield uses aggregate principal per LP

The pool stores **one cumulative principal amount per LP**, not per-deposit
lots. On a partial withdrawal it allocates that principal pro rata across the
LP's remaining shares (an average-basis approximation, not FIFO or HIFO).

What is and is not tracked per LP:

| Tracked | Not tracked |
| --- | --- |
| Share balance (`shares`) | Share price or USDC amount of each individual deposit |
| Cumulative principal (aggregate) | Deposit lots / cost basis per deposit |
| Running `yield_earned`, written on `withdraw` | Lot-level (FIFO/HIFO) yield attribution |
| `deposit_count` | A per-deposit breakdown of the position |

Consequences:

- **Total USDC returned and share ownership are unaffected.** Redemption uses
  the share formula above.
- **Reported `yield_earned` depends on the accounting method.** After deposits
  at different share prices, it can differ from a FIFO calculation for a given
  withdrawal. Example from the contracts repo: deposit 1,000 USDC, pool gains
  2%, deposit another 1,000 USDC, withdraw 500 shares for about 510 USDC. FIFO
  would record 10 USDC yield; aggregate-basis records about 5.05 USDC.
- **Do not use `yield_earned` for tax or lot-level reporting.** The portal's
  "Accrued Yield" (`position.yieldEarned`, from `/pool/position/{address}`
  `yield_earned`) is this value. A per-deposit lot ledger would require
  contract changes.
- **Unrealized gains are in share value.** `usdc_value` reflects the current
  share price; yield is realized when shares are redeemed.

## Defaults and risk

Yield is not guaranteed and principal can be lost. Docs in this repo
([pool-contract.md](../smart-contracts/pool-contract.md),
[economic-model.md](../protocol/economic-model.md)) describe a default as a
principal loss shared across shares, while the contracts repo's
`LIMITATIONS.md` currently describes the pool as recording realised loss
while leaving `TotalDeposits` and share value unchanged. These descriptions
disagree; check the deployed contract version before relying on either.

## Verify against this checkout

```bash
pnpm --filter web test -- hooks/usePool.test.ts lib/poolLiquidity.test.ts
pnpm --filter @clearledger/sdk test
pnpm --filter web test
```
