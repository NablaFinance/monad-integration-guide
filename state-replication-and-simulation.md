
# Integration Guide for Offchain state replication and simulation

Two modes of integration are possible. Offchain state replication and simulation - described in this guide and [onchain quoting](onchain-quoting.md). While the later is trivial to implement, the former is better suited for Nabla's High Frequency Pricing engine and will result with greater volumes and revenues over time.

If you are new to Nabla, we recommend you start with the [Developer overview](https://docs.nabla.fi/developers) section first and explore our  [API](https://swap-api.nabla.fi/docs#/tokens/chain_tokens_endpoint_chains__chainId__tokens_get) for reference.

We will go through the steps to **discover pools**, **replicated the setate offchain using events** and **simulate a swap quote** and finally **execute a swap** via `NablaPortal` contract.

## 1. Pool Discovery

### ABI

1. **Fetch available routers**
   [`Portal.getRouters()`](https://docs.nabla.fi/developers/portal#getrouters) → returns all supported routers.
   ```solidity
   function getRouters() external view returns (address[] memory routers)
   ```
2. **Get pools and assets of router**
   [`Portal.getRouterPools(router)`](https://docs.nabla.fi/developers/portal#getrouterpools) → returns assets and pools supported by that router.
   ```solidity
   function getRouterPools(address router) external view returns (address[] memory assets, address[] memory pools)
   ```
3. **Update pool and router list using events**
```solidity
    event AssetRegistered(address indexed sender, address router, address asset, address pool);
    event AssetUnregistered(address indexed sender, address router, address asset, address pool);
```

Note that currently (and in the forseeable future) we are deploying a single router only.

### Ethers.js example

```ts
import { ethers } from "ethers";

const portalAddress = process.env.PORTAL_ADDRESS!;
const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);

const portalAbi = [
  "event AssetRegistered(address indexed sender, address router, address asset, address pool)",
  "event AssetUnregistered(address indexed sender, address router, address asset, address pool)",
  "function getRouters() external view returns (address[])",
  "function getRouterPools(address router) external view returns (address[] assets, address[] pools)"
];

const portal = new ethers.Contract(portalAddress, portalAbi, provider);

interface Pool {
  address: string;
  asset: string;
}

interface Router {
  address: string;
  pools: Pool[];
}

const routers: Record<string, Router> = {};

// --- Pool Discovery ---
async function discoverPools() {
  const routerAddresses: string[] = await portal.getRouters();
  console.log("Routers discovered:", routerAddresses);

  for (const routerAddress of routerAddresses) {
    const [assetAddresses, poolAddresses]: [string[], string[]] = await portal.getRouterPools(routerAddress);
    const pools: Pool[] = poolAddresses.map((pool, i) => ({
      address: pool,
      asset: assetAddresses[i],
    }));

    routers[routerAddress] = {
      address: routerAddress,
      pools,
    };

    console.log(`Router: ${routerAddress}`);
    pools.forEach((p) => console.log(`  Pool: ${p.address}, Asset: ${p.asset}`));
  }
}

// --- Updates ---
function setupEventListeners() {
  portal.on("AssetRegistered", (sender, router, asset, pool) => {
    console.log(`AssetRegistered -> Router: ${router}, Asset: ${asset}, Pool: ${pool}`);

    if (!routers[router]) {
      routers[router] = { address: router, pools: [] };
    }

    // Avoid duplicates
    const exists = routers[router].pools.some(p => p.address === pool);
    if (!exists) {
      routers[router].pools.push({ address: pool, asset });
    }
  });

  portal.on("AssetUnregistered", (sender, router, asset, pool) => {
    console.log(`AssetUnregistered -> Router: ${router}, Asset: ${asset}, Pool: ${pool}`);

    if (routers[router]) {
      routers[router].pools = routers[router].pools.filter(p => p.address !== pool);
    }
  });
}

async function main() {
  await discoverPools(); // Initial discovery
  setupEventListeners(); // Keep pools up-to-date with on-chain events
}

main().catch(console.error);
```

## 2. Nabla State

Nabla consists of single-sided liquidity pools gathered under a router.
Each pool maintains its own reserve and liabilities, while an external oracle provides token prices.
Integration needs to track both metadata (fixed parameters) and dynamic state (reserves, price, etc.).

### State Shape

```ts
interface NablaPoolMeta {
  token: string;
  assetDecimals: bigint;
  curveBeta: bigint;
  curveC: bigint;
  backstopFee: bigint;
  protocolFee: bigint;
  lpFee: bigint;
}

interface NablaPoolState {
  reserve: bigint;
  reserveWithSlippage: bigint;
  totalLiabilities: bigint;
  price: bigint;
}

interface Pool {
  address: string;
  asset: string;
  meta?: NablaPoolMeta;
  state?: NablaPoolState;
}

interface Router {
  address: string;
  pools: Pool[];
}

```

### ABI

Changes to the state are completely encapsulated by 2 events 

```solidity
interface Pool {
  event ReserveUpdated(uint256 reserve, uint256 reserveWithSlippage, uint256 totalLiabilities);
}
```

Note that the emited values are absolute (not state deltas).

```solidity
interface OracleAdapter {
  function getAssetPrice(address _assetAddress) public view returns(uint256 price_);
}
```

Oracle adapter is always a singleton and provides prices for all supported tokens.

Fee changes can be followed via:

```solidity
interface Pool {
  event SwapFeesSet(address indexed sender, uint256 lpFee, uint256 backstopFee, uint256 protocolFee);
}
```

### Fetching Pool Metadata and initial state 

```solidity
interface Pool {
  function swapFees() external view returns(uint256 lpFee, backstopFee, protocolFee);
  function slippageCurve() external view returns(address curve);

  function reserve() external view returns(uint256 reserve);
  function reserveWithSlippage() external view returns(uint256 reserveWithSlippage);
  function totalLiabilities() external view returns(uint256 totalLiabilities);
}

interface SlippageCurve {
  function beta() external view returns(uint256 beta);
  function c() external view returns(uint256 c);
}
```

### Ethers.js example 

#### Pool intiialization

```ts
import { ethers } from "ethers";

const oracleAbi = [
    "event PriceUpdated(address indexed token, uint64 publishTime, int64 price)" 
];

const poolAbi = [
    "event ReserveUpdated(uint256 reserve, uint256 reserveWithSlippage, uint256 totalLiabilities)"
];

const curveAbi = [
    "function beta() view returns (int256)",
    "function c() view returns (int256)"
];


const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
const oracleAdapterAddress = process.env.ORACLE_ADAPTER!; // Singleton oracle adapter

// Store for routers (populated in previous section)
const routers: Record<string, Router> = {};

async function initializePools() {
  console.log("Initializing pool metadata and state...");

  for (const router of Object.values(routers)) {
    for (const pool of router.pools) {
      const poolContract = new ethers.Contract(pool.address, poolAbi, provider);

      // --- Fetch metadata ---
      const [lpFee, backstopFee, protocolFee] = await poolContract.swapFees();
      const curveAddress = await poolContract.slippageCurve();
      const curve = new ethers.Contract(curveAddress, curveAbi, provider);

      const [curveBeta, curveC] = await Promise.all([
        curve.beta(),
        curve.c(),
      ]);

      // asset decimals
      const tokenContract = new ethers.Contract(pool.asset, ["function decimals() view returns (uint8)"], provider);
      const assetDecimals = await tokenContract.decimals();

      pool.meta = {
        token: pool.asset,
        assetDecimals: BigInt(assetDecimals),
        curveBeta: BigInt(curveBeta),
        curveC: BigInt(curveC),
        backstopFee: BigInt(backstopFee),
        protocolFee: BigInt(protocolFee),
        lpFee: BigInt(lpFee),
      };

      // --- Fetch state ---
      const [reserve, reserveWithSlippage, totalLiabilities] = await Promise.all([
        poolContract.reserve(),
        poolContract.reserveWithSlippage(),
        poolContract.totalLiabilities(),
      ]);

      let price = 0n;

      pool.state = {
        reserve: BigInt(reserve),
        reserveWithSlippage: BigInt(reserveWithSlippage),
        totalLiabilities: BigInt(totalLiabilities),
        price,
      };

      console.log(`Initialized pool: ${pool.address}`);
      console.table(pool.meta);
      console.table(pool.state);
    }
  }
}
```

State change listeners

```ts

const oracleAbi = [
  "function getAssetPrice(address _assetAddress) public view returns(uint256 price_)" 
];
const poolAbi = [
  "event ReserveUpdated(uint256 reserve, uint256 reserveWithSlippage, uint256 totalLiabilities)"
];

function setupPoolStateEventListeners() {
  console.log("Subscribing to pool and oracle events...");

  for (const router of Object.values(routers)) {
    for (const pool of router.pools) {
      const poolContract = new ethers.Contract(pool.address, poolAbi, provider);

      // --- ReserveUpdated (state change) ---
      poolContract.on("ReserveUpdated", (reserve, reserveWithSlippage, totalLiabilities) => {
        pool.state = {
          ...pool.state!,
          reserve: BigInt(reserve),
          reserveWithSlippage: BigInt(reserveWithSlippage),
          totalLiabilities: BigInt(totalLiabilities),
        };
        console.log(`Reserve updated for pool ${pool.address}:`, pool.state);
      });

      // --- SwapFeesSet (metadata update) ---
      poolContract.on("SwapFeesSet", (sender, lpFee, backstopFee, protocolFee) => {
        if (!pool.meta) return;
        pool.meta.lpFee = BigInt(lpFee);
        pool.meta.backstopFee = BigInt(backstopFee);
        pool.meta.protocolFee = BigInt(protocolFee);
        console.log(`Fees updated for pool ${pool.address}:`, pool.meta);
      });
    }
  }

  // --- Oracle prices  (global) ---
  const blockTime = 400;
  setInterval(() => {
    for (const router of Object.values(routers)) {
   
      for (const pool of router.pools) {
        pool.state.price = await oracle.getAssetPrice(pool.asset)
      }
    }
  }, blockTime);
}
```

Full initialization

```ts
async function main() {
  await discoverPools();       // From Section 1
  await initializePools();     // Fetch metadata & state
  setupPoolStateEventListeners(); // Listen for updates
}

main().catch(console.error);
```


## 3.Swap simulation

Up to date states of two pools are enough for exact simulation of a swap. Swap logic involved relies, among other things, on Nabla slippage curve implementation. Due to the fact that said curve is not open source, examples of it's simulation will not be published. Please contact us directly whenever you are ready to implement the simulation logic.
Apart from Solidity we have typescript and Rust implementations of the logic. 

## 4.Swap execution

### ABI

[NablaPortal](https://docs.nabla.fi/developers/portal) serves as general entrypoint for swaps, single and multihop alike (for the future case of multiple routers). 

```solidity
function swapExactTokensForTokens(
    uint256 _amountIn,
    uint256 _amountOutMin,
    address[] calldata _tokenPath,
    address[] calldata _routerPath,
    address _to,
    uint256 _deadline
) external returns (uint256 amountOut_)
```

For asingle hop swap is `_tokenPath = [tokenIn, tokenOut]` and `_routerPath = [routerAddress]`

### Ethers example

```ts
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);
const portalAddress = proces.env.PORTAL_ADDRESS;

const portalAbiSwap = [
  "function swapExactTokensForTokens(uint256,uint256,address[],address[],address,uint256,bytes[]) external returns (uint256)"
];

const portalWithSigner = new ethers.Contract(
  portalAddress,
  portalAbiSwap,
  wallet
);

async function doSwap(assetA, assetB, router, quotedOut) {
  const tx = await portalWithSigner.swapExactTokensForTokens(
    ethers.parseUnits("1.0", 18),     // amount in
    quotedOut * 9975n / 10000n,       // slippage tolerance (eg 0.25%)
    [assets[0], assets[1]],           // token path
    [router],                         // router path
    wallet.address,                   // recipient
    Math.floor(Date.now() / 1000) + 1, // deadline  
);
  console.log("Swap tx:", tx.hash);
}
```
