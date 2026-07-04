# HookSwap chains

Quoting is **V2 + V3 only** on all chains (NO v4). Chains are gated by the
`SUPPORTED_CHAINS` array in `lib/handlers/injector-sor.ts`; V2 is additionally gated by the
`v2Supported` array in the same file. V3 is attempted for any supported chain.

| Chain     | chainId    | routing-api RPC env var | RPC URL                            | Wrapped native | Addresses source                                            | Subgraph status |
|-----------|------------|-------------------------|------------------------------------|----------------|-------------------------------------------------------------|-----------------|
| Sepolia   | `11155111` | `WEB3_RPC_11155111`     | your Sepolia RPC                   | WETH `0xfFf9976782d46CC05630D1f6eBAb18b2324d6B14` | Canonical Uniswap stack, already in `HooksOS/sdks` sdk-core (real) | **TODO** — using on-chain static pools |
| HyperEVM  | `999`      | `WEB3_RPC_999`          | `https://rpc.hyperliquid.xyz/evm`  | WHYPE `0x5555555555555555555555555555555555555555` | `HooksOS/sdks` fork (placeholder) ← `contracts/deployments/hyperevm.json` after deploy | **TODO** — using on-chain static pools |
| Robinhood | `4663`     | `WEB3_RPC_4663`         | `https://rpc.mainnet.chain.robinhood.com` | WETH `0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73` | **DEPLOYED** — `contracts/deployments/robinhood.json` (see address table below) | **TODO** — using on-chain static pools |
| MegaETH   | `4326`     | `WEB3_RPC_4326`         | `https://mainnet.megaeth.com/rpc`  | WETH `0x4200000000000000000000000000000000000006` | **DEPLOYED** — `contracts/deployments/megaeth.json` (see address table below) | **TODO** — using on-chain static pools |
| Ink       | `57073`    | `WEB3_RPC_57073`        | `https://rpc-gel.inkonchain.com`   | WETH `0x4200000000000000000000000000000000000006` | **DEPLOYED** — `contracts/deployments/ink.json` (see address table below) | **TODO** — using on-chain static pools |

### Deployed HookSwap stack addresses (MegaETH, Ink, Robinhood — identical, deterministic deploys)

Deployer `0xc14C897c6bff88a5Eeac31F795693b9230205125`; same nonce sequence on each chain, so
the addresses match across all three (source: `HookSwap/contracts/deployments/{megaeth,ink,robinhood}.json`).

| Contract | Address (same on 4326 / 57073 / 4663) |
|----------|----------------------------------------|
| v2Factory | `0xD1Cf664944173140AFc302c169eFD55c24966B45` |
| v2Router02 | `0xBe3729d06E3A17F3c7c5ac394c7bCbe138B6EEFA` |
| v3Factory | `0xAa1f5Bd529Be345e7FB77934554112E5ecd7D7f3` |
| QuoterV2 | `0x15cD41B273865feD20BC8B5cDF4423D7678ac78E` |
| SwapRouter02 | `0xE8526A0429aeC9a5253ac854F8b6dC964E677EE4` |
| NonfungiblePositionManager | `0xbd817036c5bF69Cb27D3A342129e39f9f908577d` |
| UniversalRouter | `0x3D30133F4d4A80684F02d8310faF572E3dc193b3` |
| WETH (wrapped native) | MegaETH + Ink: `0x4200000000000000000000000000000000000006` (canonical, reused) · Robinhood: `0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73` (official, reused) |

Init-code hashes are canonical (v2 pair `0x96e8ac42…845f`, v3 pool `0xe34f199b…8b54`) — only
factory ADDRESSES need swapping in the sdk-core / SOR forks.

Notes:

- **Permit2** is the canonical `0x000000000022D473030F116dDEE9F6B43aC78BA3` on all chains
  (per `HookSwap/contracts/config/chains.json`).
- **Sepolia** — `skipDeploy: true`; canonical v2/v3/UniversalRouter already deployed and
  wired into sdk-core with real addresses.
- **HyperEVM** — still to deploy: `v2Factory`, `v2Router`, `v3` (via `@uniswap/deploy-v3`),
  `universalRouter`. Permit2 + WHYPE already exist. Needs HyperCore big blocks enabled first.
- **Robinhood / MegaETH / Ink** — **DEPLOYED** (own full v2+v3+UR stack, addresses above).
  WETH reused (Robinhood official WETH; MegaETH/Ink canonical `0x42…06`). Permit2 canonical.
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
