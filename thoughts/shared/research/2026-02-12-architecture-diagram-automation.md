# Architecture Diagram Automation Research

**Date:** 2026-02-12
**Scope:** Diagram types, technologies, and automation strategies for Python/Polylith/AWS monorepo

---

## Part 1: Diagram Types — What Exists and When to Use Each

### C4 Model (Modern Standard)

The C4 model provides four hierarchical abstraction levels. Best-fit for our Polylith + AWS Lambda architecture.

| Level | Shows | Auto-generable? | Our Use Case |
|-------|-------|-----------------|--------------|
| **1 — System Context** | System as box + users + external systems | No (requires human boundary judgment) | FileScience + Microsoft Graph API + tenants |
| **2 — Container** | Independently deployable units | Partial (from Terraform) | Lambda functions, DynamoDB, Valkey, S3 |
| **3 — Component** | Internal structure of containers | Yes (from Python packages) | Polylith bases/components per Lambda |
| **4 — Code** | Class-level detail | Yes (Pyreverse) | Low priority — Python duck typing makes this less valuable |

**Supporting C4 views:** System Landscape (enterprise), Dynamic (runtime sequences), Deployment (infra mapping).

### Sequence Diagrams (High Value)

- **Shows:** Time-ordered interactions between services/components
- **Auto-gen:** Partial (from traces/OpenTelemetry)
- **Our use:** DynamoDB stream -> Lambda -> Valkey -> Dispatcher -> Cloud Router -> Graph API
- **Best tool:** Mermaid

### Dependency / Import Graphs (Critical for Polylith)

- **Shows:** Module/package import relationships, cyclic deps
- **Auto-gen:** Fully automatic (pydeps, Pyreverse, import-linter)
- **Our use:** Enforcing + visualizing Polylith boundaries (7 contracts already in import-linter)
- **Best tools:** pydeps (visualization), import-linter (enforcement — already using)

### Infrastructure Diagrams (Terraform/AWS)

- **Shows:** Cloud resources, networking, data flows between services
- **Auto-gen:** Fully automatic from Terraform state/HCL
- **Our use:** VPC, Lambda, DynamoDB, Valkey, NAT, Bastion topology
- **Best tools:** Rover (interactive), Inframap (static), Terragrunt `dag graph` (dependency chain)

### State Machine Diagrams

- **Shows:** States, transitions, events, guards
- **Auto-gen:** No
- **Our use:** Work-queue item lifecycle (enqueued -> leased -> processing -> finished/failed)
- **Best tool:** Mermaid

### Data Flow Diagrams

- **Shows:** Data movement through processes, transforms, stores
- **Auto-gen:** Difficult (requires business process understanding)
- **Our use:** Entity -> work-queue -> dispatcher -> cloud services -> S3 pipeline
- **Recommendation:** Use selectively for complex pipelines; C4 containers cover most needs

### Other Relevant Types

| Type | Auto-gen? | Relevance | Notes |
|------|-----------|-----------|-------|
| ER diagrams | Partial | Low | DynamoDB single-table design doesn't map well to traditional ER |
| Event flow | Partial | High | DynamoDB streams -> Lambda, async patterns |
| Call graphs | Yes (pyan, code2flow) | Medium | Useful for tracing specific flows |
| Class diagrams | Yes (Pyreverse) | Low | Python dynamic typing reduces value |

### Priority Ranking for Our Codebase

1. **Dependency/import graphs** — Critical (Polylith enforcement + visualization)
2. **Infrastructure diagrams** — High (Terraform auto-gen)
3. **C4 Container diagram** — High (system overview)
4. **Sequence diagrams** — High (discover runtime flow)
5. **State machine** — Medium (work-queue lifecycle)
6. **C4 Context diagram** — Medium (stakeholder view)
7. **Data flow** — Medium (pipeline view)
8. **Class diagrams** — Low

---

## Part 2: Technologies — Comparison

### Diagram-as-Code (Version-Controllable, Text-Based)

