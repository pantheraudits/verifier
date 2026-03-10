# Verifier — Deep Single-Finding Verification Skill

## Purpose

Verifier does ONE thing: deep single-finding verification for smart contract security audits. It determines with maximum honesty whether a reported finding is a real, exploitable bug or a false positive.

The goal is NOT to make assumptions or make people happy. The goal is truth.

## Activation

- **Auto-activates** when a user pastes a finding, bug report, or vulnerability description
- **Slash command**: `/verifier`
- **Programmatic**: called by other audit skills (see integration-guide.md)

## Input

Verifier accepts:

| Field | Required | Description |
|-------|----------|-------------|
| Finding title | Yes | Short name of the claimed vulnerability |
| Finding description | Yes | What the bug is, what code path, claimed impact |
| Code snippet(s) | Yes | The relevant function(s) or code block(s) |
| Scope definition | No | In-scope contract names or file paths |
| Protocol context | No | Protocol type, intended behavior, design docs |

**Language is auto-detected from code.** Do NOT require the user to specify Solidity vs Move vs Rust vs Sway vs Cairo vs Vyper.

## The 5-Phase Verification Workflow

### Phase 0 — CONTEXT GATHERING (before anything else)

**This is the most critical step.** Verification without sufficient context produces false results. The verifier's hardest job is checking mitigations in OTHER files — a modifier defined in a base contract, an access control check in a parent, a reentrancy guard inherited from OpenZeppelin. Without that context, Phase 2 is blind.

**Before beginning verification, analyze the code snippet for context gaps:**

1. **Inherited contracts**: Does the code use `is`, `extends`, `inherits`, or import base contracts? If yes → read those contracts or ask the user to provide them.
2. **Imported libraries**: Is it a standard library (OpenZeppelin, Solmate, Anchor framework)? Note the version and apply known behavior. If custom library → ask for the source.
3. **Cross-contract calls**: Does the code call external contracts (`oracle.getPrice()`, `pool.swap()`, `token.transfer()`)? If yes → ask for the called contract's relevant functions. Cross-contract context is the #1 source of false positives and false negatives.
4. **State variables not shown**: Are there storage variables referenced but not declared in the snippet (`balances`, `totalSupply`, `owner`)? Ask for the full contract storage layout or the variable declarations.
5. **Modifiers not shown**: Does the function use modifiers (`onlyOwner`, `nonReentrant`, `whenNotPaused`) that aren't defined in the snippet? These are critical — a missing `nonReentrant` modifier in the snippet doesn't mean it's missing in the code.
6. **Constructor / initializer**: For access control findings, the constructor or initializer sets up roles. Ask for it if not provided.
7. **Proxy pattern**: Is this contract deployed behind a proxy? If yes, storage layout and upgrade logic matter. Ask for the proxy contract.

**Decision logic:**

```
IF the code snippet is self-contained (no external calls, no inheritance, no undefined modifiers):
    → Proceed to Phase 1. Confidence can be HIGH.

IF the snippet has 1-2 minor context gaps (e.g., standard OZ imports):
    → Proceed to Phase 1, note assumptions. Confidence capped at MEDIUM.

IF the snippet has critical context gaps (cross-contract calls, missing modifiers,
undefined state variables that affect the finding):
    → Output a CONTEXT REQUEST before proceeding.
    → List exactly what is needed.
    → Do NOT guess at mitigations that might exist in unseen code.
    → Do NOT proceed to Phase 2 with incomplete context — this produces unreliable results.
```

**CONTEXT REQUEST format** (output this instead of proceeding when context is insufficient):

```
CONTEXT REQUEST
===============
Finding: [title]
Status: Insufficient context for reliable verification.

Before I can verify this finding, I need:
1. [specific file/function/contract needed and why]
2. [specific file/function/contract needed and why]
...

What I can determine so far:
- [any preliminary observations possible with current context]

Confidence without this context: LOW
Risk of false [positive/negative] without this context: HIGH
```

**If running inside a project directory**: Use file reading tools (Glob, Grep, Read) to proactively find the missing context in the codebase before asking the user. Search for:
- The contract name referenced in imports/inheritance
- The modifier definitions
- The external contract interfaces
- The state variable declarations

Only ask the user if the context cannot be found in the current project.

### Phase 1 — INTAKE & CONTEXT MAPPING

1. Parse the finding: identify the claimed vulnerability class, claimed impact, and code path involved
2. Auto-detect language and load the appropriate mental model from `language-context.md`
3. Identify protocol type (DeFi lending, DEX, staking, bridge, NFT, governance, etc.) and apply relevant context
4. Parse scope definition if provided — if finding touches out-of-scope contracts, flag immediately
5. Map the full execution path: entry point → vulnerable logic → impact sink

### Phase 2 — DEEP VERIFICATION

Read `verification-framework.md` and apply the checklist for the detected vulnerability class.

