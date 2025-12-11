# CoAuthor

An agentic, native Apple novel-writing studio that lets you describe a story in broad strokes, optionally add a chapter-wise structure, and then have AI draft the novel chapter-by-chapter — while preserving continuity, tone, and your personal style choices.

> **Note:** “CoAuthor” is a working name. Feel free to rename the app; the architecture and AGENTS remain valid.

---

## 1. High-Level Concept

CoAuthor is a **multi-agent, AI-assisted novel writing app** built entirely with **native Apple technologies** (SwiftUI, SwiftData/Core Data, CloudKit, URLSession, etc.).

You provide:

- A **high-level idea** of the novel (logline, themes, characters, arcs)
- Optional **chapter-by-chapter outline**
- **Style controls**: genre, pacing, emotion, POV, NSFW toggle, age rating, etc.

The app then:

1. Builds a **story bible** and **compacted context summary**.
2. Plans each chapter given the global outline.
3. Calls your chosen **AI engine** (OpenAI, Gemini, xAI, or more via HTTP) to draft the chapter.
4. Updates the story bible and compact summary to keep later chapters coherent.
5. Lets you **edit, branch, revise, compare versions, and export**.

---

## 2. Key Features

### 2.1 Multi-Engine AI Support

- **Pluggable AI providers** via a clean `LLMClient` protocol:
  - OpenAI (GPT family via HTTPS + URLSession)
  - Google Gemini
  - xAI Grok
  - Custom / self-hosted models (extendable)
- Per-project configuration:
  - Selected provider & model
  - Max tokens / temperature / top-p / etc.
  - Safety parameters aligned with:
    - **NSFW toggle**
    - **Age rating** (e.g., 10+, 13+, 16+, 18+)

> Implementation detail: no 3rd-party SDKs; only Apple frameworks + plain HTTP.

---

### 2.2 Story Inputs & Structure

- **Project creation**:
  - Title, working title, tagline
  - Genre(s) (e.g., fantasy, sci-fi, romance, thriller, lit-fic)
  - Target **age rating** and **NSFW** toggle
  - Target **novel length** (words) and **chapter count**
  - Tone & emotion style (e.g., melancholic, hopeful, tense)
- **Broad strokes input**:
  - Logline & synopsis
  - Key themes + motifs
  - Main characters (bio, goals, flaws)
  - High-level 3-act or 4-act structure (optional)
- **Chapter structure (optional)**:
  - Chapter titles / beats / scene ideas
  - “Must-hit moments” and constraints
  - POV and tense per chapter (e.g., 1st person past, close 3rd present)

---

### 2.3 Context Management & Story Bible

To preserve continuity, CoAuthor maintains:

- A **Story Bible**:
  - Characters (traits, history, relationships)
  - Locations, world rules, tech/magic systems
  - Timelines and key events
  - Recurring motifs and themes
- A **Compacted Context Summary** per state:
  - A distilled, token-budget friendly representation of everything written so far.
  - Continuously updated after each accepted chapter.
  - Fed into the LLM to keep later chapters coherent.

You can:

- View & edit the Story Bible manually.
- See **auto-generated “What’s happened so far”** summaries per chapter or branch.
- Compare canon across branches to detect contradictions.

---

### 2.4 Author Style Presets (Top 50 Authors)

CoAuthor offers **style presets** inspired by the **writing behaviours of top authors across genres** (not direct imitation / plagiarism):

- Preset attributes:
  - Sentence length & rhythm
  - Dialogue vs description balance
  - Pacing curve (slow burn vs rapid)
  - Emotional intensity
  - Use of figurative language vs minimalism
- Presets grouped by:
  - Genre (e.g., epic fantasy, crime, romance, literary)
  - Era (classic vs contemporary)
  - Tone (dark, hopeful, whimsical, clinical)
- You can:
  - Start from a preset and then tweak sliders.
  - Save your own **custom style profiles**.
  - Mix presets (“70% Preset A + 30% Preset B”).

> Internally, presets map to prompt templates & sampling parameters for the chosen LLM.

---

### 2.5 Fine-Grained Writing Controls

Per novel, chapter, or branch:

- **Genre & subgenre**
- **Chapter length**:
  - By approximate words, pages, or “short / medium / long”
- **Number of chapters** & total word-count target
- **POV & tense** per chapter
- **Emotion style** per chapter (e.g., “high tension”, “bittersweet calm”)
- **Violence / language / sexual content sliders**
- **NSFW toggle**:
  - When OFF, strict safe-content prompting.
  - When ON, still respects platform / provider policies, but allows more mature themes.
- **Age rating**:
  - Guides the LLM to align content with a given maturity band.

---

### 2.6 WYSIWYG + Markdown Editor

- **Dual-mode editor**:
  - WYSIWYG view with formatting toolbar (bold, italics, headings, lists, blockquotes, scene breaks).
  - Raw **Markdown** view for power users.
- Features:
  - Inline **track-changes style diff** between AI draft and your edits.
  - **Inline comments** & notes per paragraph.
  - **Split editor**: current chapter vs prior chapter / outline.
  - Minimal, distraction-free mode.

Technical:

