### [M-1] Lack of parameter checks in `StrategyMainnet::addRoute` may lead to `StrategyMainnet::claimAndSwap` always reverting if route parameters are wrong.

**Description:** 

[Curve documentation](https://docs.curve.fi/router/CurveRouterNG/#route-and-swap-parameters) determines the `_route` parameter of `Router.exchange()` function as follows: "...  The first address is always the input token, the last one always the output token. The addresses inbetween compose the route the user wants to trade.". 
Example:
```
[
    '0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2',   # initial token: wETH
    '0x7f86bf177dd4f3494b841a37e810a34dd56c829b',   # pool1: TricryptoUSDC (USDC, wBTC, wETH)
    '0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48',   # token in between: USDC
    '0x4dece678ceceb27446b35c672dc7d61f30bad69e',   # pool2: crvUSD/USDC
    '0xf939e0a03fb07f59a73314e73794be0e57ac1b4e',   # output token: crvUSD
    '0x0000000000000000000000000000000000000000',
    '0x0000000000000000000000000000000000000000',
    '0x0000000000000000000000000000000000000000',
    '0x0000000000000000000000000000000000000000',
    '0x0000000000000000000000000000000000000000',
    '0x0000000000000000000000000000000000000000',
]
```

Neither `StrategyMainnet::addRoute` nor `StrategyMainnet::claimAndSwap` check if the first and last (non-zero) addresses are equal to addresses of WETH and alETH respectively. A manager may add a wrong or incomplete route to the mappings, which can lead to following:

1. Either `router.exchange` reverts with some custom error or assertion violation.
2. Or, the last non-zero address is basically of some other token (not alETH) and `router.exchange` does not revert, the following require statement will always revert:
```js
require((balAfter - balBefore) >= _minOut, "Slippage too high");
```
`balAfter` and `balBefore` represent the contract alETH balance, so if the output token is not alETH, the function will revert with misleading error, making it harder to find the root cause of error.

**Impact:**

**Proof of Concept:**

**Recommended Mitigation:** 



### [L-1] If a corrupted route is added, it cannot be fixed/removed.

**Description:** `StrategyMainnet::addRoute` function is intended to add a new route to the Curve router and update three mappings: `routes`, `swapParams` and `pools`. As `StrategyMainnet::addRoute` has no parameter check, the route parameters may be set incorrectly (e.g., one of 11 route address is wrong) and it will be impossible to delete or re-write this route due to `StrategyMainnet::addRoute` structure: a manager is unable to choose which mapping entry to change/populate, the index is set to `nRoutes` parameter which increments after each `addRoute` call and cannot be decremented or changed manually.

**Impact:** 

**Proof of Concept:**

**Recommended Mitigation:** 