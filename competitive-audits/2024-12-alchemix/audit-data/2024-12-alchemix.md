# Alchemix Transmuter - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)


- ## Low Risk Findings
    - ### [L-01. Lack of parameter checks in `StrategyMainnet::addRoute` allows adding insufficient/wrong routes, `StrategyMainnet::claimAndSwap` will revert (with misleading error) if using a bad route. Moreover, a route cannot be changed or deleted.](#L-01)
    - ### [L-02. No functionality to update the router in `StrategyMainnet.sol`](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Alchemix

### Dates: Dec 16th, 2024 - Dec 23rd, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-12-alchemix)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 0
- Low: 2



    


# Low Risk Findings

## <a id='L-01'></a>L-01. Lack of parameter checks in `StrategyMainnet::addRoute` allows adding insufficient/wrong routes, `StrategyMainnet::claimAndSwap` will revert (with misleading error) if using a bad route. Moreover, a route cannot be changed or deleted.            



## Summary&#x20;

No sanity checks are integrated into `StrategyMainnet::addRoute` function allowing corrupted routes to be added, which then stay in the mapping forever; if such routes are passed to `StrategyMainnet::claimAndSwap`, it will revert, potentially with misleading errors.

## Vulnerability Details&#x20;

\[Curve documentation]\(<https://docs.curve.fi/router/CurveRouterNG/#route-and-swap-parameters>) determines the `_route` parameter of `Router.exchange()` function as follows: "... The first address is always the input token, the last one always the output token. The addresses inbetween compose the route the user wants to trade.".

Example from Curve documentation:

```Solidity
[

'0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2', # initial token: wETH

'0x7f86bf177dd4f3494b841a37e810a34dd56c829b', # pool1: TricryptoUSDC (USDC, wBTC, wETH)

'0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48', # token in between: USDC

'0x4dece678ceceb27446b35c672dc7d61f30bad69e', # pool2: crvUSD/USDC

'0xf939e0a03fb07f59a73314e73794be0e57ac1b4e', # output token: crvUSD

'0x0000000000000000000000000000000000000000',

'0x0000000000000000000000000000000000000000',

'0x0000000000000000000000000000000000000000',

'0x0000000000000000000000000000000000000000',

'0x0000000000000000000000000000000000000000',

'0x0000000000000000000000000000000000000000',

]

```

Neither `StrategyMainnet::addRoute` nor `StrategyMainnet::claimAndSwap` check if the first and last (non-zero) addresses are equal to addresses of WETH and alETH respectively. A manager may add a wrong or incomplete route to the mappings, which can lead to following:

1\. Either `router.exchange` reverts with some custom error or assertion violation.

2\. Or, the last non-zero address is basically of some other token (not alETH) and `router.exchange` does not revert, the following require statement will always revert:

```Solidity
require((balAfter - balBefore) >= _minOut, "Slippage too high");
```

`balAfter` and `balBefore` represent the contract alETH balance, so if the output token is not alETH, the function will revert with misleading error, making it harder to find the root cause of error.

Moreover, due to the structure of the `StrategyMainnet::addRoute` function, it is impossible to re-write route parameters or delete a bad route:

```Solidity
contract StrategyArb is BaseStrategy {

  ...

  uint256 public nRoutes = 0;

  mapping(uint256 => address[11]) public routes;
  mapping(uint256 => uint256[5][5]) public swapParams;
  mapping(uint256 => address[5]) public pools;
  ...

  function addRoute(address[11] calldata _route, uint256[5][5] calldata _swapParams, address[5] calldata _pools)
        external
        onlyManagement
    {
        routes[nRoutes] = _route;
        swapParams[nRoutes] = _swapParams;
        pools[nRoutes] = _pools;
        nRoutes++;
    }
  

  
```

The `nRotes` parameter is incremented by 1 with each `addRoute` run and cannot be changed otherwise. It represents the mapping index, at which the new entry should be stored.  Taking into account all of the above, if a bad route is added to the mapping, it cannot be adjusted or deleted.

## Impact

`StrategyMainnet::claimAndSwap` always revert if using a bad route. Keepers may anytime call `StrategyMainnet::claimAndSwap` with a bad route, as they cannot be deleted from the mapping.

## Tools Used

Manual review

## Recommendations

1. Add sanity checks at least to the `StrategyMainnet::addRoute` function&#x20;

```diff
+ error StrategyMainnet__InputTokenIsNotWETH();
+ error StrategyMainnet__OutputTokenIsNotAlETH();
+ error StrategyMainnet__BadSwapParams();

+ modifier routeIsOk(address[11] calldata _route, uint256[5][5] calldata _swapParams, address[5] calldata _pools) {
+    // Check if the first route address is equal to WETH address
+    if (_route[0] != address(underlying)) {
+         revert StrategyMainnet__InputTokenIsNotWETH();
+    }
+    // Check if the last non-zero address is equal to alETH address; _route[1] is allways a pool address
+    // Check for addresses in the slots 3-9
+    for (uint256 i = 2; i < 11; i++){
+         if (_route[i] = address(0) && _route[i-1] != address(0) && _route[i-1] != address(asset)){
+             revert StrategyMainnet__OutputTokenIsNotAlETH();        
+         }
+    }
+    // Checks if the last address is equal to the address of alETH token if there are no zero addresses
+    // (supposed there are no zero adresses inbetween non-zero addresses, for that another check is suggested)
+    if (_route[9] != address (0) && _route[10] != address(asset)){
+        revert StrategyMainnet__OutputTokenIsNotAlETH();
+    }
+
+    // Suggestions for further checks:
+    // 1. No zero adresses inbetween zero-addresses
+    // 2. There is always an odd number of non-zero addresses (due to the structure input token -> pool -> output token)
+    // 3. _swapParams[0] and _swapParams[1] must be equal to indexes of input and output tokens
+    // (check https://docs.curve.fi/router/CurveRouterNG/#_swap_params)
+
+    _;
+}

function addRoute(address[11] calldata _route, uint256[5][5] calldata _swapParams, address[5] calldata _pools)
        external
        onlyManagement
+       routeIsOk(_route, _swapParams, _pools)
    {
        routes[nRoutes] = _route;
        swapParams[nRoutes] = _swapParams;
        pools[nRoutes] = _pools;
        nRoutes++;
    }
```

Suggested tests:

* No zero adresses inbetween zero-addresses
* There is always an odd number of non-zero addresses (due to the structure input token -> pool -> output token)
* \_swapParams\[0] and \_swapParams\[1] must be equal to indexes of input and output tokens ((check <https://docs.curve.fi/router/CurveRouterNG/#_swap_params>))

1. Change the mapping population mechanism or add a functionality for Manager to change the nRoutes parameter&#x20;

```diff
    uint256 public nRoutes = 0;

+   function addOrEditRoute(address[11] calldata _route, uint256[5][5] calldata _swapParams, address[5] calldata _pools, uint256 _index, bool _add)
        external
        onlyManagement
        routeIsOk(_route, _swapParams, _pools)
    {
+       routes[_index] = _route;
+       swapParams[_index] = _swapParams;
+       pools[_index] = _pools;
+       if (_add) {
+           nRoutes++;
+       }
    }
```

This way a manager could change route parameters (in case they are outdated) or delete a corrupted route, while still keeping track of the number of routes with `nRoutes` parameter. If `_add` parameter is set to true, nRoutes are incremented. Otherwise it stays the same (if an entry gets re-written).

## <a id='L-02'></a>L-02. No functionality to update the router in `StrategyMainnet.sol`            



## Summary

The router address is hardcoded and initialized in constructor with `StrategyMainnet::_initStrategy` function. There is no way to update the router, but this functionality is intended by the developer, as it is implemented both in `StrategyArb.sol` and `StrategyOp.sol` with the `setRouter` function. &#x20;

## Impact

The management is unable to update the router address.

## Tools Used

Manual review

## Recommendations

Add the following function to `StrategyMainnet.sol`:

```Solidity
    function setRouter(address _router) external onlyManagement {
        router = ICurveRouterNG(_router);
        underlying.safeApprove(address(router), type(uint256).max);
    }
```

Be aware that `safeApprove` is deprecated. Consider using `safeIncreaseAllowance` as it was stated in the LightChaser's report.
