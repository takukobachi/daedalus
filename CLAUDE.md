# Daedalus — Claude Guidelines

@AGENTS.md

The shared rules for all agents live in `AGENTS.md` (imported above) so that
Claude Code and Codex follow a single source. Edit shared rules there; this file
holds only Claude-specific additions.

---

## Claude-Specific Additions

### Research Workflow

Use the `deep-research` skill for multi-source, fact-checked investigation that
produces a cited report; weave its findings into the relevant document and record
sources per the Citations standard.

### Script Header

Use `Author: Claude @ Daedalus` in the Script Header.

### Japanese Prose Refinement (yomiyasu)

The project enables the [yomiyasu](https://github.com/nanaism/yomiyasu) plugin
(`.claude/settings.json`) for refining AI-generated Japanese. Use it only when
explicitly asked (e.g. 「この文章を読みやすくして」), not as a default pass.
Daedalus conventions override the following yomiyasu rules and linter warnings:

- **Spacing around English terms**: keep the existing half-width spaces around
  English words and `命令セット (ISA)`-style annotations; do not remove them.
- **List / bold ratios**: ignore the bullet-ratio and bold-frequency limits.
  Tables and lists for specs, terms, comparisons, and enumerated steps are
  intended; only reasoning (methodology, causal findings, conclusions) should be
  written as prose paragraphs.
