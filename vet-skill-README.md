# vet-skill — Agentic Skill Intake Security Analysis

**Version:** 1.0.0
**Last updated:** 2026-03-19
**Methodology version:** Based on SPIN Security Audit v3 (8-pass, 3 rounds external review)

## What It Does

`vet-skill` is an on-demand security analysis command for vetting incoming skills, prompts, rules, and context files **before** they are installed into an agentic coding environment.

When you install a skill or agent configuration file, that file becomes part of the LLM's system-level instruction context and executes with full tool privileges (file writes, shell commands, git operations, network requests). A compromised file can silently modify agent behavior, exfiltrate data, or override safety rules.

This command analyzes incoming files for:
- **Invisible character injection** (Rules File Backdoor attack)
- **Instruction override / prompt injection** patterns
- **Supply chain vectors** (mutable URLs fetched at runtime)
- **Credential and sensitive data exposure**
- **Behavioral modification** beyond stated scope
- **Context poisoning** (false assertions about system architecture)
- **Tool misuse** (destructive commands, unsafe file writes)
- **Priority conflicts** with your project's safety rules

## Origin

Derived from an 8-pass security audit methodology developed for the SPIN Mental Health Clinical Staging Project (University of Calgary, 2026). The audit was grounded in:

- **OWASP Top 10 for Agentic Applications (2026)** — ASI01 through ASI07
- **Rules File Backdoor** (Pillar Security, March 2025) — invisible Unicode injection in agent config files
- **CVE-2025-55284** and IDEsaster CVEs — IDE agent exploitation vectors
- Three rounds of external security review addressing 23 weaknesses

The full audit methodology (plan + findings) is documented in the source repository's `documents/handoffs/` directory. If you are using this command in a different project, the methodology is fully self-contained in `vet-skill.md` — no external documents are required.

## Installation

### Claude Code

Copy `vet-skill.md` to your project's `.claude/commands/` directory:

```
your-project/
└── .claude/
    └── commands/
        └── vet-skill.md
```

Invoke with:
```
/project:vet-skill path/to/incoming-skill/
```

### Cursor

Copy `vet-skill.md` to `.cursor/commands/` or reference it from `.cursorrules`. Invoke from the command palette or chat.

### Other Agentic Systems

The command is a standalone markdown file. It works with any system that supports loading markdown instructions into LLM context:
- **Windsurf:** Add to `.windsurfrules` or equivalent command directory
- **Copilot Workspace:** Reference from workspace configuration
- **Aider:** Load as a context file with `--read vet-skill.md`
- **Custom agents:** Include in your system prompt or tool definitions

The only requirement is that the LLM has access to:
1. **Bash/shell** — for grep, find, and file enumeration (Steps 1-2)
2. **File read** — for structural and semantic analysis (Steps 3-5)
3. **Subagent/tool spawning** (optional) — for dual-subagent verification on large packages

If your system doesn't support subagents, Steps 4-5 run in the main context. The analysis is still valid; you lose the isolation benefit of batched subagent processing.

## Dependencies

**None.** The command uses only standard Unix tools:

| Tool | Used In | Purpose |
|------|---------|---------|
| `find` | Step 1 | File enumeration, symlink detection |
| `grep` (with `-P` for PCRE) | Steps 1-2 | Pattern matching, Unicode detection |
| `sort` | Step 1 | File listing |

All pattern matching uses POSIX-compatible regex with optional PCRE extensions for Unicode scanning. If your system's `grep` doesn't support `-P` (Perl regex), substitute with `rg` (ripgrep) or `perl -ne`.

**No external packages, APIs, or network access required.**

## How It Works

The analysis runs 5 sequential steps with go/no-go gates:

```
Step 1: File Inventory & Binary Scan ──── REJECT if invisible chars found
    │
    ▼
Step 2: Pattern Scan (grep-based) ─────── REJECT if injection signatures confirmed
    │
    ▼
Step 3: Structural Analysis ──────────── HOLD if unsafe tool invocations
    │
    ▼
Step 4: Behavioral Modification ───────── HOLD if out-of-scope modifications
    │
    ▼
Step 5: Semantic Analysis (LLM) ───────── Final safety net
    │
    ▼
Report: PASS / PASS WITH WARNINGS / HOLD / REJECT
```

### Step 1 — File Inventory & Binary Scan
Enumerates files, validates types, detects symlinks, and scans for invisible Unicode characters (the Rules File Backdoor attack vector). **No file content enters the LLM context at this step** — it's pure byte-level grep.

### Step 2 — Pattern Scan
Grep-based detection of injection signatures, credential patterns, sensitive data, and URLs. Zero-trust URL policy: every URL is flagged and classified. Again, only pattern matches enter context — not full file contents.

### Step 3 — Structural Analysis
Reads files to analyze tool invocation patterns, $ARGUMENTS flow, file write scope, external file references, and priority conflicts with the host project's safety rules. For packages with >15 files, analysis is batched into isolated subagent chunks of ~10 files each.

### Step 4 — Behavioral Modification Checklist
10-question checklist applied to every file: does it install dependencies, fetch URLs, write files, suppress warnings, skip validation, modify configs, or alter behavior outside its stated scope?

