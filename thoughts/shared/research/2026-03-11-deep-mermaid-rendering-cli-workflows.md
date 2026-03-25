---
date: 2026-03-11T17:30:00-04:00
researcher: Claude
git_commit: 5eeb027
branch: feat/ENG-2440-triage-agent-rbsm-oracle
repository: filescience
topic: "Best mechanisms for rendering Mermaid state diagrams from CLI-first workflows"
tags: [deep-research, testing, model-based-testing, developer-experience]
status: complete
research_depth: standard
iterations_completed: 3
last_updated: 2026-03-11
last_updated_by: Claude
---

# Deep Research: Mermaid Rendering in CLI-First Workflows

## Research Question
Best mechanisms for rendering/previewing Mermaid state diagrams from a CLI-first workflow (Claude Code). Need the lowest-friction path from "Claude generates diagram" to "user sees rendered visual" and codify it into our process.

## Summary
The best approach is **mermaid.live URL encoding** — Claude writes a `.mmd` file, runs `scripts/mermaid_url.py` to generate a URL, and presents both a terminal-friendly transition table and a clickable mermaid.live link. Zero install required (Python stdlib only), ~30ms to encode, and the user gets an interactive editor in their browser. This should be codified by updating `.claude/rules/model-based-testing.md` to add a rendering step — not as a standalone skill, since the rule already owns the shaping and planning phases where diagrams are generated.

## Perspectives Explored
1. **CLI tooling** — mmdc works but requires 150-300MB Chromium download and 3-8s cold starts. Too heavy for interactive refinement loops.
2. **Editor/IDE integration** — VS Code needs an extension, Obsidian has bugs with stateDiagram-v2, neither can be driven to preview from CLI. Minimum 2 manual steps.
3. **URL-based rendering** — Clear winner. mermaid.live accepts pako-encoded URLs, opens interactive editor, zero install. Helper script exists.
4. **Claude Code extension surface** — No existing rendering MCP. Best codification is a rule update, not a standalone skill. The rule already controls the workflow phases.
5. **ASCII/text alternatives** — No reliable Mermaid-to-ASCII for stateDiagram-v2. Transition tables work as a terminal supplement.

## Detailed Findings

### CLI Tooling (mmdc)
The official Mermaid CLI (`@mermaid-js/mermaid-cli`, command `mmdc`) installs via npm but requires Puppeteer as a peer dependency, which downloads a bundled Chromium browser (~150-300 MB). Cold-start render time is 3-8 seconds due to Chromium launch overhead. It supports `stateDiagram-v2` and outputs SVG, PNG, or PDF. On macOS, output files can be opened with `open <file>`.

Lightweight alternatives exist: **Kroki** (self-hostable HTTP API), **mermaid.ink** (free hosted API), and a Python-native `mmdc` using PhantomJS instead of Chromium. None require the heavy Chromium dependency.

Verdict: Too heavy for an interactive refinement loop where Claude generates → user reviews → Claude adjusts → user reviews again.

