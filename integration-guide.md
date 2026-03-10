# Integration Guide — Verifier Skill

How other Claude Code skills integrate verifier as a downstream verification layer.

---

## 1. Integration Pattern

Verifier is a downstream verification layer. The workflow:

1. **Calling skill** (e.g., move-auditor, solidity-auditor, rust-auditor) scans code and identifies candidate findings.
2. **Calling skill invokes verifier** for each candidate finding to confirm validity.
3. **Verifier runs the 5-phase verification workflow** and returns a structured report.
4. **Calling skill uses the verdict** to include, exclude, or flag the finding in its final report.

This pattern separates finding identification (the calling skill's job) from finding verification (verifier's job). The calling skill casts a wide net; verifier filters out false positives.

---

## 2. Input Contract

When invoking verifier programmatically, pass the following:

```
VERIFIER_INPUT:
  finding_title: <string>
    Required. Short name of the claimed vulnerability.
    Example: "Reentrancy in withdraw() allows double withdrawal"

  finding_description: <string>
    Required. Explain the bug, the vulnerable code path, and the claimed impact.
    Example: "The withdraw() function sends ETH via .call before updating the user's balance. An attacker can re-enter withdraw() from a fallback function to drain the pool."

  code_snippet: <string>
    Required. The relevant function(s) or code block(s). Include enough context for verifier to trace the execution path. Minimum: the vulnerable function. Ideal: the vulnerable function plus any callers and callees.

  language: <string, optional>
    If not provided, verifier auto-detects from the code snippet.
    Supported: solidity, move, rust, sway, cairo, vyper

  scope: <string, optional>
    Comma-separated list of in-scope contract names or file paths.
    Example: "LendingPool.sol, Vault.sol, Oracle.sol"
    If provided, verifier flags findings that touch out-of-scope contracts.

  protocol_context: <string, optional>
    Brief description of protocol type and intended behavior.
    Example: "DeFi lending protocol. Users deposit collateral, borrow against it. Liquidation at 80% LTV."

  severity_hint: <string, optional>
    The calling skill's initial severity assessment.
    Verifier will independently assess severity but notes the hint for comparison.
    Values: Critical, High, Medium, Low, Informational
```

---

## 3. Output Contract

Verifier always returns the structured report defined in `SKILL.md`. The full report includes:

| Field | Type | How to use it |
|-------|------|---------------|
| `VERDICT` | Valid / Invalid / Needs More Context / Out of Scope | **Primary decision field.** Include, exclude, or flag the finding. |
| `Severity` | Critical / High / Medium / Low / Informational / N/A | Set the finding severity in your report. |
| `Confidence` | High / Medium / Low | Determines if human review is needed. |
| `REASONING` | String | Include in report as the finding's technical justification. |
| `EXPLOITABILITY` | Structured | Include preconditions and attack path in the finding details. |
| `POC SKETCH` | String | Include as proof of concept if verdict is Valid. |
| `DEVIL'S ADVOCATE` | Structured | Optionally include to show thoroughness. |
| `RECOMMENDATION` | String | Include as the remediation guidance. |

### Decision Logic for Calling Skills

```
IF verdict == "Valid" AND confidence == "High":
    → Include in report with verifier's severity
IF verdict == "Valid" AND confidence == "Medium":
    → Include in report, flag for human review
IF verdict == "Valid" AND confidence == "Low":
    → Flag for human review, do not auto-include
IF verdict == "Invalid":
    → Discard from report
IF verdict == "Needs More Context":
    → Flag for human review with verifier's list of what's missing
IF verdict == "Out of Scope":
    → Exclude from report, note in scope analysis section
```

---

## 4. Example Integration (move-auditor style)

In your skill's `SKILL.md`, add a verification phase:

```markdown
### Phase N — FINDING VERIFICATION

For each candidate finding identified in previous phases:

1. Invoke the verifier skill with:
   - finding_title: the finding title
   - finding_description: the full finding description including claimed impact
   - code_snippet: the vulnerable function(s) and relevant surrounding code
   - language: "move" (or omit for auto-detection)
   - protocol_context: the protocol description from Phase 1
   - scope: the scope definition from the audit brief

2. Process verifier output:
   - VERDICT: Valid + Confidence: High/Medium → include in final report
   - VERDICT: Invalid → discard, do not include
   - VERDICT: Needs More Context → list in "Requires Human Review" section
   - VERDICT: Out of Scope → exclude, note in scope analysis

3. Use verifier's severity as the finding severity (override your initial estimate).

4. Include verifier's REASONING and RECOMMENDATION in the finding write-up.
```

---

## 5. Calling Convention

- **One finding per invocation.** Call verifier once per finding. Do not batch multiple findings in one call.
- **Do not pre-filter.** Pass all candidate findings to verifier. Let verifier make the valid/invalid call. Pre-filtering introduces bias and defeats the purpose.
- **Always pass code snippet.** Verifier without code context produces low-confidence results only. If you don't have the code, verifier will return "Needs More Context."
- **Pass maximum context.** The more code and protocol context you provide, the higher confidence verifier can achieve. Include callers, callees, state variables, and relevant modifiers/checks.
- **Respect the verdict.** If verifier says Invalid, don't override unless you have specific new information verifier didn't have. The whole point is independent verification.

---

## 6. Versioning Note

Integrating skills should pin to a specific git tag of verifier for reproducibility.

```
# In your skill's SKILL.md or README:
Depends on: verifier@v1.0.0
```

This ensures that updates to verifier's severity criteria or verification logic don't silently change the behavior of your skill. Update the pin explicitly when you want to adopt new verifier logic.
