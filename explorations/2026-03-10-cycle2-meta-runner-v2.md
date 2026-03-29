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

*(To be completed)*

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
