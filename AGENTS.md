# Daedalus Agent Instructions

This repository is a documentation and technical analysis workspace. It does not
contain application source code by default. Follow these instructions for Codex
and other coding agents working in this repository.

## Repository Layout

```tree
daedalus/
├── hardware/      # Hardware analysis: CPU, HW/SW interface, automotive, etc.
├── software/      # Software analysis: architecture, specifications, etc.
├── references/    # Reference materials: papers, books, articles
└── tools/         # Utilities and scripts
```

Every top-level directory and major subdirectory should have a `README.md` that
states its scope and indexes the documents inside it. When adding a document,
update the nearest README index in the same change. When creating a new
subdirectory, add its `README.md` in the same change.

Create subdirectories only when they are needed. Do not leave empty placeholder
directories.

## File Naming

- Documents: `{category}-{description}.md`, for example
  `isa-riscv-intro.md`
- Diagrams: `{type}-{description}.{ext}`, for example
  `diagram-system-architecture.svg`
- Scripts: `{purpose}_{description}.{ext}`, for example
  `analyze_protocol.py`

Use English-only, lowercase, hyphenated file names.

## Language Policy

- Major documents are written in Japanese.
- On first use, include the English original for technical terms, for example
  `命令セット (ISA)`.
- Code comments should usually be English.
- File names are English only.

## Document Standards

Use bilingual section titles in major documents, matching the existing style.

Analysis documents should include:

1. Overview / 概要
2. Methodology / 方法論
3. Findings / 発見事項
4. Architecture / アーキテクチャ
5. Technical Details / 技術詳細
6. Security Considerations / セキュリティ考察
7. Conclusions / 結論

Design documents should include:

1. Purpose / 目的
2. Requirements / 要件
3. Design / 設計
4. Rationale / 根拠
5. Trade-offs / トレードオフ
6. Implementation Notes / 実装ノート

Hardware specifications should include component ID, electrical characteristics,
physical dimensions, timing diagrams, operating conditions, and reference
datasheets.

Software specifications should include system requirements, API definitions,
protocol specs, data structures or schemas, error handling, performance
requirements, and version compatibility.

## Evidence, Citations, and Research

Every substantive technical claim needs evidence: a source link, file and line
reference, datasheet, paper, or clearly stated observation.

Citation formats:

- Papers: `Author, A. (Year). Title. *Journal*, vol(issue), pages. DOI`
- Web: `[Title](URL) - Accessed: YYYY-MM-DD`

Store source material under `references/papers/`, `references/books/`, or
`references/articles/`, creating the subdirectory when first needed.

For web or time-sensitive research, verify current facts before writing final
claims.

### Source Priority and Conflicts

Prefer primary, verifiable sources, and reconcile disagreements honestly instead
of choosing a single number.

- Priority, highest to lowest:
  1. Primary or peer-reviewed: official datasheets and specifications, standards
     bodies (ISO, SAE, IEEE, RFC, and similar), peer-reviewed papers, and
     first-party measurements.
  2. Reputable secondary: specialist press and expert technical explainers.
  3. Tertiary: wikis, forums, and blogs. Use these only to find sources; cite the
     primary source they reference, not the wiki or blog itself.
- Official material is authoritative for a vendor's own facts but is also
  marketing. Separate target or specification figures from demonstrated or
  as-built figures, and say which one a number is.
- An independent measurement or teardown can be stronger evidence than a
  first-party claim.
- When sources conflict, take the higher tier. When same-tier sources disagree, or
  a figure is still changing, do not assert one value: give the range, the access
  date, and the target-versus-demonstrated distinction.

Match the source to the kind of claim: enduring principles to textbooks,
peer-reviewed work, or standards; product-specific figures to first-party sources
cross-checked against an independent one; time-sensitive status to dated reporting
framed as "as of <date>". Triangulate load-bearing quantitative claims across at
least two independent sources, ideally from different tiers.

## Diagrams

- Prefer Mermaid inline diagrams for architecture, sequence, and state diagrams.
- Use PlantUML or draw.io only when Mermaid is not expressive enough.
- If using PlantUML or draw.io, commit both the source and a rendered version,
  preferably SVG.
- Include a legend for custom notation.

## Scripts

Scripts should include a short header:

```python
"""
Script: {name}
Purpose: {what it does}
Usage: {how to run it}
Dependencies: {required tools/libraries}
Author: Codex @ Daedalus
Date: {YYYY-MM-DD}
"""
```

## Quality Rules

- Default analysis depth is Level 3: complete system understanding, including
  protocols, edge cases, failure modes, security, and optimization.
- Do not leave placeholder text such as TODO or TBD in delivered documents.
- Do not commit secrets, API keys, tokens, passwords, or private credentials.
  Redact sensitive values as `█████████`.
- Keep edits scoped to the requested document or closely related index files.

## Git Conventions

Commit message format:

```text
<type>(<scope>): <subject>
```

Types: `docs`, `analysis`, `spec`, `tools`, `refactor`, `fix`, `chore`.

Changes are merged via pull requests. Since this is a documentation and analysis
repository, PRs do not require an application test plan unless scripts or tools
are changed.

## Security and Ethics

Permitted work includes public system analysis, interoperability reverse
engineering, defensive security research, educational documentation, and
protocol documentation.

Do not produce or commit malicious tools, proprietary source code, undisclosed
exploit details, piracy workflows, or instructions for unauthorized access.
