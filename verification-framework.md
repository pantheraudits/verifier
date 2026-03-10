# Verification Framework — Verifier Phase 2 Reference

Core verification logic. For each vulnerability class: trigger condition, verification steps, common false positives, and exploit path template.

---

## SOLIDITY-APPLICABLE

### 1. Reentrancy

**Subtypes**: Classic (same-function), cross-function, read-only, cross-contract.

**Trigger condition**: Finding mentions reentrancy, or code shows an external call before a state update (checks-effects-interactions violation).

**Verification steps**:
1. Identify the external call (`.call`, `.transfer`, `.send`, interface call, token transfer with callback like ERC-721 `safeTransferFrom`, ERC-777 hooks).
2. Identify what state is read AFTER the external call that was set BEFORE the call.
3. Check: is there a `nonReentrant` modifier on the function? If yes, classic reentrancy is blocked.
4. Check: can the attacker re-enter through a DIFFERENT function that reads the same state? (cross-function reentrancy — `nonReentrant` on one function doesn't protect the other unless they share the same lock).
5. Check: is the re-entered function view/pure? If yes, it's read-only reentrancy. This only matters if another contract reads this stale view in the same tx (e.g., a price oracle reading a pool's reserves mid-swap).
6. Check: does the protocol use a global reentrancy lock across all state-modifying functions?
7. Trace the exact state that becomes inconsistent and quantify the impact.

**Common false positives**:
- External call exists but all state updates happen BEFORE the call (correct CEI pattern). Not vulnerable.
- `nonReentrant` modifier is applied and covers all entry points to the shared state. Not exploitable.
- The external call goes to a TRUSTED contract (e.g., WETH, a whitelisted token). Unless the finding shows the trusted contract can be replaced.
- Read-only reentrancy is reported but no other contract reads the stale value in the same tx.

**Exploit path template**:
1. Attacker calls vulnerable function.
2. Function makes external call to attacker's contract (callback).
3. Attacker's fallback/receive re-enters the same or different function.
4. Re-entered function reads stale state (pre-update values).
5. State is manipulated: double-withdraw, incorrect balance calculation, etc.

---

### 2. Integer Overflow / Underflow

**Trigger condition**: Finding mentions overflow/underflow, or code performs arithmetic on user-controlled inputs.