### Editor/IDE Integration
VS Code does **not** render Mermaid natively in its Markdown preview — the "Markdown Preview Mermaid Support" extension (bierner.markdown-mermaid) is required. Native support is still pending as of 2026 (GitHub issue #251616). Obsidian renders Mermaid natively in preview mode but has known bugs with complex `stateDiagram-v2` syntax and is pinned to Mermaid 11.4.x.

Neither tool can be driven into preview mode from the CLI. VS Code's `code` command opens the file in the editor; preview requires Cmd+Shift+V (same tab) or Cmd+K V (side-by-side). An "Auto-Open Markdown Preview" extension exists but adds another install requirement.

Verdict: 2+ manual steps minimum. Not automatable from Claude Code.

### URL-Based Rendering (WINNER)
Both mermaid.ink and mermaid.live share the same pako encoding scheme:
1. Wrap diagram in JSON: `{"code": "<diagram>", "mermaid": {"theme": "default"}}`
2. Compress with zlib raw deflate (level 9, `wbits=-15`)
3. Base64-encode with URL-safe characters
4. Prefix with `pako:`

URLs:
- **mermaid.ink**: `https://mermaid.ink/img/pako:<encoded>` — renders static image
- **mermaid.live**: `https://mermaid.live/edit#pako:<encoded>` — opens interactive editor

A helper script at `scripts/mermaid_url.py` implements this using Python stdlib only (`zlib` + `base64`). Verified end-to-end: processes a 44-line triage agent diagram in ~30ms, producing a 1,219-character URL. Both file and inline modes work.

For a state diagram with ~7 states and ~10 transitions, the encoded URL is well under browser URL limits (~2000 chars for IE, 8000+ for modern browsers).

The mermaid.live editor is the preferred target because the user can:
- See the rendered diagram immediately
- Edit the Mermaid source and see live updates
- Copy the refined diagram back

### Claude Code Extension Surface
Chrome DevTools MCP is disabled in settings. No rendering-capable MCP servers are active. The existing `generate-state-machine-test` skill consumes `.mmd` files but has no rendering step.

The `model-based-testing.md` rule already owns the `/shape` and `/create_plan` workflow phases where diagrams are generated. Updating the rule is the right codification approach because:
- Rules drive inline process behavior during routing phases
- No `/shape` skill exists — shape is purely CLAUDE.md route B routing
- A standalone `/render-mermaid` skill would need explicit invocation by Claude rather than being automatic
- Writing the `.mmd` file as part of the rule step also feeds the downstream `generate-state-machine-test` skill

### ASCII/Text Alternatives
No Mermaid-to-ASCII tool reliably supports `stateDiagram-v2`. The `mermaid-ascii` (Go) project only handles flowcharts. General ASCII graph renderers (Graph::Easy, PHART) can handle the machine but readability degrades with edge crossings.

**Transition tables** (from-state × event → to-state) are compact, unambiguous, and scale well in terminal for machines of this size. They serve as the terminal supplement alongside the browser-rendered diagram.

## Recommendation: Dual Output via Rule Update

### What to change
Update `.claude/rules/model-based-testing.md` to add a rendering step:

**During shaping (`/shape`):**
1. Identify states, events, transitions (existing)
2. Generate Mermaid `stateDiagram-v2` (existing)
3. **NEW: Write the diagram to a `.mmd` file in the session directory**
4. **NEW: Run `python scripts/mermaid_url.py <file>` to generate a mermaid.live URL**
5. **NEW: Present BOTH a transition table in terminal AND the mermaid.live link**
6. Await user alignment before proceeding (existing)

**During planning (`/create_plan`):**
- Embed the `.mmd` file reference in the plan's State Machine section
- Re-generate the URL if the diagram changed during refinement

### Terminal output format
```
## State Machine: [feature name]

| From | Event | Guard | To | Action |
|------|-------|-------|----|--------|
| [*]  | new_message | — | processing | save_thread(...) |
| ...  | ... | ... | ... | ... |

Rendered diagram: [mermaid.live URL]
```

### Files involved
- `.claude/rules/model-based-testing.md` — add rendering step to shaping + planning
- `scripts/mermaid_url.py` — already exists, no changes needed
- `.claude/skills/generate-state-machine-test/SKILL.md` — optionally add rendering step between extraction and code generation

## Key Sources
### Codebase
- `.claude/rules/model-based-testing.md:6-13` — shaping step where diagrams are generated
- `.claude/skills/generate-state-machine-test/SKILL.md:20,40` — skill that consumes .mmd files
- `scripts/mermaid_url.py` — helper script (Python stdlib, ~30ms)
- `.claude/skills/generate-state-machine-test/examples/triage-agent.mmd` — example diagram

### External
- [mermaid.live](https://mermaid.live/) — interactive editor
- [mermaid.ink](https://github.com/jihchi/mermaid.ink) — static image rendering API
- [Pako encoding from Python](https://github.com/mermaid-js/mermaid-live-editor/discussions/1291)
- [mermaid-live-editor serde.ts](https://github.com/mermaid-js/mermaid-live-editor/blob/develop/src/lib/util/serde.ts) — canonical encoding reference
- [VS Code Mermaid extension](https://marketplace.visualstudio.com/items?itemName=bierner.markdown-mermaid)
- [Obsidian Mermaid bugs](https://forum.obsidian.md/t/mermaid-fails-to-render-a-valid-statediagram/56252)

## Open Questions
- Should the mermaid.live URL be auto-opened in the browser (`open` command), or just presented as a clickable link for the user to click? Auto-open is lower friction but may be surprising.
- Should the `.mmd` file live in the session directory (ephemeral) or alongside the plan in memory-bank (persistent)?

## Research Metadata
- Depth mode: standard
- Iterations completed: 3 / 5
- Termination reason: gaps exhausted
- Manifest: `.claude/deep-research/2026-03-11-mermaid-rendering-cli-workflows.md`
