# Integration Guide for Onchain Quoting

Two modes of integration are possible. Onchain quoting - described in this guide and [Offchain state replication and simulation](state-replication-and-simulation.md). While the former is trivial to implement the latter is better suited for Nabla's High Friquency Pricing engine and will result with greater volumes and revenues over time.

If you are new to Nabla, we recommend you start with the [Developer overview](https://docs.nabla.fi/developers) section first and explore our  [API](https://swap-api.nabla.fi/docs#/tokens/chain_tokens_endpoint_chains__chainId__tokens_get) for reference.

We will go through the steps to **discover pools** and **get an on-chain swap quote** and show you how to **execute a swap** via `NablaPortal` contract.


## 1. Pool Discovery

### ABI

1. **Fetch available routers**
   [`Portal.getRouters()`](https://docs.nabla.fi/developers/portal#getrouters) → returns all supported routers.
   ```solidity
   function getRouters() external view returns (address[] memory routers)
   ```
2. **Get pools and assets of router**
   ```solidity
   function getRouterPools(address router) external view returns (address[] memory assets, address[] memory pools)
   ```
   
   [`Portal.getRouterPools(router)`](https://docs.nabla.fi/developers/portal#getrouterpools) → returns assets and pools supported by that router.

Note that currently (and in the forseable future) we are deploying a single router only.

### Ethers.js example

```ts
import { ethers } from "ethers";

const portalAddress = process.env.PORTAL_ADDRESS;

const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);

const portalAbi = [
  "function getRouters() external view returns (address[])",
  "function getRouterPools(address router) external view returns (address[])"
];

const portal = new ethers.Contract(
  portalAddress,
  portalAbi,
  provider
);

async function discoverPools() {
  const routers = await portal.getRouters();
  console.log("Routers:", routers);

  const pools = await portal.getRouterPools(routers[0]);
  const [poolAddresses, assetAddresses] = pools;
  console.log("Pools and assets for router 0:");
  poolsAddresses.forEach((pool, i) => {
    console.log("Pool:", pool, "Asset:", assetAddresses[i])
  })

  return {router: routers[0], assets: assetAddresses};
}

discoverPools();
```

## 2. Quoting

### ABI

```solidity
function quoteSwapExactTokensForTokens(
    uint256 _amountIn,
    address[] calldata _tokenPath,
    address[] calldata _routerPath
) external view returns (uint256 amountOut_)
```

`_amountIn`: input token amount

`_tokenPath`: ordered list of token addresses (swap path). 

`_routerPath`: ordered list of routers

### Ethers.js example

```ts

const portalAddress = process.env.PORTAL_ADDRESS;

const portalQuoteAbi = [
  "function quoteSwapExactTokensForTokens(uint256 amountIn, address[] tokenPath, address[] routerPath) external view returns (uint256)"
];

const portal = new ethers.Contract(
  portalAddress,
  quoterAbi,
  provider
);

async function getQuote(assetA, assetB, router) {
  const quote = await portal.quoteSwapExactTokensForTokens(
    ethers.parseUnits("1.0", 18), 
    [assets[0], assets[1]],
    [router]
  );
  console.log("Quoted output:", quote.toString());
}
```

## 3.Swapping


### ABI

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

## 4. Full flow example

```ts
import { ethers } from "ethers";

const RPC_URL = process.env.RPC_URL;
const PRIVATE_KEY = process.env.PRIVATE_KEY;

const provider = new ethers.JsonRpcProvider(RPC_URL);
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);

// ---- ABIs ----
const portalAbi = [
  "function getRouters() external view returns (address[])",
  "function getRouterPools(address router) external view returns (address[])",
  "function quoteSwapExactTokensForTokens(uint256,uint256,address[],address[],address,uint256,bytes[]) external returns (uint256)"
  "function swapExactTokensForTokens(uint256,uint256,address[],address[],address,uint256,bytes[]) external returns (uint256)"
];

// ---- Contract addresses ----
const portalAddress = process.env.PORTAL_ADDRESS;


async function discoverPools() {
  const [router] = await portal.getRouters();

  const [poolAddresses, assets] = await portal.getRouterPools(routers[0]);

  return {router, assets};
}

async function main() {
  const portal = new ethers.Contract(PORTAL, portalAbi, wallet);

  // 1. Discover pools
  const {router, assets} = await discoverPools();

 
  // 2. Quote swap
  const amount = ethers.parseUnits("1", 18);

  const quotedOut = await portal.quoteSwapExactTokensForTokens(
    amount,
    [asset[0], asset[1]],
    [router]
  );
  console.log("Quoted output:", quotedOut.toString());

  // 3. Execute swap
  const minAmountOut = quotedOut * 9990n / 10000n; // 0.1 % slippage tolerance
  const tx = await portal.swapExactTokensForTokens(
    amount,
    minAmountOut,
    [asset[0], asset[1]],
    [router],
    wallet.address,
    Math.floor(Date.now() / 1000) + 1
  );
  console.log("Swap tx hash:", tx.hash);
}

main().catch(console.error);
```
