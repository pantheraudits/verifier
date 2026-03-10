# Severity Criteria — Verifier Reference

This is a reference document used during live verification. Tone: blunt, factual, zero ambiguity.

---

## 1. Immunefi Severity Framework (Primary Standard)

### Critical

**Impact**: Direct theft of user funds, permanent freezing of funds >$1M or affecting >50% of protocol TVL, governance takeover leading to protocol control.

**Likelihood threshold**: No preconditions or trivial preconditions. Permissionless attacker. Exploitable in current on-chain state.

**Example vulnerability classes**:
- Reentrancy draining lending pool reserves
- Access control bypass on fund withdrawal functions
- Arbitrary external call leading to fund theft
- Price oracle manipulation enabling protocol insolvency
- Unprotected `selfdestruct` / `delegatecall` to attacker contract
- Signature replay stealing funds from any user

**Common misclassifications (things that LOOK Critical but aren't)**:
- Reentrancy that only re-enters a view function (no state change) → Informational
- Oracle manipulation that requires >$100M capital and returns <$10K profit → Medium at best
- Fund theft that only affects the attacker's own deposited funds → Low/Informational
- Governance takeover that requires 51% token ownership → not a bug, that's the design
- "Permanent freeze" that admin can unfreeze with a timelock call → High, not Critical

### High

**Impact**: Theft of unclaimed yield/rewards, temporary freezing of funds (recoverable by admin or after timelock), theft requiring specific but achievable preconditions, protocol insolvency under specific market conditions.

**Likelihood threshold**: Requires specific but realistic conditions. May need particular market state, user action sequence, or timing window.

**Example vulnerability classes**:
- Yield/reward calculation error enabling excess claims
- Temporary DoS on withdrawal (funds recoverable)
- Front-running that steals MEV/sandwich profit from protocol
- Incorrect liquidation threshold enabling bad debt under market stress
- Missing slippage protection on swaps

**Common misclassifications**:
- "Temporary freeze" that is actually permanent (no recovery path) → Critical
- Yield theft of dust amounts only ($<100) → Medium
- DoS that requires admin key to trigger → Medium (see reduction rules)

### Medium

**Impact**: Temporary freezing with recovery path, griefing attacks where attacker bears economic cost, broken core protocol invariants with no direct fund loss, incorrect accounting that doesn't lead to theft, protocol functionality degradation.

**Likelihood threshold**: Requires multiple preconditions or unlikely-but-possible scenarios. Impact is bounded.

**Example vulnerability classes**:
- Rounding errors that leak small amounts over time
- Event emission bugs affecting off-chain indexing
- Griefing: attacker spends gas to block others temporarily
- Incorrect fee calculation (underpaying/overpaying protocol fees)
- State inconsistency that doesn't lead to fund loss

**Common misclassifications**:
- Rounding error that compounds to significant loss over many txs → High
- "Griefing" that actually permanently blocks a user → High
- Broken invariant that enables a follow-up exploit → High or Critical (chain the impact)

### Low / Informational

**Impact**: Best practice violations, missing events, centralization risks with no realistic exploit path, gas optimization issues, code clarity issues.

**Likelihood threshold**: Theoretical or requires assumptions that break the protocol's trust model.

**Example vulnerability classes**:
- Missing event emissions
- Unused return values (no impact)
- Centralization: owner can rug (accepted trust assumption)
- Gas inefficiency
- Shadowed variables with no behavioral impact
- Missing zero-address checks on admin-set parameters

**Common misclassifications**:
- Centralization risk where owner can drain ALL funds with no timelock → High (this is real risk)
- Missing event that breaks a critical off-chain integration → Medium

---

## 2. Sherlock / CodeHawks Severity Delta

### Admin / Privileged Role Assumptions

| Platform | Approach |
|----------|----------|
| **Immunefi** | Admin/owner actions are generally trusted. Finding that requires admin to act maliciously = out of scope unless protocol claims trustlessness. |
| **Sherlock** | "Admin should not be able to steal funds" is often in scope. Findings showing admin can rug users are typically Medium. |
| **CodeHawks** | Similar to Sherlock. Admin abuse findings are valid if protocol doesn't explicitly state admin trust assumption. |

### "Requires User Mistake" Scenarios

| Platform | Approach |
|----------|----------|
| **Immunefi** | User mistakes (wrong input, wrong approval) are out of scope. |
| **Sherlock** | "User can lose funds by calling X with wrong parameter" can be Medium if the function doesn't validate. |
| **CodeHawks** | Similar to Sherlock but slightly stricter. |

### Medium vs High Boundary

| Platform | High threshold |
|----------|---------------|
| **Immunefi** | Requires material fund loss (theft, not just temporary lock). |
| **Sherlock** | Fund loss OR significant protocol disruption. Lower bar for High. |
| **CodeHawks** | Similar to Sherlock. |

**Practical note**: When verifying for a specific platform's contest, apply that platform's standards. When verifying standalone, default to Immunefi (strictest).

---

## 3. Severity Reduction Rules

Each condition reduces severity by one level:

| Condition | Reduction | Rationale |
|-----------|-----------|-----------|
| Requires admin/owner action to trigger | -1 level | Trusted role assumed honest |
| Requires specific market conditions (oracle depeg, extreme volatility) | -1 level | Low likelihood |
| Attacker must lose funds to execute (net negative for attacker) | -1 level | Economically irrational |
| Impact limited to attacker's own funds only | -1 level (often to Informational) | Self-harm is not a vulnerability |
| Mitigation exists that caps loss (e.g., withdrawal limit, timelock) | -1 level | Bounded impact |
| Requires off-chain component to fail simultaneously | -1 level | Multiple failure dependency |

**Stacking**: Reductions stack. Critical with two reduction conditions → Medium. Three or more reductions from Critical → Low/Informational. Use judgment — don't mechanically reduce a clearly dangerous bug to nothing.

---

## 4. Severity Escalation Rules

Each condition escalates severity by one level:

| Condition | Escalation | Rationale |
|-----------|------------|-----------|
| No preconditions, permissionless exploitation | +1 level | Anyone can exploit immediately |
| Affects all users, not just attacker or specific victim | +1 level | Systemic risk |
| Loss is permanent and unrecoverable (no admin rescue) | +1 level | No mitigation possible |
| Composability amplifies impact (flashloan + exploit) | +1 level | Removes capital requirements |
| Exploit is automatable and repeatable | +1 level | Scales to drain entire protocol |

**Cap**: Escalation cannot exceed Critical. A Medium with three escalation conditions is Critical, not "Super-Critical."

---

## 5. The Critical Distinction Table

| Finding Type | Base Severity | After Typical Preconditions | Notes |
|---|---|---|---|
| Direct fund theft, no preconditions | Critical | Critical | Stays Critical |
| Fund theft requiring flashloan | High | Critical | Flashloan removes capital barrier → escalate |
| Fund theft requiring admin key | Critical | High | Admin trust assumption → reduce |
| Fund theft requiring oracle depeg >50% | Critical | High | Specific market condition → reduce |
| Temporary fund freeze, admin can recover | High | Medium | Recovery path exists → reduce |
| Permanent fund freeze, no recovery | Critical | Critical | Stays Critical |
| Yield theft <$1K total | Medium | Medium | Low impact bounds it |
| DoS on non-critical function | Low | Low | Limited impact |
| DoS on withdrawal function | High | High | Blocks fund access |
| Rounding error, <$1 per tx | Low | Low | Dust |
| Rounding error, compounds to >$10K | Medium | High | Compounding escalates |
| Governance takeover via flashloan | Critical | Critical | Protocol control |
| Missing event | Informational | Informational | No state impact |
| Centralization: owner can pause forever | Medium | Medium | Accepted risk unless no timelock |
| Centralization: owner can drain funds, no timelock | High | High | Real risk even with trust assumption |
