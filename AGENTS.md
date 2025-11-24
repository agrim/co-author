# AGENTS.md – NovelForge

This document describes the **agentic architecture** of NovelForge – the roles, responsibilities, and collaboration patterns of the agents that together implement the app.

The intent is to give any future AI assistant (or human developer) a precise playbook for extending or maintaining the system.

---

## 1. Vision & Non-Goals

### 1.1 Vision

Build a **native Apple, AI-assisted novel studio** that:

- Lets authors move from **idea → outline → draft → revision → export** with minimal friction.
- Keeps **story continuity** rock-solid via a Story Bible and compacted context.
- Offers **deep control** over style, genre, emotion, and rating.
- Uses **multiple LLM providers** behind a clean abstraction.
- Treats the author’s words and privacy with **respect and safety**.

### 1.2 Non-Goals

- Not a collaborative Google Docs clone (multi-user real-time editing is out of scope for v1).
- Not a typesetting/typesetting-perfect publishing tool (EPUB/PDF exports are “good enough drafts”).
- Not a plagiarism engine; **author presets are behavioural approximations**, not imitation of exact prose.

---

## 2. Global Constraints & Shared Concepts

### 2.1 Technical Constraints

- **Swift + SwiftUI only** for app logic and UI.
- **Apple-native frameworks only**:
  - SwiftUI, AppKit/UIKit as needed
  - SwiftData/Core Data
  - CloudKit
  - URLSession
  - Keychain
  - StoreKit (if monetization added)
- No third-party SDKs, no cross-platform frameworks.

### 2.2 Core Domain Models

All agents share these conceptual entities:

- `NovelProject`
  - `id`, `title`, `tagline`
  - `ageRating`, `nsfwEnabled`
  - `targetWordCount`, `targetChapters`
  - `globalStyleProfile`
  - `storyBibleId`, `rootBranchId`
- `StoryBible`
  - Characters, locations, rules, timelines, themes.
- `Branch`
  - `id`, `name`, `parentBranchId?`, `forkedFromChapterId?`
  - Ordered list of `Chapter` ids.
- `Chapter`
  - `id`, `title`, `index`, `branchId`
  - `outline` (beats, scenes, POV, notes)
  - `currentRevisionId`
- `Revision`
  - `id`, `chapterId`, `createdAt`, `createdBy` (AI or user)
  - `contentMarkdown`
  - `summary` (short)
- `StyleProfile`
  - Genre, pacing, sentence complexity, dialogue ratio, emotion curve.
- `AIProviderConfig`
  - Provider (`openai`, `gemini`, `xai`, `custom`)
  - Model, temperature, maxTokens, etc.
- `SafetyProfile`
  - Age rating, NSFW flag, sliders for violence/language/sexuality.

---

## 3. Agent Overview

High-level agents:

1. **Conductor Agent**
2. **Product & UX Agent**
3. **Architecture & Platform Agent**
4. **Data & Persistence Agent**
5. **AI Provider Abstraction Agent**
6. **Preset & Style Agent**
7. **Novel Outline Agent**
8. **Chapter Planning Agent**
9. **Story Bible & Continuity Agent**
10. **Context Summarization Agent**
11. **Chapter Drafting Agent**
12. **Editing & WYSIWYG Agent**
13. **Branching & Version Control Agent**
14. **Sync & Cloud Agent**
15. **Safety & Rating Agent**
16. **Export & Formatting Agent**
17. **QA & Testing Agent**
18. **Docs & Examples Agent**

Each section below describes an agent’s purpose, inputs, outputs, and success criteria.

---

## 4. Conductor Agent

**Role:** Overall orchestrator. Decides which agents to call in what order for a given user action.

- **Inputs:**
  - User intent (e.g., “Generate Chapter 3 draft”).
  - Current `NovelProject` state.
- **Outputs:**
  - Updated domain entities (chapters, revisions, summaries).
- **Responsibilities:**
  - Map high-level flows:

    - New project → Outline → Chapters.
    - Generate next chapter.
    - Regenerate a chapter.
    - Fork branch.
    - Export novel.

  - Route calls:
    - Outline Agent → Story Bible Agent → Summarization Agent → Chapter Planning Agent → Chapter Drafting Agent → Safety Agent → Persistence Agent.
- **Success:** All complex user actions resolve into a predictable, repeatable pipeline.

---

## 5. Product & UX Agent

**Role:** Ensures workflows and UI flows are intuitive and aligned with writer needs.

- **Inputs:**
  - User stories, feature descriptions.
- **Outputs:**
  - UX flows, screen maps, SwiftUI view architecture guidelines.
- **Responsibilities:**
  - Define key screens:
    - Project list
    - Project dashboard (stats, branches, outline)
    - Chapter editor (with WYSIWYG + Markdown modes)
    - Story Bible view
    - Branch tree / timeline
    - Settings (AI, style presets, safety).
  - Keep UI minimal and writing-centric.
- **Success:** Authors can onboard and produce a first chapter without documentation.

---

## 6. Architecture & Platform Agent

**Role:** Defines app architecture using only Apple frameworks.

