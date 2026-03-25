---
type: state
created: 2026-03-21
status: active
touched: 2026-03-21
tags: [akasha, ux, interaction-design, manifesto]
related:
  - thoughts/shared/briefs/2026-03-21-akasha-interaction-design-manifesto.md
  - thoughts/shared/briefs/2026-03-20-personal-life-ontology-agent-platform.md
  - thoughts/shared/briefs/2026-03-20-akasha-capability-platform.md
  - thoughts/shared/briefs/2026-03-09-generative-ui.md
  - topics/dashboard-facelift.md
---
# Akasha Interaction Design Manifesto

> For a product whose moat is temporal irreplaceability, the feel is the product. Users won't compare Akasha to a dashboard or a chatbot. They'll compare it to texting a friend, to a tool that anticipates what they need. That's the bar.

This manifesto defines the interaction philosophy for every Akasha surface -- the FileScience dashboard, the consumer agent, and generative UI components. It is a reference document, not a component spec. Downstream plans cite it; implementers use its decision framework to make correct timing, rhythm, and presence decisions without asking.

**Origin:** [[2026-03-21-akasha-interaction-design-manifesto|Shaped 2026-03-21]] from brainstorm notes on perceived performance psychology, grounded in academic research and industry precedent.

**Structure:** Each principle follows the format used by the most-adopted design documents (Material Design, Apple HIG, Atlassian): named label, one-sentence stand, "versus what?" test, rationale, application guidance, backfire risk, and concrete do/don't examples.

---

## Founding Principles

Four principles, ordered from most universal to most context-specific. Every interaction across every Akasha surface should be traceable to at least one of these.

---

### Principle 1: Never Leave Them in the Dark

**Stand:** Every user action gets visible acknowledgment within 300ms. For operations longer than 1 second, show meaningful progress or partial results. Never let the user stare at nothing.

**Versus what?** "Ship it fast and let the UI catch up" -- the belief that raw speed excuses lack of feedback. Speed is necessary but not sufficient. A 200ms operation with no visual acknowledgment feels broken. A 5-second operation with progressive disclosure feels fast.

**Rationale:**
- Perceived performance research (Mejtoft, Langstrom & Soderstrom, ECCE 2018): Interactive loading states lead to higher satisfaction, especially for long waits. Skeleton screens reduce perceived wait duration by 20-30% vs. spinners.
- Maister's Laws (David Maister, "The Psychology of Waiting Lines," 1985): Unoccupied time feels longer than occupied time. Pre-process waits feel longest. Uncertain waits feel longer than known waits. The formula: Satisfaction = Perception - Expectation.
- The Houston airport case: Baggage claim complaints didn't drop when wait times were reduced -- they dropped when arrival gates were moved farther away, so passengers walked longer (occupied time) and found luggage already there.

**Apply when:**
- Any user-initiated action (click, tap, send, submit) -- acknowledge within 300ms
- Any operation exceeding 1 second -- show progress, partial results, or a meaningful status message
- Any state transition (loading, processing, saving, sending) -- make it visible
- Pre-process waits (before the system starts working) -- acknowledge receipt immediately

**Don't apply when:**
- Background sync or ambient updates that the user didn't initiate -- these should be silent unless they produce notable results
- Micro-interactions under 100ms (hover states, cursor changes) -- the browser/OS handles these natively

**Backfire risk:**
- Skeleton screens only buy time -- if the actual delay is long (> 10s), they become as frustrating as a spinner. At that point, switch to P2 (Show the Work) with real progress information.
- Optimistic UI requires careful rollback handling. A "liked" post that silently un-likes erodes trust more than a loading state would have. Only use optimistic UI for low-risk, easily reversible actions.

**Do:** User clicks "Apply filter" on a data table. The table instantly shows a skeleton shimmer over the data rows (< 300ms). Within 500ms, the first rows appear. The rest stream in progressively.

**Don't:** User clicks "Apply filter." The screen freezes for 1.2 seconds, then the full table appears all at once. The user wonders if their click registered.

---

### Principle 2: Show the Work

**Stand:** For complex operations, make the effort visible. Transform invisible processing into visible narratives so users feel included in system activity.

**Versus what?** "Just show the result" -- the belief that users only care about output, not process. Users use visible effort as a proxy for quality. An agent that shows "checking your sleep data against spending patterns..." earns more trust than one that instantly says "you overspent." The work IS the value signal.

