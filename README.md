# Verifier

A Claude Code skill for deep single-finding verification in smart contract security audits. Verifier takes a reported finding (bug report, vulnerability description, audit issue) and determines whether it is a real, exploitable vulnerability or a false positive. It outputs a structured verification report with verdict, severity, confidence, exploit path analysis, and devil's advocate counterarguments. It does not generate findings — it verifies them.

## Install

```bash
git clone https://github.com/iftikharuddin/verifier.git
cp -r verifier/ ~/.claude/commands/verifier/
```

Or symlink for easy updates:

```bash
ln -s $(pwd)/verifier ~/.claude/commands/verifier
```

## Usage

### Slash command

```
/verifier
```

Then paste a finding with:
- Finding title
- Finding description (what the bug is, what code path, claimed impact)
- Code snippet(s)

Verifier auto-detects the language (Solidity, Move, Rust, Sway, Cairo, Vyper) and runs the 5-phase verification workflow.

### Direct paste

Paste a finding description and code snippet into Claude Code. Verifier auto-activates when it detects a finding/vulnerability being described.

## Input Format

| Field | Required | Description |
|-------|----------|-------------|
| Finding title | Yes | Short name of the vulnerability |
| Finding description | Yes | Bug explanation, code path, claimed impact |
| Code snippet(s) | Yes | Relevant function(s) or code block(s) |
| Scope | No | In-scope contract names or file paths |
| Protocol context | No | Protocol type and intended behavior |

## Output

Every invocation produces a structured **VERIFIER REPORT** containing:

- **Verdict**: Valid / Invalid / Needs More Context / Out of Scope
- **Severity**: Critical / High / Medium / Low / Informational
- **Confidence**: High / Medium / Low
- **Reasoning**: Why the finding is valid or invalid, with specific code references
- **Exploitability**: Preconditions, attack path, attacker type
- **PoC sketch**: Pseudocode proof of concept (if valid)
- **Devil's advocate**: Counterarguments tested against the verdict
- **Recommendation**: Specific fix direction

See `sample-finding.md` for complete worked examples.

## Severity Standard

**Primary**: Immunefi severity framework (Critical / High / Medium / Low / Informational).

**Noted**: Sherlock and CodeHawks deltas where classification differs (admin trust assumptions, user mistake handling, Medium/High boundary).

See `severity-criteria.md` for the full framework.

## Integration

Verifier is designed to be called by other audit skills (move-auditor, solidity-auditor, etc.) as a downstream verification layer. See `integration-guide.md` for the input/output contract and calling conventions.

## Disclaimer

Verifier accelerates finding verification. It does not replace human judgment on ambiguous findings. Confidence: Low verdicts and Needs More Context results require manual review. Use verifier as a first pass, not a final authority.

## What It Does NOT Do

- **Does not replace a full audit.** It verifies individual findings, not entire codebases.
- **Does not batch verify.** One finding per invocation. Pass each finding separately.
- **Does not generate findings.** It only verifies findings that are already identified. Use audit skills (move-auditor, etc.) for finding generation.
- **Does not access on-chain state.** Verification is based on the code provided. It does not query blockchain data.
- **Does not guarantee correctness.** It is a tool, not an oracle. Low-confidence results exist for a reason.
