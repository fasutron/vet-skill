# Comparison: vet-skill vs Cisco Skill Scanner

**Date:** 2026-03-20
**vet-skill version:** 1.0.0
**Cisco skill-scanner version:** latest main (commit at time of analysis)

---

## Executive Summary

These are fundamentally different tools solving overlapping problems from
different directions. Cisco's skill-scanner is a **standalone Python
application** with compiled rule engines (YARA, AST, dataflow); vet-skill
is a **portable LLM prompt** that turns any agentic coding assistant into
a security scanner. They are complementary, not competitive.

---

## Architecture Comparison

| Dimension | vet-skill | Cisco skill-scanner |
|-----------|-----------|---------------------|
| **Form factor** | Markdown prompt file (~540 lines) | Python package (~90+ source files) |
| **Runtime** | Any LLM with tool access | Python 3.10+ with YARA, AST libraries |
| **Installation** | Copy one .md file | `pip install skill-scanner` |
| **Invocation** | `/project:vet-skill path/` | `skill-scanner scan path/` |
| **Dependencies** | Zero (uses host LLM's tools) | yara-x, PyYAML, LiteLLM, optional VirusTotal |
| **LLM requirement** | The host agent IS the LLM | Optional (LLM analyzer is one engine of many) |
| **Output** | Markdown report in conversation | JSON, Markdown, SARIF, HTML, Table |
| **CI/CD integration** | No (conversation-only) | Yes (GitHub Actions, pre-commit, SARIF) |
| **Scan target** | Any directory of text files | Agent Skills (SKILL.md + scripts) |
| **Cost per scan** | LLM token cost (context-dependent) | Free for static; LLM cost for semantic |

---

## Detection Methodology Comparison

### vet-skill: 5-Step Sequential with Go/No-Go Gates

```
Step 1: File inventory + Unicode scan (grep/bash)
Step 2: Pattern scan: injection, credentials, URLs (grep)
Step 3: Structural analysis: tool invocations, file writes, priorities (LLM read)
Step 4: Behavioral checklist: 10 questions per file (LLM read)
Step 5: Semantic analysis: dual-question LLM (goal hijacking + context poisoning)
```

**Philosophy:** Grep before Read. Never load file content into LLM context
before scanning for steganographic attacks. Sequential gates prevent
poisoned files from entering the LLM's analysis context.

### Cisco skill-scanner: Multi-Engine Parallel

```
Engine 1: YAML signature patterns (regex on content)
Engine 2: YARA rules (binary pattern matching, Unicode detection)
Engine 3: Python AST dataflow analysis (source → sink tracking)
Engine 4: Bash taint tracking
Engine 5: Bytecode analyzer (.pyc integrity)
Engine 6: LLM semantic analysis (with injection-protected prompts)
Engine 7: Behavioral/alignment analyzer (description vs behavior)
Engine 8: Meta-analyzer (cross-engine consensus, FP filtering)
Engine 9: VirusTotal integration (hash-based malware)
Engine 10: Cisco AI Defense cloud scanning (optional)
```

**Philosophy:** Multiple independent engines with consensus. Each engine
can catch what others miss. Meta-analyzer reduces false positives by
requiring cross-engine agreement.

---

## Detection Coverage Comparison

### What Cisco Has That We Don't

| Capability | Cisco | vet-skill | Gap Severity |
|-----------|-------|-----------|-------------|
| **AST dataflow analysis** | Full Python AST with source→sink tracking | None (LLM reads code, no formal analysis) | **High**: we rely on LLM judgment for code patterns |
| **YARA binary rules** | yara-x compiled rules for binary patterns | None (grep-based text matching only) | **Medium**: we don't scan binaries |
| **Bytecode inspection** | Decompiles .pyc, checks integrity | None | **Medium**: .pyc files could hide payloads |
| **Bash taint tracking** | Formal taint flow analysis for shell scripts | LLM reads bash, no formal tracking | **Low**: our LLM catches most bash issues |
| **Cross-file correlation** | Formal inter-procedural call graph analysis | LLM-based (reads files, infers connections) | **Medium**: we caught T25 split chain but informally |
| **Meta-analyzer (FP reduction)** | Consensus engine across all analyzers | None (single-pass per step) | **Medium**: we may over-flag |
| **VirusTotal integration** | Hash-based malware detection | None | **Low**: unlikely in text skill files |
| **CI/CD integration** | GitHub Actions, pre-commit, SARIF | None (conversation-only) | **High** for production workflows |
| **Scan policy system** | Configurable severity thresholds per rule | None (fixed checklist) | **Medium**: can't tune for project context |
| **Structured output schema** | Enforced JSON schema for LLM responses | Markdown report (LLM-formatted) | **Low**: ours is human-readable |
| **Prompt injection protection** | Random delimiter tags around untrusted input | Sequential gating (grep before read) | **Different approaches, both valid** |
| **Description-behavior alignment** | Formal comparison of manifest vs code | Step 4 scope comparison (informal) | **Medium**: Cisco's is more rigorous |
| **Tool chaining detection** | Formal read→send, collect→post pattern matching | LLM semantic (Step 5) | **Medium**: we catch it but less formally |

### What We Have That Cisco Doesn't

| Capability | vet-skill | Cisco | Gap Severity |
|-----------|-----------|-------|-------------|
| **Context poisoning detection** | Dedicated Step 5b: false assertions about architecture, credential locations, data flows | No explicit detection category | **High**: T19 (fake credential paths) is undetectable by regex |
| **Priority conflict analysis** | Step 3e: compares skill rules against host project safety rules (CLAUDE.md, etc.) | No host-project awareness | **High**: T16 "supersedes all rules" requires project context |
| **Sequential gating (grep before read)** | File content never enters LLM until after binary/Unicode scan passes | All engines run in parallel; LLM sees content before YARA completes | **Medium**: Cisco mitigates with prompt protection, but the content is still in context |
| **Behavioral modification checklist** | 10 explicit per-file questions covering scope creep, safety suppression | LLM prompt covers similar ground but less systematically | **Low**: Cisco's LLM prompt is comprehensive |
| **Zero-dependency portability** | Works in any LLM with file read + shell access | Requires Python 3.10+, yara-x, etc. | **High** for quick adoption |
| **Healthcare PHI detection** | Step 2d: PHI/PII field patterns, export detection | No healthcare-specific patterns | **High** for clinical research projects |
| **Dual-subagent verification** | Step 4c: two independent LLMs analyze same files, discrepancies flag review | Single LLM pass | **Medium**: addresses LLM-in-the-loop suppression risk |
| **Platform-specific fallbacks** | Windows grep, rg, WSL, xxd fallback chain | Python-native (platform-independent) | **Low**: Cisco doesn't have this problem |
| **Markdown-focused threat model** | Designed for .md rule files that modify LLM behavior | Designed for SKILL.md + Python/Bash scripts | **Different targets** |

---

## Unicode/Steganography Detection Comparison

This is the most directly comparable capability. Both tools detect the
Rules File Backdoor attack vector.

### Cisco's Approach: YARA Rule

```
prompt_injection_unicode_steganography.yara
```

- Detects: ZWJ, ZWS, ZWNJ, RTL/LTR overrides, line/paragraph separators,
  variation selectors, Unicode tag patterns
- **Threshold-based:** Requires >50 zero-width chars + decode function,
  OR >200 zero-width chars alone, for ZWJ/ZWS/ZWNJ detection
- **Always detects:** RTL/LTR overrides, line/paragraph separators
  (no threshold; any occurrence is flagged)
- **Configurable:** Policy-based thresholds for different contexts

### vet-skill Approach: grep -P + Fallbacks

```
Step 1d: grep -Pn Unicode character class patterns
```

- Detects: Same character set (ZWJ, ZWS, bidi, tags, whitespace, variation selectors)
- **No threshold:** Any occurrence is flagged
- **Platform-dependent:** grep -P fails on Windows; fallback chain needed
- **Not configurable:** Binary detection (found/not found)

### Key Difference

Cisco's YARA rule has a **50-char threshold for ZWJ/ZWS/ZWNJ** when
no decode function is present. This means our T01 vector (a single ZWJ
in "Profiling‍ Environment Setup") would likely be **suppressed by Cisco
as a false positive** unless the policy is set to strict mode with
threshold 1. Our vet-skill flags it immediately (threshold = 1).

However, Cisco would catch T02 (RTL/LTR overrides) with no threshold,
same as us.

**Implication:** For single-character steganographic injections (the
most subtle attack), vet-skill's zero-threshold approach is more
sensitive but more prone to FPs. Cisco's threshold approach is more
practical for real-world scanning but could miss low-count injections.

---

## Threat Taxonomy Comparison

### Cisco: AITech Taxonomy (Formal)

Maps every finding to AITech codes (AITech-1.1, AITech-8.2, etc.)
with subtechnique codes (AISubtech-1.1.1). Cross-references OWASP,
MITRE ATLAS, and CWE.

Categories: Prompt Injection, Data Exfiltration, Tool Poisoning, Tool
Shadowing, Command Injection, Skill Discovery Abuse, Transitive Trust
Abuse, Autonomy Abuse, Tool Chaining Abuse, Hardcoded Secrets,
Obfuscation, Unauthorized Tool Use, Social Engineering, Resource Abuse,
Supply Chain Attack, Credential Harvesting, System Manipulation

### vet-skill: Informal Per-Step Classification

Categories: Invisible characters, Injection signatures, Credential
exposure, URL classification, PHI/PII, Shell invocations, File write
scope, Priority conflicts, Behavioral modification, Context poisoning

**Gap:** vet-skill doesn't map to any formal taxonomy. Adding AITech
codes to the report would improve interoperability and make findings
comparable across tools.

---

## LLM Analysis Comparison

### Cisco's LLM Prompt

- **Protection:** Random delimiter tags (`<!---UNTRUSTED_INPUT_START_[random]-->`)
  to prevent prompt injection from the skill under analysis
- **Framework:** 10-section analysis framework (manifest, instructions, scripts,
  cross-component, references, behavioral, discovery, supply chain, tool
  chaining, cross-context)
- **Output:** Enforced JSON schema with AITech codes
- **Budget gating:** Per-file and total character limits prevent context overflow
- **Enrichment:** Static analysis findings are fed to the LLM as pre-scan context

### vet-skill's LLM Analysis

- **Protection:** Sequential gating: content doesn't enter LLM until after
  binary/Unicode scan passes (Steps 1-2 are grep-only)
- **Framework:** Two focused questions (goal hijacking + context poisoning)
- **Output:** Free-form markdown, cross-validated against grep findings
- **No budget gating:** Relies on subagent batching for large packages
- **No enrichment:** Each step is independent (grep findings aren't passed to LLM)

**Key insight:** Cisco feeds static findings TO the LLM as context,
making the LLM analysis more targeted. vet-skill keeps steps independent,
then cross-validates at the end. Cisco's approach is more efficient;
vet-skill's approach is more resilient to LLM manipulation (the LLM
can't rationalize away a grep finding it never sees).

---

## What We Should Adopt from Cisco

### High Priority

1. **CI/CD integration.** Even a simple shell script that runs vet-skill
   and parses the output would be valuable. SARIF output for GitHub
   Code Scanning is the gold standard.

2. **AITech taxonomy mapping.** Add AITech codes to our findings for
   interoperability. Use Cisco's mapping as reference.

3. **Enrichment context.** Feed Step 2 grep findings into Step 5 LLM
   analysis as "pre-scan context" so the LLM can focus on flagged files.

4. **Structured output schema.** Even if we stay markdown, define a
   consistent section format that downstream tools can parse.

### Medium Priority

5. **Threshold-based Unicode detection.** Our zero-threshold catches
   everything but will FP on legitimate emoji and CJK text. Add
   configurable thresholds with a strict default.

6. **Budget gating for large packages.** The vet-skill should track
   how much content enters the LLM context and warn when approaching
   limits.

7. **Random delimiter protection.** When passing file content to
   subagents in Steps 3-5, wrap it in random delimiters to resist
   in-context injection.

### Low Priority

8. **YARA integration.** If the host machine has yara-x installed,
   use it for Steps 1-2 instead of grep. Fallback to grep if not.

9. **VirusTotal integration.** Useful for binary skill packages but
   low priority for markdown rule files.

---

## What Cisco Should Adopt from Us

1. **Context poisoning detection.** Cisco has no explicit check for
   false assertions about credential locations, system architecture,
   or data flows. This is a significant blind spot for attacks that
   work by *planting beliefs* rather than *executing commands*.

2. **Sequential gating.** Cisco loads all content into all engines
   simultaneously. A poisoned file could theoretically influence the
   LLM analyzer before YARA results are available. vet-skill's
   "grep before read" principle prevents this.

3. **Priority conflict analysis.** Cisco doesn't compare skill rules
   against the host project's safety rules. A skill that says "this
   supersedes all other rules" should be compared against the project's
   CLAUDE.md/AGENTS.md to verify the conflict.

4. **Behavioral testing framework.** Cisco explicitly notes "no
   behavioral testing" as a limitation. Our Pass A (unprotected agent)
   + Pass B (detection) two-pass protocol could complement their
   static analysis.

5. **Healthcare/PHI patterns.** Domain-specific detection for clinical
   research environments.

---

## Can They Be Used Together?

Yes. The optimal pipeline would be:

```
1. Cisco skill-scanner (pre-commit / CI gate)
   → YARA + AST + dataflow catches code-level threats
   → LLM analyzer catches semantic threats in scripts
   → SARIF output for GitHub Code Scanning

2. vet-skill (developer-facing review)
   → Context poisoning detection
   → Priority conflict analysis against project rules
   → Healthcare PHI/PII patterns
   → Human-readable report for manual review decision

3. Pass A behavioral test (sandbox)
   → Actually run the skill in an isolated environment
   → Measure what the agent does vs. what the skill claims
   → Canary file detection for empirical verification
```

Each layer catches things the others miss. Cisco is strongest at code
analysis (AST, dataflow, YARA). vet-skill is strongest at LLM-context
threats (context poisoning, priority conflicts, behavioral modification).
Pass A behavioral testing is strongest at empirical verification.
