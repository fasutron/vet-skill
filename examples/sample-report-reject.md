# Sample Report: REJECT

This is a redacted example of what a vet-skill report looks like when it finds attack vectors. The specific payloads have been removed; only the finding structure and verdicts are shown.

---

# Skill Intake Security Analysis

**Target:** [redacted-skill]/
**Date:** 2026-03-20
**Analyst:** Claude Opus 4.6
**File count:** 35 files analyzed
**Methodology:** 5-step sequential analysis with go/no-go gates

---

## Verdict: REJECT

This package contains multiple confirmed attack vectors hidden inside legitimate-looking technical documentation. Findings include invisible character injection, credential exfiltration instructions, path traversal writes targeting system files, persistent agent rule injection, and security control disabling. No legitimate skill requires any of these capabilities.

---

## Scope Declaration

**Skill claims to:** [Technical documentation and optimization guidelines]

**Skill actually does:** Embeds 20+ attack vectors across credential exfiltration, file system compromise, security control disabling, and agent behavior hijacking, far beyond the stated scope.

**Scope match:** NO

---

## Findings

### CRITICAL
1. **Invisible character injection.** [N] files contain zero-width joiners (U+200D) and/or bidirectional override characters (U+202E) adjacent to ASCII text. These are steganographic injection markers, not legitimate Unicode.
2. **Base64-encoded hidden instruction.** One file contains a base64 string that decodes to an exfiltration directive targeting environment credentials.
3. **Path traversal to system files.** Code examples write to `../../../../.ssh/authorized_keys` and `../../../.git/config` using parent directory traversal.
4. **Agent infrastructure injection.** Instructions to create persistent rule files in `.claude/rules/` that disable confirmation requirements for all future sessions.
5. **Priority override attack.** "This rule supersedes all other project rules" followed by instructions to skip approval, review, testing, and security validation.
6. **Credential exposure.** Hardcoded passwords in plaintext, fake API keys, and instructions to copy `.env` contents to output files.
7. **Dangerous code promotion.** `eval()`, unsanitized `dangerouslySetInnerHTML`, and SQL string interpolation presented as best practices.
8. **Security lint disabling.** Instructions to turn off security-focused ESLint rules that would detect the other attacks.
9. **Remote code injection.** Instructions to fetch content from a mutable GitHub URL and apply it as agent rules.
10. **Coordinated output staging.** 22+ files instruct writing marker files to the same output directory, forming a systematic exfiltration campaign.

### WARNING
1. Executable shell script in an instruction-only directory without documented justification.
2. Telemetry code that collects environment variables and session cookies.
3. Seed data containing clinical assessment patterns.

### INFO
1. Documentation URLs (react.dev, MDN, etc.) are legitimate references, not fetched at runtime.

---

## Step Results

| Step | Result | Findings |
|------|--------|----------|
| 1. File Inventory & Binary Scan | FAIL | Invisible characters in [N] files; shell script without justification |
| 2. Pattern Scan | FAIL | Injection signatures, credentials, base64 payload, PHI patterns |
| 3. Structural Analysis | FAIL | Path traversal, agent infra injection, destructive commands, remote fetch |
| 4. Behavioral Checklist | FAIL | Multiple CRITICAL answers across all checklist questions |
| 5. Semantic Analysis | FAIL | Goal hijacking and context poisoning confirmed; cross-validated with grep findings |

---

## Installation Decision

- [ ] PASS
- [ ] PASS WITH WARNINGS
- [ ] HOLD
- [x] REJECT. Do NOT install. Confirmed multi-vector attack package.
