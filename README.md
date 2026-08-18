# PMM Integration Guide

Complete guide for integrating Mystic's Private Market Maker (PMM) as a liquidity source in the KyberSwap aggregator.


## Overview

The PMM exposes two endpoints:

- **Price Levels**  a cached, two-sided book per chain, polled offline to compute expected amount out. Commits us to nothing.
- **Firm Quote** signs an executable order against real inventory and returns ready-to-execute calldata.

Settlement runs through the **1inch Limit Order Protocol v4**, so no custom verifier contract is needed. The PMM signs an RFQ order; KyberSwap's executor fills it.

| Property | Value |
|----------|-------|
| Base URL | `https://pmm-api.mysticfinance.xyz` |
| Settlement contract | `0x111111125421cA6dc452d289314280a0f8842A65` (1inch LOP v4) |
| Quote validity | 120 seconds |
| Price transport | HTTP polling (WebSocket planned) |
| Inventory | Held in a Safe; orders signed via EIP-1271 |

---

## Integration Flow

```
┌────────────────────────────────┐
│ GET /kyber/v1/prices?chainId=1 │  ← polled every 1-2s, cached by Kyber
└───────────────┬────────────────┘
                │
       ┌────────▼─────────┐
       │ Compute amount   │  ← offline, no call to PMM
       │ out from levels  │
       └────────┬─────────┘
                │
         user confirms swap
                │
┌───────────────▼────────────────┐
│ POST /kyber/v1/firm?chainId=1  │  ← once per swap, PMM signs here
└───────────────┬────────────────┘
                │
       ┌────────▼──────────┐
       │ code === 0 ?      │
       └───┬───────────┬───┘
           │           │
          yes          no ──► route around PMM (see Error Codes)
           │
    ┌──────▼──────────────────────┐
    │ Response gives:             │
    │  • router (call target)     │
    │  • calldata                 │
    │  • amountInOffset           │
    │  • expiry                   │
    └──────┬──────────────────────┘
           │
    ┌──────▼──────────────────────┐
    │ Executor calls `router`     │
    │ with calldata before expiry │
    └─────────────────────────────┘
```

---

## API Reference

All responses use the standard envelope. **`code: 0` means success**; any non-zero `code` is a  refusal returned with HTTP 200 (not an outage).

```typescript
{
  code: number;      // 0 = OK
  message: string;
  data?: T;
}
```

### 1. Get Price Levels

**Endpoint:** `GET /kyber/v1/prices?chainId=<id>`

**Response:**
```typescript
{
  code: 0,
  message: "OK",
  data: {
    prices: {
      // key format: "0x<baseToken>/0x<quoteToken>" (lowercase)
      "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48": {
        last_updated: number;      // unix seconds, fractional
        bids: Array<[number, number]>;  // PMM BUYS base  — [price, size]
        asks: Array<[number, number]>;  // PMM SELLS base — [price, size]
      }
    }
  }
}
```

**Level semantics:**

- `price` is **quote per base**. `size` is denominated in the **base token**.
- Levels are **exclusive** total tradable depth is the cumulative sum across levels.
- The **first level's size is the minimum trade size**, published as a split level at the same price (e.g. `[[1893.10, 0.005], [1893.10, 16.87], ...]` means a 0.005 minimum).
- A pair we cannot currently price, stale reference price, or no inventory is **omitted from the response entirely**. It is never published with a zero price or an empty ladder. Absence means "route around us".

**Example:**
```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "prices": {
      "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48": {
        "last_updated": 1787048272.153,
        "bids": [[1893.1003, 0.005], [1893.1003, 16.8731], [1891.9627, 16.8781]],
        "asks": []
      }
    }
  }
}
```

> **Note on one-sided books.** Mystic's inventory is stablecoin-denominated (USDC/AUSD), so the **bid** side carries depth and `asks` may legitimately be empty until fills leave us holding the base token.

### 2. Request Firm Quote

**Endpoint:** `POST /kyber/v1/firm?chainId=<id>`

