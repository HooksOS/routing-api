# HookSwap self-hosted quoting service (routing-api fork)

This folder documents how the forked **`HooksOS/routing-api`** (Uniswap's classic
smart-order-router-based quoting engine) is wired for the three HookSwap chains:

| Chain     | chainId    | Protocols wired |
|-----------|------------|-----------------|
| Sepolia   | `11155111` | v2 + v3         |
| HyperEVM  | `999`      | v2 + v3         |
| Robinhood | `4663`     | v2 + v3         |

**No v4.** These chains are deliberately kept out of every `v4Supported` / `mixedSupported`
gate. Quoting is V2 + V3 only.

---

## 1. Architecture (who calls what)

```
HookSwap interface (web app)
        │   speaks the "Trading API" schema (quote/swap/approval), NOT routing-api's schema
        ▼
Trading API  (a.k.a. unified-routing-api / trading-api)   ← YOU MUST ALSO HOST THIS (see §6)
        │   translates Trading API requests → routing-api GET /quote?...  requests
        ▼
routing-api  (THIS repo)  ── classic SOR quote server (AWS Lambda + API Gateway via CDK)
        │
        ▼
@uniswap/smart-order-router (SOR)   ← the AlphaRouter engine
        │
        ├── RPC provider   (per-chain JSON-RPC, read as WEB3_RPC_<chainId>)
        └── pools/liquidity (V2/V3 SubgraphProvider  OR  on-chain Static*SubgraphProvider)
```

> **IMPORTANT:** The HookSwap interface does **not** call this `routing-api` directly.
> The interface is built against the **Trading API** (the schema exposed at
> `trading.gateway.uniswap.org` / `TRADING_API_URL_OVERRIDE`). `routing-api` is the
> lower-level "classic" quoting engine that a Trading API (unified-routing-api) wraps.
> Standing up `routing-api` alone gives you a working `GET /quote` endpoint, but the
> interface will not talk to it until you also provide a Trading API layer (§6).

### How chains are gated

`routing-api` gates supported chains in one primary place and several supporting maps:

- **Primary gate:** `SUPPORTED_CHAINS` in `lib/handlers/injector-sor.ts`. This array is
  re-exported and used by the request schema (`lib/handlers/quote/schema/quote-schema.ts`)
  to `Joi.valid(...)` the `tokenInChainId` / `tokenOutChainId` params. A chain not in this
  array is rejected at validation time.
- **Per-protocol gates (inside `buildContainerInjected` in the same file):**
  `v2Supported`, `v4Supported`, `mixedSupported`. We add the HookSwap chains to
  `v2Supported` only. V3 is always attempted for any `SUPPORTED_CHAINS` entry (there is no
  `v3Supported` allow-list — V3 is the default protocol at `quote.ts` when none requested).

### How RPC URLs are read per chain

In `lib/handlers/injector-sor.ts` → `buildContainerInjected`:

```ts
url = process.env[`WEB3_RPC_${chainId.toString()}`]!
```

So each chain needs a `WEB3_RPC_<chainId>` env var. (A newer "RPC gateway" path exists via
`lib/rpc/GlobalRpcProviders.ts` + `lib/config/rpcProviderProdConfig.json`, but the HookSwap
chains are **not** in the gateway config, so they fall through to the simple
`WEB3_RPC_<chainId>` env-var path — which is what we want for self-hosting.)

Required env vars:

| Chain     | env var              | value                              |
|-----------|----------------------|------------------------------------|
| Sepolia   | `WEB3_RPC_11155111`  | your Sepolia RPC                   |
| HyperEVM  | `WEB3_RPC_999`       | `https://rpc.hyperliquid.xyz/evm`  |
| Robinhood | `WEB3_RPC_4663`      | Robinhood chain RPC                |

> Note: the Foundry deploy kit in `HookSwap/contracts/config/chains.json` names its RPC
> env vars `SEPOLIA_RPC_URL` / `HYPEREVM_RPC_URL` / `ROBINHOOD_RPC_URL`. `routing-api` uses
> a **different** convention (`WEB3_RPC_<chainId>`). Set both if you share one `.env`.

### How pools / liquidity are obtained

`instantiateSubgraphProvider` (in `injector-sor.ts`) does the following per chain/protocol:

1. Tries an **AWS S3 pool-cache-backed** subgraph provider
   (`V2AWSSubgraphProvider` / `V3AWSSubgraphProvider` `.EagerBuild(...)`), which reads a
   pre-computed pool snapshot from an S3 bucket populated by the `cache-pools` cron.
2. **On any error** (no S3 bucket, chain not present in `lib/cron/cache-config.ts`
   `chainProtocols`, subgraph unreachable) it **falls back** to the SOR **on-chain static
   providers**: `StaticV2SubgraphProvider` / `StaticV3SubgraphProvider`.

The HookSwap chains have **no subgraph deployed**, and we intentionally did **not** add them
to `chainProtocols`. As a result they take the **static on-chain path** automatically: SOR
discovers V2 pairs / V3 pools by computing candidate pool addresses and reading them
on-chain via multicall. This lets the service route as soon as contracts + real pools exist,
with no subgraph dependency.

