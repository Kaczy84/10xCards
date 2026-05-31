---
project: "10xCards"
version: 1
status: draft
created: 2026-05-26
context_type: greenfield
product_type: web-app
target_scale:
  users: small
  qps: low
  data_volume: small
timeline_budget:
  mvp_weeks: 5
  hard_deadline: null
  after_hours_only: true
---

## Vision & Problem Statement

Creating high-quality flashcards manually is time-consuming, which discourages self-directed adult learners from using spaced repetition — an effective but adoption-blocked learning method. The learner knows what a good card looks like and has the study material; the conversion step is the bottleneck, not the knowledge.

The insight: LLMs can now generate high-quality flashcards from raw text in seconds. Existing tools (Anki, Quizlet) pre-date practical AI generation and haven't closed this gap. The timing is right to remove the conversion friction entirely.

## User & Persona

**Primary persona: Self-directed adult learner**

A professional, language learner, or certification candidate who is studying on their own initiative. They have access to study material (articles, notes, textbooks) and want to retain it using spaced repetition. The barrier is the card creation step — not the studying itself.

The moment they reach for this product: they have a block of text they want to learn from and know that manually writing flashcards for it would take longer than the learning value justifies.

## Success Criteria

### Primary
- At least 75% of AI-generated flashcards are accepted by the user (not edited beyond recognition or deleted).
- At least 75% of all flashcards in the system are created via the AI generation flow, not manually.

### Secondary
- Users return for a second review session — a signal that the spaced repetition loop is engaging enough to revisit.

### Guardrails
- User flashcard data must never be lost or mixed between accounts. Data integrity failure invalidates the product regardless of AI quality.
- AI-generated cards must always be editable and deletable by the user. The user remains in control of their deck at all times.
- The app must remain usable (manual card creation + review of existing cards) if the AI generation service is unavailable.

## User Stories

### US-01: Learner generates flashcards from a text

- **Given** a logged-in learner who has pasted a block of study text
- **When** they trigger AI generation
- **Then** they see a bulk list of suggested flashcards, can select/deselect each, edit any card's front or back, and save the chosen set to their account

#### Acceptance Criteria
- The batch of suggestions appears after a single trigger action (no multi-step wizard)
- Each card in the list is independently selectable; deselected cards are discarded, not saved
- Editing a card's text does not remove it from the selectable list
- After saving, the accepted cards appear in the learner's card list

### US-02: Learner reviews flashcards

- **Given** a logged-in learner with at least one saved flashcard
- **When** they start a review session
- **Then** the app presents their cards one at a time in random order; the learner responds (remembered / not remembered); the app records the response

#### Acceptance Criteria
- All saved cards are included in the session (no scheduling filter for MVP)
- Each response is recorded
- When no cards exist, the app shows an informative empty state (not a broken screen)

## Functional Requirements

### Authentication
- FR-001: Learner can register an account with email and password. Priority: must-have
  > Socrates: Counter-argument considered: "registration friction is the #1 drop-off in new apps; anonymous mode first." Resolution: kept as written — cloud storage and persistence are core to the product promise; anonymous mode is not needed for MVP.
- FR-002: Learner can log in and log out of their account. Priority: must-have
  > Socrates: Counter-argument considered: "session management complexity may be underestimated." Resolution: kept as written — implementation choice (managed auth service) is downstream; the FR itself is not at risk.

### AI Generation
- FR-003: Learner can paste a block of text and receive a batch of AI-generated flashcard suggestions. Priority: must-have
  > Socrates: Counter-argument considered: "inconsistent AI quality erodes trust faster than no AI." Resolution: kept as written — AI generation is the core value proposition. Quality risk is mitigated by the bulk review step (FR-004) which keeps the learner as the final quality gate.
- FR-004: Learner can review the generated batch (bulk list), select which cards to save, and discard the rest. Priority: must-have
  > Socrates: Counter-argument considered: "bulk list encourages rubber-stamping, inflating the 75% acceptance metric without proving quality." Resolution: kept as written — the learner is the quality gate; rubber-stamping would show up as poor retention in the review data.
- FR-005: Learner can edit a generated card's front or back text before saving it. Priority: must-have
  > Socrates: Counter-argument considered: "inline editing adds friction to the bulk selection flow." Resolution: kept as essential — without editing, borderline cards get discarded rather than improved; this recovers value from near-correct suggestions.

### Manual Creation
- FR-006: Learner can create a flashcard manually by writing the front and back. Priority: must-have
  > Socrates: Counter-argument considered: "if 75% of cards come from AI, manual creation dilutes MVP focus." Resolution: kept as safety net — when AI fails (bad quality, service outage, niche topic), the learner must have a fallback path. Removing it makes the product brittle.

### Card Management
- FR-007: Learner can view all their saved flashcards in a list. Priority: must-have
- FR-008: Learner can edit a saved flashcard's front or back text. Priority: must-have
- FR-009: Learner can delete a saved flashcard. Priority: must-have
  > Socrates (FR-007–009): Counter-argument considered: "adding full card management CRUD alongside AI generation and review dilutes MVP focus." Resolution: all three kept as must-have — a learner who can't fix or remove bad cards will churn. The deck degrades without edit/delete; view is how learners access cards outside review sessions.

### Review
- FR-010: Learner can start a review session; the app presents their saved cards in random order. Priority: must-have
- FR-011: Learner can respond to each card during review (remembered / not remembered); the app records the response. Priority: must-have
  > Socrates (FR-010–011): Counter-argument considered: "integrating a spaced repetition algorithm is the riskiest scope item; defer to v2." Resolution: REVISED — SRS integration deferred to v2 (FR-012). MVP uses a simple random shuffle for review. The learning loop is still proven (users return, respond to cards); the scheduling optimization follows once the loop is validated.
- FR-012: Integration with a spaced repetition algorithm (replacing the random shuffle) to schedule cards by learning priority. Priority: nice-to-have (v2)

## Non-Functional Requirements

- A learner's flashcards are visible only to that learner; no card or account data is accessible to or mixed with another user's session, even under partial failure conditions.

## Business Logic

The app evaluates pasted text and decides which content is worth turning into a flashcard.

The input is a block of plain text provided by the learner — a paragraph, a chapter excerpt, a set of notes, or any freeform prose. The output is a batch of flashcard suggestions, each structured as a front (question or prompt) and back (answer or definition). The learner encounters this as a bulk list: they did not specify what to extract or how to phrase it; the app made those decisions.

The rule is not just extraction — it also involves phrasing: a learnable fact is reformatted into a question-answer pair suitable for active recall practice. How the app determines what qualifies as "worth learning" from a given text, and how it phrases the resulting card, is the core domain decision.

## Access Control

Multi-user web app. Authentication via email and password only (no OAuth for MVP). Flat user model — all authenticated users have equal access to their own flashcard sets. No admin panel, no tiers in MVP. Unauthenticated users cannot access flashcard features.

## Non-Goals

- No multi-format import (PDF, DOCX, images, OCR): input is text paste only. File upload pipeline is a v2 concern.
- No flashcard sharing between users: each deck is private to one account. No public decks, no export-to-friend, no collaborative editing.
- No mobile apps (iOS/Android): web browser only for MVP. Native builds are explicitly out of scope.
- No custom spaced repetition algorithm: MVP uses a simple random shuffle for review. Integration with a ready-made SRS algorithm (FR-012) is deferred to v2. No building of proprietary SM-2 or SuperMemo variants.
- No integrations with other educational platforms (LMS, Duolingo, Anki export, etc.).

## Open Questions

None. All gray areas were resolved during shaping (quality_check_status: accepted).
