# Betting Skills

How to place bets on the on-chain prediction protocol that runs both
**Candle Rush** and **Price Arena**. Protocol-level reference — not tied
to any particular bot or language. Should be enough to implement a
client in any language that can sign EVM transactions (Go, Rust, TS,
Python, Java, …).

## What you can bet on

| Product | Question being answered | Resolution window |
|---|---|---|
| **Candle Rush** | Will the next candle of asset X close GREEN (close > open) or RED (close < open)? | One candle (5m, 15m, or 30m) |
| **Price Arena — UP/DOWN** | Will asset X close above or below its entry price after N minutes? | A configurable period (e.g. 5m, 15m, 30m, 60m) |
| **Price Arena — RELATIVE** | Within a basket of N assets, which one will have the highest (or lowest) % move after N minutes? | A configurable period |

All three products are entry points on the same Diamond contract. USDC
is the only payment token.

## Network and addresses

- **Chain:** Monad (EVM), **chain ID `143`**
- **RPC:** `https://rpc.monad.xyz` (any Monad RPC works; use the chain ID
  above when signing — txs signed for the wrong chain ID are rejected)
- **Diamond contract:** `0x928dc8afe312df45576b15b08c086c5427fd8207`
- **USDC (payment token, 6 decimals):** `0x754704Bc059F8C67012fEd69BC8A327a5aafb603`

Asset price feeds come from **Pyth** via the Hermes HTTP/SSE gateway
(public default: `https://hermes.pyth.network`).

Asset contract addresses (the value you pass as `Asset` /
`PredictionPairBase` / inside `Assets[]`):

| Symbol | Address | Pyth price ID (hex, no 0x prefix) |
|---|---|---|
| BTC | `0x0555E30da8f98308EdB960aa94C0Db47230d2B9c` | `e62df6c8b4a85fe1a67db44dc12de5db330f7ac66b72dc658afedf0f4a415b43` |
| ETH | `0xEE8c0E9f1BFFb4Eb878d8f15f368A02a35481242` | `ff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace` |
| SOL | `0xea17E5a9efEBf1477dB45082d67010E2245217f1` | `ef0d8b6fda2ceba41da15d4095d1da392a0d2f8ed0c6c7bc0f4cfac8c280b56d` |

Confirm any additional assets (commodities, equities) against the
protocol's own asset registry — feed exponents differ (see §"Pyth →
1e8 conversion" below).

---

# Step 0 — Pre-bet preparation (do once per wallet)

Before any of the three product flows works, the betting wallet must:

### 1. Hold USDC

The wallet's USDC balance must be ≥ the bet's `AmountIn` (in base
units; USDC has 6 decimals, so 5 USDC = `5_000_000`).

### 2. Have approved the Diamond contract to spend USDC

```
allowance = USDC.allowance(wallet, DIAMOND)
if allowance < AmountIn:
    USDC.approve(DIAMOND, MAX_UINT256)
    wait for tx receipt, ensure status == 1
```

Approving `MAX_UINT256` once is the common pattern — it avoids per-bet
approval txs.

### 3. Hold enough native gas

Every bet is one ERC-20 spend + one Diamond call. Recommended gas
limits (well above empirical maxes):

- **Candle Rush bet:** `800_000`
- **Price Arena bet (either flavor):** `1_000_000`

---

# Product 1 — Candle Rush

## Function

```
diamond.placeBet(input)
```

## Input struct (ABI shape)

| Field | Type | Description |
|---|---|---|
| `recipient` | `address` | Wallet that receives the winnings. Usually the signer. |
| `asset` | `address` | The asset contract address (BTC/ETH/SOL, see table above). |
| `intervalSeconds` | `uint32` | `300` (5m), `900` (15m), or `1800` (30m). |
| `openTime` | `uint64` | Unix seconds of the candle's **open**. See "openTime math" below. |
| `side` | `uint8` | `0` = GREEN/UP (close > open). `1` = RED/DOWN (close < open). |
| `tokenIn` | `address` | USDC address. |
| `amountIn` | `uint96` | Bet size in USDC base units (6 decimals). |
| `broker` | `uint24` | Broker ID assigned by the protocol. `1` is the default broker. |

> **Exact types matter.** `amountIn` is `uint96` and `broker` is `uint24` —
> not `uint256`. The function selector is `keccak256` of the full type
> signature, so a client generated from the wrong types computes a
> different selector and every call reverts. Encode these as the widths
> shown.

## openTime math

