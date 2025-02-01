### [H-1]  `MathMasters::mulWadUp` function in imprecise in specific situations resulting in incorrect calculations of edge cases

**Description:** `mulWadUp` has a specific case: If x / y results in exactly 1, x is adjusted upward by 1. This results in imprecise rounding.

**Impact:** Imprecise rounding in edge cases.

**Proof of Concept:** 
Add following contract to the test. Required installed Halmos:

```js
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity ^0.8.3;

import {Base_Test, console2} from "./Base_Test.t.sol";
import {MathMasters} from "src/MathMasters.sol";

contract HalmosTest is Base_Test {
    // halmos --function symEx_testMulWadUp --solver-timeout-assertion 0
    function symEx_testMulWadUp(uint256 x, uint256 y) public {
        if (x == 0 || y == 0 || y <= type(uint256).max / x) {
            uint256 result = MathMasters.mulWadUp(x, y);
            uint256 expected = x * y == 0 ? 0 : (x * y - 1) / 1e18 + 1;
            assert(result == expected);
        }
    }
}
```

Or, alternatively, add the `certora` folder in your test suite, export the certora key with following command
```shell
export CERTORAKEY=*KEY*
```
and run 
```shell
certoraRun ./certora/MulWadUp.conf
```

**Recommended Mitigation:** 

Remove following line:

```diff
function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
    /// @solidity memory-safe-assembly
    assembly {
        // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
        if mul(y, gt(x, or(div(not(0), y), x))) {
            mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
            revert(0x1c, 0x04)
        }
-       if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
        z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
    }
}
```

### [H-2] `MathMasters::sqrt` incorrectly checks the lt of a right shift resulting in incorrect calculations of edge cases 

**Description:** The Babylonian method relies on a good initial estimate for fast convergence. The bit shifts help approximate the number of bits needed for x, but 0xffff2a introduces an error. Using 0xffff2a instead of 0xffffff changes the initial r calculation slightly, which can lead to potential precision issues in edge cases.

**Impact:** 
1. Possible imprecise calculations in edge cases.
2.  With a bad start, the correct # of bits increases ~linearly each iteration instead of ~quadratically.

**Proof of Concept:**

