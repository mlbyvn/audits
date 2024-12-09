## High

### [H-1] `TSwapPool::getInputAmountBasedOnOutput` uses wrong values for the fee calculation resulting in lost fees

**Description:** `getInputAmountBasedOnOutput` scales the amount of output tokens by 10_000 instead of 1_000.

**Impact:** Protocol takes a lot more fees than expected.

**Recommended Mitigation:** 
```diff
    function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
-       return ((inputReserves * outputAmount) * 10000) / ((outputReserves - outputAmount) * 997);
+       return ((inputReserves * outputAmount) * 1000) / ((outputReserves - outputAmount) * 997);
    }
```

### [H-2] No slippage protection in `TSwapPool::swapExactOutput`, bad exchange rate transactions may be validated 

**Description:** `swapExactOutput` calculates how much input tokens are needed to get the exact output token amount. If the exchange rate changes drastically before the `swapExactOutput` transaction gets validated, a user may need to pay a lot more input tokens than expected. There is also a possibility of MEV even with the deadline check: if the deadline parameter is set too large, a malicious node operator will be able to front-run the transaction and force the changes in exchange rate, which will result in user's money loss.

**Impact:** The user could get a much worse swap with potential money loss.

**Recommended Mitigation:** 
```diff
function swapExactOutput(
        IERC20 inputToken,
        IERC20 outputToken,
        uint256 outputAmount,
        uint64 deadline
+       uint256 maxInput
    )
        public
        revertIfZero(outputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 inputAmount)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        inputAmount = getInputAmountBasedOnOutput(outputAmount, inputReserves, outputReserves);

+       if (maxInput < inputAmout){
+           revert();
+       }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }

```

### [H-3] `TSwapPool::sellPoolTokens` calls the wrong function for selling pool tokens, resulting in incorrect token amount received.

**Description:** According to the documentation, `sellPoolTokens` is a "wrapper function to facilitate users selling pool tokens in exchange for WETH". The `poolTokenAmount` is a parameter that specifies the amount of pool tokens to sell, thus `swapExactInput` must be used instead of `swapExactOutput`.

**Impact:** User sells incorrect amount of pool tokens.

**Recommended Mitigation:** Add following changes in the code:
```diff
    function sellPoolTokens(uint256 poolTokenAmount) external returns (
        uint256 wethAmount,
+       uint256 minWethToReceive) {
-       return swapExactOutput(i_poolToken, i_wethToken, poolTokenAmount, uint64(block.timestamp));
+       return swapExactInput(i_poolToken, poolTokenAmount, i_wethToken, minWethToReceive, uint64(block.timestamp));
    }
```

### [H-4] `TSwapPool::_swap` breaks the protocol invariant

**Description:** The AMM uses a constant formula to calculate the ratio between two token amounts: x * y = k, where x - the balance of pool token and y - the weth balance. Whenever there is a balance change in one of the pools, the ratio between two amounts should be the same. But "extra incentive" in `TSwapPool::_swap` breaks this invariant
**Impact:** Due to the extra token rewards the protocol funds will be drained after some time.

**Proof of Concept:** The following function breaks the invariant
```js
function _swap(IERC20 inputToken, uint256 inputAmount, IERC20 outputToken, uint256 outputAmount) private {
        if (_isUnknown(inputToken) || _isUnknown(outputToken) || inputToken == outputToken) {
            revert TSwapPool__InvalidToken();
        }

        swap_count++;
        if (swap_count >= SWAP_COUNT_MAX) {
            swap_count = 0;
            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
        }
        emit Swap(msg.sender, inputToken, inputAmount, outputToken, outputAmount);

        inputToken.safeTransferFrom(msg.sender, address(this), inputAmount);
        outputToken.safeTransfer(msg.sender, outputAmount);
    }

    function _isUnknown(IERC20 token) private view returns (bool) {
        if (token != i_wethToken && token != i_poolToken) {
            return true;
        }
        return false;
    }
```
<details>
<summary>Proof Of Code</summary>
Add `invariant` and `mocks` folders into your test folder and run invariant tests.
</details>