> ### ⚠️ Prerequisites: this endpoint will reject you until both are done
>
> Unlike `/prices`, this endpoint hands out signed orders against real inventory, so it is closed by default. Before your first call, send us:
>
> **1. Your egress IP addresses / CIDR ranges.**
> We allowlist them. Any other source gets `HTTP 403` with no response body. Send updates whenever they change, a rotated IP looks identical to an attack from our side.
>
> **2. Your executor addresses, per chain.**
> These are the values you will send as `rfq_sender`. We whitelist them, and every order we sign pins `allowedSender` to that address on-chain. Any `rfq_sender` outside the list is refused with `1009`, regardless of your IP. Currently we whitelisted `0x0F4A1D7FdF4890bE35e71f3E0Bbc4a0EC377eca3` as the `rfq_sender` on our supported chains

**Request:**
```typescript
{
  orders: Array<{
    maker_asset: string;        // token the PMM provides
    taker_asset: string;        // token the PMM receives
    taker_amount: string;       // amount of taker_asset, in wei (decimal string)
    alpha_fee_amount?: string;  // fee to withhold, in wei
    alpha_fee_token?: string;   // defaults to maker_asset; may be taker_asset
  }>;
  allow_partial_success?: boolean;  // default false (all-or-nothing)
  request_id: string;
  rfq_sender: string;           // KyberSwap executor — REQUIRED, whitelisted
  recipient?: string;           // where maker asset lands; defaults to rfq_sender
  user_address: string;         // end user, for risk profiling / rate limiting
  partner?: string;             // e.g. "llamaswap" — drives rate-limit exemptions
}
```

**Response:**
```typescript
{
  code: 0,
  message: "OK",
  data: {
    orders: Array<{
      maker_asset: string;
      taker_asset: string;
      taker_amount: string;
      maker_amount: string;      // EXACTLY what settles at this taker_amount
      alpha_fee_amount: string;
      calldata: string;          // call this against `router`
      amountInOffset: number;    // byte offset of amountIn within calldata
    }>;
    expiry: number;              // unix seconds; order is unfillable after this
    router: string;              // transaction target (1inch LOP v4)
    maker: string;               // Mystic's Safe address
    taker: string;               // echoes rfq_sender
  }
}
```

**Example:**
```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "orders": [{
      "maker_asset": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
      "taker_asset": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
      "taker_amount": "1000000000000000000",
      "maker_amount": "1892600264",
      "alpha_fee_amount": "500000",
      "calldata": "0x56a75868...",
      "amountInOffset": 292
    }],
    "expiry": 1787048333,
    "router": "0x111111125421ca6dc452d289314280a0f8842a65",
    "maker": "0x327f3f2d0945e877147cd58db0ab6637623ac174",
    "taker": "0x70997970c51812dc3a010c7d01b50e0d17dc79c8"
  }
}
```

---

## Complete Integration Example

```typescript
const PMM_BASE = 'https://pmm-api.mysticfinance.xyz';

// ── 1. Poll price levels (cache these; do not call per user) ──────────────
async function getPrices(chainId: number) {
  const res = await fetch(`${PMM_BASE}/kyber/v1/prices?chainId=${chainId}`);
  const body = await res.json();
  if (body.code !== 0) throw new Error(`PMM prices: ${body.message}`);
  return body.data.prices;
}

// ── 2. Compute expected output offline (levels are exclusive) ────────────
function amountOutFromBids(bids: [number, number][], baseIn: number) {
  let remaining = baseIn, quoteOut = 0;
  for (const [price, size] of bids) {
    if (remaining <= size) return quoteOut + remaining * price;
    quoteOut += size * price;
    remaining -= size;
  }
  return null;  // exceeds published depth
}

// ── 3. Request a firm quote at swap-confirmation time ────────────────────
async function getFirmQuote(chainId: number, params: {
  makerAsset: string; takerAsset: string; takerAmount: string;
  alphaFeeAmount: string; executor: string; userAddress: string;
}) {
  const res = await fetch(`${PMM_BASE}/kyber/v1/firm?chainId=${chainId}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      orders: [{
        maker_asset: params.makerAsset,
        taker_asset: params.takerAsset,
        taker_amount: params.takerAmount,
        alpha_fee_amount: params.alphaFeeAmount,
      }],
      allow_partial_success: false,
      request_id: crypto.randomUUID(),
      rfq_sender: params.executor,
      recipient: params.executor,
      user_address: params.userAddress,
      partner: 'kyberswap',
    }),
  });

  const body = await res.json();
  if (body.code !== 0) {
    // Business refusal, not an outage — route around the PMM.
    console.warn(`PMM declined (${body.code}): ${body.message}`);
    return null;
  }
  return body.data;
}

