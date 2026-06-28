# Daedalus - Claude Guidelines

Technical analysis repository. See README.md for project overview and philosophy.

---

## Language Policy

- All major documents: Japanese
- Technical terms: keep the English original alongside the Japanese (e.g. `命令セット (ISA)`) on first use
- Code comments: English preferred
- File names: English only, lowercase with hyphens

---

## Documentation Templates

### Analysis Documents - Required Sections

1. Overview / 概要
2. Methodology / 方法論
3. Findings / 発見事項
4. Architecture / アーキテクチャ
5. Technical Details / 技術詳細
6. Security Considerations / セキュリティ考察
7. Conclusions / 結論

### Design Documents - Required Sections

1. Purpose / 目的
2. Requirements / 要件
3. Design / 設計
4. Rationale / 根拠
5. Trade-offs / トレードオフ
6. Implementation Notes / 実装ノート

### Hardware Specs Must Include

Component ID, electrical characteristics, physical dimensions, timing diagrams, operating conditions, reference datasheets.

### Software Specs Must Include

System requirements, API definitions, protocol specs, data structures/schemas, error handling, performance requirements, version compatibility.

---

## File Naming

| Type | Format | Example |
|------|--------|---------|
| Documents | `{category}-{description}.md` | `api-specification.md` |
| Diagrams | `{type}-{description}.{ext}` | `diagram-system-architecture.png` |
| Scripts | `{purpose}_{description}.{ext}` | `analyze_protocol.py` |

---

## Diagrams

- Use Mermaid, PlantUML, or draw.io
- Include architecture, sequence, and state diagrams where applicable
- Consistent notation (UML, C4), legend if custom
- Provide source + rendered versions (SVG or high-res PNG)

---

## Code in Documents

Always include: language spec, context explanation, source location, line numbers.

```
**File**: `src/network/protocol.c` (lines 145-160)
**Purpose**: Message framing logic
```

Pseudocode: language-agnostic, include complexity analysis, document assumptions.

---

## Script Header

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

## Citations

- Papers: `Author, A. (Year). Title. *Journal*, vol(issue), pages. DOI`
- Web: `[Title](URL) - Accessed: YYYY-MM-DD`
- Store in `references/papers/`, `references/books/`, `references/articles/`

---

## Commit Message Format

```
<type>(<scope>): <subject>
```

Types: `docs`, `analysis`, `spec`, `tools`, `refactor`, `fix`

---

## Quality Rules

- Default analysis depth: Level 3 (complete system understanding, all protocols, edge cases, security, optimization)
- No placeholder text (TODO, TBD)
- No sensitive information (API keys, tokens, passwords) - redact as `█████████`
- Evidence required for all claims
- All sections complete before delivery

---

## Security & Ethics

Permitted: public system analysis, interoperability RE, defensive security research, educational docs, protocol documentation.

Prohibited: copy protection circumvention for piracy, malicious tools, proprietary code sharing, undisclosed exploits, unauthorized analysis.