**Verification steps**:
1. **Check Solidity version**. >= 0.8.0: arithmetic reverts on overflow by default.
2. If >= 0.8.0: is the vulnerable arithmetic inside an `unchecked {}` block? If not, it's a false positive.
3. If < 0.8.0 or inside `unchecked`: does the code use SafeMath or equivalent checks?
4. Can the attacker control the input values to trigger overflow?
5. What is the impact of the overflow? (e.g., balance wraps to max uint, check passes when it shouldn't)
6. For underflow: is the subtraction order correct? Can an attacker make `a - b` underflow by controlling `b > a`?

**Common false positives**:
- Solidity >= 0.8.0 without `unchecked` — the compiler catches it. Most common false positive.
- Overflow exists but the overflowed value is never used in a security-relevant check.
- The input is bounded by other checks (e.g., `require(amount <= balance)`) that prevent the overflow from being reachable.

**Exploit path template**:
1. Attacker provides input that causes arithmetic to overflow/underflow.
2. Overflowed value passes a check it shouldn't (e.g., `balance - withdrawal` underflows to max uint, passing `require(balance >= withdrawal)`).
3. Attacker extracts funds or manipulates state based on the incorrect value.

---

### 3. Access Control Bypass

**Trigger condition**: Finding claims a restricted function can be called by unauthorized users.

**Verification steps**:
1. Identify the access control mechanism: modifier, require statement, if-revert, role-based (AccessControl), ownership (Ownable).
2. Trace ALL paths to the vulnerable function: direct external call, internal call from another external function, delegatecall from proxy.
3. Check: is the modifier applied to all external/public entry points? Missing on one entry point = bypass.
4. Check: can the role be obtained by the attacker? (e.g., anyone can call `grantRole` if the admin role is misconfigured).
5. Check: initializer functions — can the attacker call `initialize()` on an uninitialized proxy to become owner?
6. Check: `msg.sender` vs `tx.origin` — is the check on the right value?

**Common false positives**:
- Finding reports missing access control on a function that is `internal` or `private` — can't be called externally.
- The function is behind a proxy that enforces its own auth before delegating.
- The "unrestricted" function only modifies the caller's own state (self-serve pattern, by design).

**Exploit path template**:
1. Attacker identifies a privileged function missing access control or with a bypassable check.
2. Attacker calls the function directly (or through an unprotected entry point).
3. Attacker gains elevated privileges or performs unauthorized state changes.

---

### 4. Oracle Manipulation / Price Manipulation

**Trigger condition**: Finding involves token price, exchange rate, collateral value, or any value derived from on-chain state that can be manipulated in the same tx.

**Verification steps**:
1. Identify the price source: Chainlink feed, TWAP oracle, spot AMM price, internal exchange rate.
2. If spot AMM price (e.g., `reserve0/reserve1`): this is manipulable via flashloan. Check if the protocol uses this for anything security-relevant (liquidation, collateral valuation, swap execution).
3. If TWAP: what's the observation window? Short TWAP (<30 min) is still manipulable with sustained capital. Long TWAP (>24h) is generally safe.
4. If Chainlink: check for stale price (missing `updatedAt` check), L2 sequencer uptime check, heartbeat validation.
5. Quantify: how much capital is needed to manipulate the price, and what's the profit from the manipulation?
6. Check: does the protocol have price deviation checks, circuit breakers, or multi-oracle comparison?

**Common false positives**:
- Oracle uses TWAP with a long window — not manipulable in practice.
- Chainlink oracle with proper staleness checks — not manipulable unless Chainlink itself fails (accepted risk).
- Price manipulation is possible but requires capital exceeding the protocol's TVL — economically infeasible.
- The price is only used for display/events, not for any state-changing logic.

**Exploit path template**:
1. Attacker takes flashloan for large capital.
2. Attacker manipulates AMM reserves (swap large amount, skewing the ratio).
3. In the same tx, attacker calls the vulnerable protocol that reads the manipulated price.
4. Protocol executes based on incorrect price (over-values attacker's collateral, under-values liquidation threshold).
5. Attacker profits from the mispriced operation.
6. Attacker repays flashloan.

---

### 5. Flash Loan Attack Surface

**Trigger condition**: Finding involves flash loans, or the vulnerability becomes exploitable only with large capital that could be flash-borrowed.

**Verification steps**:
1. Does the exploit require large capital? If yes, can it be flash-borrowed? (Almost always yes — Aave, dYdX, Balancer, Uniswap V3 all offer flash loans.)
2. Can the entire exploit execute atomically in one transaction? Flash loans must be repaid in the same tx.
3. Does the protocol have any flash loan protection? (e.g., `require(block.number > lastDepositBlock)` to prevent same-block deposit+exploit)
4. Is the finding actually about flash loans, or is the flash loan just the capital source for a different underlying vulnerability? (Usually the latter — flash loans amplify other bugs.)

**Common false positives**:
- Finding says "flash loan attack" but the underlying vulnerability doesn't exist — the flash loan just provides capital for an attack that isn't actually possible.
- Protocol has same-block protection that prevents flash loan exploitation.

**Exploit path template**:
(Flash loans are usually an amplifier, not the bug itself. The template is: flash loan → other vulnerability → repay.)

---

### 6. Front-Running / MEV

**Trigger condition**: Finding involves transaction ordering, sandwich attacks, or profit extraction through tx sequencing.

**Verification steps**:
1. What value is extractable through front-running? (Token swap slippage, liquidation bonus, arbitrage)
2. Is the victim a user or the protocol?
3. Does the protocol have slippage protection? (`minAmountOut` parameter)
4. Is the front-running a design-level issue (inherent to AMMs) or a code-level bug (missing protection)?
5. Is this MEV that validators extract (accepted reality) or a code bug that enables unusual MEV?

**Common false positives**:
- Reporting sandwich attacks on AMM swaps as a bug — this is inherent to AMM design, not a code vulnerability.
- Slippage protection exists but the finding ignores it.
- The "front-runnable" function has no MEV value (nothing to extract from ordering).

**Exploit path template**:
1. Attacker monitors mempool for victim tx.
2. Attacker submits tx before victim (front-run) to move price.
3. Victim's tx executes at worse price.
4. Attacker submits tx after victim (back-run) to capture the profit.

---

### 7. Storage Collision (Proxy Patterns)

**Trigger condition**: Finding involves proxy contracts, `delegatecall`, upgradeable patterns, or mentions storage slot collisions.

**Verification steps**:
1. What proxy pattern is used? Transparent, UUPS, Beacon, Diamond?
2. Check: do the proxy and implementation share storage slot 0, 1, etc.? If yes, is there a collision?
3. For UUPS: is the `_authorizeUpgrade` function properly access-controlled?
4. For Transparent: is the admin slot in the correct EIP-1967 location?
5. Check: unstructured storage (EIP-1967) vs structured storage — is the gap (`__gap`) maintained for upgrades?
6. Check: does the new implementation add storage variables in the correct position?

**Common false positives**:
- EIP-1967 compliant proxy with correct slot usage — no collision.
- Finding assumes naive storage layout but the proxy uses unstructured storage.
- Storage gap is present and correctly sized.

**Exploit path template**:
1. Implementation contract writes to a storage slot that overlaps with the proxy's admin/implementation slot.
2. Attacker triggers the write to overwrite the admin or implementation address.
3. Attacker upgrades the proxy to a malicious implementation.

---

### 8. Signature Replay / Missing Nonce

**Trigger condition**: Finding involves EIP-712, `ecrecover`, permit, gasless transactions, or off-chain signatures.

**Verification steps**:
1. Is there a nonce that increments after each signature use? Without it, the same signature can be replayed.
2. Is the chain ID included in the signed data? Without it, signature is replayable across chains.
3. Is there a deadline/expiry? Without it, old signatures remain valid forever.
4. Does `ecrecover` check for `address(0)`? If signature is invalid, `ecrecover` returns `address(0)` which might pass a check.
5. Is the domain separator correct (contract address included)?
6. For permit: is the permit nonce separate from the transfer nonce?

**Common false positives**:
- Nonce is present but the finding missed it.
- The function can only be called once per state anyway (e.g., one-time claim), making replay harmless.
- Chain ID and deadline are included in the EIP-712 domain.

**Exploit path template**:
1. Attacker obtains a valid signature (from public tx, leaked, or previously used).
2. Attacker replays the signature to execute the same action again (double-withdraw, double-approve, cross-chain replay).

---

### 9. Improper Input Validation

**Trigger condition**: Finding claims user input is not validated, leading to unexpected behavior.

**Verification steps**:
1. What input is unvalidated? (Amount, address, array length, bytes data)
2. What happens with extreme values? (0, max uint, empty array, zero address)
3. Does the unvalidated input reach a security-relevant operation? (Transfer, state update, external call)
4. Is validation done elsewhere in the call chain? (Upstream function, modifier, library)

**Common false positives**:
- Input is validated by the calling function before reaching the internal function.
- Zero-address check is missing but the function is admin-only (admin can be trusted to pass correct values).
- The "invalid" input doesn't lead to any harmful outcome.

---

### 10. Rounding Errors / Precision Loss

**Trigger condition**: Finding involves division, percentage calculation, exchange rate, shares-to-assets conversion, or any financial math.

**Verification steps**:
1. Identify where division occurs. In Solidity, integer division truncates (rounds down).
2. Who benefits from the rounding direction? (Protocol or user? Depositor or withdrawer?)
3. Can an attacker exploit the rounding? (e.g., deposit 1 wei, get rounded up to 1 share, withdraw more than deposited)
4. Does the rounding compound? (1 wei per tx × millions of txs = significant loss)
5. Check: is there a minimum deposit/withdrawal amount that prevents dust attacks?
6. ERC-4626 vaults: check the share-to-asset conversion for the first depositor attack (inflation attack).

**Common false positives**:
- Rounding exists but always rounds in the protocol's favor (by design — this is correct).
- Rounding loss is < 1 wei per operation and doesn't compound.
- Minimum deposit amount prevents the dust manipulation.

**Exploit path template (ERC-4626 inflation attack)**:
1. Attacker is first depositor. Deposits 1 wei, gets 1 share.
2. Attacker donates large amount to vault (direct transfer, skipping deposit).
3. Now 1 share = large amount. Exchange rate is inflated.
4. Victim deposits. Due to rounding, victim's deposit rounds down to 0 shares.
5. Attacker withdraws, capturing victim's deposit.

---

### 11. Denial of Service

**Trigger condition**: Finding claims a function can be made to revert permanently or become too expensive to call.

**Verification steps**:
1. Identify the DoS vector: unbounded loop, push to unbounded array, external call that can revert, block gas limit.
2. Can an attacker grow the array/loop indefinitely? Or is it bounded?
3. Does the function have a gas-intensive fallback if an external call fails?
4. Is the DoS permanent (no recovery) or temporary (admin can fix)?
5. What function is DoS'd? Critical (withdraw, liquidate) or non-critical (view function)?

**Common false positives**:
- The array is bounded by a reasonable constant (e.g., max 100 elements).
- The function is not user-facing and only called by admin/keeper.
- External call failure is handled with try/catch or doesn't block the main logic.

---

## MOVE-APPLICABLE

### 12. Missing Signer / Capability Checks

**Trigger condition**: Finding claims a function that should require authorization doesn't check for signer or capability.

**Verification steps**:
1. Does the function take a `signer` or `&signer` parameter? If not, any account can call it.
2. If it takes a signer, does it use the signer meaningfully (e.g., `signer::address_of(account)` to check authorization)?
3. Is there a capability resource that should be checked? (`assert!(exists<AdminCap>(addr))`)
4. Can the function be called through a public entry point that doesn't require signer?

**Common false positives**:
- Function is `public(friend)` and only callable by trusted modules.
- Function is `fun` (not `public`) — only callable within the module.
- Signer check is done in the calling function.

---

### 13. Resource Leak / Double-Spend

**Trigger condition**: Finding claims a resource can be duplicated or lost.

**Verification steps**:
1. Move's type system prevents copy/drop of resources without the ability. Is the resource type marked with `copy` or `drop`?
2. If the finding claims a logic-level double-spend: can the same resource be used in two different operations through a sequence of calls?
3. Check: are resources properly destroyed or stored after use?

**Common false positives**:
- Move's type system already prevents this at compile time for resources without `copy` ability.
- The "double-spend" is actually two separate resources being used once each.

---

### 14. Phantom Type Confusion

**Trigger condition**: Finding involves generic types, phantom type parameters, or type-based access control in Move.

**Verification steps**:
1. Are phantom type parameters used for access control (e.g., `Pool<CoinA, CoinB>`)?
2. Can an attacker create a type that satisfies the phantom constraint but represents something different?
3. Is there on-chain validation of the actual coin type beyond the generic parameter?

---

### 15. Shared Object Race Conditions (Sui)

**Trigger condition**: Finding involves concurrent access to a shared object on Sui.

**Verification steps**:
1. Is the object shared (`transfer::share_object`)? Owned objects are not subject to race conditions.
2. Can concurrent transactions create inconsistent state? (Sui's consensus orders txs, but the resulting state depends on the order.)
3. Is there a time-of-check-time-of-use (TOCTOU) issue between reading state and modifying it?

---

### 16. Missing Abort in Error Paths (Aptos)

**Trigger condition**: Finding claims an error condition is not handled, allowing execution to continue with invalid state.

**Verification steps**:
1. Does the error path use `abort` or `assert!` to halt execution?
2. If the error is handled with a return value, does the caller check it?
3. What happens if execution continues past the error — what state is corrupted?

---

## RUST / SEALEVEL-APPLICABLE

### 17. Missing Account Ownership Checks

**Trigger condition**: Finding claims an account passed to a Solana instruction isn't validated for ownership.

**Verification steps**:
1. Does the program check `account.owner == expected_program_id`?
2. In Anchor: is the account type constraint properly set? (`#[account(owner = program_id)]`)
3. Can an attacker pass an account they own with crafted data to bypass logic?

---

### 18. Missing Signer Checks

**Trigger condition**: Finding claims a Solana instruction doesn't verify that a required account has signed.

**Verification steps**:
1. Does the program check `account.is_signer`?
2. In Anchor: is the `Signer` type or `#[account(signer)]` constraint used?
3. What action does the unsigned account authorize? (Transfer, state change, admin operation)

---

### 19. Integer Overflow (Rust no_std)

**Trigger condition**: Arithmetic on Solana programs compiled with release mode (overflow wraps silently in release).

**Verification steps**:
1. Is the program compiled in release mode? (Default for Solana programs — overflow wraps.)
2. Does the code use `checked_add`, `checked_sub`, `checked_mul`? Or raw operators?
3. Can the attacker control inputs to cause overflow?
4. In Anchor: does the program use `#[cfg(not(feature = "no-overflow-checks"))]`?

---

### 20. Arbitrary CPI

**Trigger condition**: Finding claims a program makes a CPI to an attacker-controlled program.

**Verification steps**:
1. Is the target program ID hardcoded or passed as an account?
2. If passed as an account: is the program ID validated against expected value?
3. Can the attacker substitute a malicious program that returns success but performs different actions?

---

### 21. Account Data Reuse / Missing Discriminator

**Trigger condition**: Finding claims an account of one type can be passed where another type is expected.

**Verification steps**:
1. Does the program check a discriminator field in the account data?
2. In Anchor: is the 8-byte discriminator automatically checked? (It is for `#[account]` types.)
3. For raw (non-Anchor) programs: is there a manual type check?
4. Can an attacker create an account that matches the expected data layout but has different semantics?

---

## CROSS-LANGUAGE

### 22. Business Logic Bug

**Trigger condition**: Finding claims the protocol's business logic doesn't match its intended behavior. Not a technical vulnerability class — a semantic one.

**Verification steps**:
1. What is the intended behavior? (Check docs, comments, function names, protocol description.)
2. What does the code actually do? (Trace execution step by step.)
3. Where do intention and implementation diverge?
4. Is the divergence harmful? (Some divergences are harmless or even beneficial.)
5. Who is harmed by the divergence? (Protocol, specific users, all users?)

**Common false positives**:
- The auditor misunderstands the intended behavior. The code is correct.
- The divergence exists but has no impact on any user or the protocol.
- The behavior is intentional but undocumented (ask the team).

---

### 23. Missing Event Emission

**Trigger condition**: Finding claims a state-changing function doesn't emit an event.

**Verification steps**:
1. Is the function state-changing? View/pure functions don't need events.
2. Does an off-chain system depend on this event? (Indexer, frontend, monitoring)
3. Is this a compliance requirement? (Some standards require specific events.)

**Severity**: Almost always Informational. Elevate to Low/Medium only if a critical off-chain system depends on the missing event.

---

### 24. Centralization Risk

**Trigger condition**: Finding claims an admin/owner has too much power.

**Verification steps**:
1. What can the admin do? (Pause, upgrade, set fees, withdraw, change parameters)
2. Is there a timelock? (Reduces severity — users can exit before admin action takes effect.)
3. Is the admin a multisig? (Reduces likelihood of malicious action but doesn't eliminate it.)
4. Can the admin directly steal user funds? (If yes: this is a real finding. If no: informational.)
5. Does the protocol documentation declare the trust assumption?

**Severity**: Informational if documented trust assumption with timelock. Medium if admin can cause significant harm. High if admin can directly drain funds with no timelock.

---

### 25. Economic Attack (Incentive Misalignment)

**Trigger condition**: Finding describes an economic exploit that doesn't require a code bug — just rational economic behavior that harms the protocol.

**Verification steps**:
1. Is this a code bug or a design/incentive issue?
2. Is the attack profitable for the attacker?
3. Does the attack require irrational behavior from other participants?
4. Would this be caught by economic modeling / game theory review rather than code audit?

**Common false positives**:
- "Attacker can dump tokens to lower price" — this is market activity, not a vulnerability.
- "Whales can dominate governance" — this is a design tradeoff, not a bug.
- Economic attacks that require collusion among many actors.

**Severity**: Usually Medium at most. These are design issues, not code bugs. Only escalate if the economic attack can be executed atomically (flashloan + governance).
