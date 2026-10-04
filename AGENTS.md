# Daedalus — Agent Guidelines

Technical analysis repository (documentation only; no application source code).
See `README.md` for the project overview and philosophy.

This file is the single source of shared rules for every coding agent: Codex
reads it directly, and Claude Code imports it from `CLAUDE.md`. Edit shared rules
here; agent-specific additions belong in that agent's own file (e.g.
`CLAUDE.md`).

These guidelines are kept in English for precision. The documents agents
*produce*, however, follow the Language Policy below.

---

## 1. Repository Structure & Conventions

Top-level layout:

```tree
daedalus/
├── hardware/      # Hardware analysis (CPU, HW/SW interface, automotive, ...)
├── software/      # Software analysis (architecture, specifications, ...)
├── foundations/   # Foundational theory (mathematics, physics, ...) behind the analyses
└── references/    # Reference materials (papers, books, articles)
```

Conventions:

- **Per-directory README**: every top-level directory and every major
  subdirectory contains a `README.md` that states its scope and indexes the
  documents inside it. When adding a new document, update the nearest README's
  index. When creating a new subdirectory, add its `README.md` in the same change.
- **Create subdirectories on demand**: e.g. `references/papers/`,
  `references/books/`, `references/articles/` are created when the first item of
  that kind is added — do not leave empty placeholder directories.
- **Scope edits**: keep changes to the requested document and its closely related
  index files.

### File Naming

| Type | Format | Example |
|------|--------|---------|
| Documents | `{category}-{description}.md` | `isa-riscv-intro.md` |
| Diagrams | `{type}-{description}.{ext}` | `diagram-system-architecture.svg` |
| Scripts | `{purpose}_{description}.{ext}` | `analyze_protocol.py` |

File names: English only, lowercase with hyphens.

---

## 2. Language Policy

- **Major documents**: Japanese.
- **Technical terms**: keep the English original alongside the Japanese on first
  use, e.g. `命令セット (ISA)`.
- **Code comments**: English preferred.
- **File names**: English only (see above).

---

## 3. Document Templates

Section headings are `## 日本語 (English)`, e.g. `## 概要 (Overview)` — the
lists below give the English names. Every analysis and foundations document
ends with `## 参考文献 (References)`. In non-software domains (aerospace,
automotive, …) Security Considerations may be replaced by a domain-appropriate
section, e.g. `## 運用・安全考察 (Operational & Safety Considerations)`.

### Analysis Documents — Required Sections

1. Overview / 概要
2. Methodology / 方法論
3. Findings / 発見事項
4. Architecture / アーキテクチャ
5. Technical Details / 技術詳細
6. Security Considerations / セキュリティ考察
7. Conclusions / 結論

### Design Documents — Required Sections

1. Purpose / 目的
2. Requirements / 要件
3. Design / 設計
4. Rationale / 根拠
5. Trade-offs / トレードオフ
6. Implementation Notes / 実装ノート

### Foundations Documents — Required Sections

For documents under `foundations/` (mathematics, physics, and other theory):

1. Overview / 概要
2. Prerequisites / 前提知識 — assumed background and prerequisite documents in the series
3. Main Content / 本論 — definitions, theorems, and intuition, built up from the basics
4. Engineering Connections / 工学への接続 — why the theory matters for engineered systems
5. Applications within Daedalus / Daedalus内の応用 — relative links to the analysis documents this theory supports
6. References / 参考文献 — textbooks and papers per the Citations standard

### Hardware Specs Must Include

Component ID, electrical characteristics, physical dimensions, timing diagrams,
operating conditions, reference datasheets.

### Software Specs Must Include

System requirements, API definitions, protocol specs, data structures/schemas,
error handling, performance requirements, version compatibility.

---

## 4. Writing Standards

### Code in Documents

When quoting code from an analyzed system, always include the language spec, a
context explanation, and the source location with line numbers:

```
**File**: `src/network/protocol.c` (lines 145-160)
**Purpose**: Message framing logic
```

Pseudocode: language-agnostic, include complexity analysis, document assumptions.

### Diagrams