If you later deploy subgraphs, add per-chain entries to `lib/cron/cache-config.ts`
(`chainProtocols`) and the `v2SubgraphUrlOverride` / `v3SubgraphUrlOverride` switches, and
supply `SUBGRAPH_URL_<chain>` values (left as TODO in `.env.example`).

---

## 2. Files edited to add the 3 chains

All edits are in the forked `routing-api` (this repo), nothing in the interface or the
`sdks` fork was touched.

| File | Change |
|------|--------|
| `lib/handlers/injector-sor.ts` | Added `ChainId.HYPEREVM`, `ChainId.ROBINHOOD` to the exported **`SUPPORTED_CHAINS`** array (Sepolia already present). Added `ChainId.SEPOLIA`, `ChainId.HYPEREVM`, `ChainId.ROBINHOOD` to the **`v2Supported`** array inside `buildContainerInjected`. **Not** added to `v4Supported` / `mixedSupported` (v2+v3 only). |
| `lib/util/newCachedRoutesRolloutPercent.ts` | Added `HYPEREVM`/`ROBINHOOD` entries (exhaustive `{ [chain in ChainId]: number }` map — would fail to compile without them once the fork enum adds the members). |
| `lib/util/tenderlyNewEndpointRolloutPercent.ts` | Same; set to `0` (no Tenderly project for these chains → uses `eth_estimateGas` fallback). |
| `lib/util/extraV4FeeTiersTickSpacingsHookAddresses.ts` | Added `HYPEREVM`/`ROBINHOOD` → `emptyV4FeeTickSpacingsHookAddresses` (exhaustive map; no v4). |
| `lib/util/hooksAddressesAllowlist.ts` | Added `HYPEREVM`/`ROBINHOOD` → `[ADDRESS_ZERO]` (exhaustive map; no-hook pools only). |
| `lib/util/onChainQuoteProviderConfigs.ts` | Added `HYPEREVM`/`ROBINHOOD` to the two exhaustive maps `NEW_QUOTER_DEPLOY_BLOCK` (→ `0`, **TODO** real deploy block) and `LIKELY_OUT_OF_GAS_THRESHOLD` (→ `0` = disabled). |
| `lib/util/defaultBlocksToLiveRoutesDB.ts` | Added `HYPEREVM`/`ROBINHOOD` → `60` (**TODO** tune to real block time). |

The "exhaustive map" edits are only strictly required **once `@uniswap/sdk-core` is
resolved to the fork** (see §3) — that is when `ChainId.HYPEREVM` / `ChainId.ROBINHOOD`
become real enum members and TypeScript demands every `{ [chain in ChainId]: T }` literal
list them.

---

## 3. `ChainId` must resolve to the HooksOS/sdks fork (dependency override — TODO)

`routing-api` imports `ChainId` from `@uniswap/sdk-core`, and `SUPPORTED_CHAINS`/`v2Supported`
reference `ChainId.HYPEREVM` (999) and `ChainId.ROBINHOOD` (4663). **Upstream `@uniswap/sdk-core`
does not define these**, so the edits only compile when the dependency is pointed at the
HookSwap fork `github.com/HooksOS/sdks` (which defines `HYPEREVM = 999`, `ROBINHOOD = 4663`,
`SEPOLIA = 11155111`).

Two dependencies must be overridden to the forks:

- `@uniswap/sdk-core`         → `HooksOS/sdks` (`sdks/sdk-core`) — so `999` / `4663` are known ChainIds.
- `@uniswap/smart-order-router` → `HooksOS/smart-order-router` — so the SOR engine itself
  recognises the chains and has their contract addresses (see §5). Upstream SOR
  `4.31.8` (the version pinned in `package.json`) knows nothing about 999 / 4663.

Options to wire the overrides (pick one, **not yet applied** here):

1. **npm `overrides` / yarn `resolutions`** in `package.json`, pointing both packages at the
   built fork artifacts (published to a private registry, or a git URL that builds on install).
2. **Local link:** `npm link` / `yarn link` the locally built `HooksOS/sdks` sdk-core and
   `HooksOS/smart-order-router` into `node_modules` (the sibling clones live at
   `../smart-order-router` and a checkout of `../sdks`).

Until this override is in place, `npm run build` / `tsc` will error on the new `ChainId`
members — that is expected and is the honest current state.

---

## 4. Environment variables

See `.env.example`. Minimum to build + serve quotes for the 3 chains:

- `WEB3_RPC_11155111`, `WEB3_RPC_999`, `WEB3_RPC_4663` — per-chain RPC (required).
- `SUBGRAPH_URL_11155111`, `SUBGRAPH_URL_999`, `SUBGRAPH_URL_4663` — **TODO/placeholder**;
  only needed if you switch these chains off the on-chain static path onto a subgraph.
- Tenderly / Alchemy / Goldsky / AWS keys — optional; not needed for on-chain static routing.

---

## 5. Filling contract addresses