**Rationale:**
- Effort heuristic (Buell & Norton, "The Labor Illusion," Management Science, 2011): 266 participants rated service value 8% higher when shown a live list of actions being performed vs. a plain progress bar. Five experiments across travel and dating confirmed the effect. Mechanism: attribute substitution + reciprocity -- when quality is hard to assess directly, the brain substitutes "how hard did they work?" as the quality signal.
- Kruger et al. (2004) confirmed the underlying heuristic causally: identical outputs rated higher in quality and monetary value when described as requiring more effort.
- UXmatters "Designing for Autonomy" (2025): "Transform invisible processing into visible narratives so users feel included in system activity." This is the principle that separates AI agents from black boxes.
- Kayak's flight search (the literal study case): live "searching 400+ sites" list during search. Domino's pizza tracker: step-by-step preparation visible in-app.

**Apply when:**
- Agent performs multi-step reasoning (show each step as it happens)
- Dashboard generates a complex report or export (show "joining datasets... applying filters... rendering...")
- Generative UI builds a view from multiple data sources (show source attribution)
- Any operation where the user can't easily assess output quality (the effort signal helps them trust the result)

**Don't apply when:**
- The operation is simple and fast (< 1 second) -- P1 acknowledgment is sufficient
- The output quality is poor -- visible effort on bad output amplifies dissatisfaction (Buell's follow-up experiment confirmed this)
- Expert users in power-user mode who have explicitly opted for speed over narrative (defer to P4)

**Backfire risk:**
- The effort heuristic amplifies existing opinions, not reverses them. If the agent's answer is bad, showing "checking 5 databases... cross-referencing 3 sources... analyzing patterns..." makes the user MORE dissatisfied than if they'd just seen the bad answer appear. The quality must be there -- showing work is not a substitute for good output.
- Replication attempts (N=1,405) found support for quality/liking effects but weaker support for monetary value effects -- the effect may be narrower than the original claimed. Don't over-index on it.

**Do:** User asks the agent "how's my backup health this month?" The agent responds within 300ms: "Checking your backup status..." Then streams: "Scanned 3 cloud connections... 847 items backed up... 2 failures detected in OneDrive..." Then delivers the full summary with the failures highlighted.

**Don't:** User asks "how's my backup health this month?" The agent waits 4 seconds in silence, then delivers a complete summary. The user doesn't know if their message was received, if the system is working, or if it's stuck.

---

### Principle 3: Earn Trust Through Friction

**Stand:** For high-stakes or irreversible actions, deliberate slowdowns signal care. Instant execution of life-altering actions feels reckless.

**Versus what?** "Fastest wins" -- the belief that all latency is bad. Some latency is a trust signal. A system that instantly deletes all your backups feels dangerous. A system that says "verifying permissions... checking for active restores... confirming scope..." feels like it cares about getting it right.

**Rationale:**
- Deliberate friction research: Wells Fargo deliberately slowed their retinal scanner because users didn't trust it at full speed. Facebook pads account security checks to ~10 seconds (the actual check takes milliseconds). TurboTax's "calculating your refund" progress bar runs the same duration for all users -- pure theater that increases confidence in results.
- Effort justification (Festinger, 1957): People value outcomes more when they perceive effort was spent obtaining them. A mortgage approval that takes 3 seconds feels suspicious. One that takes 15 seconds with a visible credit check feels thorough.
- ACM (2021) formal definition: Friction is "a momentary perturbation in an otherwise uninterrupted interaction that does not compromise the user's experience in the long run or disrupt trust in the service."
- Microsoft UX for Agents (2025): "Anchor points" -- intentional moments where the system pauses and puts a human back in the loop. These are design features, not bugs.

**Apply when:**
- Write-back actions that change external state ("reschedule my day," "pay this bill," "reply to that email")
- Destructive operations (delete, revoke, disconnect)
- Financial transactions (payments, refunds, billing changes)
- Actions that affect other people (sending messages, publishing, sharing)
- First-time actions where the user hasn't established a pattern yet

**Don't apply when:**
- Routine, low-risk, easily reversible actions (toggling a filter, changing a view, bookmarking)
- Actions the user performs frequently and has established muscle memory for
- Read-only operations -- friction on reads feels bureaucratic

**Backfire risk:**
- Security theater is a double-edged sword: if users discover the delay is fake (e.g., via developer tools or by noticing the timing never varies), trust collapses more severely than if no delay had existed. Show real steps where possible rather than pure theater.
- Overuse creates abandonment: friction applied broadly or frequently becomes fatigue. It works in specific, high-stakes moments -- not as a general UI philosophy. "The Crying Wolf" anti-pattern.
- Context mismatch: luxury/high-touch contexts support friction. Consumer apps with speed expectations have a narrower window.

**Do:** User tells the agent "delete all backups for Tenant X." The agent responds: "This will permanently delete 2,341 backed-up items across 3 cloud connections for Tenant X. Checking for active restores... none found. Verifying your admin permissions... confirmed." Then presents a clear confirmation with the scope spelled out.

**Don't:** User tells the agent "delete all backups for Tenant X." The agent responds "Done." 2,341 items are gone. The user's stomach drops.

---

### Principle 4: Adapt to the Human

**Stand:** Interaction rhythm varies by user expertise and context stakes. Justify delays textually, not just with timing. One experience does not fit all.

**Versus what?** "One experience fits all" -- the belief that consistent pacing equals good UX. Consistent pacing is consistently wrong for someone. A novice user needs warmth and narrative. An expert user needs speed and control. Treating them the same satisfies neither.

**Rationale:**
- Anthropomorphism research (IJHCI 2025): Typing indicators and variable response timing increase social presence perception for novice users and increase usage intention. But for experienced users, human-like delays are interpreted as slowness, not warmth. The effect is not universal.
- Textual justification (ACM CUI 2024, N=194): Justifying delays with text ("I'm analyzing your question...") builds more trust than timing manipulation alone. This works across expertise levels because it provides information, not just pacing.
- Trust in AI declining (Gartner 2025): Only 42% of customers trust businesses to use AI ethically, down from 58% in 2023. Anthropomorphism alone doesn't fix this. Transparency + competence does.
- Google Conversation Principles: Turn-taking rhythm should match the context -- social conversations allow longer pauses; task-oriented exchanges demand efficiency.

**Apply when:**
- The product serves both novice and expert users (which Akasha does across B2B and B2C)
- The same action can have different stakes depending on context (e.g., an agent query about weather vs. about health data)
- A user has explicitly signaled a preference (power-user mode, compact view, "just tell me")
- Delays need explanation -- always justify textually, regardless of user segment
- Timing variation for authenticity (avoiding "The Chatbot Tell") applies at ALL expertise levels. Even novice mode uses variable timing -- the typing bubble duration should correlate with response complexity. The novice/expert split governs warmth and narrative density, not timing authenticity. Never use fixed uniform timing regardless of mode.

**Don't apply when:**
- P3 (Earn Trust Through Friction) applies -- irreversible actions get friction regardless of expertise
- The user hasn't been classified yet (default to novice-friendly behavior until signals accumulate)
- Channel constraints make adaptation impossible (e.g., Apple Messages for Business doesn't expose user expertise metadata)

**Backfire risk:**
- False intimacy: users who over-trust an anthropomorphized agent are more harmed when it fails. Don't create warmth you can't sustain.
- Cognitive load: anthropomorphic behavior that isn't task-relevant adds extraneous load, especially in expert or high-stakes contexts (legal, financial, clinical).
- Uncanny valley: near-human behavior that falls slightly short triggers suspicion rather than warmth. Better to be clearly artificial-but-helpful than almost-human-but-off.
- Misclassification: treating an expert as a novice (patronizing) or a novice as an expert (overwhelming) both erode trust. Default to novice, escalate to expert based on observed behavior.

**Do (novice):** User texts the agent for the first time: "what's going on with my backups?" The agent shows typing indicator, pauses 1.5s, then responds warmly: "Hey! Let me check on that for you. Looking at your 3 connected cloud accounts..." Streams findings with context and explanation.

**Do (expert):** Power user texts: "backup status." The agent responds within 500ms with a concise summary: "3 connections active. 847/849 items synced. 2 failures: OneDrive/Reports/Q4.xlsx (permission denied), Box/Legal/contract-v3.pdf (file locked). Details?" No typing bubbles, no warm-up, no narrative.

**Don't:** Expert user gets the full novice treatment -- typing bubbles, warm greeting, step-by-step narration of a process they understand. They switch to a competitor.

---

## Sources

| Citation | Key Finding | Used In |
|----------|------------|---------|
| Buell & Norton, "The Labor Illusion," Management Science, 2011 | Visible effort increases perceived value by ~8%; amplifies existing opinions | P2 |
| Kruger et al., 2004 | Identical outputs rated higher when described as requiring more effort | P2 |
| Maister, "The Psychology of Waiting Lines," 1985 | Unoccupied time > occupied time; S = P - E | P1 |
| Mejtoft, Langstrom & Soderstrom, ECCE 2018 | Interactive loading screens lead to higher satisfaction; skeleton screens reduce perceived wait 20-30% | P1 |
| IJHCI 2025 (anthropomorphism study) | Typing indicators increase social presence for novices; experienced users interpret delays as slowness | P4 |
| ACM CUI 2024, N=194 | Textual justification of delays builds more trust than timing manipulation alone | P4 |
| Festinger, 1957 (effort justification) | People value outcomes more when they perceive effort was spent obtaining them | P3 |
| ACM 2021 (friction definition) | Friction = "momentary perturbation that doesn't compromise long-run experience or trust" | P3 |
| Gartner 2025 | AI trust declining: 42% trust (down from 58% in 2023) | P4 |
| Microsoft "UX Design for Agents," 2025 | Temporal axis (past/present/future); "nudging > notifying"; anchor points | P3, P4 |
| UXmatters "Designing for Autonomy," 2025 | "Transform invisible processing into visible narratives" | P2 |
| Google Conversation Principles (~2018, referenced in UX literature) | Turn-taking rhythm should match context: social conversations allow longer pauses; task-oriented exchanges demand efficiency | P4 |
| Jehad Affoneh, "Versus What?" test | A principle that doesn't reject an alternative isn't opinionated enough | All |
| NN/g Design Principles | Principles that resolve real disagreements get referenced; platitudes don't | All |

---

## Surface-Specific Patterns

Each surface maps all 4 principles to concrete interaction patterns. These are specific enough to implement from but described in prose, not code.

### FileScience Dashboard (B2B)

**Stack:** Next.js App Router + shadcn/ui + Tailwind CSS. Greenfield -- these patterns should be baked in from the start, not retrofitted.

| Principle | Pattern | Implementation Notes |
|-----------|---------|---------------------|
| **P1: Never Leave in Dark** | **Skeleton screens** for every data container (tables, charts, cards). **Optimistic UI** for filter toggles, view preferences, and settings changes -- update immediately, sync in background, rollback silently on failure. | shadcn/ui Skeleton component exists. Use React Suspense boundaries to trigger skeletons automatically. Optimistic updates via React Query's `onMutate`. |
| **P1: Never Leave in Dark** | **Progressive data streaming** for tables and charts. First rows/data points appear within 500ms; remainder streams in. Never show a full-page loader. | Next.js streaming SSR + React Server Components. Use `loading.tsx` convention for route-level skeletons. |
| **P2: Show the Work** | **Staged progress** for exports, reports, and bulk operations. Show "Querying 3 databases... Joining datasets... Applying filters... Rendering PDF..." -- not just a percentage bar. | Custom progress component that accepts step descriptions. Backend sends SSE or WebSocket progress events. |
| **P2: Show the Work** | **Query explain** for search results. Show what was searched and why results were ranked this way. "Searched across Clio matters, OneDrive files, and Box folders. 23 results ranked by relevance." | Display as a collapsible detail row above results. |
| **P3: Earn Trust** | **Multi-step confirmation** for destructive actions. "Delete all backups for Tenant X" -> scope summary -> active restore check -> permission verification -> explicit "Delete permanently" button with danger styling. | shadcn/ui AlertDialog with sequential steps. Red/destructive variant for the final action. |
| **P3: Earn Trust** | **Visible validation** before data exports. "Verifying your permissions... Checking export format compatibility... Preparing 2,341 items..." Even if fast, show the steps. | Brief staged animation (minimum 800ms total) with real validation steps. |
| **P4: Adapt to Human** | **Compact/power-user mode** toggle. Strips transition animations, reduces skeleton durations, collapses verbose status messages to icons, increases data density. Persisted per user. | CSS custom property `--motion-preference` + `prefers-reduced-motion` media query respect. User preference stored in localStorage + synced to profile. |

### Akasha Agent (B2C Text/Call)

**Channels:** iMessage (Apple Messages for Business), WhatsApp Flows, voice calls. The agent is the interface -- there is no dashboard to fall back to.

| Principle | Pattern | Implementation Notes |
|-----------|---------|---------------------|
| **P1: Never Leave in Dark** | **Immediate acknowledgment** within 300ms of user message. In text: show typing indicator or send "On it." In voice: verbal acknowledgment ("Got it, let me check."). Never leave the user wondering if their message was received. | iMessage: native typing indicator API. WhatsApp: simulated via webhook delay before response. Voice: immediate verbal filler. |
| **P1: Never Leave in Dark** | **Stream partial findings** during complex reasoning. Don't wait for the complete answer. "I can see your sleep data looks normal this week... now checking spending patterns..." | Token streaming for text channels. For voice: periodic verbal updates ("Still looking at your financial data..."). |
| **P2: Show the Work** | **Show reasoning steps textually.** "Checking your sleep data against spending patterns... Found a correlation: your spending spikes 2 days after poor sleep..." The narrative IS the value -- it teaches the user how the agent thinks. | Each reasoning step sent as a separate message or appended to a streaming response. NOT just timing delays with a typing bubble. |
| **P2: Show the Work** | **Source attribution.** When the agent draws from multiple data domains, name them. "Based on your Oura sleep data, Plaid transaction history, and Google Calendar..." | Inline citation format. Collapsible "sources" section in rich message formats. |
| **P3: Earn Trust** | **Visible verification for write-back actions.** "Reschedule my day" -> "Checking your calendar... You have 3 meetings. Moving standup to 2pm, pushing design review to tomorrow. The client call at 4pm can't move (external attendee). Here's the new schedule -- confirm?" | Preview of changes + explicit confirmation required before execution. Show what CAN'T be changed and why. |
| **P3: Earn Trust** | **Staged execution for multi-step actions.** Don't batch-execute 5 changes silently. Show each one: "Rescheduling standup... done. Moving design review... done. Client call stays at 4pm. All set." | Sequential execution with per-step status. Allows user to intervene mid-sequence. |
| **P4: Adapt to Human** | **Novice mode (default):** Full conversational rhythm -- typing indicator, 1-2s pause before responses, warm language ("Hey! Let me check on that"), step-by-step explanations, proactive context ("This means your backup is healthy"). | Default for new users. Gradually reduce warmth as interaction count increases (learned, not toggled). |
| **P4: Adapt to Human** | **Expert mode (earned):** Fast responses (< 500ms when possible), terse acknowledgments, no typing bubbles, data-dense formatting, skip explanations unless asked. "backup status" -> instant summary with numbers. | Triggered by: user explicitly asks ("just tell me"), user has 50+ interactions, user sends terse queries consistently. Always provide justification text for delays even in expert mode. |

### Generative UI Components (Both Surfaces)

**Stack direction:** CopilotKit (runtime + FastAPI SDK) + assistant-ui (chat primitives) with constrained shadcn/ui component catalog. These patterns apply to UI generated at runtime by the LLM.

| Principle | Pattern | Implementation Notes |
|-----------|---------|---------------------|
| **P1: Never Leave in Dark** | **Content-aware entrance animations.** Components don't just pop in -- they animate from skeleton to populated state. Tables fill row by row. Charts draw their axes, then data points. Cards reveal header, then body. | CSS `@keyframes` or Framer Motion `AnimatePresence`. Skeleton-to-content transition should take 200-400ms per component. |
| **P1: Never Leave in Dark** | **Placeholder-first rendering.** When the LLM selects a component, render its skeleton immediately (within 300ms of tool call). Data fills in as it arrives from the backend. | Component catalog includes skeleton variants for every component type. LLM tool call triggers skeleton; data callback fills content. |
| **P2: Show the Work** | **Source attribution on generated views.** Every generated component displays which data sources contributed. "Built from Clio matters + OneDrive audit log + Box file metadata." | Footer or header badge on every generated component. Click to expand source details. |
| **P2: Show the Work** | **Generation narrative.** When the LLM builds a complex view, show the assembly process: "Querying Clio for active matters... Found 47. Cross-referencing with backup status... 3 have stale backups." | Streaming narrative above the component as it builds. Collapses to a summary line after generation completes. |
| **P3: Earn Trust** | **Approval gates for action components.** If a generated component includes buttons that trigger write-back actions (approve, delete, send), the component renders in a "preview" state with a prominent "Approve and Execute" gate. | Preview state: actions are visible but disabled. Gate: single confirmation button. Component visually signals "pending approval" (muted colors, dashed border). |
| **P3: Earn Trust** | **Change preview for generated mutations.** When a component proposes changes (e.g., a generated "fix these 3 backup failures" card), show exactly what will change before the user confirms. | Diff-style preview: current state -> proposed state. Red/green for deletions/additions. |
| **P4: Adapt to Human** | **Adaptive component density.** Novice users get simple cards with large text and clear CTAs. Expert users get dense tables with sortable columns and inline actions. The LLM selects component complexity based on user segment. | User segment passed as context to the LLM tool call. Component catalog has "simple" and "dense" variants for key component types. |

---

## Channel Constraints

Each channel has different capabilities for implementing the principles. These constraints must be checked before any implementation plan that targets a specific channel.

### iMessage (Apple Messages for Business)
*Verified as of: 2026-03-21*

| Capability | Status | Notes |
|------------|--------|-------|
| Typing indicator | Native | Available via Messages for Business API. No rate limits on indicator display. |
| Interactive Forms | iOS 18.4+ | Rich forms with text fields, pickers, date selectors. Built-in loading states. |
| Fallback (iOS < 18.4) | Rich link previews | Manual refresh required. Use rich link previews with embedded data for older devices. |
| Apple Pay | Native | Mandatory Apple-controlled timing for payment flow -- not designable. Do not attempt to add friction to the Apple Pay flow itself. |
| Read receipts | User-controlled | Do not rely on read receipts for presence signaling. Users may have them disabled. |
| Message size | Limited | Large responses must be chunked. Use this as a natural progressive disclosure mechanism. |
| Rich bubbles | Supported | Custom interactive message extensions for complex UI inline in chat. |

### WhatsApp Flows
*Verified as of: 2026-03-21*

| Capability | Status | Notes |
|------------|--------|-------|
| Typing indicator | Simulated | No native API. Simulate by introducing a webhook delay before sending the response. Risk: WhatsApp enforces per-phone-number message rate limits -- excessive simulated typing can trigger throttling. |
| Interactive forms | Supported | WhatsApp Flows provide multi-step forms with loading spinners. |
| Fallback | Plain text + emoji | For devices or contexts where Flows aren't available, use plain text with emoji status indicators (checkmark, hourglass, warning). |
| Template messages | 24-hour window | Proactive agent messages require pre-approved templates and can only be sent within 24 hours of the last user message. Limits P2 (Show the Work) for async processes. |
| Media messages | Supported | Images, documents, and audio. Can be used for richer "show the work" narratives (charts as images, reports as PDFs). |

### Web Dashboard
*Verified as of: 2026-03-21*

| Capability | Status | Notes |
|------------|--------|-------|
| All interaction patterns | Full control | No platform constraints. Framer Motion, CSS transitions, Web Animations API, React Suspense all available. |
| Animation | Unlimited | This is the highest-fidelity surface. Use it to set the standard that other channels approximate. |
| Progressive loading | Native | Next.js streaming SSR, React Server Components, Suspense boundaries. |
| User preference persistence | Full | localStorage + server sync for compact/power-user mode, motion preferences. |
| Fallback (older browsers / reduced-motion) | Degrade gracefully | Respect `prefers-reduced-motion` -- all skeleton animations and entrance transitions degrade to instant state changes. Minimum browser baseline: Chrome 100+, Safari 15.4+, Firefox 100+. Below baseline: remove Framer Motion AnimatePresence transitions, retain skeleton structure, fall back to CSS opacity transitions. |

### Voice
*Verified as of: 2026-03-21. Constraints only -- full voice interaction design is a separate domain.*

| Capability | Status | Notes |
|------------|--------|-------|
| Silence tolerance | Max 2 seconds | Silence > 2s feels broken. Use filler phrases ("Let me check on that...") as audio skeleton screens. |
| Tone as signal | Available | Tone shifts from conversational to focused signal "I'm processing." Return to conversational tone signals "I have your answer." |
| Turn-taking | Critical | Interrupting the user mid-sentence breaks trust. Wait for clear turn boundaries. |
| Pacing | Contextual | Slow, deliberate pacing for sensitive topics (health, finance). Brisk pacing for routine queries. Match the user's pace when possible. |
| Fallback (TTS latency > 3s) | Escalating fills | At 2s: second verbal fill ("Still working on this, give me one more moment"). At 5s: offer channel pivot ("This is taking longer than expected -- I can send you the details in a text message when ready."). Never exceed 5s of silence without a verbal update. |

---

## Anti-Patterns

Named failure modes, each tied to a specific principle violation. If you find yourself implementing one of these, stop and reconsider.

### "The Silent Spinner"
**Violates:** P1 (Never Leave Them in the Dark)

Any delay > 300ms with no visible feedback. A loading spinner with no context is only slightly better than nothing -- it says "I'm working" but not "I'm working on YOUR thing" or "I'll be done in about 5 seconds."

**Fix:** Replace with a skeleton screen (layout-aware), a contextual status message, or progressive content delivery.

### "The Fake Think"
**Misapplies:** P3 (Earn Trust Through Friction)

Artificial delays on operations that complete instantly, in routine contexts. The TurboTax progress bar works because tax calculations feel high-stakes. A fake 3-second delay on "save preferences" feels insulting. Friction is for moments that warrant it.

**Fix:** Reserve deliberate friction for irreversible, high-stakes, or first-time actions. Routine operations should be as fast as possible with P1 acknowledgment.

### "The Chatbot Tell"
**Violates:** P4 (Adapt to the Human)

Instant, uniform response timing that signals "this is a bot." Every response arrives in exactly 200ms regardless of complexity. The timing is inhuman -- no person reads and responds to "what's the weather?" and "analyze my health trends for the past 6 months" in the same time.

**Fix:** Response timing should correlate with actual processing complexity. Simple queries: fast (< 500ms). Complex queries: acknowledge fast (< 300ms), then stream the work.

### "The Patronizing Pause"
**Violates:** P4 (Adapt to the Human)

Full human-like conversational rhythm applied to an expert user who has signaled they want speed. Typing bubbles, warm greetings, step-by-step narration -- for someone who texted "backup status" and wants three numbers.

**Fix:** Detect expertise signals (terse queries, high interaction count, explicit "just tell me"). Shift to expert mode: fast responses, data-dense formatting, no theater.

### "Theater Without Substance"
**Misapplies:** P2 (Show the Work)

Showing elaborate "work" narratives ("Analyzing your data across 5 dimensions... Cross-referencing with industry benchmarks... Applying AI-powered insights...") when the output is generic, wrong, or unhelpful. The effort heuristic amplifies existing opinions -- visible effort on bad output makes the user MORE dissatisfied.

**Fix:** Before streaming a work narrative, assess: (1) Is the system actually performing distinct steps, or generating in a single pass? (2) Is the output confidence above a threshold you'd show to a user? If yes to both, show the work. If the system is doing a single-shot generation with no intermediate steps, suppress the narrative and deliver the result directly with a P1 acknowledgment. A fabricated multi-step narrative on a single-shot output is the anti-pattern -- not the absence of narration.

### "The Uncanny Response"
**Misapplies:** P4 (Adapt to the Human)

Near-human conversational behavior that falls slightly short of human -- variable typing speeds, occasional "hmm" interjections, simulated pauses that don't quite match human rhythm. Instead of warmth, users feel unease. The uncanny valley applies to interaction patterns, not just visual avatars.

**Fix:** Be clearly helpful rather than almost-human. A system that openly signals "I'm an AI assistant checking your data" earns more trust than one that tries to pass as human and fails.

### "The Crying Wolf"
**Misapplies:** P3 (Earn Trust Through Friction)

Deliberate friction on routine, low-stakes actions. "Are you sure you want to change this filter?" "Confirm: sort by date ascending?" When every action requires confirmation, users stop reading confirmations. When the real high-stakes moment arrives ("Delete all backups for Tenant X?"), they click through it reflexively.

**Fix:** Reserve friction exclusively for irreversible, high-stakes, or consequential actions. Everything else should be fast and optimistic with easy undo.

---

## Conflict Resolution

When two principles apply to the same scenario, use these rules to determine which takes precedence.

### P1 (Never Leave in Dark) vs P2 (Show the Work)

**Threshold: 1 second.**

Operations completing in < 1s: P1 wins. Simple acknowledgment -- skeleton screen, optimistic update, brief status flash. Don't interrupt a fast operation with a "showing work" narrative.

Operations taking > 1s: P2 kicks in. Show progress, partial results, or a step-by-step narrative. P1 still applies for the initial acknowledgment (< 300ms), then P2 takes over for the duration.

Operations taking > 10s: P2 is mandatory. At this duration, the user needs real information about what's happening and an estimate of when it will finish.

### P1 (Never Leave in Dark) vs P3 (Earn Trust Through Friction)

**These compose, not compete.**

P1 governs the initial acknowledgment window (< 300ms). P3 governs the verification sequence that follows. For high-stakes actions: acknowledge receipt within 300ms, then begin the friction flow. P1 is never skipped -- even when P3 demands a deliberate multi-step verification, the user gets immediate confirmation that their action was received.

Example: User says "delete all backups for Tenant X." P1: within 300ms, agent responds "Got it -- let me verify that." P3: then runs the friction flow ("Checking for active restores... Verifying admin permissions... Confirming scope: 2,341 items across 3 connections...").

### P2 (Show the Work) vs P3 (Earn Trust Through Friction)

**Tie-breaker: intent. Safety steps vs. quality steps.**

At the implementation level, P2 and P3 can produce visually similar patterns (visible step sequences). The distinction is intent, which determines whether the steps can be suppressed for expert users:

- **P3 steps (safety):** Steps that verify safety, permissions, scope, or reversibility ("checking for active restores," "verifying admin permissions"). These are trust anchors -- they CANNOT be skipped regardless of user expertise. P4 never overrides P3.
- **P2 steps (quality):** Steps that signal thoroughness and effort ("cross-referencing 5 data sources," "analyzing patterns across 3 domains"). These are quality signals -- they CAN be abbreviated or skipped for expert users (P4 overrides P2 for reversible actions).

When a sequence contains both P3 safety steps and P2 quality steps, show the P3 steps always. Show the P2 steps according to the P2 vs P4 rules (novice: show; expert on reversible: skip).

### P2 (Show the Work) vs P4 (Adapt to Human)

**Tie-breaker: user expertise + action reversibility.**

- **Novice user:** P2 takes precedence. Always show the work -- novices need the narrative to build understanding and trust. Even if it's slower, the educational value and trust-building justify it.
- **Expert user, reversible action:** P4 takes precedence. Skip the narrative, deliver results fast. Expert users interpret unnecessary narration as patronizing.
- **Expert user, irreversible action:** P2 takes precedence. Even experts need to see the work when the stakes are high. The narrative here serves as a final safety check, not education.

### P3 (Earn Trust Through Friction) vs P4 (Adapt to Human)

**Tie-breaker: action reversibility.**

- **Irreversible actions:** P3 always wins, regardless of user expertise. Deleting data, sending messages, making payments -- everyone gets friction. Experts get a single, clear confirmation. Novices get a multi-step flow with scope explanation.
- **Reversible high-stakes actions:** P4 modulates. Experts get a streamlined confirmation (one step, no explanation). Novices get the full friction flow (multi-step, scope explanation, "here's how to undo this").
- **Routine actions:** Neither P3 nor P4 suggests friction. P1 handles these.

### Default Rule

When in doubt, bias toward P1 (Never Leave Them in the Dark). It is never wrong to tell the user what's happening. An unnecessary status message is mildly redundant. A missing status message makes the user wonder if the system is broken.

---

## Decision Framework

The implementer's cheat sheet. Given an action type and user segment, this matrix tells you which principle applies and what pattern to use.

| Action Type | Primary Principle | Pattern | Novice Modifier | Expert Modifier | Channel Notes |
|-------------|------------------|---------|-----------------|-----------------|---------------|
| **Read data** (view, search, query) | P1: Never Leave in Dark | Skeleton screen; stream results progressively | Add contextual tooltips explaining what's loading | Show data density indicators; skip tooltips | All channels support this |
| **Filter / sort** | P1: Never Leave in Dark | Optimistic UI -- update immediately, sync in background | Brief "Filtering..." toast | Silent update, no toast | Web only (agent channels don't have filters) |
| **Generate view** (report, dashboard, analysis) | P2: Show the Work | Staged progress with step descriptions; source attribution | Full narrative ("Querying your Clio matters... Found 47...") | Abbreviated progress ("47 matters, 3 stale backups") | iMessage: chunk into multiple messages. WhatsApp: use media for complex views |
| **Agent coaching response** (insight, advice, analysis) | P2: Show the Work | Stream reasoning steps; name data sources | Warm language, step-by-step explanation, proactive context | Data-dense, terse, skip explanation unless asked | Voice: periodic verbal updates during processing |
| **Agent write-back** (reschedule, pay, send) | P3: Earn Trust Through Friction | Preview changes; require explicit confirmation; staged execution | Multi-step flow with scope explanation + "here's how to undo" | Single confirmation with change summary | iMessage: use Interactive Forms for confirmation. WhatsApp: use Flows. Voice: read back and wait for verbal "yes" |
| **Destructive action** (delete, disconnect, revoke) | P3: Earn Trust Through Friction | Scope summary -> validation checks -> explicit confirmation with danger styling | Full scope explanation + "this cannot be undone" warning + 5s cooldown | Scope summary + single "Delete permanently" confirmation | All channels: no shortcuts, no "don't show again" |
| **Onboarding flow** (first-time setup) | P4: Adapt to Human (novice default) | Warm, guided, conversational. Show what each step accomplishes. Progress indicator for multi-step flows. | Full guidance is the default -- this IS the novice experience | Allow "skip intro" or "set up later" for users who know what they want | iMessage/WhatsApp: interactive forms for structured setup. Web: wizard UI with progress bar |
| **Long-running task** (> 10s processing) | P1 + P2 (compound) | Acknowledge within 300ms (P1); stream partial findings as available (P2); provide time estimate if possible ("checking 3 sources, usually ~20s") | Warm verbal updates every 5s; explain each step | Terse status only; suppress explanations; provide estimated completion | Voice: verbal fill every 2s mandatory. WhatsApp: intermediate message every ~5s. iMessage/web: progressive streaming |
| **Error / recovery** | P1: Never Leave in Dark + P2: Show the Work | Explain what went wrong in plain language. Show what was attempted. Offer a clear next step. | "Something went wrong" + plain language explanation + one-click retry | Error code + technical detail + manual retry option | Voice: "I hit a problem. Here's what happened..." -- never silence after an error |

---

## Review Cadence

This manifesto is a living document, not a monument.

- **Annual:** Refresh academic sources. Check if cited findings have been replicated, extended, or challenged. Update confidence levels.
- **Quarterly:** Check against shipping product reality. Are the patterns being followed? Are new patterns emerging that should be codified?
- **Before implementation:** Re-verify channel constraints for the specific channel being built. API capabilities change. iOS versions advance. WhatsApp policies evolve.
- **After user research:** When real users interact with Akasha products, update the manifesto with empirical findings. Replace "research suggests" with "our data shows."