#### Mermaid — RECOMMENDED PRIMARY

- **Pros:** Native GitHub rendering (since 2022), VS Code markdown preview, simple syntax, 12+ diagram types, venture-backed active development
- **Cons:** C4 support still experimental, limited customization vs PlantUML, some GitHub rendering gaps (icons, hyperlinks)
- **Diagram types:** Flowchart, sequence, class, state, ER, Gantt, Git graph, C4 (experimental), mindmap, timeline
- **Best for:** READMEs, onboarding docs, PR descriptions, quick diagrams — anywhere speed matters
- **GitHub integration:** Native (fenced `mermaid` code blocks render inline)

#### D2 — RECOMMENDED FOR PRESENTATION QUALITY

- **Pros:** Modern minimal syntax (`hello -> w: there`), professional dark-mode output, native container edges (critical for architecture), LaTeX + code snippets in diagrams, auto-formatting
- **Cons:** Smaller ecosystem than Mermaid, less GitHub integration
- **Best for:** Presentation-quality architecture diagrams, complex nested systems
- **GitHub integration:** Native rendering

#### PlantUML

- **Pros:** Full UML spec coverage, extensive customization/theming, battle-tested since 2009
- **Cons:** Requires Java runtime + server, no native GitHub rendering (must pre-render), GPL 3.0
- **Best for:** Formal architecture reviews, regulated documentation
- **Recommendation:** Skip unless formal UML required — Mermaid covers our needs

#### Structurizr (C4-focused DSL)

- **Pros:** Single model/multiple views, Python bindings (`structurizr-python`, `PyStructurizr`, `buildzr`)
- **Cons:** Complexity overhead, C4-only
- **Recommendation:** Consider later if formally adopting C4; Mermaid C4 syntax is sufficient to start

#### Graphviz/DOT

- **Pros:** Most widely used graph visualization engine, advanced layout algorithms
- **Cons:** Verbose syntax, no sequence diagrams
- **Role:** Backend for other tools (pydeps, Terraform graph, PlantUML) — rarely used directly

### Python-Specific Code Visualization

| Tool | What It Does | Output | Best For |
|------|-------------|--------|----------|
| **pydeps** | Import/dependency graph from bytecode | SVG/PNG via Graphviz | Polylith boundary visualization |
| **Pyreverse** (Pylint) | Class + package diagrams from code | DOT, PlantUML, Mermaid | Reverse-engineering class hierarchies |
| **py2puml** | Python modules -> PlantUML class diagrams (AST) | PlantUML | Type-annotation-driven class docs |
| **code2flow** | Call graphs from Python source | DOT/SVG | Tracing specific execution paths |
| **pyan** | Call graph analysis | DOT | Static call analysis (maintenance concerns) |

### Infrastructure Visualization

| Tool | Input | Output | Status (2026) | Notes |
|------|-------|--------|---------------|-------|
| **Rover** | tfstate/plan | Interactive web UI | Active | Best for exploring plan changes |
| **Inframap** | tfstate/HCL | Clean architecture diagrams | Active | Focuses on relevant infra, high-level |
| **Terraform Graph** | HCL (built-in) | DOT | Active (HashiCorp) | Quick dependency analysis, verbose output |
| **Terravision** | HCL | AWS architecture diagrams | Active | Auto-gen from Terraform code |
| **Terragrunt `dag graph`** | Terragrunt configs | DOT | Built-in | Module dependency chain visualization |
| **Diagrams** (Python lib) | Python code | PNG/SVG | Active (resumed maintenance) | Programmatic cloud architecture diagrams |
| **Blast Radius** | — | — | **DEPRECATED** | Terraform 1.x incompatible — avoid |

### Visual/Interactive Tools

