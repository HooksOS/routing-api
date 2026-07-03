# HookSwap chains

Quoting is **V2 + V3 only** on all three chains (NO v4). Chains are gated by the
`SUPPORTED_CHAINS` array in `lib/handlers/injector-sor.ts`; V2 is additionally gated by the
`v2Supported` array in the same file. V3 is attempted for any supported chain.

| Chain     | chainId    | routing-api RPC env var | RPC URL                            | Wrapped native | Addresses source                                            | Subgraph status |
|-----------|------------|-------------------------|------------------------------------|----------------|-------------------------------------------------------------|-----------------|
| Sepolia   | `11155111` | `WEB3_RPC_11155111`     | your Sepolia RPC                   | WETH `0xfFf9976782d46CC05630D1f6eBAb18b2324d6B14` | Canonical Uniswap stack, already in `HooksOS/sdks` sdk-core (real) | **TODO** — using on-chain static pools |
| HyperEVM  | `999`      | `WEB3_RPC_999`          | `https://rpc.hyperliquid.xyz/evm`  | WHYPE `0x5555555555555555555555555555555555555555` | `HooksOS/sdks` fork (placeholder) ← `contracts/deployments/hyperevm.json` after deploy | **TODO** — using on-chain static pools |
| Robinhood | `4663`     | `WEB3_RPC_4663`         | Robinhood chain RPC                | WETH — deploy via Foundry kit | `HooksOS/sdks` fork (placeholder) ← `contracts/deployments/robinhood.json` after deploy | **TODO** — using on-chain static pools |

Notes:

- **Permit2** is the canonical `0x000000000022D473030F116dDEE9F6B43aC78BA3` on all three
  (per `HookSwap/contracts/config/chains.json`).
- **Sepolia** — `skipDeploy: true`; canonical v2/v3/UniversalRouter already deployed and
  wired into sdk-core with real addresses.
- **HyperEVM** — deploy `v2Factory`, `v2Router`, `v3` (via `@uniswap/deploy-v3`),
  `universalRouter`. Permit2 + WHYPE already exist.
- **Robinhood** — deploy `weth9`, `v2Factory`, `v2Router`, `v3`, `universalRouter`.
  Permit2 already exists.
- **Subgraph status = TODO for all:** no subgraph deployed. SOR's on-chain
  `StaticV2SubgraphProvider` / `StaticV3SubgraphProvider` are used automatically (via the
  fallback in `instantiateSubgraphProvider`), so routing works from on-chain pool reads once
  contracts + real pools exist. Deploying subgraphs later requires registering the chains in
  `lib/cron/cache-config.ts` and setting `SUBGRAPH_URL_<chainId>`.

### Address maps that must carry these chains (in the SOR + sdk-core forks)

- sdk-core (`HooksOS/sdks`): `ChainId` enum (already has `HYPEREVM=999`, `ROBINHOOD=4663`,
  `SEPOLIA`), plus per-chain address maps.
- smart-order-router (`HooksOS/smart-order-router`): `src/util/chains.ts`
  (`SUPPORTED_CHAINS`, `V2_SUPPORTED`, `ID_TO_CHAIN_ID`, `ID_TO_NETWORK_NAME`,
  `WRAPPED_NATIVE_CURRENCY`) and `src/util/addresses.ts` (`V3_CORE_FACTORY_ADDRESSES`,
  `V2_FACTORY_ADDRESSES`, `QUOTER_V2_ADDRESSES`, `NEW_QUOTER_V2_ADDRESSES`,
  `SWAP_ROUTER_02_ADDRESSES`, multicall).