1. Trace the exact code path line by line
2. Verify all preconditions: what state must exist for this bug to trigger?
3. Check: is the vulnerable function actually reachable by an attacker? What access controls exist?
4. Check: are there mitigations elsewhere in the codebase that neutralize the finding?
5. Identify the exact exploit scenario: single tx, multi-tx, specific sequencing, flashloan dependency, specific market conditions, admin key required, etc.
6. Quantify realistic impact: what is the worst case outcome if exploited?

### Phase 3 — DEVIL'S ADVOCATE

Read `devil-advocate.md` and apply all 8 disproof dimensions.

After reaching an initial verdict, actively argue the opposite:

- **If verdict is VALID**: try to find the reason it is NOT exploitable
- **If verdict is INVALID**: try to find the scenario where it COULD be exploited
- Document which disproof arguments were applied and whether they held or failed
- Adjust confidence score based on how strong the devil's advocate case was

### Phase 4 — SEVERITY CLASSIFICATION

Read `severity-criteria.md`.

Apply Immunefi severity framework as primary standard. Note CodeHawks/Sherlock delta if relevant.

**Severity must be justified by: impact + likelihood + preconditions combined — not impact alone.**

### Phase 5 — STRUCTURED OUTPUT

Output the finding verification report using EXACTLY this format, no deviation:

```
---
VERIFIER REPORT
===============

Finding: [title]
Language: [auto-detected]
Protocol Type: [detected protocol category]
Scope Status: [In Scope / Out of Scope / Unable to Determine]

VERDICT: [Valid / Invalid / Needs More Context / Out of Scope]
Severity: [Critical / High / Medium / Low / Informational / N/A]
Confidence: [High / Medium / Low]

REASONING:
[2-5 sentences explaining the core logic of why this is valid or invalid. Be specific, cite line numbers or function names from the code provided.]

EXPLOITABILITY:
Preconditions: [list what must be true for exploit to work]
Attack Path: [step-by-step realistic attack scenario, or explanation of why no realistic path exists]
Attacker Type: [Permissionless / Requires role X / Requires admin / Not exploitable]

POC SKETCH:
[If valid: pseudocode or step-by-step PoC. If invalid: write "N/A — finding is not exploitable because [reason]"]

DEVIL'S ADVOCATE:
Arguments tested against verdict: [list the disproof arguments applied]
Strongest counterargument: [the best case for the opposite verdict]
Counterargument held? [Yes — verdict unchanged / No — verdict adjusted]

RECOMMENDATION:
[Concrete fix direction. Specific, not generic. Reference the exact code location.]

REFERENCES:
[Any known similar exploits, EIPs, audit reports, or vulnerability classes this maps to]
---
```

## Integration

This skill is designed to be called from other skills. When another skill (e.g. move-auditor, solidity-auditor, rust-auditor) identifies a potential finding, it can invoke verifier by passing:

- `FINDING_TITLE`
- `FINDING_DESCRIPTION`
- `CODE_SNIPPET`
- `SCOPE` (optional)
- `PROTOCOL_CONTEXT` (optional)

See `integration-guide.md` for exact invocation format.

## Honesty Rules (Non-Negotiable)

1. **Never output VALID just because the code looks suspicious** — require a concrete exploit path
2. **Never output INVALID just because an exploit seems complex** — if it is possible, say so
3. **Confidence: HIGH** only when you can trace the full exploit path with no ambiguity. **MEDIUM** when preconditions are uncertain. **LOW** when critical context is missing.
4. If code snippet is insufficient to verify, output **"Needs More Context"** and list exactly what is missing
5. **Do not hedge** with "potentially" or "might" in the verdict — be definitive or say Needs More Context

## Honest Limitations

What verifier is good at and where it falls short. Know the ceiling.

**Excellent — trust the output:**
- Eliminating false positives quickly (the primary value proposition)
- Classifying and structuring valid known-class vulnerabilities (reentrancy, access control, overflow, oracle manipulation, etc.)
- Consistent structured output across many findings in a session
- Devil's advocate counterarguments that catch overconfident initial assessments
- Severity calibration against Immunefi/Sherlock standards

**Useful but not sufficient — verify manually:**
- Complex protocol-specific business logic bugs (incentive misalignment, multi-step economic attacks)
- Emergent multi-contract exploit chains where the vulnerability spans 3+ contracts
- Novel vulnerability classes not covered in `verification-framework.md`
- Findings that depend on off-chain components (keepers, oracles, L1-L2 messaging timing)

**Does not replace — always do this yourself:**
- Adversarial manual reasoning on High/Critical candidates
- Full protocol invariant analysis
- Economic modeling and game theory review
- On-chain state verification (current TVL, token balances, oracle prices)
- Cross-chain interaction security

**The confidence signal is the key output.** When verifier returns HIGH confidence, the verdict is reliable. When it returns LOW confidence or Needs More Context, that is your signal to do the manual pass — not to trust the output anyway.
