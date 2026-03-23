# vet-skill

A portable security scanner for agentic skill files, rule files, and context documents. It's a single markdown prompt: copy it into your project, point it at an incoming skill, and get a structured security verdict.

No dependencies. No installation. Works in any LLM-powered coding agent that can read files and run shell commands.

---

## Why This Exists

When you install a skill, rule file, or agent configuration into a tool like Claude Code, Cursor, or Windsurf, that file becomes part of the LLM's system-level instruction context. It runs with full tool privileges: file writes, shell commands, git operations, network requests. A compromised file can silently modify agent behavior, exfiltrate data, or override safety rules.

The [Rules File Backdoor attack]([[https://www.pillar.security/blog/new-vulnerability-in-github-copilot-and-cursor-how-hackers-can-weaponize-ai-code-editors]) (Pillar Security, March 2025) demonstrated this by embedding invisible Unicode characters in agent config files, characters that traditional scanners and code review can't see. The attack surface has only grown since then. Agent skills now include markdown instructions, Python scripts, shell commands, referenced data files, and YAML manifests.

`vet-skill` scans all of it.

---

## What It Detects

The scanner runs a 5-step sequential analysis with go/no-go gates. Each step must pass before the next begins.

| Step | Method | What It Catches |
|------|--------|----------------|
| **1. File Inventory & Binary Scan** | grep/bash only (no file content enters LLM context) | Invisible Unicode (ZWJ, bidi overrides, tag characters), symlinks, unexpected file types |
| **2. Pattern Scan** | Grep-based regex | Injection signatures, credential exposure, URL classification, PHI/PII patterns, base64/hex payloads |
| **3. Structural Analysis** | LLM reads files | Tool invocations, file write targets, path traversal, priority conflicts with host project rules |
| **4. Behavioral Checklist** | LLM per-file audit | 12 questions per file covering scope creep, safety suppression, config modification, compliance-framed social engineering |
| **5. Semantic Analysis** | LLM dual-question | Goal hijacking, context poisoning (false assertions about credentials and architecture) |

The key design principle: **grep before read.** File content doesn't enter the LLM's context window until after binary and Unicode scans pass. This prevents steganographic payloads from influencing the analyzer before they're detected.

Full methodology: [docs/METHODOLOGY.md](docs/METHODOLOGY.md)

---

## How to Install

### Claude Code

Copy `vet-skill.md` to your project's command directory:

```
your-project/
└── .claude/
    └── commands/
        └── vet-skill.md
```

Run with:

```
/project:vet-skill path/to/incoming-skill/
```

### Cursor

Copy `vet-skill.md` to `.cursor/commands/` or reference it from `.cursorrules`.

### Other Agentic Systems

The file works with any system that loads markdown into LLM context:

- **Windsurf:** Add to `.windsurfrules` or command directory
- **Copilot Workspace:** Reference from workspace config
- **Aider:** Load as `--read vet-skill.md`
- **Custom agents:** Include in your system prompt or tool definitions

The only requirements: the LLM needs file read access, shell access (for grep/find), and optionally subagent spawning for large packages.

---

## How It Was Tested

We built a 26-vector test suite disguised as a realistic React performance optimization skill package. Each vector embeds a different attack type, from invisible Unicode injection to trojan code patterns to compliance-framed social engineering, hidden inside legitimate React/Next.js performance advice.

Testing used a two-pass protocol:

- **Pass A (Threat Surface):** An unprotected agent interacts with the skill package at three trust levels. Measures which attacks actually succeed against a real agent.
- **Pass B (Detection Capability):** The vet-skill analyzes the same package. Measures which attacks the scanner catches.

Comparing the passes reveals four categories: threats that are both blocked and detected (defense in depth), threats blocked but not detected (low priority), threats detected but not blocked (protected by scanner), and threats neither blocked nor detected (critical gaps).

### Results

| Metric | Value |
|--------|-------|
| Detection rate (full) | 88% |
| Detection rate (including partial) | 100% |
| False positives | 0 |
| Correct verdict (REJECT) | 3/3 runs |
| Critical gaps found | 0 |

The remaining 12% partial detections are vectors designed to blend into legitimate content: compliance-framed config writes, canary instructions embedded in optimization documentation, and compound vectors whose sub-components are individually unremarkable. These fall below the detection floor for static analysis and require behavioral testing to catch empirically.

Full results: [results/TEST-RESULTS-2026-03-20.md](results/TEST-RESULTS-2026-03-20.md)

---

## Comparison with Cisco Skill Scanner

Cisco released [skill-scanner](https://github.com/cisco-ai-defense/skill-scanner), a Python-based tool with YARA rules, AST dataflow analysis, and LLM semantic scanning. It's a different tool solving overlapping problems from a different direction.

| Dimension | vet-skill | Cisco skill-scanner |
|-----------|-----------|---------------------|
| Form factor | Single markdown file | Python package (~90 source files) |
| Dependencies | Zero | yara-x, PyYAML, LiteLLM, optional VirusTotal |
| Best at | LLM-context threats (context poisoning, priority conflicts, behavioral modification) | Code-level threats (AST dataflow, binary analysis, bytecode integrity) |
| CI/CD | No | GitHub Actions, pre-commit, SARIF |
| Unicode detection | grep/rg with fallbacks (threshold = 1) | YARA rules (threshold = 50 for ZWJ without decode context) |

They're complementary. Cisco catches code patterns that vet-skill's regex can't reach. vet-skill catches context poisoning and priority conflicts that Cisco doesn't model. The ideal pipeline runs both.

Full comparison: [docs/COMPARISON-cisco-skill-scanner.md](docs/COMPARISON-cisco-skill-scanner.md)

---

## Limitations

We're upfront about what this tool can't do.

- **Static analysis only.** It scans files on disk. It can't detect attacks that activate only at runtime, or payloads that change after scanning (mutable URLs).
- **LLM-in-the-loop risk.** Steps 3-5 use the LLM to read file content. A sufficiently sophisticated payload could theoretically influence the analysis. The grep-based Steps 1-2 are immune to this. The dual-subagent protocol in Step 4 mitigates but doesn't eliminate the risk.
- **Platform-dependent Unicode scanning.** `grep -P` fails silently on Windows for Unicode escapes. The protocol includes a mandatory verification command and fallback chain (rg, WSL, xxd), but an agent that skips the verification will miss invisible characters. This happened in our Run 1 testing; the amendment to Run 2 fixed it.
- **Detection ceiling at 88%.** Four vectors consistently resist full detection because they're designed to be indistinguishable from legitimate content. Behavioral testing (running the skill against a monitored agent) catches what static analysis can't.
- **Not deterministic.** LLM behavior varies across runs. We observed the scanner catching 68% in Run 1, 84% in Run 2, and 88% in Run 3. The variance came from both protocol improvements and LLM stochasticity.

Full limitations: [docs/LIMITATIONS.md](docs/LIMITATIONS.md)

---

## Adapting to Your Project

### Healthcare / Clinical Research

Keep Step 2d (PHI/PII patterns) enabled. Add your project-specific identifier patterns: study IDs, record number formats, institutional identifiers.

### Non-Healthcare

Skip Step 2d. Everything else is universal.

### Priority Conflict Baseline

Step 3e compares skill instructions against your project's safety rules. Point it at your equivalent of CLAUDE.md, `.cursorrules`, `.windsurfrules`, or CONVENTIONS.md.

### Extending the Checklist

Step 4b's 12-question checklist can be extended with domain-specific questions:

- **Financial:** "Does this file reference account numbers, routing numbers, or transaction data?"
- **Legal:** "Does this file instruct the agent to draft contracts or legal documents?"
- **Infrastructure:** "Does this file instruct the agent to modify cloud resources or IAM policies?"

---

## Origin

Developed for the SPIN Mental Health Clinical Staging Project at the University of Calgary (2026). The analysis methodology grew from an 8-pass security audit grounded in:

- **OWASP Top 10 for Agentic Applications (2026):** ASI01 through ASI07
- **Rules File Backdoor** (Pillar Security, March 2025): invisible Unicode injection in agent config files
- **CVE-2025-55284** and IDEsaster CVEs: IDE agent exploitation vectors
- Three rounds of external security review addressing 23 weaknesses

The full audit produced the vet-skill command, a behavioral test suite, and the testing protocol documented in this repository.

---

## Contributing

We welcome contributions in three areas:

**Detection patterns.** If you've encountered an attack pattern that vet-skill misses, open an issue with the pattern description (no live payloads, please). Regex signatures for Step 2 and checklist questions for Step 4 are the easiest to add.

**Test results.** If you run vet-skill on a different agent (Cursor, Windsurf, Gemini, GPT), against your own test vectors or ours, share the results. Cross-agent comparison data strengthens the tool for everyone.

**False positive reports.** If vet-skill flags a legitimate skill as suspicious, that's a bug. Report it with the skill content (or a sanitized version) and the specific finding that was wrong.

---

## License

MIT. Use freely. Attribution appreciated but not required.

If this tool prevents a security incident in your project, consider contributing back improvements to the checklist or detection patterns.