For large packages (>30 files), uses dual-subagent verification — two independent LLM subagents analyze each batch, and discrepancies trigger manual review.

### Step 5 — Semantic Analysis
Two LLM-driven questions catch what regex and checklists miss:
1. **Behavioral modification (ASI01)** — text that alters agent behavior even when phrased as advice or examples
2. **Context poisoning (ASI06)** — false assertions about system architecture, credential locations, or data flows

Results are cross-validated against Step 2 grep findings.

## Verdicts

| Verdict | Meaning |
|---------|---------|
| **PASS** | Zero CRITICAL findings. Safe to install. |
| **PASS WITH WARNINGS** | Zero CRITICAL findings. Notable but non-threatening concerns documented. |
| **HOLD** | CRITICAL findings that might have legitimate justification. Do NOT install until a human reviews. |
| **REJECT** | Confirmed malicious patterns. Do NOT install. |

## Threat Model

The command addresses these threat actors:

| Actor | Attack | Detection Step |
|-------|--------|---------------|
| **Malicious contributor** | Poisoned PR adding injection to skill files | Steps 1, 2, 5 |
| **Supply chain compromise** | Third-party skill package with hidden instructions | Steps 2, 4, 5 |
| **Invisible character injection** | Rules File Backdoor — zero-width Unicode in config files | Step 1 |
| **Context poisoning** | False assertions about credential locations or architecture | Step 5 |
| **Dynamic loading chain** | Skill references mutable external URL, content changes after review | Step 2 |
| **Scope creep** | Skill does more than it claims — modifies configs, installs packages | Steps 3, 4 |

### OWASP Agentic Top 10 (2026) Coverage

| ASI | Category | Covered In |
|-----|----------|------------|
| ASI01 | Agent Goal Hijacking | Steps 2, 4, 5 |
| ASI02 | Tool Misuse | Steps 3a, 3b, 3c |
| ASI03 | Identity & Privilege Abuse | Steps 2c, 3e |
| ASI04 | Agentic Supply Chain | Steps 2b, 4b (Q2) |
| ASI05 | Unexpected Code Execution | Steps 1d, 3a, 3b |
| ASI06 | Memory & Context Poisoning | Steps 5a, 5b |
| ASI07 | Insecure Inter-Agent Communication | Step 3d |

## Limitations

1. **Not a runtime monitor.** This is a pre-installation static analysis. It cannot detect attacks that activate only at runtime (e.g., time-delayed payloads, conditional logic based on context window content).

2. **Semantic analysis has inherent limits.** The LLM performing Step 5 could itself be influenced by a sufficiently sophisticated injection in the file it's analyzing. The dual-subagent protocol mitigates this but does not eliminate it.

3. **Single-point-in-time.** The analysis reflects the files as they exist when scanned. If a skill fetches content from a mutable URL (even if flagged as CRITICAL), the fetched content can change after analysis.

4. **No behavioral testing.** This command performs static analysis only. It does not load the skill into a sandbox and test what happens. For behavioral testing, run the skill in an isolated environment with monitored tool access.

5. **Regex is a floor, not a ceiling.** Steps 1-2 catch known patterns. Novel injection phrasing, multilingual content, or encoded payloads may bypass grep. Step 5 (semantic analysis) is the safety net, but it is not guaranteed to catch everything.

6. **LLM-in-the-loop analysis risk.** Steps 3-5 are performed by the same LLM that reads the file content. A sufficiently sophisticated payload could attempt to influence the analysis itself (e.g., suppress a finding or rationalize a vulnerability). The grep-based Steps 1-2 are immune to this because file content does not enter the LLM context. The dual-subagent protocol (Step 4c) mitigates but does not fully eliminate this structural risk.

## Adapting to Your Project

### Healthcare / Clinical Projects
Keep Step 2d (PHI/PII patterns) enabled. Add your project-specific identifier patterns (study IDs, record formats).

### Non-Healthcare Projects
Remove or skip Step 2d. The remaining steps are universal.

### Priority Conflict Baseline
Step 3e compares skill instructions against your project's safety rules. Point it at your equivalent of:
- `CLAUDE.md` (Claude Code)
- `.cursorrules` (Cursor)
- `.windsurfrules` (Windsurf)
- `CONVENTIONS.md` or `AGENTS.md` (custom setups)

### Extending the Checklist
Step 4b's 10-question checklist can be extended. Add domain-specific questions:
- Financial: "Does this file reference account numbers, routing numbers, or transaction data?"
- Legal: "Does this file instruct the agent to draft contracts or legal documents?"
- Infrastructure: "Does this file instruct the agent to modify cloud resources or IAM policies?"

## Example Usage

```bash
# Analyze a skill directory before installing it
/project:vet-skill .agents/skills/incoming-skill/

# Analyze a single file
/project:vet-skill downloaded-rules.md

# Clone and analyze a GitHub skill package
git clone https://github.com/org/cool-skill /tmp/cool-skill
/project:vet-skill /tmp/cool-skill/
```

## License

MIT. Use freely. Attribution appreciated but not required.

If this analysis prevents a security incident in your project, consider contributing back improvements to the checklist or detection patterns.