- **Mermaid inline** is the default — it is version-controllable and renders on
  GitHub. Use it for architecture, sequence, and state diagrams.
- Use PlantUML or draw.io only when Mermaid cannot express the diagram; in that
  case commit both the source and a rendered version (SVG preferred, or high-res
  PNG).
- Use consistent notation (UML, C4). Include a legend for any custom notation.

### Citations

- Papers: `Author, A. (Year). Title. *Journal*, vol(issue), pages. DOI`
- Web: `[Title](URL) — Accessed: YYYY-MM-DD`
- Store source material under `references/papers/`, `references/books/`, or
  `references/articles/` (creating the subdirectory as needed).

### Source Hierarchy & Conflicts

Prefer primary, verifiable sources, and reconcile disagreements honestly rather
than picking a single number.

- **Priority (high → low)**:
  1. **Primary / peer-reviewed** — official datasheets & specifications, standards
     bodies (ISO, SAE, IEEE, RFC, …), peer-reviewed papers, first-party measurements.
  2. **Reputable secondary** — specialist press, expert technical explainers.
  3. **Tertiary** — wikis, forums, blogs. Use only to *discover* sources; cite the
     primary source they point to, not the wiki/blog itself.
- **Official ≠ neutral**: first-party material is authoritative for a vendor's own
  facts but is also marketing. Distinguish *target / spec* figures from
  *demonstrated / as-built* ones, and label which a number is.
- **Independent verification can outrank authority**: a third-party measurement or
  teardown may be stronger evidence than a vendor's claim.
- **On conflict**: take the higher tier. When same-tier sources disagree, or a
  fast-moving figure is in flux, do not assert one value — give the range, the
  access date, and the target-vs-demonstrated distinction.

### Script Header

```python
"""
Script: {name}
Purpose: {what it does}
Usage: {how to run it}
Dependencies: {required tools/libraries}
Author: {Agent} @ Daedalus
Date: {YYYY-MM-DD}
"""
```

`{Agent}` is the authoring agent, e.g. `Claude` or `Codex`.

---

## 5. Research Workflow

- Evidence is required for every claim — link to the source, file/line, or
  datasheet that supports it.
- Verify web or time-sensitive facts at writing time rather than relying on
  memory.
- Match the source to the claim: enduring principles → textbooks / peer-reviewed /
  standards; product-specific figures → first-party official cross-checked with an
  independent source; time-sensitive status → dated reporting framed "as of
  <date>".
- Triangulate load-bearing quantitative claims across at least two independent
  sources, ideally crossing tiers; if they cannot be reconciled, report the
  uncertainty per **Source Hierarchy & Conflicts**.

---

## 6. Quality Rules

**Analysis depth** — default to Level 3:

- **Level 1 (Surface)**: overview, main components, basic data/control flow.
- **Level 2 (Functional)**: detailed behavior, key protocols, common edge cases.
- **Level 3 (Complete, default)**: full system understanding — all protocols,
  edge cases, failure modes, security, and optimization.

Other rules:

- No placeholder text (TODO, TBD); all required sections complete before delivery.
- No sensitive information (API keys, tokens, passwords) — redact as `█████████`.
- Every claim is backed by evidence.

---

## 7. Git Conventions

Commit message format:

```
<type>(<scope>): <subject>
```

Types: `docs`, `analysis`, `spec`, `tools`, `refactor`, `fix`, `chore`.
Use `docs(<area>)` for adding or updating documents, where `<area>` is the
subdirectory name (e.g. `docs(compilers-runtimes)`, `docs(foundations)`);
use `chore(<agent>)` for agent configuration (e.g. `chore(claude)`).

Changes are merged via pull requests. Since this is a documentation/analysis
repository (not implementation), PRs do not require a test plan unless scripts
or tools are changed.

---

## 8. Security & Ethics

**Permitted**: public system analysis, interoperability reverse engineering,
defensive security research, educational documentation, protocol documentation.

**Prohibited**: copy-protection circumvention for piracy, malicious tools,
proprietary code sharing, undisclosed exploits, unauthorized analysis.
