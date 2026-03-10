# Language Context — Verifier Reference

Loaded on demand based on auto-detected language from the code snippet. Each section defines the security mental model for that language's execution environment.

---

## 1. Solidity / EVM

### Execution Model
- **Call stack**: External calls transfer execution to the callee. The callee can call back (reentrancy). Internal calls stay in the same execution context.
- **msg.sender**: The immediate caller. Changes on external calls. Does NOT change on internal calls or delegatecalls.
- **tx.origin**: The original EOA that initiated the transaction. Never changes. Using it for auth is a known antipattern.
- **delegatecall**: Executes callee code in the context of the caller (caller's storage, caller's msg.sender). This is how proxies work and how storage collisions happen.
- **State changes are atomic per transaction**: Either all changes commit or all revert. No partial state updates.

### Security Primitives
- **Modifiers** (`onlyOwner`, `nonReentrant`): Applied before function body. Check that they can't be bypassed by calling the internal function directly.
- **require / revert**: State-reverting checks. Verify they're in the right place (before state changes, not after).
- **Events**: Don't affect state. Missing events are Informational only unless an off-chain system depends on them.

### Key Risk Areas
- **delegatecall to untrusted address**: Arbitrary code execution in caller's context.
- **Assembly / low-level calls**: Bypass Solidity safety checks. `call`, `staticcall`, `delegatecall` return bool — if unchecked, failure is silently ignored.
- **External contract interactions**: Any `.call`, `.transfer`, `.send`, or interface call to an untrusted contract.
- **Solidity version matters**: Pre-0.8.0 has unchecked arithmetic by default. Post-0.8.0 reverts on overflow unless inside `unchecked {}`.

### What "Exploitable" Means in EVM
- **Single-tx exploit**: Attacker deploys a contract that executes the full attack in one transaction (most powerful — atomic, no front-running risk).
- **Multi-tx exploit**: Requires multiple transactions, potentially across blocks. Subject to front-running and state changes between txs.
- **EOA vs Contract caller**: Some exploits only work from a contract (reentrancy). If the function has `require(msg.sender == tx.origin)`, contract callers are blocked (but this is an antipattern itself).
- **Flashloan-enabled**: Attacker borrows unlimited capital for one tx. Eliminates capital requirements for many exploits.

### KEY QUESTION TO ASK
**"Can an unprivileged external caller reach this code path and cause a state change that benefits them at the expense of the protocol or other users?"**

---

## 2. Move (Sui + Aptos)

### Ownership Model
- **Resources cannot be copied or dropped implicitly**: The type system enforces this. A resource must be explicitly moved, stored, or destroyed.
- **No reentrancy in the Solidity sense**: Move's execution model doesn't allow a callee to call back into the caller mid-execution. But there are other vectors.
- **Module-level access control**: Only the module that defines a type can create, modify, or destroy it. This is enforced by the compiler.

### Signer & Capability Patterns
- **Signer**: Represents an authenticated account. Functions taking `&signer` require the account's authorization.
- **Capability pattern**: A resource stored in a specific account that grants permission. Missing capability check = access control bypass.
- **Friend modules**: Can call `public(friend)` functions. Check that friend declarations aren't overly permissive.

### Sui-Specific
- **Shared objects**: Accessible by anyone. Require consensus for ordering. Race conditions are possible if multiple txs modify the same shared object.
- **Owned objects**: Only the owner can use them in txs. Safe from race conditions but transferable.
- **Object ownership transfer**: Verify that ownership can't be transferred to an attacker through unexpected code paths.
- **Fastpath vs Consensus**: Owned object txs use fastpath (faster, no ordering). Shared object txs go through consensus.

### Aptos-Specific
- **Resource account pattern**: Autonomous accounts that own resources. Check that the signer capability for resource accounts is properly protected.
- **Module `friends`**: Overly permissive friend declarations can bypass access control.
- **`move_to` / `move_from`**: Resource storage operations. Verify that `move_from` doesn't allow unauthorized withdrawal of resources.

### What "Exploitable" Means in Move
- No reentrancy attacks in the traditional sense — but **logic bugs in resource handling** are the primary vector.
- **Missing signer checks**: Functions that should require authorization but don't.
- **Shared object manipulation** (Sui): Race conditions or unexpected state from concurrent access.
- **Type confusion**: Using a phantom type or generic incorrectly to bypass checks.

### KEY QUESTION TO ASK
**"Does this function properly verify the signer/capability and ensure the resource ownership model is preserved through every code path?"**

---

## 3. Rust / Sealevel (Solana)

### Account Model
- **Everything is an account**: Programs, data, tokens — all accounts with an owner, lamports, and data field.
- **Programs are stateless**: They process instructions that reference accounts. The program itself stores no state.
- **Account ownership**: Each account has an `owner` field (a program ID). Only the owner program can modify the account's data.

### Instruction Handler Pattern
- Instructions are deserialized from transaction data. The program receives a list of accounts and instruction data.
- **Account validation is the program's responsibility**: The runtime doesn't validate that the right accounts are passed. The program must check every account.

### Key Checks
- **Owner check**: Verify `account.owner == expected_program_id`. Without this, an attacker can pass a fake account owned by a different program with arbitrary data.
- **Signer check**: Verify `account.is_signer == true` for accounts that should have signed the transaction.
- **Discriminator check**: Anchor adds 8-byte discriminators. Without this, an attacker can pass an account of the wrong type (e.g., pass a Vault account where a User account is expected).
- **PDA derivation**: Program Derived Addresses must be re-derived and compared. Don't trust user-supplied PDAs without verification.
- **Lamport balance checks**: After CPI calls, verify lamport balances haven't been manipulated.