The SOR engine needs, **per chain**, the addresses of: V3 core factory, V2 factory,
QuoterV2 (+ NEW_QUOTER_V2), SwapRouter02 / UniversalRouter, and Multicall2, plus the wrapped
native token. In SOR these live in `src/util/addresses.ts` (`V3_CORE_FACTORY_ADDRESSES`,
`QUOTER_V2_ADDRESSES`, `NEW_QUOTER_V2_ADDRESSES`, `SWAP_ROUTER_02_ADDRESSES`, …) and
`src/util/chains.ts` (`WRAPPED_NATIVE_CURRENCY`, `ID_TO_NETWORK_NAME`, `SUPPORTED_CHAINS`,
`V2_SUPPORTED`).

Sources of the real values:

- **Sepolia (11155111):** canonical Uniswap v2/v3/UniversalRouter stack is already deployed
  and already wired into `sdk-core` with real addresses (`HookSwap/contracts/config/chains.json`
  → `skipDeploy: true`). Nothing to fill.
- **HyperEVM (999) & Robinhood (4663):** addresses are **placeholders** in the sdks fork until
  the Foundry kit deploys the contracts. After deploy, `HookSwap/contracts/scripts` writes
  `HookSwap/contracts/deployments/<chain>.json` (currently empty — not deployed yet). Copy
  those addresses into the **`HooksOS/sdks` fork** (sdk-core address maps) and the
  **`HooksOS/smart-order-router` fork** (`addresses.ts` / `chains.ts`), then rebuild the forks
  so this `routing-api` picks them up via the §3 override.

WHYPE (HyperEVM wrapped native) = `0x5555555555555555555555555555555555555555`; Permit2 is the
canonical `0x000000000022D473030F116dDEE9F6B43aC78BA3` on all three chains (per
`contracts/config/chains.json`).

---

## 6. Running locally + the interface ↔ Trading API ↔ routing-api gap

### Running routing-api

`routing-api` is **not a plain Express server** — it is an AWS Lambda + API Gateway app
defined with CDK (`bin/app.ts`, `bin/stacks/*`). There is no built-in `npm start`/`listen()`
harness. Practical options:

- **Build/typecheck:** `npm install && npm run build` (runs typechain codegen + `tsc`).
  Requires the §3 fork override to compile with the new ChainIds.
- **Deploy (the supported path):** configure AWS + CDK, then `cdk deploy RoutingAPIStack`
  (see the repo root `README.md`). Yields `https://…/quote`.
- **Invoke:** `scripts/get_quote.ts` posts to `process.env.UNISWAP_ROUTING_API + 'quote'`,
  and the e2e mocha tests hit a deployed URL. Example GET:
  `/quote?tokenInAddress=0x..&tokenInChainId=999&tokenOutAddress=0x..&tokenOutChainId=999&amount=1000000&type=exactIn&protocols=v2,v3`
- **Fully-local option (not included here):** wrap the exported quote Lambda handler
  (`lib/handlers/quote/…`) in a thin Express adapter that maps an HTTP request into the
  Lambda `event` shape. This is the lightest way to get a local `GET /quote` without AWS.

### The interface connection gap (honest)

The HookSwap interface speaks the **Trading API** schema, not routing-api's native
`GET /quote` schema. To connect the interface to this self-hosted stack you must do **one**
of:

- **(A) Also self-host a Trading API** (`unified-routing-api` / `trading-api`) that wraps this
  `routing-api`, and point the interface's `TRADING_API_URL_OVERRIDE` at it. This is the
  faithful architecture and what production uses.
- **(B) Point `TRADING_API_URL_OVERRIDE` at a small adapter** you write that accepts the
  interface's Trading-API requests and translates them into `routing-api` `GET /quote` calls
  (and back). Lighter than (A) but you own the schema mapping and swap-calldata assembly.

There is **no** interface setting that makes it call `routing-api`'s classic schema directly.

---

## 7. What remains before it can actually quote

1. **Deploy contracts** on HyperEVM (999) and Robinhood (4663) via the Foundry kit; capture
   `contracts/deployments/<chain>.json`.
2. **Fill addresses** into the `HooksOS/sdks` and `HooksOS/smart-order-router` forks
   (factory / quoter / router / multicall / wrapped-native) and rebuild them.
3. **Add the 3 chains to the SOR fork** `src/util/chains.ts` (`SUPPORTED_CHAINS`,
   `V2_SUPPORTED`, `ID_TO_CHAIN_ID`, `ID_TO_NETWORK_NAME`, `WRAPPED_NATIVE_CURRENCY` — the last
   is itself an exhaustive `{ [chain in ChainId]: Token }` map) and `src/util/addresses.ts`.
   *(This routing-api fork is wired; SOR is a separate dependency that also needs these edits —
   see §3.)*
4. **Apply the dependency override** (§3) so `routing-api` resolves both packages to the forks;
   then `npm run build` compiles.
5. **Provide liquidity** — real V2/V3 pools must exist on-chain for the static providers to
   find routes (or deploy subgraphs and register them in `cache-config.ts`).
6. **Set `WEB3_RPC_*`** env vars and deploy/serve routing-api.
7. **Stand up a Trading API layer** (§6) and point the interface's `TRADING_API_URL_OVERRIDE`
   at it.