- Built with **SwiftUI** text views & custom layout.
- Markdown parsing/rendering via native tools (e.g., `AttributedString` + Markdown support).
- No non-Apple UI frameworks.

---

### 2.7 Chapter-Wise Storage & Story Branches

Each project contains:

- Chapters in a **timeline**:
  - `Chapter 1`, `Chapter 2`, … `Chapter N`
- Each chapter maintains:
  - AI draft(s)
  - User edits
  - **Snapshots / revisions**
  - Notes & metadata (POV, emotional beat, etc.)

**Branching:**

- Fork the story from:
  - Any chapter
  - Any revision
- Maintain multiple endings or alternate arcs:
  - `Mainline`
  - `Alt Ending A`
  - `What-if: character X dies in Ch. 7`
- Branch management:
  - Branch tree view
  - Per-branch metrics (word count, tone summary)
  - Merge or cherry-pick passages between branches.

---

### 2.8 Revisions, Version Control & History

- Automatic **snapshots** before and after:
  - AI regeneration
  - Major user edits
- Inline **revision comparison**:
  - Side-by-side diff
  - Highlighted changes
- **Named versions**:
  - “First draft”, “Editor pass”, “Beta version”
- Per-novel **timeline**:
  - Chronological history of events and changes.

Implementation:

- Backed by SwiftData/Core Data entities (`Project`, `Chapter`, `Revision`, `Branch`).
- Efficient local storage with incremental saving.

---

### 2.9 Cloud Sync & Cross-Device Experience

- All data stored locally with option to sync via **CloudKit & iCloud**:
  - Projects, chapters, branches, revisions.
  - Story Bible and style presets.
  - AI settings (keys never leave secure storage unless needed for API call).
- Targets:
  - macOS (primary)
  - iPadOS (secondary)
  - iOS (optional, focus on reading & light editing)

Design:

- **Offline-first**:
  - Full functionality without network (except AI calls).
- **Conflict-aware sync**:
  - Merge simple edits automatically.
  - Show conflict dialogs for complex cases.

---

### 2.10 Safety, Ratings & Content Controls

- Global project **rating** and **NSFW flag** are first-class citizens.
- LLM prompts always include:
  - Safe content guidelines derived from rating.
  - Clarity about what is *not* allowed.
- Optional **content scanner**:
  - After generation, a separate pass checks for rating alignment (as far as LLM’s self-check allows).
- Age rating influences:
  - Violence / gore descriptions
  - Sexual / romantic explicitness
  - Language intensity
  - Themes (e.g., self-harm, addiction handled with care)

---

### 2.11 Export & Integration

Export options:

- **Markdown** (per chapter or whole novel)
- **EPUB** (draft-quality)
- **Plain text**
- **PDF** (with chapter headings & page numbers)

Other:

- Simple **live word-count stats**:
  - Per chapter, per branch, per project, per POV.
- Basic **outline export**:
  - Act structure, key beats, character arcs.

---

## 3. Architecture Overview

CoAuthor is built using a **multi-agent orchestration model** internally:

- **Conductor**:
  - Orchestrates the agents for planning, drafting, revising, and summarizing.
- **Agents**:
  - Outline Planner, Chapter Planner, Chapter Writer, Style Manager, Story Bible Manager, Summarizer, Safety Checker, Sync Manager, etc.

Each agent:

- Is pure Swift logic.
- Communicates through in-memory models (`NovelProject`, `Chapter`, `StoryBible`, `StyleProfile`).
- Uses only Apple frameworks (SwiftUI, SwiftData/Core Data, CloudKit, URLSession, etc.).

> See **AGENTS.md** for a detailed description of each agent and their contracts.

---

## 4. Tech Stack

- **Language**: Swift
- **UI**: SwiftUI (macOS, iPadOS, iOS)
- **State / Reactivity**: Observation (Swift macros), Combine where needed.
- **Persistence**: SwiftData or Core Data (configurable), plus local filesystem for exports.
- **Sync**: CloudKit (iCloud containers).
- **Networking**: URLSession for calling AI providers.
- **Security**:
  - Keychain for API keys / tokens.
  - No 3rd-party SDKs.

---

## 5. Getting Started (Development)

1. **Requirements**
   - Xcode (latest stable)
   - macOS (latest stable)
   - Apple Developer account (for CloudKit / iCloud features)

2. **Clone & open**
   - Open the Xcode project/workspace.
   - Select the `CoAuthor` scheme.

3. **Configure AI providers**
   - In app settings, add API keys for:
     - OpenAI
     - Gemini
     - xAI
   - Keys should be stored using Keychain.

4. **Run**
   - Build & run on macOS or simulator.
   - Create a new project and experiment with:
     - High-level synopsis input.
     - Auto-generated outline and chapters.
     - Branching & revisions.

---

## 6. Roadmap Ideas

- Character relationship graphs and timeline visualisation.
- Collaboration mode (share read-only branch with beta readers).
- Integrated note cards / corkboard view.
- AI-assisted line-editing and copy-editing modes.
- Multilingual support for writing in non-English languages.
- Better EPUB export and formatting options.

---

## 7. License

*TBD* — choose an appropriate open-source or proprietary license as needed.