### CPI Security Model
- **Cross-Program Invocation (CPI)**: A program can call another program. The callee sees the caller as `msg.sender`.
- **Arbitrary CPI**: If a program takes an arbitrary program ID as input and CPIs to it, an attacker can redirect to their own malicious program.
- **Signer seeds in CPI**: When a PDA signs a CPI, the seeds must be verified to prevent PDA substitution.

### What "Exploitable" Means in Solana
- **Account substitution**: Passing a different account than expected. This is the #1 attack vector on Solana.
- **Missing checks**: The program doesn't validate an account, so the attacker passes one they control.
- **CPI manipulation**: Redirecting a CPI to a malicious program or spoofing a PDA.

### KEY QUESTION TO ASK
**"Are ALL accounts passed to this instruction fully validated — owner, signer, discriminator, PDA derivation — with no assumptions?"**

---

## 4. Sway (Fuel)

### Execution Model
- **UTXO + Contract hybrid**: Fuel uses a UTXO model for native assets and a contract model for stateful logic. This is unlike both Ethereum and Bitcoin.
- **Predicates**: Stateless scripts that evaluate to true/false. They gate UTXO spending. No state modification, no side effects.
- **Scripts**: Stateless entry points that orchestrate contract calls. Similar to Solana instructions.
- **Contracts**: Stateful programs with storage. Called by scripts or other contracts.

### Asset Handling
- **Native assets are first-class**: Unlike EVM where ETH and ERC-20 are handled differently. In Fuel, all assets (including the base asset) use the same UTXO model.
- **msg_amount() and msg_asset_id()**: Check that both are validated. A function expecting one asset might receive a different asset ID.
- **Forwarded coins**: Contracts can receive coins attached to calls. Verify that unexpected coins don't break accounting.

### What "Exploitable" Means in Fuel
- **Predicate bypass**: If a predicate's spending conditions can be satisfied by an attacker.
- **Asset confusion**: Sending the wrong asset ID to a function that doesn't validate it.
- **Script-level manipulation**: Crafting a script that calls contracts in an unexpected sequence.
- **Storage manipulation through re-entrancy-like patterns**: Fuel's call model is different from EVM but cross-contract calls still exist.

### KEY QUESTION TO ASK
**"Does this function validate both msg_amount() and msg_asset_id(), and does the UTXO accounting remain consistent across all call paths?"**

---

## 5. Cairo (StarkNet)

### Felt Arithmetic
- **Felts (field elements)**: The native number type. Arithmetic wraps modulo a large prime (P ≈ 2^251). This means:
  - Subtraction can underflow and wrap to a very large number
  - Division is modular inverse, not integer division
  - Comparisons (`<`, `>`) on felts are not intuitive — they compare field elements, not integers
- **u256, u128, etc.**: Cairo has proper integer types that panic on overflow. Check which type is used.

### Storage Variables
- **Storage is explicitly declared and accessed**: No storage slot collisions like Solidity proxies (unless using raw storage access).
- **Mappings use hash-based slots**: Similar to Solidity's keccak256 slot computation but with Pedersen hash.

### Account Abstraction
- **All accounts are smart contracts**: There are no EOAs. This means:
  - Signature validation is customizable per account
  - Replay protection is the account's responsibility
  - `get_caller_address()` always returns a contract address
- **Multicall**: Account contracts can batch multiple calls. Verify that batching doesn't create unexpected state interactions.

### What "Exploitable" Means in StarkNet
- **Felt overflow**: Arithmetic on felts wrapping unexpectedly, bypassing balance checks.
- **Account abstraction exploits**: Custom validation logic that can be tricked.
- **L1-L2 messaging**: Messages between L1 and L2 can be replayed or spoofed if not properly validated.
- **Missing access control**: Similar to other languages but with Cairo-specific patterns.

### KEY QUESTION TO ASK
**"Is the arithmetic performed on felts or proper bounded integer types, and are all comparisons meaningful given the modular arithmetic?"**

---

## 6. Vyper

### Differences from Solidity That Affect Security Reasoning
- **Built-in reentrancy lock**: Vyper has `@nonreentrant("lock_name")` decorator. Unlike Solidity, this is a first-class language feature. Check if it's applied to vulnerable functions.
- **No inline assembly**: Vyper does not support inline assembly. This eliminates an entire class of bugs (low-level memory manipulation, manual storage access).
- **No function overloading**: Each function has one implementation. Reduces confusion about which function is called.
- **No modifiers**: Vyper uses `@internal`, `@external`, `@view`, `@pure` decorators and explicit `assert` statements instead of Solidity-style modifiers.
- **Integer overflow**: Vyper has always checked for overflow by default (unlike pre-0.8 Solidity). However, specific Vyper versions had bugs in this (e.g., 0.2.15-0.3.0 had a reentrancy lock bug).
- **No `delegatecall`**: Vyper does not expose `delegatecall`. Proxy patterns are not natively supported. If present, they use raw `CALL` and must be audited carefully.
- **Compiler bugs**: Vyper has had significant compiler bugs (e.g., the Curve pool reentrancy from a Vyper compiler bug in 2023). Always check Vyper version.

### What to Check That Differs from Solidity Auditing
- **Vyper version**: Check for known compiler vulnerabilities for the specific version used.
- **Reentrancy lock syntax**: Verify `@nonreentrant` is applied correctly and uses consistent lock names.
- **`raw_call`**: The Vyper equivalent of Solidity's low-level call. Check for unchecked return values and untrusted targets.
- **`create_minimal_proxy_to`**: Vyper's proxy creation. Verify the target is trusted.

### KEY QUESTION TO ASK
**"What Vyper version is this, and does the finding account for Vyper's built-in safety features (overflow checks, reentrancy decorators) that may already mitigate the issue?"**
