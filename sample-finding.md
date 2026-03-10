# Sample Findings — Verifier Worked Examples

Three complete examples demonstrating verifier input and output format.

---

## Example 1 — VALID Critical (Reentrancy in Lending Protocol)

**Why this example is included**: Demonstrates a textbook valid finding with high confidence, showing the full exploit path and PoC sketch.

### Input (as a user would paste it)

**Finding**: Reentrancy in `withdraw()` allows draining of lending pool

**Description**: The `withdraw()` function in LendingPool.sol sends ETH to the user via a low-level `.call` before updating their balance. An attacker can deploy a contract with a `receive()` fallback that re-enters `withdraw()`, draining the pool since the balance check passes each time (balance hasn't been zeroed yet).

**Code**:
```solidity
// LendingPool.sol
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount, "Insufficient balance");

    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");

    balances[msg.sender] -= amount;
    totalDeposits -= amount;

    emit Withdrawal(msg.sender, amount);
}
```

### Verifier Output

```
---
VERIFIER REPORT
===============

Finding: Reentrancy in withdraw() allows draining of lending pool
Language: Solidity
Protocol Type: DeFi Lending
Scope Status: In Scope

VERDICT: Valid
Severity: Critical
Confidence: High

REASONING:
The withdraw() function violates the checks-effects-interactions pattern. The ETH transfer via .call on line 4 occurs before the balance update on line 7. Since .call forwards all available gas, the recipient contract's receive() function can re-enter withdraw(). On re-entry, balances[msg.sender] still holds the original value, so the require check on line 2 passes again. This allows repeated withdrawals until the pool is drained. There is no nonReentrant modifier on this function.

EXPLOITABILITY:
Preconditions:
- Attacker must have a non-zero balance in the lending pool (any amount, even 1 wei)
- Pool must hold ETH (i.e., other users have deposited)
Attack Path:
1. Attacker deposits minimal ETH into LendingPool
2. Attacker deploys a contract with receive() that calls withdraw() again
3. Attacker calls withdraw() from the malicious contract
4. On each re-entry, the balance check passes (balance not yet updated)
5. Loop continues until pool is drained or gas runs out
6. Final iteration: all balance updates execute in reverse order, but pool is already empty
Attacker Type: Permissionless

POC SKETCH:
contract Attacker {
    LendingPool pool;
    uint256 count;

    constructor(address _pool) { pool = LendingPool(_pool); }

    function attack() external payable {
        pool.deposit{value: msg.value}();
        pool.withdraw(msg.value);
    }

    receive() external payable {
        if (count < 10) {  // or until pool is drained
            count++;
            pool.withdraw(msg.value);  // re-enter
        }
    }
}

DEVIL'S ADVOCATE:
Arguments tested against verdict:
- Dimension 1 (Access Control): PASSED — withdraw() is external with no access control, permissionless
- Dimension 2 (Precondition Feasibility): PASSED — only needs any deposit, trivially achievable
- Dimension 3 (Mitigating Code): PASSED — no nonReentrant modifier, no global reentrancy lock
- Dimension 4 (Economic Infeasibility): PASSED — attacker profits massively (drains pool minus gas cost)
- Dimension 5 (Protocol Context Mismatch): PASSED — function behaves as described
- Dimension 6 (Timing): PASSED — single tx attack, no timing requirements
- Dimension 7 (Known Duplicate): PASSED — no indication this is accepted risk
- Dimension 8 (Missing Code Context): PASSED — function is complete as shown
Strongest counterargument: If there is a nonReentrant modifier applied at a higher level (e.g., a proxy wrapper or inherited contract) not shown in this snippet, the finding would be invalid. Based on the code provided, no such protection exists.
Counterargument held? No — verdict unchanged

RECOMMENDATION:
Apply the checks-effects-interactions pattern: move balances[msg.sender] -= amount and totalDeposits -= amount BEFORE the .call. Additionally, add a nonReentrant modifier (OpenZeppelin ReentrancyGuard) to withdraw() and all other functions that interact with balances.

REFERENCES:
- The DAO hack (2016) — same pattern
- OpenZeppelin ReentrancyGuard
- SWC-107: Reentrancy
---
```

---

## Example 2 — INVALID (False Positive: Overflow on Solidity 0.8.x)

**Why this example is included**: Demonstrates correct identification of a false positive — the most common type of incorrect finding from automated tools and AI agents.

### Input (as a user would paste it)

**Finding**: Integer overflow in `addReward()` allows minting unlimited reward tokens

**Description**: The `addReward()` function adds a user-supplied `amount` to `totalRewards` without overflow protection. An attacker can pass a very large value that causes `totalRewards` to overflow, wrapping to zero or a small number, then claim more rewards than the pool holds.

**Code**:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract RewardPool {
    mapping(address => uint256) public rewards;
    uint256 public totalRewards;

    function addReward(address user, uint256 amount) external onlyAdmin {
        rewards[user] += amount;
        totalRewards += amount;
    }

    function claimReward() external {
        uint256 reward = rewards[msg.sender];
        require(reward > 0, "No rewards");
        rewards[msg.sender] = 0;
        totalRewards -= reward;
        (bool success, ) = msg.sender.call{value: reward}("");
        require(success, "Transfer failed");
    }
}
```

### Verifier Output

```
---
VERIFIER REPORT
===============

Finding: Integer overflow in addReward() allows minting unlimited reward tokens
Language: Solidity
Protocol Type: Reward Distribution
Scope Status: In Scope

VERDICT: Invalid
Severity: N/A
Confidence: High

REASONING:
This is a false positive. The contract uses Solidity ^0.8.19, which has built-in overflow/underflow protection on all arithmetic operations by default. The additions in addReward() (rewards[user] += amount and totalRewards += amount) will revert with a panic if they overflow — there is no unchecked block wrapping these operations. An attacker cannot cause overflow; the transaction will simply revert. Additionally, addReward() is gated by onlyAdmin, so even if overflow were possible, only the admin could trigger it.

EXPLOITABILITY:
Preconditions: N/A — exploit is not possible
Attack Path: No realistic attack path exists. Solidity 0.8.x arithmetic reverts on overflow. The function is also admin-only.
Attacker Type: Not exploitable

POC SKETCH:
N/A — finding is not exploitable because Solidity ^0.8.19 automatically reverts on integer overflow. Any attempt to overflow totalRewards or rewards[user] will cause the transaction to revert with a Panic(0x11) error.

DEVIL'S ADVOCATE:
Arguments tested against verdict:
- Dimension 1 (Access Control): FAILED (for attacker) — onlyAdmin modifier blocks unprivileged callers
- Dimension 2 (Precondition Feasibility): FAILED — overflow impossible due to 0.8.x protection
- Dimension 3 (Mitigating Code): FAILED — Solidity compiler itself is the mitigation
- Dimension 5 (Protocol Context): finding assumes pre-0.8.x arithmetic behavior, which is incorrect
Strongest counterargument: If this code were compiled with solc < 0.8.0, or if the arithmetic were inside an unchecked block, the overflow would be real. A future refactor that adds unchecked for gas optimization could re-introduce this vulnerability.
Counterargument held? No — verdict unchanged. The code as written is safe.

RECOMMENDATION:
No code change needed. The finding is invalid. For defense in depth, the team could add an explicit require(totalRewards + amount >= totalRewards) but this is redundant with Solidity 0.8.x behavior and wastes gas.

REFERENCES:
- Solidity 0.8.0 release notes: automatic overflow/underflow checks
- SWC-101: Integer Overflow and Underflow (not applicable to 0.8.x without unchecked)
---
```

---

## Example 3 — NEEDS MORE CONTEXT (Incomplete Cross-Contract Interaction)

**Why this example is included**: Demonstrates the correct response when the code snippet is insufficient for verification — verifier asks for specific missing information rather than guessing.

### Input (as a user would paste it)

**Finding**: Price oracle manipulation allows undercollateralized borrowing

**Description**: The `borrow()` function in Vault.sol reads the collateral price from `oracle.getPrice()`. If the oracle returns a spot price from an AMM, an attacker could flashloan to manipulate the pool, inflate their collateral value, borrow more than they should, and leave the vault with bad debt.

**Code**:
```solidity
function borrow(uint256 borrowAmount) external {
    uint256 collateralValue = getCollateralValue(msg.sender);
    require(collateralValue >= borrowAmount * MIN_COLLATERAL_RATIO / 1e18, "Undercollateralized");

    borrows[msg.sender] += borrowAmount;
    IERC20(borrowToken).transfer(msg.sender, borrowAmount);
}

function getCollateralValue(address user) public view returns (uint256) {
    uint256 price = oracle.getPrice(collateralToken);
    return collateralBalances[user] * price / 1e18;
}
```

### Verifier Output

```
---
VERIFIER REPORT
===============

Finding: Price oracle manipulation allows undercollateralized borrowing
Language: Solidity
Protocol Type: DeFi Lending
Scope Status: Unable to Determine

VERDICT: Needs More Context
Severity: Potentially Critical (pending oracle implementation review)
Confidence: Low

REASONING:
The finding's validity depends entirely on the implementation of oracle.getPrice(). If the oracle reads a spot AMM price (e.g., UniswapV2 reserve ratio), this is a valid Critical finding — flashloan manipulation is straightforward. If the oracle uses Chainlink price feeds with proper staleness checks, or a TWAP with a sufficiently long window (>30 minutes), the finding is likely invalid. The borrow() function itself has no manipulation protection beyond the collateral ratio check, so the oracle is the sole defense. Without seeing the oracle contract, a definitive verdict is impossible.

EXPLOITABILITY:
Preconditions:
- Oracle must read a manipulable price source (spot AMM price)
- Attacker needs access to a flashloan pool for the collateral token
- Vault must hold sufficient borrowToken for the borrow
Attack Path (IF oracle is manipulable):
1. Attacker flashloans large amount of collateral token
2. Attacker swaps into AMM pool, inflating collateral token price
3. Attacker calls borrow() — getCollateralValue returns inflated value
4. Attacker borrows maximum amount against inflated collateral
5. Attacker swaps back, deflating price
6. Attacker repays flashloan, keeps borrowed tokens
7. Vault is left with bad debt (collateral is now worth less than borrow)
Attacker Type: Permissionless (if oracle is manipulable)

POC SKETCH:
N/A — cannot construct PoC without oracle implementation. If oracle reads spot price, PoC follows the standard flashloan oracle manipulation template above.

DEVIL'S ADVOCATE:
Arguments tested against verdict:
- Dimension 3 (Mitigating Code): INCONCLUSIVE — oracle may have TWAP, Chainlink feed, or other manipulation resistance
- Dimension 8 (Missing Code Context): FAILED — oracle implementation is critical and missing
- Dimension 4 (Economic Feasibility): INCONCLUSIVE — depends on flashloan availability and pool depth
Strongest counterargument: The oracle likely uses Chainlink or a TWAP — most production protocols don't use raw spot prices. If so, this finding is invalid.
Counterargument held? Cannot determine without oracle code.

RECOMMENDATION:
Provide the oracle contract implementation (specifically the getPrice() function) for definitive verification. If it reads spot AMM prices, replace with Chainlink feed or implement a TWAP with >=30 minute window. Additionally, consider adding a price deviation check (revert if price changed >X% from last known good price within one block).

MISSING CONTEXT NEEDED:
1. Oracle contract source code — specifically the getPrice() function implementation
2. What price source does the oracle use? (Chainlink, TWAP, spot AMM, custom)
3. Are there any price deviation circuit breakers in the oracle or vault?
4. What is the oracle contract address / name so it can be reviewed?

REFERENCES:
- Euler Finance exploit (March 2023) — flash loan + oracle manipulation
- Mango Markets exploit (October 2022) — oracle manipulation for undercollateralized borrowing
- Chainlink Price Feed documentation
---
```