- **Inputs:**
  - Feature list, technical constraints.
- **Outputs:**
  - Overall module layout, dependency graph, naming conventions.
- **Responsibilities:**
  - Enforce layering:
    - **Domain** (pure Swift) vs **Infrastructure** (CloudKit, URLSession) vs **UI** (SwiftUI).
  - Use:
    - SwiftData/Core Data for persistent store.
    - CloudKit for sync.
    - URLSession for LLM HTTP calls.
  - Keep the `LLMClient` abstraction thin and testable.
- **Success:** Codebase is modular, testable, and extendable without breaking Apple-only constraint.

---

## 7. Data & Persistence Agent

**Role:** Model and persist all domain objects.

- **Inputs:**
  - Domain model definitions.
- **Outputs:**
  - SwiftData/Core Data schemas, migration strategies.
- **Responsibilities:**
  - Map domain concepts (Project, Branch, Chapter, Revision, StoryBible) to persistent storage.
  - Design indexes for:
    - Query by project, branch, chapter index.
    - Fast lookups for story bible entries.
  - Implement:
    - Auto-save & manual save.
    - Efficient revision storage (avoid full duplication if possible).
- **Success:** Data remains consistent and resilient, even under frequent writes and sync.

---

## 8. AI Provider Abstraction Agent

**Role:** Provide a unified interface to multiple LLM providers.

- **Inputs:**
  - `AIProviderConfig`, prompt, context objects.
- **Outputs:**
  - Text completions / chat responses.
- **Responsibilities:**
  - Define `LLMClient` protocol, e.g.:

    ```swift
    protocol LLMClient {
        func complete(prompt: LLMRequest) async throws -> LLMResponse
    }
    ```

  - Implement adapters for:
    - OpenAI
    - Gemini
    - xAI
  - Handle:
    - URLSession usage
    - JSON encoding/decoding
    - Error handling + retries + rate-limit backoff
  - Enforce injection of:
    - Safety constraints
    - Style presets
- **Success:** Adding a new provider is easy; callers don’t care which provider is in use.

---

## 9. Preset & Style Agent

**Role:** Manage author style presets and custom style profiles.

- **Inputs:**
  - Library of behaviour descriptions for top 50 authors.
  - User-defined style settings.
- **Outputs:**
  - Final `StyleProfile` and prompt fragments for LLM requests.
- **Responsibilities:**
  - Encode each preset as:
    - Narrative behaviours (dialogue ratio, pacing, etc.).
    - Parameter ranges (temperature, etc.).
    - Prompt string describing style.
  - Apply user modifications on top of presets.
  - Expose a UI for:
    - Choosing presets.
    - Saving custom styles.
- **Constraints:**
  - Avoid explicit mimicry instructions; describe style abstractly.
- **Success:** Users feel noticeable differences between style presets and can tune them.

---

## 10. Novel Outline Agent

**Role:** Help the user move from broad idea to a structured outline.

- **Inputs:**
  - Project synopsis, themes, characters.
  - Style & genre info.
- **Outputs:**
  - Act structure (e.g., 3-act).
  - Chapter list with rough summaries and goals.
- **Responsibilities:**
  - Ask the LLM to propose outline variations.
  - Allow user adjustments:
    - Add/remove/reorder chapters.
    - Edit beats and notes.
  - Store outline per chapter.
- **Success:** The outline reflects both the user’s intention and structural best practices.

---

## 11. Chapter Planning Agent

**Role:** Plan each chapter in detail before drafting.

- **Inputs:**
  - Chapter’s outline.
  - Story Bible.
  - Compacted context summary.
  - Style & safety profiles.
- **Outputs:**
  - Detailed chapter plan:
    - Scene list
    - Emotional beats
    - Key reveals
    - POV and tense confirmation
- **Responsibilities:**
  - Generate structured plan (e.g., JSON or structured Swift type) via LLM.
  - Ensure that the plan advances arcs and respects continuity.
- **Success:** Chapter drafts written by the LLM feel guided and purposeful.

---

## 12. Story Bible & Continuity Agent

**Role:** Maintain the Story Bible and enforce continuity.

- **Inputs:**
  - Outline, chapter plans, final chapter texts.
- **Outputs:**
  - Updated `StoryBible`.
  - Continuity warnings.
- **Responsibilities:**
  - Extract & update:
    - Character bios & arc summaries.
    - Locations, rules, timelines.
  - Use LLM to detect:
    - Obvious continuity errors (age/time/location contradictions, etc.).
  - Provide quick reference views for the writer.
- **Success:** Long novels remain internally consistent as far as possible.

---

## 13. Context Summarization Agent

**Role:** Distil the entire story so far into compact LLM-friendly context.

- **Inputs:**
  - Story Bible, all accepted chapters (or branch subset).
- **Outputs:**
  - `CompactedContext` structure:
    - “Previously on…” summary.
    - Bullet list of key unresolved threads.
    - Short character state snapshots.
- **Responsibilities:**
  - Maintain the summary after each accepted chapter.
  - Ensure summary size stays under a configurable token budget.
