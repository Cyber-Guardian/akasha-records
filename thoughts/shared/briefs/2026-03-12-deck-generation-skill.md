---
type: event
created: 2026-03-12
status: active
---
# Idea Brief: Claude Code Deck Generation Skill

**Date:** 2026-03-12
**Status:** Shaped -> Planning

## Problem
Developers accumulate rich structured context (plans, briefs, research, decisions) but translating that into stakeholder-friendly formats is a manual, high-friction context switch. The knowledge already exists in memory-bank markdown artifacts — the presentation layer is missing. Primary use case: selling stakeholders on new ideas or strategies.

## Constraints
- Output format: PDF (stakeholders don't need to edit)
- Content source: memory-bank markdown artifacts (briefs, plans, research, decisions)
- Must handle Mermaid diagrams (used throughout the codebase for state machines, architecture)
- Python-native monorepo — Node.js is acceptable as a tool dependency (npx one-shot) but not a code dependency
- Chrome/Chromium required on machine for Marp PDF rendering
- Skill must fit the existing `.claude/skills/` pattern

## Options Considered

### Marp Pipeline
Markdown-in, PDF-out via Marp CLI. Claude generates Marp-flavored markdown from memory-bank artifacts, Marp renders to PDF. Custom CSS theme for branding. Mermaid diagrams pre-rendered via `mmdc` (mermaid-cli) to PNG/SVG before embedding.
- Gains: Beautiful CSS-themed PDF output, simple directive syntax (20 directives, all YAML), markdown-native input matches existing artifacts, fast CLI
- Costs: Node.js/npx + Chrome dependency for PDF, Mermaid requires pre-render step via mmdc
- Complexity: Low-Medium

### Quarto Multi-format
Single `.qmd` source renders to reveal.js HTML and/or PDF. Native Mermaid rendering. Could also produce PPTX if needs change.
- Gains: Native Mermaid, multi-format from one source, most complete solution
- Costs: Quarto binary install, heavier tool, more to learn, overkill for PDF-only use case
- Complexity: Medium

### LLM-Structured -> python-pptx
Claude generates structured JSON, Python script fills branded .pptx template via python-pptx, export to PDF via LibreOffice.
- Gains: Total design control, Python-native, no Node.js
- Costs: More moving parts (JSON intermediate, python-pptx script, LibreOffice for PDF), verbose, clunky PDF conversion
- Complexity: High

## Chosen Approach
**Marp Pipeline** — simplest path to beautiful PDF output from markdown. The directive syntax is trivially LLM-generatable (validated via ground check). The Mermaid gap is addressable with a pre-render step using `mmdc`. CSS theming gives full branding control.

## Key Context Discovered During Shaping
- Marp PPTX output is image-based (not editable text) — but PDF is the target, so this doesn't matter
- Marp has NO native Mermaid support (blocked by font scaling in SVG foreignObject) — pre-rendering to PNG/SVG via mermaid-cli is the reliable workaround
- Chrome/Chromium is required on the machine for PDF generation (Marp shells out to headless browser)
- The full Marp directive surface is ~20 directives — well within reliable LLM generation territory
- Prior research on [[2026-02-12-architecture-diagram-automation|Architecture Diagram Automation]] already identified D2 for "stakeholder decks" — this skill supersedes that direction
- Existing `scripts/mermaid_url.py` shows the repo already has Mermaid tooling patterns

## Next Step
- [Plan] -> `/create_plan memory-bank/thoughts/shared/briefs/2026-03-12-deck-generation-skill.md`