// ── 4. Execute ─────────────────────────

async function execute(quote, actualAmountIn?: bigint) {
  const order = quote.orders[0];

  if (Math.floor(Date.now() / 1000) >= quote.expiry) {
    throw new Error('Quote expired — request a new one');
  }

  const calldata = order.calldata;

  // Executor must be `rfq_sender` — the order is locked to it on-chain.
  return executor.call({ to: quote.router, data: calldata });
}
```

---

## Key Implementation Details

### 1. Patching `amountIn`, and reading `amountInOffset`

The integration example above executes the calldata exactly as returned, which is correct for the common case. This section covers when you need to change it, and how.

#### When you need to patch

Fills are **pro rata at a fixed rate**: `makerOut = amountIn × signedMaking / signedTaking`, floor-rounded. The rate you were quoted is the rate you get, whatever amount you fill at.

**Our leg is the whole route, or its first hop → no patching.** The amount you requested is the amount that arrives. Execute the calldata unchanged.

**Downward → always works, costs you nothing.** On a multi-hop route our leg's input is the previous hop's on-chain output, so the exact figure is only known at execution. Write the real amount in and you receive proportionally less at the same rate.

**Upward → supported up to 1%.** We sign each order with 1% headroom on both legs, so a previous-hop estimate that lands slightly high still fills. Beyond that the order is not large enough and the fill revert, request a fresh quote at the larger size.

```
route:  USDC ──[Uniswap]──> WETH ──[Mystic PMM]──> LINK

  request firm quote for 1.000 WETH
    ├─ delivered 0.994 → patch amountIn to 0.994e18 → fills at the quoted rate
    ├─ delivered 1.008 → patch amountIn to 1.008e18 → fills (within 1%)
    └─ delivered 1.050 → exceeds headroom → re-quote
```

To keep corrections small, **request the amount your route simulation expects the previous hop to deliver**, rather than its minimum, you already compute that number when building the route. If your slippage tolerance means corrections regularly exceed 1%, tell us and we will raise the headroom to match.

#### How to patch

`amountInOffset` is a **byte offset into the returned calldata**. Overwrite the 32-byte big-endian word at that position and change nothing else:

```typescript
function patchAmountIn(calldata: string, offset: number, amountIn: bigint): string {
  const bytes = Buffer.from(calldata.slice(2), 'hex');
  const word = Buffer.from(amountIn.toString(16).padStart(64, '0'), 'hex'); // 32 bytes
  word.copy(bytes, offset);
  return '0x' + bytes.toString('hex');
}

// usage
const order = quote.orders[0];
const calldata = actualAmountIn
  ? patchAmountIn(order.calldata, order.amountInOffset, actualAmountIn)
  : order.calldata;
