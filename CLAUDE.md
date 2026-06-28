# Daedalus — Claude Guidelines

Technical analysis repository (documentation only; no application source code).
See `README.md` for the project overview and philosophy.

These guidelines are operational instructions for Claude and are kept in English
for precision. The documents Claude *produces*, however, follow the Language
Policy below.

---

## 1. Repository Structure & Conventions

Top-level layout:

```tree
daedalus/
├── hardware/      # Hardware analysis (CPU, HW/SW interface, automotive, ...)
├── software/      # Software analysis (architecture, specifications, ...)
├── references/    # Reference materials (papers, books, articles)
└── tools/         # Utilities and scripts
```

Conventions:

- **Per-directory README**: every top-level directory and every major
  subdirectory contains a `README.md` that states its scope and indexes the
  documents inside it. When adding a new document, update the nearest README's
  index. When creating a new subdirectory, add its `README.md` in the same change.
- **Create subdirectories on demand**: e.g. `references/papers/`,
  `references/books/`, `references/articles/` are created when the first item of
  that kind is added — do not leave empty placeholder directories.

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

Section titles are bilingual (`English / 日本語`) to match existing documents.

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

### Script Header

```python
"""
Script: {name}
Purpose: {what it does}
Usage: {how to run it}
Dependencies: {required tools/libraries}
Author: Claude @ Daedalus
Date: {YYYY-MM-DD}
"""
```

---

## 5. Research Workflow

- Use the `deep-research` skill for multi-source, fact-checked investigation that
  produces a cited report; weave its findings into the relevant document and
  record sources per the Citations standard.
- Evidence is required for every claim — link to the source, file/line, or
  datasheet that supports it.

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

Changes are merged via pull requests. Since this is a documentation/analysis
repository (not implementation), PRs do not require a test plan.

---

## 8. Security & Ethics

**Permitted**: public system analysis, interoperability reverse engineering,
defensive security research, educational documentation, protocol documentation.

**Prohibited**: copy-protection circumvention for piracy, malicious tools,
proprietary code sharing, undisclosed exploits, unauthorized analysis.