| Tool | Version Control | Best For | VS Code Integration |
|------|----------------|----------|-------------------|
| **draw.io / diagrams.net** | `.drawio.svg` (good diffs) | Visual editing, stakeholder-facing | Excellent (inline editing) |
| **Excalidraw** | `.excalidraw` (JSON, verbose) | Whiteboarding, brainstorming | Good |
| **Eraser.io** | Diagram-as-code underneath | AI-assisted hybrid (drag-drop updates code) | Extension available |
| **Lucidchart** | Proprietary cloud | Enterprise collaboration only | N/A |

### Quick Decision Matrix

```
Quick docs in markdown          -> Mermaid
Presentation-quality arch       -> D2
Python code -> AWS diagrams     -> Diagrams library
Terraform state/plan viz        -> Rover (interactive) or Inframap (static)
Terragrunt dependency chain     -> terragrunt dag graph
Python dependency graphs        -> pydeps
Reverse-engineer Python UML     -> Pyreverse
Visual editing + git            -> draw.io (VS Code extension)
Whiteboarding                   -> Excalidraw
```

---

## Part 3: Automation Strategy

### Tier 1 — Fully Automatic (CI/CD, Zero Manual Effort)

**Python dependency graphs:**
```bash
# Generate Polylith dependency visualization
pydeps --max-bacon 2 -o docs/diagrams/discover-deps.svg bases/filescience/discover
pydeps --max-bacon 2 -o docs/diagrams/queue-processor-deps.svg bases/filescience/valkey_queue_processor
```

**Architecture boundary enforcement (already active):**
```bash
make lint-imports  # import-linter, 7 contracts, runs in hooks + CI
```

**Infrastructure diagrams from Terraform:**
```bash
# Terragrunt module dependency chain
cd infrastructure/live/non-prod/us-east-1/dev && terragrunt dag graph | dot -Tsvg -o docs/diagrams/terragrunt-deps.svg

# Clean infrastructure diagram from state
inframap generate infrastructure/live/non-prod/us-east-1/dev --output docs/diagrams/aws-infra.svg
```

### Tier 2 — Semi-Automatic (On-Demand + Review)

**LLM-assisted Mermaid generation:**
- ~40% time savings with AI skeleton + manual refinement
- Store prompts in version control for reproducibility
- Always validate syntax before committing
- We already have v1 Mermaid diagrams in discover research doc

**Class/package diagrams on demand:**
```bash
pyreverse -o mmd -p discover bases/filescience/discover  # Mermaid output
py2puml bases/filescience/discover filescience.discover > docs/diagrams/discover-classes.puml
```

**Interactive Terraform exploration:**
```bash
docker run -p 9000:9000 im2nguyen/rover  # Interactive plan visualizer
```

### Tier 3 — Manual (Quarterly Updates)

- C4 System Context diagram (stakeholder view)
- High-level architecture decisions
- Cross-system integration diagrams
- Tools: Mermaid in markdown or D2 for presentation

### Keeping Diagrams in Sync — Anti-Drift Strategy

1. **Auto-generated diagrams in CI** — regenerated every build, committed if changed
2. **import-linter as living architecture doc** — contracts ARE the boundary spec
3. **Mermaid in markdown** — lives next to code, reviewed in PRs
4. **Manual diagrams have documented update schedule** — quarterly for high-level views
5. **Diagram sources versioned** — never commit only rendered output

### Recommended File Organization

```
docs/
├── diagrams/
│   ├── infrastructure/    # Auto-generated from Terraform (CI/CD)
│   ├── dependencies/      # Auto-generated from pydeps (CI/CD)
│   └── architecture/      # Manual/semi-auto C4, sequence, state
└── README.md              # Links + update schedule
```

Or embed Mermaid directly in relevant markdown files (current approach in memory-bank research docs).

---

## Part 4: Recommended Stack for FileScience

### Primary (Start Here)

| Tool | Purpose | Setup Effort | Automation |
|------|---------|-------------|------------|
| **Mermaid** | Sequence, C4, state, flowchart diagrams in markdown | Zero | Manual + LLM-assisted |
| **pydeps** | Python import/dependency graphs | `uv add --dev pydeps` + Graphviz | Fully auto (Makefile/CI) |
| **import-linter** | Architecture boundary enforcement | Already active | Fully auto (hooks + CI) |
| **Terragrunt `dag graph`** | Module dependency chain | Built-in | Fully auto |