- **Success:** Later chapters feel consistent and relevant, with no heavy repetition of past text.

---

## 14. Chapter Drafting Agent

**Role:** Turn a chapter plan + context into a prose draft via the LLM.

- **Inputs:**
  - Chapter plan.
  - Compacted context.
  - StyleProfile.
  - SafetyProfile.
- **Outputs:**
  - Draft chapter in Markdown.
  - Short per-chapter summary.
- **Responsibilities:**
  - Craft prompts combining:
    - Story so far
    - Scene plan
    - Style specification
    - Safety & rating rules
  - Provide:
    - First full draft
    - Optional “alternatives” for tricky scenes.
- **Success:** Drafts are coherent, stylistically aligned, and mostly safe. Good enough for revision.

---

## 15. Editing & WYSIWYG Agent

**Role:** Power the rich text editing and Markdown rendering.

- **Inputs:**
  - Markdown text from drafts or revisions.
- **Outputs:**
  - Attributed text for SwiftUI views.
  - Updated Markdown after user edits.
- **Responsibilities:**
  - Provide dual-mode editor:
    - WYSIWYG (formatted)
    - Raw Markdown
  - Implement:
    - Formatting actions (bold, italics, headings, lists).
    - Inline comments and notes.
    - Diff view between revisions.
- **Success:** Editing is fast, stable, and visually pleasant; no phantom formatting glitches.

---

## 16. Branching & Version Control Agent

**Role:** Manage branches, forks, and revisions.

- **Inputs:**
  - User actions (fork branch, revert chapter, save snapshot).
  - Existing project state.
- **Outputs:**
  - Updated branch tree.
  - New `Revision` entries.
- **Responsibilities:**
  - Support:
    - Forking at any chapter.
    - Keeping independent timelines per branch.
    - Naming and tagging branches.
    - Recording revision history.
  - Provide diffing logic:
    - Between two revisions of a chapter.
    - Between branches at the same chapter index.
- **Success:** Users can experiment freely without losing work or getting confused.

---

## 17. Sync & Cloud Agent

**Role:** Handle iCloud/CloudKit sync and basic conflict resolution.

- **Inputs:**
  - Local database mutations.
  - CloudKit updates.
- **Outputs:**
  - Synced project data across devices.
  - Conflict resolution prompts when necessary.
- **Responsibilities:**
  - Map domain entities to CloudKit records.
  - Batch updates efficiently.
  - Handle:
    - Offline queues.
    - Basic merges.
- **Success:** Projects transparently appear on all devices with minimal user intervention.

---

## 18. Safety & Rating Agent

**Role:** Enforce age ratings, NSFW settings, and general safety.

- **Inputs:**
  - `SafetyProfile`, chapter drafts.
- **Outputs:**
  - Revised prompts, guidance to LLM.
  - Warnings or suggestions for user.
- **Responsibilities:**
  - Inject rating constraints into prompts.
  - Optionally run a second “self-check” LLM pass to detect potential misalignment.
  - Provide UI warnings:
    - “This draft may exceed your selected rating.”
- **Success:** Users generally receive content aligned with their selected rating and comfort level.

---

## 19. Export & Formatting Agent

**Role:** Compile and export the novel.

- **Inputs:**
  - Selected branch & revision set.
  - Export settings.
- **Outputs:**
  - Markdown, EPUB, PDF files.
- **Responsibilities:**
  - Build a clean export:
    - Proper heading levels, scene breaks, front matter.
  - Generate:
    - Table of contents
    - Basic title page
  - Use native Apple APIs for PDF/EPUB creation.
- **Success:** Users can easily take their work to editors, readers, or publishing tools.

---

## 20. QA & Testing Agent

**Role:** Ensure correctness, performance, and UX quality.

- **Inputs:**
  - Specifications from other agents.
- **Outputs:**
  - Test plans, Swift test suites.
- **Responsibilities:**
  - Define unit tests for:
    - Branching logic
    - Persistence
    - Summarization length bounds
  - Define integration tests:
    - New project → first chapter flow
    - Forking and export flows.
- **Success:** Regressions are caught early; core workflows remain solid.

---

## 21. Docs & Examples Agent

**Role:** Maintain user-facing and developer-facing documentation.

- **Inputs:**
  - Feature set, architecture decisions.
- **Outputs:**
  - README, AGENTS (this file), inline docs, example projects.
- **Responsibilities:**
  - Keep docs synchronized with implementation.
  - Provide:
    - Quickstart examples
    - “How to add a new AI provider”
    - “How to customize style presets”
- **Success:** New contributors or AI assistants can become productive quickly.

---

## 22. Collaboration Rules Between Agents

- Agents communicate only via **domain models**, not via UI objects.
- Conductor orchestrates; agents don’t call each other directly without going through Conductor (unless explicitly permitted).
- When in doubt:
  - Prefer **stateless, pure logic**.
  - Avoid circular dependencies.

---

With this agentic blueprint, NovelForge can evolve from a single-author prototype into a sophisticated AI-powered novelist’s studio while remaining understandable and maintainable by both human developers and AI assistants.