```

**Why editing signed calldata is safe:** the signature covers the **order struct** (assets, amounts, maker traits), not the fill parameters. `amount` is an argument to the fill call, so rewriting it leaves the signature valid. Rewriting any other word will invalidate it and the fill will revert.

#### Reading the offset

Never hardcode it, the value depends on which fill entrypoint the order uses:

| Maker type | Entrypoint | `amountInOffset` |
|------------|-----------|------------------|
| EOA | `fillOrderArgs` | 324 |
| Safe (current) | `fillContractOrderArgs` | 292 |

It is computed per-order and will follow any future change in maker configuration, so read it from the response every time.

The order can be filled **once**, at any amount up to the signed size.

### 2. `rfq_sender` is enforced on-chain

Every order pins `allowedSender` to the `rfq_sender` supplied in the request. Only that address can fill it. `rfq_sender` must also appear in our configured executor whitelist or the request is refused with `1009` — see [What We Need From You](#what-we-need-from-you).

### 3. Alpha fee handling

The fee is **withheld from the taker's receiving amount**, not transferred separately:

```
gross maker amount  = ladder walk at taker_amount
maker_amount        = gross − alpha_fee_amount
```

The difference stays in Mystic's inventory for offline settlement. If `alpha_fee_token` is the taker asset, it is converted to maker-asset terms at that quote's own rate. We record fee token, wei amount, and USD value at signing time for monthly reconciliation.

### 4. Expiry

Orders are valid for **120 seconds** and enforce expiry on-chain. Requesting a quote significantly ahead of broadcast will produce reverts.

---

## API Endpoints Summary

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/kyber/v1/prices?chainId=<id>` | GET | Fetch price levels for a chain |
| `/kyber/v1/firm?chainId=<id>` | POST | Request a signed, executable quote |
| `/health` | GET | Quoting status, per-chain quotable pair count |

---

## What We Need From You

To complete the integration we need:


1. **Egress IP ranges** for the `/firm` allowlist.

---


## Error Codes

All returned as HTTP 200 with a non-zero `code`.

| Code | Meaning | Recommended action |
|------|---------|--------------------|
| `0` | Success | Proceed |
| `1001` | Unsupported chain | Do not route to PMM on this chain |
| `1002` | Unsupported pair | Do not route this pair |
| `1003` | Insufficient liquidity | Reduce size or route elsewhere |
| `1004` | Stale price | Retry shortly; PMM has no fresh reference |
| `1005` | Rate limited | Back off; see rate limits below |
| `1006` | Notional cap exceeded | Reduce size |
| `1007` | Quoting disabled | PMM is paused; skip until prices return |
| `1008` | Invalid alpha fee | Fee ≥ maker amount, or unknown fee token |
| `1009` | Unauthorised `rfq_sender` | Executor not whitelisted — contact us |
| `1500` | Internal error | Retry or route elsewhere |

HTTP-level responses:

| Status | Meaning |
|--------|---------|
| `400` | Malformed request (bad address, non-decimal wei string, missing `chainId`) |
| `403` | Caller IP not in allowlist (`/firm` only) |

---

## Common Issues

### Issue: "Pair missing from /prices response"
**Causes:**
- No inventory or no router allowance on that token → we publish nothing rather than a zero
- Reference price is stale (> 10s old)
- Pair not configured on that chain

**Solution:** Treat absence as "PMM has no liquidity right now" and route around us. It is not an error state.

### Issue: `asks` array is empty
**Cause:** Mystic's inventory is stablecoin-denominated, so we quote the **bid** side (buying the volatile token, paying stable). Asks populate only once fills leave us holding the base token.

**Solution:** This is expected behaviour, not a fault. Note that ask-only test tooling will need base-side inventory seeded first — tell us before running the acceptance test and we will fund both sides.

### Issue: "Transaction reverted"
**Causes:**
- Quote expired → request a fresh quote (120s validity)
- `amountIn` scaled **up by more than 1%** → beyond the signed headroom; re-quote at the larger size (see Key Implementation Details §1)
- Caller is not `rfq_sender` → the order is locked to that address
- Order already filled → single-fill only

### Issue: Quote succeeds but the fill reverts
**Cause:** During early integration our staging PMM may publish **simulated depth** so the book is non-empty before the Safe is funded. Quotes are real and signed, but the maker cannot deliver, so fills revert.

**How to tell:** `GET /health` reports `"mockInventoryEnabled": true`, and each affected inventory row is flagged `"mocked": true`.

**Solution:** Use staging to validate request/response handling and calldata parsing, not settlement. We will confirm when the Safe is funded and simulation is off. Production never runs with this enabled.

### Issue: `1009 Unauthorised rfq_sender`
**Solution:** Your executor address is not in our whitelist for that chain. Send us the executor addresses per chain and we will configure them.

### Issue: `403 Forbidden` on /firm
**Solution:** Your egress IPs are not in our allowlist. Send us the current list.

---