A valid `openTime` is the start of a **future** candle bucket:

```
now = current Unix time (UTC, seconds)
currentBucket = floor(now / intervalSeconds)
nextOpen      = (currentBucket + 1) * intervalSeconds
```

Guardrails:

- If `nextOpen - now < 10` seconds, skip ahead by one interval (your tx
  probably won't land before the candle opens otherwise).
- To bet on multiple consecutive candles, advance: `nextOpen +=
  intervalSeconds` for each.
- Never re-bet on an `openTime` that's already passed — the contract
  rejects it.

## Minimal flow (pseudo-code, any language)

```
ensureUsdcAllowance(wallet, DIAMOND, betAmountUsdcBaseUnits)

openTime = nextCandleOpen(intervalSeconds = 300)        // 5m candle

tx = diamond.placeBet({
    recipient:       wallet.address,
    asset:           BTC_ADDRESS,
    intervalSeconds: 300,
    openTime:        openTime,
    side:            0,                                  // GREEN
    tokenIn:         USDC_ADDRESS,
    amountIn:        5_000_000,                          // 5 USDC
    broker:          1,
}, gasLimit = 800_000)

wait(tx); require(receipt.status == 1)
```

## Hedged-pair pattern

The recommended bot pattern is to bet **both sides** of the same candle
from two **different wallets**, so the bot has on-chain coverage no
matter which way the candle closes:

- Wallet A places `side = 0` (GREEN) on `(asset, intervalSeconds, openTime)`.
- Wallet B places `side = 1` (RED) on the same `(asset, intervalSeconds, openTime)`.

Two separate `placeBet` transactions. Two separate signers. Distinct
recipients (so the protocol sees two independent positions).

The protocol doesn't enforce this — it's a betting strategy, not a
requirement.

---

# Product 2 — Price Arena, UP/DOWN

## Function

```
diamond.predictAndBet(params)
```

## Input struct (ABI shape)

| Field | Type | Description |
|---|---|---|
| `recipient` | `address` | Wallet receiving the winnings. |
| `predictionPairBase` | `address` | The asset to bet on. |
| `isUp` | `bool` | `true` = closing price will be **above** entry. `false` = below. |
| `period` | `uint8` | Period code (see "Periods" below). |
| `tokenIn` | `address` | USDC address. |
| `amountIn` | `uint96` | Bet size in USDC base units. |
| `price` | `uint64` | **Acceptable** entry price in 1e8 fixed-point — see "Slippage math" below. |
| `broker` | `uint24` | Broker ID (e.g. `1`). |

## Periods

`period` is a small enum the contract uses to map to a real duration.
Discover the active set by reading the protocol's market metadata
(commonly served via a GraphQL endpoint that returns each active
market's `periods` list — each period has a numeric code and a
human-readable name like `"5m"`, `"15m"`, `"30m"`, `"60m"`).

Cast the numeric code to `uint8` before sending.

## Slippage math (the `price` field)

Pyth gives you a current price as `(rawPrice, expo)`. The contract
expects a `uint64` in 1e8 fixed-point — i.e. dollars × 10^8.

**Step 1 — convert Pyth to 1e8** (see §"Pyth → 1e8 conversion").

**Step 2 — apply tolerance.** Let `B` be tolerance in basis points
(e.g. `100` = 1.00%, `500` = 5.00%):

```
if isUp:
    acceptable = price1e8 * (10_000 + B) / 10_000
else:
    acceptable = price1e8 * (10_000 - B) / 10_000
```

UP bets bump the cap **up** (so the contract accepts you even if the
real entry price drifted a bit higher between your read and the tx
landing). DOWN bumps it down. The tx reverts if the on-chain oracle
price at execution exceeds `acceptable` for an UP bet (or falls below
for DOWN).

Reasonable defaults:

| Asset class | Tolerance bps |
|---|---|
| Crypto (BTC/ETH/SOL) | `100` (1%) |
| Commodities (GOLD, SILVER, WTI) | `300–500` |
| Equities | `500–1000` (during market hours) |

Coarser feeds (commodities, equities) update less often than crypto, so
the price you read can be stale by the time the tx lands — wider
tolerance prevents preventable refunds.

---

# Product 3 — Price Arena, RELATIVE

## Function

```
diamond.predictRelativeAndBet(params)
```

## Input struct (ABI shape)

| Field | Type | Description |
|---|---|---|
| `recipient` | `address` | Wallet receiving the winnings. |
| `ruleType` | `uint8` | `0` = bet picked asset will be **highest**, `1` = **lowest**. |
| `period` | `uint8` | Period code. |
| `sideIndex` | `uint8` | Index into `assets[]` of the asset you're betting on. |
| `tokenIn` | `address` | USDC address. |
| `amountIn` | `uint96` | Bet size in USDC base units. |
| `broker` | `uint24` | Broker ID. |
| `assets` | `address[]` | Every asset in the basket, **in market order**. Do not reorder. |
| `acceptableEntryPrices` | `uint64[]` | One acceptable entry price per asset in `assets`, same indexing. |

## acceptableEntryPrices math

Same idea as the UP/DOWN `price` field, but per-asset and the direction
depends on `ruleType`.

For each asset `i` in the basket:

- Fetch current Pyth price → convert to 1e8 (`price1e8[i]`).
- Apply tolerance:
  - **`ruleType = 0`** (betting "highest"): for the asset you picked
    (`i == sideIndex`), bump price **up**; for every other asset, bump
    **down**.
  - **`ruleType = 1`** (betting "lowest"): for the asset you picked,
    bump **down**; for every other, bump **up**.

The intuition: each asset has a "favorable" direction for your
prediction, and tolerance is applied so the contract accepts entry as
long as oracle prices move ≤ B bps **against** you between read and
execution.

If you don't want to think per-asset, a permissive simplification is:
**use the same `acceptable` for all assets, computed in the direction
that favors your prediction for the picked asset.** This costs you a
slightly worse entry on average but avoids edge cases when the basket
has feeds with very different update cadences.

## Tolerance across the basket

When assets in the basket have different feed quality, take the **max
of the per-asset tolerances** (most permissive). Otherwise the laggiest
feed in the basket dominates and you'll get unnecessary refunds.

---

# Pyth → 1e8 conversion

Pyth encodes a price as `(rawPrice, expo)` where the real value is
`rawPrice × 10^expo`. The contract wants `realValue × 10^8`. Therefore:

```
expoDiff = expo - (-8) = expo + 8

if expoDiff > 0:
    price1e8 = rawPrice * 10^expoDiff
elif expoDiff < 0:
    price1e8 = rawPrice / 10^(-expoDiff)   // integer truncation
else:
    price1e8 = rawPrice
```

Worked examples:

| Feed | rawPrice | expo | expoDiff | price1e8 | Real value |
|---|---:|---:|---:|---:|---|
| BTC | `4_678_323_540_000` | -8 | 0 | `4_678_323_540_000` | $46,783.24 |
| WTI crude | `9_027_110` | -5 | +3 | `9_027_110_000` | $90.27 |
| GOLD | `4_478_361` | -3 | +5 | `447_836_100_000` | $4,478.36 |

Always check feed exponent at runtime — Pyth can rotate exponents.
Never hard-code `expo = -8`; it's only correct for the major crypto
feeds.

**Defensive check:** if `rawPrice < 0`, refuse the bet — a negative
oracle price means something is wrong upstream and the contract math
will overflow or revert.

# Fetching fresh prices from Pyth

Two options, both via Hermes (`https://hermes.pyth.network` by
default):

**HTTP poll** (simple, good enough for low-frequency bots):

```
GET /v2/updates/price/latest?ids[]=<priceId1>&ids[]=<priceId2>...&encoding=hex&parsed=true&ignore_invalid_price_ids=true
```

Returns a `parsed[]` array; each entry has `id` and a `price` object
`{ price, expo, publish_time, conf }`. Compare `publish_time` to your
local clock — drop and retry if age > `maxPriceAge` (good default:
30s). `ignore_invalid_price_ids=true` keeps one bad ID from failing the
whole batch.

> Older docs reference `/api/latest_price_feeds` — that's the legacy v1
> path. Use the `/v2/updates/price/latest` form above.

**SSE stream** (lower-latency, recommended for bots placing many
bets/second):

```
GET /v2/updates/price/stream?ids[]=<priceId1>&ids[]=<priceId2>...&encoding=hex&parsed=true
```

Maintain a local cache keyed by price ID; each event replaces the
cached entry. Read from cache at bet time, validate `publish_time`
freshness before signing.

Always re-fetch or re-read **right before signing** the bet. A price
that was fresh when you picked the market can be stale by the time
you've built the tx.

---

# Market metadata

To know which markets are currently active (and what periods they
support, which assets are in each RELATIVE basket, what `ruleType`
they're configured for, etc.) you need the protocol's market registry.
This is typically exposed as a GraphQL endpoint that returns two
collections — `upDown` and `relative` markets — each with:

- `base` (the asset address, for UP/DOWN) or `assets[]` (for RELATIVE)
- `periods[]` — list of supported period codes for that market
- `isActive` flag
- `ruleType` (RELATIVE only)
- the Pyth `priceId`s your bot will need

You can also bypass discovery and hard-code known active markets — the
contract calls only need the addresses, period code, and price IDs.
The registry is just a convenience for figuring out what's tradeable
right now.

---

# Putting it together — full bet pseudo-code

```
// One-time, per wallet:
ensureUsdcAllowance(wallet, DIAMOND, MAX_UINT256)
ensureNativeGas(wallet, minimumGasReserve)

// Per bet:

// 1. Pick a market (from your discovery layer, or hard-coded).
market = pickMarket()       // returns {kind, asset(s), period, priceIds, ...}

// 2. Read fresh prices for every priceId.
priceData = pyth.fetchFresh(market.priceIds, maxAge = 30s)
for each id, p in priceData:
    require(now - p.publishTime <= 30s)
    require(p.rawPrice >= 0)
    price1e8[id] = convertPythTo1e8(p.rawPrice, p.expo)

// 3. Compute acceptable price(s) using tolerance bps.
if market.kind == "CANDLE_RUSH":
    openTime = nextCandleOpen(market.intervalSeconds)
    input = {
        recipient:        wallet.address,
        asset:            market.asset,
        intervalSeconds:  market.intervalSeconds,
        openTime:         openTime,
        side:             pickSide(),        // 0 GREEN, 1 RED
        tokenIn:          USDC,
        amountIn:         betAmountUsdcBaseUnits,
        broker:           1,
    }
    tx = diamond.placeBet(input, gasLimit = 800_000)

elif market.kind == "UP_DOWN":
    isUp = pickDirection()
    acceptable = applyTolerance(price1e8[market.priceIds[0]], toleranceBps, isUp)
    params = {
        recipient:           wallet.address,
        predictionPairBase:  market.asset,
        isUp:                isUp,
        period:              market.periodCode,
        tokenIn:             USDC,
        amountIn:            betAmountUsdcBaseUnits,
        price:               acceptable,
        broker:              1,
    }
    tx = diamond.predictAndBet(params, gasLimit = 1_000_000)

elif market.kind == "RELATIVE":
    sideIndex = pickAssetInBasket(market.assets)
    acceptablePrices = [
        applyToleranceRelative(price1e8[pid], toleranceBps, market.ruleType, i == sideIndex)
        for i, pid in enumerate(market.priceIds)
    ]
    params = {
        recipient:              wallet.address,
        ruleType:               market.ruleType,
        period:                 market.periodCode,
        sideIndex:              sideIndex,
        tokenIn:                USDC,
        amountIn:               betAmountUsdcBaseUnits,
        broker:                 1,
        assets:                 market.assets,
        acceptableEntryPrices:  acceptablePrices,
    }
    tx = diamond.predictRelativeAndBet(params, gasLimit = 1_000_000)

// 4. Wait for the receipt and check status.
receipt = wait(tx, timeout = 60s)
require(receipt.status == 1)
```

---

# Failure modes to expect

| Symptom | Likely cause | Fix |
|---|---|---|
| `placeBet` reverts immediately | `openTime` has already passed, or is more than one interval in the future | Recompute `openTime` from current Unix time, just before signing. |
| Bet refunded after submission | Real entry price drifted outside `acceptablePrice` between read and execution | Widen `toleranceBps` (especially for non-crypto feeds), or re-read price closer to tx submission. |
| `predictRelativeAndBet` reverts | `assets[]` order or length doesn't match the on-chain market definition | Re-fetch market metadata; never reorder `assets` between read and call. |
| ERC-20 spend failure | Missed approval, or allowance < `amountIn` | Run the allowance check before every bet (or use `MAX_UINT256` once). |
| Tx underpriced / stuck | Gas limit too low for the contract path | Use `800_000` for Candle Rush, `1_000_000` for Price Arena. Re-broadcast with a bumped fee if needed. |
| `convertPythTo1e8` produces a garbage number | `expo` was assumed to be `-8` but the feed actually publishes a coarser exponent | Always read `expo` at runtime and apply the `expoDiff` formula. |
| Period code rejected | Period isn't enabled for this market | Read the market's `periods[]` list and use one of those codes. |