### Secondary (Add When Needed)

| Tool | Purpose | When |
|------|---------|------|
| **D2** | Presentation-quality diagrams | Architecture reviews, stakeholder decks |
| **Rover** | Interactive Terraform plan review | Before major infra changes |
| **Diagrams** (Python) | Programmatic AWS architecture diagrams | When infra grows complex |
| **Pyreverse** | Class/package diagrams | Onboarding, refactoring planning |
| **draw.io** (VS Code) | Visual editing for non-technical stakeholders | Cross-team communication |

### Deferred

| Tool | Reason to Defer |
|------|----------------|
| PlantUML | Mermaid covers our needs; PlantUML adds Java dependency |
| Structurizr | Overkill until formally adopting C4 at scale |
| Eraser.io | Proprietary SaaS, vendor lock-in risk |
| Lucidchart | Only for enterprise collaboration needs |

---

## Part 5: Implementation Roadmap

### Phase 0 — Foundation (Already Done)
- [x] import-linter contracts enforcing Polylith boundaries
- [x] v1 Mermaid diagrams in discover research doc (context + sequence)

### Phase 1 — Makefile Targets + pydeps
- Add `make diagram-deps` target using pydeps for all bases
- Add `make diagram-infra` target for Terragrunt DAG
- Install Graphviz + pydeps as dev dependencies

### Phase 2 — Diagram Library in Docs
- Create `docs/diagrams/` structure
- Embed key Mermaid diagrams in relevant READMEs
- Add C4 container diagram for full system view

### Phase 3 — CI Integration
- Auto-regenerate dependency diagrams on PR
- Optionally diff diagrams to detect architecture drift
- Add diagram generation to GitHub Actions

### Phase 4 — Polish (When Needed)
- D2 for presentation-quality versions
- Rover for infra review workflow
- Diagrams Python library for programmatic AWS views

---

## Part 6: Excalidraw + Claude Code Skill Approach (Deep Dive)