**Recommended Mitigation:** Remove the "extra incentive" mechanics:
```diff
-       if (swap_count >= SWAP_COUNT_MAX) {
-           swap_count = 0;
-           outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
-       }
``` 
It is also recommended to follow the [FREI-PI](https://www.nascent.xyz/idea/youre-writing-require-statements-wrong) and integrate invariant checks direct in constracts.

## Medium

### [M-1] Absence of deadline check in `TSwapPool::deposit` allows transaction completion even after deadline

**Description:** The uint64 parameter `deadline` is according to the documentation "the deadline for the transaction to be completed by", but is never used or checked. This opens a door for an MEV or [sandwich attacks](https://support.uniswap.org/hc/en-us/articles/19387081481741-What-is-a-sandwich-attack).

**Impact:** A malicious node operator is a liquidity provider for the TSwap, he would be able to see pending deposits into the protocol, the practical effect of which would be that their shares and fees are lowered as new `LPTokens` are minted. The node operator would have the power to delay the processing of this `deposit` transaction in favor of validating more swap transactions maximizing the fees they would obtain from the protocol at the expense of the new depositer.

**Proof of Concept:** `deadline` check missing.

**Recommended Mitigation:** Consider using the `TSwapPool::revertIfDeadlinePassed` modifier:
```diff
function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        // @audit Deadline not being used
        uint64 deadline
    )
        external
        revertIfZero(wethToDeposit)
+       revertIfDeadlinePassed(deadline)
        returns (uint256 liquidityTokensToMint)
    {...}
```

### [M-2] Wierd ERC20 tokens may break the protocol invariant

**Description:** There is no whitelist of allowed tokens to create pool with. Some token implementations may break the protocol invariant or open new attack vectors.
1. Fee-on-transfer tokens will break the invariant as in the discussed issue with `TSwapPool::_swap`
2. ERC777 allow reentrant calls on transfer resulting in possibility for an attacker to steal the liquidity pool. The same issue was [reported in the audit for Uniswap V1](https://github.com/Consensys/Uniswap-audit-report-2018-12?tab=readme-ov-file#31-liquidity-pool-can-be-stolen-in-some-tokens-eg-erc-777-29).
3. Upgradable tokens (as USDC for example) are upgradable, allowing the token owners to make arbitrary modifications to the logic of the token at any point in time.
Other potential dangers of wierd ERC20 tokens are listed [here](https://github.com/d-xo/weird-erc20?tab=readme-ov-file). 

**Recommended Mitigation:** Get acquainted with token integration checklist by the [Trail of Bits](https://github.com/crytic/building-secure-contracts/blob/master/development-guidelines/token_integration.md) or [Consensys](https://diligence.consensys.io/blog/2020/11/token-interaction-checklist/).

## Low

### [L-1] `TSwapPool::_addLiquidityMintAndTransfer` emits an event with parameters in wrong order

**Description:** `TSwapPool::_addLiquidityMintAndTransfer` emits an event `LiquidityAdded` with values logged in an incorrect order.

**Impact:** Event emission is incorrect, leading to off-chain functions potentially malfunctioning.

**Recommended Mitigation:** 
```diff
- emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit);
+ emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit);
```

## Informational

### [I-1] Unused error `PoolFactory::PoolFactory__PoolDoesNotExist`

**Description:** The error `PoolFactory__PoolDoesNotExist` is not used and should be removed.

```diff
-    error PoolFactory__PoolDoesNotExist(address tokenAddress);
```

### [I-2] Lacking zero-address checks in `PoolFactory::constructor` and `TSwapPool::constructor`

**Description:** Constructors lack zero-address checks. Reference: https://github.com/crytic/slither/wiki/Detector-Documentation#missing-zero-address-validation

```diff
    constructor(address wethToken) {
+       if (wethToken == address(0)) {
+           revert
+       }
        i_wethToken = wethToken;
    }
```

```diff
    constructor(
        address poolToken,
        address wethToken,
        string memory liquidityTokenName,
        string memory liquidityTokenSymbol
    )
        ERC20(liquidityTokenName, liquidityTokenSymbol)
    {
+       if (wethToken || poolToken == address(0)) {
+           revert();
+       }
        i_wethToken = IERC20(wethToken);
        i_poolToken = IERC20(poolToken);
    }
```

### [I-3] `PoolFactory::createPool` uses the wrong IERC20 function to get the token symbol

**Description:** IERC20 has a distinct function that returns the symbol of the token. 

```diff
-       string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
+       string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol());
```

### [I-4] `TSwapPool::getOutputAmountBasedOnInput` and `TSwapPool::getInputAmountBasedOnOutput` use "magic" numbers

**Description:** In order to make the code more readable and avoid typos it is suggested to use declared constants instead of "magic" numbers.

**Impact:** Usage of "magic" numbers leads to a huge business logic failure in `TSwapPool::getInputAmountBasedOnOutput`, that was already covered in "High" section of this report. 

**Recommended Mitigation:** Use constants for values that are used to calculate fees:
```diff
+   uint256 public constant FEE_CONST = 997;
+   uint256 public constant FEES_PRECISION = 1000; 
```

### [I-5] Missing natspec

**Description:** Functions `getOutputAmountBasedOnInput` `getInputAmountBasedOnOutput` and `swapExactInput` have no natspec.  

**Recommended Mitigation:** Add missing documentation.