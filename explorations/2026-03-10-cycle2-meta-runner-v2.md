# Exploration: Meta-runner v2 — Make It Truly Agentic

**Started:** 2026-03-10
**Status:** Active
**Cycle:** 2
**Source Repo:** [nlscng/meta-runner](https://github.com/nlscng/meta-runner)

---

## Monday: Framing

**Who is the user?**
> Myself — a Netrunner player who already has a working quiz tool (meta-runner v1: 639 LOC, 67 tests, 6 quiz types, 2460 cards, meta-aware questions) and wants to learn how to build real agent behavior.

**What is the idea?**
> Transform meta-runner from a command-driven quiz tool into an agent that demonstrates memory (remembers what you know and don't), planning (proposes and executes a learning strategy per session), and initiative (says things unprompted — adapts difficulty, flags weak areas, redirects mid-session).

**What is the desired outcome?**
> An agent that passes the behavioral test: "You keep undervaluing ice costs — that matters because glacier wins by taxing the runner. Let's explore glacier matchups today." *without the user requesting a topic.* Concretely: ≥3 unprompted agent actions per session grounded in meta reasoning, adaptive concept selection based on understanding history, and a session-opening plan targeting meta knowledge gaps.

**Why does it matter?**
> Cycle 1's key learning was "agent ≠ tool." This project isolates the exact skills that failed — memory, planning, initiative — on top of infrastructure that's already done. Day 1 work IS agent work.

### Research Summary

#### Redefining the Unit of Learning (Revised 2026-03-28)

**Original plan:** SM-2 scheduling over `(card_code, quiz_type)` pairs — "do you know Engolo's install cost?" This is fact recall (trivia), not meta understanding.

**Problem:** In card games, "meta" = the metagame — which decks are winning, why, what counters them, how the landscape shifts. SM-2 optimizes memorization scheduling, but knowing a card's install cost isn't meta knowledge. Knowing *why that cost matters in the context of a glacier matchup* is meta knowledge.

**Revised approach:** The unit of learning becomes a **meta concept** — an archetype, a matchup, a synergy pattern, a strategic principle. SM-2 still works as the scheduling engine, but it schedules concept review, not card trivia.

| Original Unit | Revised Unit | Example |
|--------------|-------------|---------|
| `01042_cost_quiz` | `glacier-ice-economics` | "Why does ice cost matter more in glacier than rush?" |
| `01042_identify_ice` | `kill-deck-counterplay` | "What should you check before running against Jinteki PE?" |
| Card fact recall | `archetype-recognition` | "This deck has 18 ice, avg cost 4+, and scores behind remotes. What archetype?" |
| Popularity stat | `meta-shift-reasoning` | "Endurance was everywhere, then disappeared. Why?" |

#### Meta Knowledge Source (for Track A)

The agent needs *some* meta knowledge to teach meta concepts. Full knowledge graph is Track B. For Track A, use **NRDB community reviews as lightweight meta knowledge:**

- Pull `/api/2.0/public/reviews` — single API call, returns all reviews with comments
- Store as `reviews.json` (~500-1000 reviews with multi-paragraph meta analysis)
- Reviews contain archetype discussion, synergy explanations, matchup advice, meta positioning
- Keyword search over reviews for relevant context when the agent needs to explain a concept

This is "poor man's RAG" — no vector DB, no graph, just community-written meta knowledge searchable by card/topic. Enough to ground sessions 1-3 in real meta reasoning.

#### SM-2 Spaced Repetition (the scheduling engine)

The algorithm is repurposed from card-fact scheduling to **meta-concept scheduling**:
```
State per meta concept: n (rep count), EF (easiness factor, init 2.5), I (interval in days)

if grade >= 3 (correct):
    if n == 0: I = 1
    elif n == 1: I = 6
    else: I = round(I * EF)
    n += 1
else (incorrect):
    n = 0, I = 1

EF = EF + (0.1 - (5 - grade) * (0.08 + (5 - grade) * 0.02))
EF = max(EF, 1.3)
```

**Minimum viable: ~30 lines of Python.** Hardcode parameters, use Anki's 4-button mapping (Wrong→0, Hard→3, Good→4, Easy→5).

**Meta concept adaptations:**
- Track per `(concept_id, quiz_type)` — a player might understand glacier ice selection but not glacier matchup counterplay
- Concepts map to groups of related cards, not individual cards
- SM-2 schedules *which concepts to explore*, the agent then generates contextual questions from cards within that concept

#### Agent Memory Pattern

**Use concept-level user model.** Store meta concept understanding + session summaries in a JSON file:

```json
{
  "concepts": {
    "glacier-matchups": {"times_tested": 8, "times_correct": 3, "ef": 1.8, "interval": 2, "last_seen": "2026-03-10"},
    "kill-deck-counterplay": {"times_tested": 5, "times_correct": 4, "ef": 2.6, "interval": 10, "last_seen": "2026-03-08"},
    "econ-denial-strategy": {"times_tested": 0, "ef": 2.5, "interval": 0, "last_seen": null}
  },
  "sessions": [
    {"date": "2026-03-10", "concepts_explored": 4, "accuracy": 0.55, "weak_areas": ["glacier-matchups", "asset-spam-counterplay"]}
  ],
  "meta_concepts_catalog": "meta_concepts.json",
  "version": 2
}
```

Alongside, a **meta concepts catalog** (`meta_concepts.json`) defines the concept space:

```json
{
  "glacier-matchups": {
    "name": "Glacier Matchups",
    "side": "corp",
    "description": "Understanding glacier strategy: tall servers, expensive ice, scoring behind remotes",
    "related_cards": ["30039", "30050", "35052", "35053"],
    "related_archetypes": ["glacier-pd", "glacier-rh"],
    "sample_questions": [
      "Why does glacier prefer expensive ice over cheap ice?",
      "What runner strategy best counters a glacier corp?"
    ]
  }
}
```

The catalog can start curated (20-30 concepts) and grow via NRDB review mining later.

#### Session Planning (pure function)

```python
def plan_session(concept_memory, reviews_index, target=4) -> list[Topic]:
    # Priority 1: Overdue meta concepts (next_review <= today)
    # Priority 2: Weak concepts (accuracy < 60%, tested >= 3x)
    # Priority 3: Untested concepts
    # Priority 4: Revisit strong concepts with new angle
```

**Data needed:** concept memory (JSON file) + meta concepts catalog + NRDB reviews for context. All local files.

#### Agent Initiative (no SDK changes needed)

The MS Agents SDK ActivityHandler is request-response, but initiative doesn't require proactive messaging. **Piggyback on every response:**

1. **Post-answer commentary:** "Knowing Punitive costs 6 matters because kill decks need you to steal first — experienced players check corp credits before running." (Session 1)
2. **Session opening plan:** "Last time you struggled with glacier counterplay. Today: 2 overdue concepts on ice economics, 1 new concept on econ denial." (Session 2)
3. **Mid-session course correction:** "You keep missing questions about ice costs in context — let's shift to *why* ice cost matters for different archetypes." (Session 3)

All implemented within existing `on_message_activity` and `on_members_added_activity` methods. No new SDK features, no proactive messaging infrastructure.

#### Key Insight from Research

Nobody has built an adaptive quiz agent for card games. SRS tools (Anki, Mnemosyne) are pure tools. AI agents are chatbots. The intersection — an agent that autonomously manages domain-specific learning — barely exists. This is genuinely novel territory.

### Anti-Trap Constraint (from ideation phase)

**"No new infrastructure in sessions 1-3."**
- No schema changes. No new SQLite tables. No SDK refactoring.
- One API call to pull NRDB reviews (one-time). Two JSON files (concept memory + meta concepts catalog). Three pure functions.
- Piggybacked onto existing response flow.
- Graph DB and deeper infra in Track B / sessions 4+ AFTER agent behavior works.

### Gate Check

If by end of session 3 the agent has not made an unprompted meta-grounded observation, **STOP and switch to Idea #8 (Standup Summarizer, git-only).**

### Session 1-3 Execution Plan

| Session | Build | Agent Behavior Gained |
|---------|-------|----------------------|
| **1** | `concept_memory.json` + `meta_concepts.json` (curated catalog of ~25 concepts) + SM-2 scheduling over concepts + wire into quiz handler + pull NRDB reviews once | Agent remembers which meta concepts you understand well vs poorly, across sessions |
| **2** | `plan_session()` + session opening message grounded in concept gaps | Agent greets you with "Last time you struggled with kill deck counterplay — let's work on that today" |
| **3** | Post-answer meta commentary + mid-session course correction using reviews as context | Agent connects quiz answers to meta reasoning: "That ice cost matters because glacier needs to tax 4+ credits per run to stay ahead" |

### Next Steps

- Tuesday: Identify specific barriers to each session's deliverable
- Wednesday: Decompose into components with SMART success metrics

---

## Tuesday: Barriers

Reviewed the v1 codebase (`quiz_agent.py`, `card_data.py`, test fixtures) and identified barriers across the 3 session deliverables. Originally framed around card-fact recall; revised 2026-03-28 to reflect meta-concept learning.

### Session 1: Memory (SM-2 over meta concepts + `concept_memory.json`)

| # | Barrier | Why It Matters | Mitigation |
|---|---------|---------------|------------|
| 1 | No persistence hook — CLI loop has no save/load lifecycle | Ctrl+C exits lose all concept memory data | Write after every answer (cheap for JSON); `atexit` handler as safety net |
| 2 | `session_cards_seen` initialized but never used | Can't prevent within-session concept repeats | Replace with SM-2's "already reviewed today" check at the concept level |
| 3 | **Card selection is random** (60% meta-biased `random.choice`) | SM-2 needs priority-based selection: overdue concept → weak concept → unseen concept → reinforcement. Rewires the core quiz loop. | New `_select_next_concept()` method that consults concept memory, then generates a card-level question within that concept |
| 3a | **Meta concepts catalog doesn't exist yet** | Agent needs a defined concept space to schedule over | Curate `meta_concepts.json` with ~25 concepts: archetypes (glacier, rush, kill, asset spam, aggro, econ denial), matchup pairs, strategic principles. Map each concept to related cards from existing card DB |
| 3b | **NRDB reviews not yet ingested** | Agent needs community meta knowledge to generate contextual explanations | One-time pull of `/api/2.0/public/reviews` → `reviews.json`. Index by card code for keyword lookup |

**Barrier #3 remains the critical path** — but now it's "select next concept" not "select next card." The concept layer sits above the card layer.

### Session 2: Planning (`plan_session()` + opening message)

| # | Barrier | Why It Matters | Mitigation |
|---|---------|---------------|------------|
| 4 | No session boundary concept — `while True` with no start/end | Can't inject a session-opening plan before the first question | Add `_start_session()` called once before the loop; prints plan |
| 5 | Cold start — first session has no history to plan from | `plan_session()` returns empty when concept_memory.json doesn't exist | Default to "meta fundamentals intro" plan — start with highest-priority archetypes (glacier, rush, kill); explain to user what future plans will look like |

### Session 3: Initiative (meta commentary + course correction)

| # | Barrier | Why It Matters | Mitigation |
|---|---------|---------------|------------|
| 6 | `handle_message()` returns a single string | Initiative means meta commentary appended to quiz results | Response builder: accumulate parts, join at return |
| 7 | No rolling performance tracking | Course correction needs per-concept accuracy and trend, not just cumulative totals | Add `_session_tracker` with recent-N window (last 5-10 answers), tracked at the concept level |
| 7a | **Meta commentary requires context** | "That matters because..." needs the agent to explain *why* in meta terms | Search `reviews.json` for relevant card/concept context; inject into response builder |

### Cross-cutting

| # | Barrier | Why It Matters | Mitigation |
|---|---------|---------------|------------|
| 8 | Test fixtures bypass `__init__` via `patch.object` | New init behavior (load memory, set up SM-2) silently skipped in tests | Update fixtures to either call real init with test memory file, or explicitly set up SM-2 state |

### Critical Path

```
Barrier #3a (meta concepts catalog) → Barrier #3 (concept selection) → Barrier #1 (persistence) → Barrier #4 (session boundary) → Barrier #6 (response builder) → Barrier #7a (meta context from reviews)
```

The concept catalog (#3a) is now the very first thing to build — it defines the concept space the entire system operates on. Everything downstream depends on having well-defined meta concepts.

---

## Wednesday: Decomposition & Metrics

Audited the meta-runner-2 codebase (1,567 LOC, 93 tests) against the Track A session plan. Significant progress: the repo setup session (March 28) pre-built much of the scaffolding. The decomposition below identifies what's **done**, what's **partially done**, and what's **missing** — with SMART metrics for each component.

### What Already Exists (Baseline)

| Component | Status | LOC | Tests |
|-----------|--------|-----|-------|
| SM-2 algorithm (`sm2_update`) | ✅ Complete | ~30 | 9 tests |
| Concept memory persistence | ✅ Complete | ~50 | 2 tests |
| Concept selection priority (`select_next_concepts`) | ✅ Complete | ~40 | 4 tests |
| Meta concepts catalog (17 concepts, 4 categories) | ⚠️ Partial — `key_cards` empty | ~200 | 3 tests |
| Session planning (`start_session`) | ⚠️ Partial — generic, no history reference | ~30 | 4 tests |
| Meta commentary (`_get_meta_commentary`) | ⚠️ Partial — low quality, no reviews data | ~15 | 0 tests |
| Course correction (`_check_course_correction`) | ⚠️ Partial — only 2 triggers | ~10 | 4 tests |
| Review index (search/load) | ✅ Complete | ~80 | 8 tests |
| NRDB reviews fetching | ❌ Missing — no `fetch_reviews()` | 0 | 0 tests |
| Web UI (Gradio) | ✅ Complete (PR #1) | ~470 | 10 tests |

**Assessment:** The infrastructure is built. What's missing is the **quality layer** — the parts that make this an *agent* rather than a quiz tool with an SM-2 scheduler. The gaps are all in the "intelligence" functions: commentary quality, plan personalization, course correction depth, and actual meta data to operate on.

---

### Component Decomposition

#### Session 1: Memory

**C1.1 — NRDB Reviews Ingestion**
- **Status:** Missing. `review_index.py` has `load_reviews()` and `search_reviews()` but no API fetch function. Without reviews, the agent has zero meta knowledge to draw from.
- **Work:** Add `fetch_reviews()` to `card_data.py` or `review_index.py`. Pull from NRDB API `/api/2.0/public/reviews`. Cache as `data/reviews.json`. Add to `.gitignore`.
- **Barrier addressed:** #3b
- **SMART metric:** `fetch_reviews()` pulls ≥100 reviews from NRDB API; `search_reviews()` returns relevant results for 5 test queries (`"glacier"`, `"kill deck"`, `"ice cost"`, `"asset spam"`, `"econ denial"`); reviews.json is created and gitignored.
- **Effort:** Small — single API call, JSON dump, one function.

**C1.2 — Key Cards Mapping**
- **Status:** All 17 concepts have `"key_cards": []`. The agent can't connect concepts to actual cards.
- **Work:** Populate `key_cards` for each concept. Two approaches: (a) manual curation from game knowledge, or (b) heuristic from card DB (match faction + keywords + type patterns). Manual is more accurate; heuristic scales but needs validation.
- **Barrier addressed:** #3a (partial)
- **SMART metric:** ≥15 of 17 concepts have ≥3 key cards; mapped card codes exist in the card DB; at least 2 concepts per category have cards.
- **Effort:** Medium — requires Netrunner domain knowledge or card DB analysis.

**C1.3 — Memory Persistence Lifecycle**
- **Status:** `save_concept_memory()` is called after every grade submission and at session end. No `atexit` handler. Ctrl+C during a session loses the current session's data.
- **Work:** Add `atexit.register(save_concept_memory)` in `__init__`. Verify data survives abnormal exits.
- **Barrier addressed:** #1
- **SMART metric:** `concept_memory.json` persists after: (a) normal quit, (b) Ctrl+C mid-question, (c) 3 consecutive sessions. Data is recoverable in all 3 scenarios.
- **Effort:** Small — 3 lines of code + test.

**C1.4 — Concept Memory Cross-Session Validation**
- **Status:** SM-2 unit tests exist but no integration test verifying multi-session learning.
- **Work:** Add a test that simulates: session 1 (answer 4 concepts) → save → session 2 (verify overdue/weak concepts are prioritized) → save → session 3 (verify learning curve changes concept selection).
- **Barrier addressed:** #8 (test coverage for new behavior)
- **SMART metric:** Integration test passes: session 2 plan differs from session 1 based on grades given; session 3 correctly identifies overdue concepts from session 1.
- **Effort:** Medium — requires test fixture design.

#### Session 2: Planning

**C2.1 — Cold Start Plan**
- **Status:** First session shows "🆕 new concept" for all concepts, picks first 4 in untested order (essentially catalog order). No explanation of what the agent does or why.
- **Work:** Add a first-session path in `start_session()` that explains the learning system and picks foundational concepts (archetypes first: glacier, rush, kill, fast-advance — not matchups or meta-knowledge which depend on understanding archetypes).
- **Barrier addressed:** #5
- **SMART metric:** First-ever session opening: (a) explains the learning model in ≤3 sentences, (b) selects ≥3 archetype concepts (not matchups or meta-knowledge), (c) differs visually from returning-user plan.
- **Effort:** Small — conditional logic + copy.

**C2.2 — Historical Session Summary in Plan**
- **Status:** `start_session()` shows concept priorities but no reference to past sessions. The `memory["sessions"]` array tracks history but isn't surfaced.
- **Work:** Add "Last session (March 28): explored 4 concepts at 62% accuracy. Weak areas: glacier-matchups, kill-deck-counterplay" to the opening message when session history exists.
- **Barrier addressed:** #4 (partial — session boundary exists, but plan doesn't use history)
- **SMART metric:** When ≥1 past session exists, opening message includes: (a) date of last session, (b) number of concepts explored, (c) accuracy %, (d) specific weak areas if any. When 0 past sessions, falls back to cold start (C2.1).
- **Effort:** Small — read `memory["sessions"][-1]`, format string.

**C2.3 — Plan Adaptation Over Time**
- **Status:** `select_next_concepts()` priority ordering works (overdue → untested → weak → reinforcement) but hasn't been validated with realistic multi-session data.
- **Work:** Simulate 5-session learning arcs with different grade patterns. Verify plans evolve: session 1 = all new → session 2 = mix of overdue + new → session 3 = weak areas dominate → session 4 = reinforcement of strong + overdue from session 1.
- **Barrier addressed:** Overall quality assurance
- **SMART metric:** 5-session simulation test: each session's plan differs from previous; by session 5, ≥2 concepts that were graded "easy" in session 1 reappear for reinforcement review.
- **Effort:** Medium — requires test design with controlled dates.

#### Session 3: Initiative

**C3.1 — Meta Commentary Quality**
- **Status:** `_get_meta_commentary()` searches reviews by concept name, takes first 200 chars of first match. Problems: (a) concept names like "Glacier" may match irrelevant reviews, (b) first 200 chars often include HTML artifacts or preamble, (c) no fallback if no reviews exist yet.
- **Work:** Improve search to use concept description keywords + key card names. Extract meaningful sentences (not arbitrary 200 chars). Add fallback to concept description when no reviews match.
- **Barrier addressed:** #7a
- **SMART metric:** Manual evaluation on 10 concepts: ≥6/10 produce commentary that a Netrunner player would find relevant (not generic or garbled). Fallback produces something useful for all 10.
- **Effort:** Medium — search refinement + text extraction logic.

**C3.2 — Course Correction Depth**
- **Status:** Two triggers: 3 wrong in a row → "shift to something easier"; 3 easy in a row → "push harder". No concept-specific insight.
- **Work:** Add: (a) same-concept repeated wrong → "Let's approach [concept] from a different angle"; (b) cross-concept pattern → "You're strong on archetypes but struggling with matchups — that's normal, matchups build on archetype knowledge"; (c) session fatigue → after 10+ questions, "Good session — you've covered a lot. Want to wrap up?"
- **Barrier addressed:** #7
- **SMART metric:** Course correction triggers in ≥4 distinct scenarios (current 2 + at least 2 new). Each trigger produces a message that references specific concepts or categories (not generic).
- **Effort:** Medium — pattern detection + message templates.

**C3.3 — Post-Answer Explanation Quality**
- **Status:** Wrong answers show concept description. Correct answers show "Strong understanding!" No meta reasoning ("why this matters").
- **Work:** On wrong: show description + one relevant review excerpt. On correct: connect to broader strategy ("This is key for glacier matchups because..."). Use reviews and concept relationships for grounding.
- **Barrier addressed:** #6 (response builder)
- **SMART metric:** Post-answer responses for wrong grades include ≥1 sentence beyond the concept description for 5/5 tested concepts. Post-correct responses for grade=4 include a connecting insight (not just "✅ Strong understanding!").
- **Effort:** Medium — requires review integration + concept relationship logic.

#### Cross-Cutting

**CC.1 — Behavioral Integration Test**
- **Status:** 93 tests exist but all are unit-level. No test validates the agent behavior sequence (start → quiz → grade → commentary → next concept → course correction → quit → restart).
- **Work:** Add an integration test that scripts a full session lifecycle. Verify all three behaviors (memory, planning, initiative) fire in sequence.
- **Barrier addressed:** #8
- **SMART metric:** Integration test covers: start_session with history → quiz → grade → verify commentary appears → grade 3 more → verify course correction fires → quit → restart → verify plan references last session.
- **Effort:** Medium.

**CC.2 — Gate Check Test**
- **Status:** Gate check defined ("≥3 unprompted meta-grounded observations per session") but no automated way to validate it.
- **Work:** Define a scripted 3-session play-through scenario. Count unprompted agent actions (commentary, course correction, plan adaptation). Log counts. The gate check is manual evaluation but the counting should be automated.
- **SMART metric:** Scripted 3-session play-through produces ≥3 unprompted agent messages per session that reference specific meta concepts. If <3, Track A fails the gate check.
- **Effort:** Large — requires scripted scenario design + evaluation criteria.

---

### Dependency Graph

```
C1.1 (fetch reviews) ──┬──→ C3.1 (commentary quality)
                        └──→ C3.3 (post-answer explanations)
C1.2 (key cards) ───────┬──→ C3.1 (commentary quality — search by card name)
                        └──→ C3.3 (connect answers to specific cards)
C1.3 (persistence) ────────→ C1.4 (cross-session validation)
C2.1 (cold start) ─────────→ C2.2 (history summary — needs cold start as fallback)
C1.4 (cross-session) ──────→ C2.3 (plan adaptation — needs multi-session data)
C3.1 + C3.2 + C3.3 ────────→ CC.1 (behavioral integration test)
CC.1 ───────────────────────→ CC.2 (gate check)
```

### Parallel Tracks

These can be worked in parallel:

| Track | Components | Dependency |
|-------|-----------|------------|
| **Data** | C1.1 (reviews fetch) + C1.2 (key cards) | None — independent of each other |
| **Memory lifecycle** | C1.3 (atexit) + C1.4 (cross-session test) | C1.3 before C1.4 |
| **Planning** | C2.1 (cold start) + C2.2 (history summary) | C2.1 before C2.2 |
| **Initiative** | C3.1 + C3.2 + C3.3 | Depends on Data track completing first |
| **Validation** | CC.1 + CC.2 | Depends on everything above |

### Recommended Build Order

| Phase | Components | What It Proves |
|-------|-----------|---------------|
| **Phase 1** | C1.1, C1.2, C1.3, C2.1 | Agent has data to operate on + foundational behaviors |
| **Phase 2** | C2.2, C2.3, C1.4 | Planning is personalized and adapts across sessions |
| **Phase 3** | C3.1, C3.2, C3.3 | Agent produces unprompted, relevant meta insight |
| **Phase 4** | CC.1, CC.2 | Gate check: does this feel like an agent? |

### Evaluation Strategy

**Automated (unit + integration tests):**
- SM-2 correctness: ✅ already covered (9 tests)
- Concept selection: ✅ already covered (4 tests)
- Memory persistence: needs C1.3 + C1.4 tests
- Plan adaptation: needs C2.3 simulation tests
- Course correction triggers: needs C3.2 tests
- Full behavioral sequence: needs CC.1

**Manual (play-through evaluation):**
- Commentary relevance (C3.1): 10-concept manual review
- Plan quality (C2.1, C2.2): first-session vs returning-session comparison
- Gate check (CC.2): 3-session scripted play-through, count unprompted actions

**Quantitative targets:**
- ≥3 unprompted meta-grounded observations per session (gate check)
- ≥60% commentary relevance on manual evaluation
- 5-session plan evolution is monotonically improving (weak areas diminish)
- Test coverage: current 93 → target ≥110 after all components

---

## Thursday: Agents & Tools

*(To be completed)*

---

## Friday: Execution & Iteration

*(To be completed)*

---

## Journal

### Week 1

**2026-03-10 (Tuesday):** Completed Monday framing phase. Researched SM-2, agent memory patterns, session planning, and initiative in request-response frameworks. Key finding: all three missing agent behaviors (memory, planning, initiative) can be added with zero new infrastructure — one JSON file, three pure functions, piggybacked response content. Created execution plan for sessions 1-3.

### Week 2

**2026-03-21 (Saturday):** Completed Tuesday barriers phase. Reviewed v1 codebase in detail and identified 8 barriers across 3 session deliverables. Critical path is Barrier #3: card selection rewiring from random to SM-2-driven priority picking. Also evaluated MCP server for Cycle 2 — deferred (no behavioral test, infrastructure trap). See barriers tables above for mitigations.

### Week 3

**2026-03-28 (Saturday):** Two significant developments:

1. **Knowledge graph feasibility study** — Researched NetrunnerDB API (v2 + v3 preview) and ABR API. Designed a full graph schema (14 node types, 15+ edge types) for representing cards, decks, reviews, tournaments, archetypes, synergies, and temporal meta shifts. Recommended Kuzu (embedded) → Neo4j (production RAG). Assessed as clearly feasible but categorized as Track B / Cycle 3 infrastructure. See `journal/2026-03-28.md`.

2. **Revised Track A: unit of learning shift** — Identified that SM-2 over card facts (trivia recall) ≠ meta understanding. "Meta" in card games means the metagame — archetypes, matchups, counter-strategies, meta shifts. Revised Track A to operate on **meta concepts** instead of card facts. SM-2 stays as the scheduling engine but schedules concepts like "glacier-matchups" and "kill-deck-counterplay" instead of "01042_cost_quiz." Added NRDB reviews as lightweight meta knowledge source (single API call). Updated barriers: new #3a (meta concepts catalog) and #7a (meta context from reviews). Critical path now starts with defining the concept space. Updated session plan, framing, and gate check accordingly.

### Week 4

**2026-03-30 (Sunday):** Completed Wednesday decomposition phase. Audited meta-runner-2 codebase (1,567 LOC, 93 tests) against barriers. Key finding: infrastructure is built but the quality layer is missing — commentary grabs arbitrary text, planning ignores session history, key_cards is empty, reviews data doesn't exist yet. Decomposed Track A into 13 components across 4 phases with SMART metrics. Identified that Phase 1 (data + foundational behaviors) is entirely independent work that can be parallelized. See decomposition tables above for full detail.