Triggered by [Yee Fei's article](https://medium.com/@ooi_yee_fei/custom-claude-code-skill-auto-generating-updating-architecture-diagrams-with-excalidraw-431022f75a13) on using Claude Code skills for auto-generating Excalidraw diagrams.

### Why Excalidraw Over Mermaid for Architecture Diagrams

**Yee Fei's rationale (Terraform/GCP infra with 15+ resources):**
- Mermaid gets **cluttered fast** at scale — a Terraform stack with 15+ resources becomes unreadable
- Mermaid doesn't support **freeform layout** needed for architecture diagrams
- Manual Excalidraw is beautiful and collaborative but **always outdated** because it requires manual updates
- Solution: Claude Code skill that **generates and updates** `.excalidraw` JSON from codebase analysis

**Key insight:** Excalidraw is not diagram-as-code (it's JSON, not human-readable), but with an LLM skill it becomes **LLM-as-code** — Claude reads the codebase and writes the JSON.

### Three Existing Implementations

#### 1. ooiyeefei/ccc — Pure Skill, Codebase-Aware ([repo](https://github.com/ooiyeefei/ccc))

**Approach:** Claude Code Skill (markdown knowledge file) that teaches Claude:
- How to analyze any codebase (monorepo, microservices, IaC, API, frontend)
- How to generate valid `.excalidraw` JSON with correct bindings and routing
- Critical workarounds for Excalidraw JSON quirks

**Workflow:**
1. User says: "Generate an architecture diagram for this project"
2. Claude analyzes codebase using Glob/Grep/Read
3. Identifies components, services, databases, APIs
4. Maps relationships and data flows
5. Generates valid `.excalidraw` JSON → `docs/architecture/`

**Update workflow:** Claude compares current codebase against existing diagram and updates it.

**Critical Excalidraw JSON quirks documented:**
- Diamond shapes have **broken arrow connections** in raw JSON — use styled rectangles instead
- Labels require **TWO elements** (shape + text with `containerId`), not a `label` property
- Elbow arrows need `elbowed: true`, `roundness: null`, `roughness: 0`
- Arrow x,y must be **edge coordinates**, not center
- Arrow width/height = bounding box of points array

**Layout convention:**
```
Row 1: Users/Entry points       (y: 100)
Row 2: Frontend/Gateway          (y: 230)
Row 3: Orchestration             (y: 380)
Row 4: Services                  (y: 530)
Row 5: Data layer                (y: 680)
Columns: x = 100, 300, 500, 700, 900
```

#### 2. robtaylor/excalidraw-diagrams — Python Library ([repo](https://github.com/robtaylor/excalidraw-diagrams))

**Approach:** Python `excalidraw_generator` library installed as Claude Code skill with helper classes:
- `Diagram` — base class with `box()`, `arrow_between()`, `text_box()`
- `Flowchart` — auto-positioning with `start()`, `process()`, `decision()`, `connect()`
- `AutoLayoutFlowchart` — hierarchical layout using `grandalf` package
- `ArchitectureDiagram` — `component()`, `database()`, `service()`, `user()`, `connect()`
- Configurable styles: `DiagramStyle`, `FlowchartStyle`, `ArchitectureStyle`, `BoxStyle`
- Color schemes: default, monochrome, corporate, vibrant, earth

**Extra features:**
- PNG export via Playwright (headless Chromium → excalidraw.com → screenshot)
- Google Drive integration (upload, share, round-trip editing)
- Auto-sizing boxes to text content

**Example for our use case:**
```python
from excalidraw_generator import ArchitectureDiagram

arch = ArchitectureDiagram()
arch.user("tenant", "Tenant", x=400, y=50)
arch.service("trigger", "Entity Discovery\nTrigger", x=350, y=180, color="violet")
arch.service("processor", "Valkey Queue\nProcessor", x=350, y=330, color="violet")
arch.component("manager", "Discovery\nManager", x=150, y=480, color="blue")
arch.component("dispatcher", "Dispatcher V2", x=350, y=480, color="blue")
arch.component("queue", "Work Queue", x=550, y=480, color="blue")
arch.database("dynamo", "DynamoDB\nwork-queue", x=550, y=180, color="green")
arch.component("valkey", "Valkey Cluster", x=700, y=330, color="orange")
arch.service("graph", "Microsoft\nGraph API", x=150, y=630, color="red")
arch.connect("trigger", "dynamo", "writes entities")
arch.connect("dynamo", "processor", "DynamoDB Stream")
arch.connect("processor", "manager", "orchestrates")
arch.connect("manager", "dispatcher", "dispatches")
arch.connect("dispatcher", "queue", "enqueues work")
arch.connect("queue", "valkey", "throttle context")
arch.connect("manager", "graph", "API calls")
arch.save("docs/architecture/filescience-discover.excalidraw")
```

#### 3. rnjn/cc-excalidraw-skill — Pure Knowledge File ([repo](https://github.com/rnjn/cc-excalidraw-skill))

**Approach:** Comprehensive reference docs (5 files) teaching Claude the Excalidraw JSON format:
- `SKILL.md` — element types, quick-start JSON templates, color palette, styling properties
- `diagram-patterns.md` — professional patterns per diagram type
- `examples.md` — complete working JSON examples
- `element-reference.md` — full property reference
- `best-practices.md` — layout, color theory (60-30-10 rule), typography guidelines

**Simpler than ooiyeefei's** (no codebase analysis workflow), but solid JSON reference.

### Comparison: Mermaid vs Excalidraw for Our Codebase

| Criterion | Mermaid | Excalidraw (via Skill) |
|-----------|---------|----------------------|
| **Readability at scale** | Degrades with 15+ nodes | Freeform layout handles complexity |
| **Version control** | Text diffs (excellent) | JSON diffs (noisy but functional) |
| **GitHub rendering** | Native inline rendering | Requires VS Code extension or excalidraw.com |
| **Human-editable source** | Yes (text syntax) | No (JSON), but editable in visual tool |
| **Update workflow** | Edit text manually or LLM-regenerate | LLM compares codebase vs existing diagram |
| **Setup effort** | Zero | Install skill + VS Code extension |
| **Output quality** | Functional but rigid layout | Professional, hand-drawn aesthetic |
| **Automation potential** | Embed in markdown, CI render to SVG | LLM generates from codebase analysis |
| **PR review** | Renders inline in PR description | Must open `.excalidraw` file separately |

### Recommendation: Complementary Use

**Mermaid for:**
- Sequence diagrams (discover runtime flow) — Mermaid excels here
- Quick inline diagrams in PRs and READMEs
- State machines (work-queue lifecycle)
- Anything that needs to render in GitHub markdown

**Excalidraw (via skill) for:**
- System architecture overview (C4 container level) — freeform layout matters
- Infrastructure topology — complex AWS layouts
- Stakeholder-facing diagrams — professional aesthetic
- Diagrams that need manual refinement after generation

**Implementation path:**
1. Start with ooiyeefei/ccc skill (pure knowledge, no dependencies)
2. Customize for Polylith/AWS/Terraform patterns
3. Add robtaylor's Python library later if programmatic generation proves valuable
4. Store `.excalidraw` files in `docs/architecture/`
5. View/edit with VS Code Excalidraw extension

---

## Part 7: D2 + Terrastruct Deep Dive (2026-02-12)

### D2 at a Glance

| Metric | Value |
|--------|-------|
| Stars | 23k |
| Latest | v0.7.1 (Aug 2024) |
| License | MPL-2.0 (open source) |
| Language | Go |
| CI/Docker | Yes (`terrastruct/d2` on Docker Hub) |
| Output | SVG, PNG, PDF, ASCII |
| Python lib | `py-d2` (89 stars, v1.0.2, MIT) |

### Terrastruct (the company)

- 2-10 employees, $2M funding (Bloomberg Beta)
- D2 is open source; **TALA layout engine is proprietary** ($20/mo bundled, or standalone license)
- Without TALA: Dagre (unmaintained since 2018) or ELK (maintained, decent)
- D2 Studio: web IDE with bidirectional text↔visual sync

### D2 Features Relevant to Our Decision

**Native C4 support (v0.7+):**
- `suspend`/`unsuspend` keywords for multi-view from one model
- `c4-person` shape, C4 theme, legend support
- Container nesting maps naturally to C4 levels

**Scenarios = sliced views from single file:**
```d2
# Base layer (all elements)
lambda_trigger: Entity Discovery Trigger
dynamodb: DynamoDB work-queue
lambda_processor: Valkey Queue Processor

# Scenario: just the discover subsystem
scenarios: {
  discover: {
    manager: Discovery Manager
    dispatcher: Dispatcher V2
    lambda_processor -> manager
    manager -> dispatcher
  }
}
```

**Imports = composable architecture across files:**
- Import files like programming language modules
- Shared styles, icons, common components
- Domain experts contribute separate files, combine via imports

**AWS/cloud icons:** Built-in via icons.terrastruct.com (Lambda, S3, DynamoDB, etc.)

**Programmatic generation (`py-d2`):**
```python
from py_d2 import D2Diagram, D2Shape, D2Connection, D2Style
shapes = [
    D2Shape(name="trigger", style=D2Style(fill="orange")),
    D2Shape(name="processor", style=D2Style(fill="orange")),
]
connections = [D2Connection(shape_1="trigger", shape_2="processor", label="DynamoDB Stream")]
diagram = D2Diagram(shapes=shapes, connections=connections)
```

### D2 vs Structurizr vs Mermaid — Honest Comparison

| Criterion | D2 | Structurizr | Mermaid |
|-----------|-----|-------------|---------|
| **Model vs render** | Hybrid (scenarios + imports give model-like features) | True model (define once, render many views) | Pure render |
| **C4 support** | Native (v0.7+) | Native (purpose-built) | Experimental |
| **Sliced views** | Scenarios + imports | Workspace views + filters | None |
| **GitHub rendering** | No (must pre-render SVG/PNG) | No (must export to Mermaid) | **Yes (native)** |
| **Python generation** | `py-d2` (89 stars, simple API) | `buildzr` (8 stars, type-safe) | String templates |
| **Layout quality** | TALA (paid) >> ELK > Dagre | Via export to rendering tool | Single engine, limited |
| **Terraform auto-gen** | tf2d2 (12 stars, immature) | None | None |
| **Excalidraw bridge** | **None** | Via Mermaid → mermaid-to-excalidraw | **Yes** (mermaid-to-excalidraw) |
| **Platform risk** | Small company (2-10 people), TALA proprietary | Consolidating to vNext (active, Simon Brown) | Venture-backed, massive community |
| **CI/Docker** | Yes (official image) | Yes (CLI in Docker) | Yes (CLI) |
| **Syntax simplicity** | Very simple, easy to template | DSL, more verbose | Compact per diagram type |

### Key D2 Limitations for Our Stack

1. **No GitHub rendering** — Can't embed D2 in PRs/READMEs like Mermaid
2. **No D2→Excalidraw converter** — Separate rendering paths required
3. **TALA lock-in** — Free layout engines (Dagre/ELK) are notably worse for architecture diagrams
4. **No Terraform/Polylith auto-gen** — We build bridge scripts regardless of tool choice
5. **Small company risk** — 2-10 employees, if revenue dries up TALA stalls (D2 OSS continues)

### Where D2 Genuinely Wins

1. **Simpler syntax** — Easier to generate programmatically than Structurizr DSL
2. **Scenarios + imports** — Gets you 80% of Structurizr's model power with less ceremony
3. **Better output quality** (with TALA) — Purpose-built for architecture layout
4. **AWS icons built-in** — No external library needed
5. **Watch mode** — `d2 --watch` with live browser reload, fully offline

### Practical Assessment for FileScience

**D2 could replace Structurizr in the "at 20 Lambdas" tier** of our scaling plan. The tradeoff:
- We lose: Structurizr's formal model, Mermaid export (and thus Excalidraw bridge)
- We gain: Simpler syntax, scenarios for sliced views, better rendering (if paying for TALA)
- Neutral: We build bridge scripts (Terraform→X, Polylith→X) either way

**D2 cannot replace Mermaid** for inline GitHub/PR diagrams (no native rendering).

**D2 cannot replace Excalidraw** for stakeholder-facing diagrams (no conversion path, different aesthetic).

---

## Part 8: Structurizr ↔ Excalidraw Pipeline Research (2026-02-12)

### The Core Question

Can we produce deterministic Structurizr output and feed it into Excalidraw for nicer rendering?

### Answer: Yes, Via Mermaid Bridge

**Pipeline:** `buildzr (Python) → Structurizr JSON → CLI export → Mermaid → @excalidraw/mermaid-to-excalidraw → .excalidraw`

**Key tool:** [@excalidraw/mermaid-to-excalidraw](https://github.com/excalidraw/mermaid-to-excalidraw) (699 stars, maintained by Excalidraw team, v1.1.0 July 2024)

**Limitation:** Layout is regenerated (not preserved from Structurizr), so manual refinement or Claude Code skill polish is needed for presentation quality.

### Ecosystem Gaps (No Existing Tooling Found)

| Integration | Exists? | What Would Be Needed |
|-------------|---------|---------------------|
| Terraform HCL → Structurizr model | No | Parse HCL/state, map AWS resources to C4 containers |
| Terragrunt DAG → Structurizr | No | Parse `terragrunt dag graph` DOT output → C4 relationships |
| Polylith workspace → Structurizr | No | Parse `poly info` + import-linter contracts → C4 components |
| Structurizr → Excalidraw direct | No | Two-step via Mermaid bridge |
| D2 → Excalidraw | No | Separate rendering paths |

### Python Structurizr Libraries

| Library | Stars | Status | Recommendation |
|---------|-------|--------|---------------|
| `structurizr-python` | 67 | **ARCHIVED** | Avoid |
| `pystructurizr` | 140 | Active | OK (CLI + kroki.io rendering) |
| `buildzr` | 8 | Active (179 commits) | **Best option** — Pythonic context managers, type-safe, cloud themes |

### Structurizr CLI & Platform Status

- **CLI:** Archived Feb 2026, consolidated into vNext. Still functional.
- **Structurizr Cloud:** EOL September 2026
- **Structurizr Lite:** EOL, single-user only, not CI-suitable
- **On-premises:** Docker + API, still supported
- **Export formats:** PlantUML, C4-PlantUML, Mermaid, D2, DOT, WebSequenceDiagrams, Ilograph, JSON, HTML, custom

### Decision Recorded

See [[2026-02-12-c4-diagram-methodology-and-tooling|2026-02-12-c4-diagram-methodology-and-tooling.md]]

---

## Sources

### Diagram Types & Standards
- [C4 Model](https://c4model.com/)
- [Miro: C4 Model Guide](https://miro.com/diagramming/c4-model-for-software-architecture/)
- [UML in 2024: Is UML Obsolete?](https://jaystechbites.com/posts/2024/uml-alternatives-in-2024/)

### Tools
- [Mermaid](https://mermaid.js.org/) | [GitHub](https://github.com/mermaid-js/mermaid)
- [D2](https://d2lang.com/) | [Text-to-Diagram Comparison](https://text-to-diagram.com/)
- [pydeps](https://github.com/thebjorn/pydeps)
- [Pyreverse](https://pylint.readthedocs.io/en/stable/pyreverse.html)
- [py2puml](https://github.com/lucsorel/py2puml)
- [code2flow](https://github.com/scottrogowski/code2flow)
- [Diagrams (Python)](https://diagrams.mingrammer.com/)
- [Rover](https://github.com/im2nguyen/rover)
- [Inframap](https://github.com/cycloidio/inframap)
- [Terravision](https://github.com/patrickchugh/terravision)
- [Structurizr](https://structurizr.com/)

### Excalidraw + Claude Code Skills
- [Yee Fei: Custom Claude Code Skill for Excalidraw (Medium)](https://medium.com/@ooi_yee_fei/custom-claude-code-skill-auto-generating-updating-architecture-diagrams-with-excalidraw-431022f75a13)
- [ooiyeefei/ccc — Claude Code Custom Plugins (GitHub)](https://github.com/ooiyeefei/ccc)
- [robtaylor/excalidraw-diagrams — Python generator skill (GitHub)](https://github.com/robtaylor/excalidraw-diagrams)
- [rnjn/cc-excalidraw-skill — Knowledge file skill (GitHub)](https://github.com/rnjn/cc-excalidraw-skill)

### Automation & Best Practices
- [Automate Technical Diagrams with LLMs](https://cosmo-edge.com/automate-technical-diagrams-llm-mermaid-plantuml-cicd/)
- [Pros and Cons of Diagram-as-Code](https://icepanel.io/blog/2025-02-05-the-pros-and-cons-of-diagram-as-code-for-software-architecture)
- [Top 5 Terraform Visualization Tools 2026](https://spacelift.io/blog/terraform-visualization)
- [Terragrunt dag graph](https://terragrunt.gruntwork.io/docs/reference/cli/commands/dag/graph/)
- [Hava CI/CD Integration](https://www.hava.io/blog/integrate-diagrams-in-your-ci/cd-pipelines-using-github-actions)
