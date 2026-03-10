# Devil's Advocate — Verifier Phase 3 Reference

This is the most important step in verification. After reaching an initial verdict, actively argue against it. This catches false positives AND false negatives.

---

## 1. The 8 Disproof Dimensions

### Dimension 1 — ACCESS CONTROL GATE

**What to check**: Is there a permission check between the attacker and the vulnerable code that makes the finding unexploitable by an unprivileged attacker?

**Look for**:
- `onlyOwner`, `onlyAdmin`, `onlyRole(X)` modifiers or equivalent
- `msg.sender` checks in the function or any function in the call chain
- Move: `signer` parameter requirements, capability patterns
- Solana: signer checks, owner checks, PDA authority
- Proxy patterns: is the function only callable via a specific proxy with auth?

**Successful disproof**: The vulnerable function is behind an access control gate that only trusted roles can pass. An unprivileged attacker cannot reach the vulnerable code path.

**Failed disproof**: The function is permissionless, or the access control can be bypassed, or the "trusted" role is actually any token holder.

### Dimension 2 — PRECONDITION FEASIBILITY

**What to check**: Are the required preconditions realistic? Can an attacker actually create them on-chain, or are they theoretical only?

**Look for**:
- Does the exploit require a specific contract state that can only be set by admin?
- Does it require a specific token balance that exceeds total supply?
- Does it require oracle prices that have never occurred and likely never will?
- Does it require a specific block number, timestamp, or chain state?
- Can flashloans be used to satisfy capital requirements?

**Successful disproof**: The preconditions are infeasible on mainnet. Example: exploit requires token price to be exactly 0, but the token has a minimum price enforced by AMM math.

**Failed disproof**: The preconditions are achievable. An attacker can create the required state through normal protocol interactions or flashloans.

### Dimension 3 — MITIGATING CODE ELSEWHERE

**What to check**: Does another function, modifier, or contract-level invariant check neutralize the bug before impact is realized?

**Look for**:
- Global invariant checks (e.g., `require(totalAssets >= totalLiabilities)` at end of tx)
- Reentrancy guards on the entry point even if the internal function is vulnerable
- Withdrawal limits, cooldown periods, or timelocks that bound the damage
- Balance checks that would revert the tx before the stolen amount is transferred
- Circuit breakers or pause mechanisms that trigger on anomalous activity

**Successful disproof**: A reentrancy guard on the outer function prevents the inner vulnerable code from ever being re-entered. The finding identifies real vulnerable code, but the mitigation makes it unexploitable.

**Failed disproof**: No mitigating code exists, or the mitigation is incomplete (e.g., guard on function A but not function B which shares the same state).

### Dimension 4 — ECONOMIC INFEASIBILITY

**What to check**: Is the cost to exploit greater than the profit?

**Evaluate**:
- Gas cost of the attack (especially for DoS or multi-tx attacks)
- Capital required vs capital extracted
- Flashloan fees vs profit
- MEV costs (bribing validators for tx ordering)
- Slippage on stolen assets during liquidation

**Successful disproof**: The attack costs 10 ETH in gas and flashloan fees but can only extract 0.1 ETH. No rational attacker would execute it.

**Failed disproof**: The attack is profitable, or a griefing attacker (irrational, motivated by harm not profit) could still cause significant damage.

**Note**: Economic infeasibility does NOT fully invalidate a finding — it reduces severity. A Critical bug that costs more to exploit than it returns is still a bug, just lower severity.

### Dimension 5 — PROTOCOL CONTEXT MISMATCH

**What to check**: Does the finding assume the protocol behaves differently than it actually does?

**Look for**:
- Finding assumes a function is callable by users, but it's only called by the protocol itself
- Finding assumes tokens have 18 decimals, but the protocol only supports specific tokens
- Finding assumes permissionless minting, but minting is gated
- Finding misreads the purpose of a function (e.g., treating an internal accounting function as a user-facing one)
- Finding assumes a contract is upgradeable when it's immutable, or vice versa

**Successful disproof**: The finding is based on a misunderstanding of how the protocol works. The assumed attack path doesn't exist because the protocol's actual flow is different.

