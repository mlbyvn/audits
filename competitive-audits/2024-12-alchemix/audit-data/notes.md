Functionality:
1. Constructor:
    - Sets the transmutor, asset and underlying token (WETH)
    - Initializes the router
2. _initStrategy - used in constructor to initialize the router
3. addRoute:
    - Adds new route to be passed into Curve Router with swap parameters and pools
    - Callable only by the manager
4. _deployFunds:
    - Can deploy up to '_amount' of 'asset' in the yield source.
    - This function is called at the end of a {deposit} or {mint} call.
5. claimAndSwap:
    - onlyKeepers

Qs
1. Sandwich attack/MEV?  -> they are aware of sandwich
2. How do we check if alETH depegs? -> periphery not in scope
3. What if price oracle goes stale while alETH is depegged? -> periphery not in scope
4. What if alETH stays depegged for a long time? -> periphery not in scope
5. Break requires with forcing eth?
6. What happens if a router dies?
7. Any troubles using 0.8.18 now?
8. What if route params are wrong?
9. Are all params in place when calling functions from interfaces?

Notes 
1.`StrategyMainnet::useOracle` is never used
2. Slippage Check May Be Insufficient
```js
require((balAfter - balBefore) >= _minOut, "Slippage too high");
```
Issue: This slippage check assumes that balAfter - balBefore accurately represents the swap output. However, it does not account for tokens stuck due to failed swaps or incomplete approvals.
Mitigation: Ensure that all tokens involved in the swap are fully accounted for and verify balances of both input and output tokens.
3. Lack of Emergency Withdraw Mechanism

    Issue: There is no function to handle emergency withdrawals in case of a transmuter or router failure.
    Mitigation: Introduce a function (e.g., emergencyWithdraw) that allows the withdrawal of assets directly to the strategy owner or management in case of an emergency.
4. Incorrect Minimum Output in Swaps
```js
require(_minOut > _amountClaim, "minOut too low");
```
Issue: The _minOut > _amountClaim condition is unclear and does not necessarily ensure that the output is profitable. This check might fail to account for slippage accurately.
Mitigation: Use an external price oracle to compute _minOut based on the actual price impact, fees, and slippage.
5. Missing Input Validation in addRoute
```
routes[nRoutes] = _route;
swapParams[nRoutes] = _swapParams;
pools[nRoutes] = _pools;
```
Issue: The function addRoute does not validate the inputs _route, _swapParams, or _pools. Incorrect or malicious data could result in failed swaps or unintended consequences.
Mitigation: Add validation checks for:

    _route length and correctness (e.g., valid token addresses).
    _swapParams compatibility with the CurveRouterNG.
    _pools alignment with _route.
6. onlyKeepers Modifier Usage
```
function claimAndSwap(uint256 _amountClaim, uint256 _minOut, uint256 _routeNumber) external onlyKeepers {
```
Issue: The onlyKeepers modifier ensures only authorized accounts can call this function. However, if an authorized keeper account is compromised, they could manipulate _amountClaim and _minOut for profit.
Mitigation: Add safeguards to verify _amountClaim and _minOut against reasonable bounds or integrate external price oracles for validation.
7. Possible issues with approve: reentrancy if a router is malicious?
```
asset.safeApprove(address(transmuter), type(uint256).max);
underlying.safeApprove(address(router), type(uint256).max);
```
Issue: Using safeApprove with type(uint256).max can lead to reentrancy issues if transmuter or router are compromised. A malicious contract could manipulate approvals during the execution of these functions.
Mitigation: Consider using safeIncreaseAllowance or resetting approval to 0 before setting it again.