Fuzzing does not help to spot the issue even with high # of fuzz runs. Running symbolic execution tools to prove that `MathMasters::sqrt` is equivalent to `solmateSqrt` or `uniSqrt` fails due to [path explosion](https://docs.certora.com/en/latest/docs/user-guide/out-of-resources/timeout.html#path-explosion) problem (check run output in `timeout_certora_sqrt.txt`). 
The solution is the [modular verification](https://docs.certora.com/en/latest/docs/user-guide/out-of-resources/timeout.html#modular-verification): both functions are divided into modules, that are pairwise compared by certora. If devided in two parts, `MathMasters::sqrt` and `solmateSqrt` have exact same bottom parts, i.e., we need to compare only the top ones.

Quickstart:

1. Register by Certora
2. Get a personal key
3. Add the `certora` folder and the `HarnessCertora.sol`
4. Import personal key with `export CERTORAKEY=*KEY*` command
5. Run `certoraRun ./certora/Sqrt.conf`

Certora report shows, that for the value x = 0xffffffffffffffffffffff the top half of both `MathMasters::sqrt` and `solmateSqrt` functions produce different outputs (bottom halfs are same).  That means, that intermediate result, that is further fed to same subfunctions, is different, thus may potentially result in imprecise square root calculation.

```
*---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------*
| Rule name                           | Verified     | Time (sec) | Description | Local vars         |
| ----------------------------------- | ------------ | ---------- | ----------- | ------------------ |
| solmateSqrtModularized(uint256)     | Not violated | 0          |             | no local variables |
| mathMastersSqrtModularized(uint256) | Not violated | 0          |             | no local variables |
*---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------*
Results for all:
*---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------*
| Rule name                         | Verified     | Time (sec) | Description                                                 | Local vars                             |
| --------------------------------- | ------------ | ---------- | ----------------------------------------------------------- | -------------------------------------- |
| envfreeFuncsStaticCheck           | Not violated | 0          |                                                             | no local variables                     |
| testSqrtAgainstSolmateModularized | Violated     | 4          | Assert message: mathMastersSqrtModularized(x) == solmate... | HarnessCertora=HarnessCertora (0x2711) |
|                                   |              |            | - certora/Sqrt.spec line 29                                 | ecrecover=ecrecover (1)                |
|                                   |              |            |                                                             | identity=identity (4)                  |
|                                   |              |            |                                                             | sha256=sha256 (2)                      |
|                                   |              |            |                                                             | x=0xffffffffffffffffffffff (2^88 - 1)  |
*---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------*
```

Link to the report: https://prover.certora.com/output/4829620/cc45d38b85ae4bcea65abde2dc535e78?anonymousKey=664fd8b4a4dec7083dcdf3e5a896149345cac808&params=%7B%222%22%3A%7B%22index%22%3A0%2C%22ruleCounterExamples%22%3A%5B%7B%22name%22%3A%22rule_output_3.json%22%2C%22selectedRepresentation%22%3A%7B%22label%22%3A%22STR%22%2C%22value%22%3A0%7D%2C%22callResolutionSingleFilter%22%3A%22%22%2C%22variablesFilter%22%3A%22%22%2C%22callTraceFilter%22%3A%22%22%2C%22variablesOpenItems%22%3A%5Btrue%2Ctrue%5D%2C%22callTraceCollapsed%22%3Atrue%2C%22rightSidePanelCollapsed%22%3Afalse%2C%22rightSideTab%22%3A%22%22%2C%22callResolutionSingleCollapsed%22%3Atrue%2C%22viewStorage%22%3Atrue%2C%22expandedArray%22%3A%22%22%2C%22orderVars%22%3A%5B%22%22%2C%22%22%2C0%5D%2C%22orderParams%22%3A%5B%22%22%2C%22%22%2C0%5D%2C%22scrollNode%22%3A0%2C%22currentPoint%22%3A0%2C%22trackingChildren%22%3A%5B%5D%2C%22trackingParents%22%3A%5B%5D%2C%22trackingOnly%22%3Afalse%2C%22singleCallResolutionOpen%22%3A%5B%5D%7D%5D%7D%7D&generalState=%7B%22fileViewOpen%22%3Atrue%2C%22fileViewCollapsed%22%3Atrue%2C%22mainTreeViewCollapsed%22%3Atrue%2C%22callTraceClosed%22%3Afalse%2C%22mainSideNavItem%22%3A%22rules%22%2C%22globalResSelected%22%3Afalse%2C%22isSideBarCollapsed%22%3Afalse%2C%22isRightSideBarCollapsed%22%3Afalse%2C%22selectedFile%22%3A%7B%22uiID%22%3A%2216af9d%22%2C%22output%22%3A%22.certora_sources%2Fcertora%2FSqrt.spec%22%2C%22name%22%3A%22Sqrt.spec%22%7D%2C%22fileViewFilter%22%3A%22%22%2C%22mainTreeViewFilter%22%3A%22%22%2C%22contractsFilter%22%3A%22%22%2C%22globalCallResolutionFilter%22%3A%22%22%2C%22currentRuleUiId%22%3A2%2C%22counterExamplePos%22%3A1%2C%22expandedKeysState%22%3A%223-10-1-1-03-1-1-1-1-1-1-1%22%2C%22expandedFilesState%22%3A%5B%5D%2C%22outlinedfilterShared%22%3A%22000000000%22%7D

**Recommended Mitigation:** 

```diff
    function sqrt(uint256 x) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly {
            z := 181

            // This segment is to get a reasonable initial estimate for the Babylonian method. With a bad
            // start, the correct # of bits increases ~linearly each iteration instead of ~quadratically.
            let r := shl(7, lt(87112285931760246646623899502532662132735, x))
            r := or(r, shl(6, lt(4722366482869645213695, shr(r, x))))
            r := or(r, shl(5, lt(1099511627775, shr(r, x))))

-           r := or(r, shl(4, lt(16777002, shr(r, x))))
+           r := or(r, shl(4, lt(16777215, shr(r, x))))
            z := shl(shr(1, r), z)

            // There is no overflow risk here since `y < 2**136` after the first branch above.
            z := shr(18, mul(z, add(shr(r, x), 65536))) // A `mul()` is saved from starting `z` at 181.

            // Given the worst case multiplicative error of 2.84 above, 7 iterations should be enough.
            z := shr(1, add(z, div(x, z)))
            z := shr(1, add(z, div(x, z)))
            z := shr(1, add(z, div(x, z)))
            z := shr(1, add(z, div(x, z)))
            z := shr(1, add(z, div(x, z)))
            z := shr(1, add(z, div(x, z)))
            z := shr(1, add(z, div(x, z)))

            // If `x+1` is a perfect square, the Babylonian method cycles between
            // `floor(sqrt(x))` and `ceil(sqrt(x))`. This statement ensures we return floor.
            // See: https://en.wikipedia.org/wiki/Integer_square_root#Using_only_integer_division
            z := sub(z, lt(div(x, z), z))
        }
    }
```