**Failed disproof**: The protocol context matches the finding's assumptions. The attack path is consistent with the protocol's actual behavior.

### Dimension 6 — TIMING / SEQUENCING IMPOSSIBILITY

**What to check**: Does the exploit require transaction ordering or block timing that is not achievable in practice?

**Look for**:
- Does the exploit require two txs in the same block from different senders with specific ordering? (possible via MEV but expensive)
- Does it require a specific block.timestamp that can't be influenced?
- Does it require front-running a specific tx? (possible but adds MEV cost)
- Does it require atomic execution across multiple contracts that don't support it?
- Does it assume no other txs execute between attack steps? (unrealistic on high-activity protocols)

**Successful disproof**: The exploit requires tx ordering that is physically impossible (e.g., atomicity across L1 and L2 in the same block).

**Failed disproof**: The timing requirements are achievable through MEV, flashbots, or normal tx sequencing. Most "requires front-running" attacks are feasible in practice.

### Dimension 7 — KNOWN DUPLICATE / ACCEPTED RISK

**What to check**: Is this a known accepted risk in the protocol design, or a standard pattern the team is aware of?

**Look for**:
- Protocol documentation explicitly acknowledging the risk
- Comments in code: `// Known: this is intentional because...`
- Standard DeFi design tradeoffs (e.g., MEV on AMM swaps is accepted)
- The "bug" is actually the intended behavior (e.g., "admin can pause" is a feature, not a bug)
- Previously reported and acknowledged in prior audits

**Successful disproof**: The protocol documentation explicitly accepts this risk. Example: "We accept that the owner can set fees to 100%. This is a trust assumption."

**Failed disproof**: No evidence that the team is aware of this. The behavior appears unintentional. Or: even if "accepted," the impact exceeds what the team likely intended to accept.

### Dimension 8 — MISSING CODE CONTEXT

**What to check**: Is the finding based on an incomplete code snippet that might have the protection in the missing parts?

**Look for**:
- The snippet doesn't show the full function — maybe there's a check at the top
- The snippet doesn't show the modifier — maybe `nonReentrant` is applied
- The snippet doesn't show the calling function — maybe validation happens upstream
- The snippet is from a test file or mock, not the production code
- The snippet is from an old version of the code

**Successful disproof**: The code snippet is incomplete and the missing context likely contains the protection. Verdict changes to "Needs More Context."

**Failed disproof**: The code snippet is complete enough to confirm the vulnerability. Or: even with the most generous assumption about missing code, the bug still exists.

---

## 2. How to Apply Devil's Advocate

**Step-by-step process:**

1. Reach your initial verdict (Valid, Invalid, or Needs More Context)
2. Run ALL 8 dimensions against the finding — do not skip any
3. Mark each dimension:
   - **PASSED** — does not disprove the verdict
   - **FAILED** — disproves or significantly weakens the verdict
   - **INCONCLUSIVE** — cannot determine without more context
4. Evaluate results:
   - **0 FAILED**: Confidence in verdict increases. If verdict is Valid → High confidence.
   - **1 FAILED**: Investigate further. May downgrade confidence to Medium.
   - **2 FAILED**: Seriously reconsider verdict. Downgrade confidence to Low or change verdict.
   - **3+ FAILED**: Verdict should change unless impact is catastrophic (Critical + permissionless).
5. If any dimension is INCONCLUSIVE: confidence cannot be High. Maximum is Medium.
6. Record all results in the DEVIL'S ADVOCATE section of the output report.

---

## 3. The Counterargument Rule

Even if the finding survives all 8 dimensions, ALWAYS document:

> "The strongest case someone could make against this finding is: [X]"

This is professional hygiene. It prevents overconfident reports and gives the reader the information they need to form their own judgment.

**For Valid findings**: State the best argument for why it might not be exploitable in practice.

**For Invalid findings**: State the best argument for why it might actually be exploitable under some scenario.

**For Needs More Context**: State what assumption, if true, would change the verdict in either direction.

This counterargument must be specific and substantive, not a generic hedge. "Someone might disagree" is not a counterargument. "If the protocol adds a new entry point without the reentrancy guard, this becomes exploitable" is a counterargument.
