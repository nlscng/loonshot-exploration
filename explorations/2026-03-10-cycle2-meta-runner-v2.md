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
> An agent that passes the behavioral test: "I notice you're weak on ICE subtypes — let's focus there today" *without the user requesting a topic.* Concretely: ≥3 unprompted agent actions per session, adaptive card selection based on history, and a session-opening plan.

**Why does it matter?**
> Cycle 1's key learning was "agent ≠ tool." This project isolates the exact skills that failed — memory, planning, initiative — on top of infrastructure that's already done. Day 1 work IS agent work.

### Research Summary

#### SM-2 Spaced Repetition (the memory engine)

The algorithm per card:
```
State: n (rep count), EF (easiness factor, init 2.5), I (interval in days)

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

**Card game adaptations:**
- Track per `(card_code, quiz_type)` — a player might know Engolo's cost but not its subtypes
- Meta weighting: `effective_interval = interval * (0.7 if in_meta else 1.0)` — meta cards reviewed more often
- No new DB tables needed — JSON file suffices for sessions 1-3

#### Agent Memory Pattern

**Use entity memory pattern.** Store per-card state + session summaries in a JSON file:

```json
{
  "cards": {
    "01042_identify_ice": {"times_seen": 12, "times_correct": 8, "ef": 2.36, "interval": 6, "last_seen": "2026-03-10"},
    "01042_cost_quiz": {"times_seen": 5, "times_correct": 5, "ef": 2.7, "interval": 15, "last_seen": "2026-03-08"}
  },
  "sessions": [
    {"date": "2026-03-10", "cards_reviewed": 15, "accuracy": 0.73, "weak_areas": ["ice_subtypes"]}
  ],
  "version": 1
}
```

**Zero infrastructure cost.** `json.load()` on startup, `json.dump()` on exit. Migrate to SQLite in sessions 4+.

#### Session Planning (pure function)

```python
def plan_session(card_memory, meta_cards, target=10) -> list[Topic]:
    # Priority 1: Overdue reviews (next_review <= today)
    # Priority 2: Weak areas (accuracy < 60%, seen >= 3x)
    # Priority 3: Unseen meta cards
    # Priority 4: Random reinforcement
```

**Data needed:** card memory (JSON file) + existing meta_stats table + existing card DB. All already available.

#### Agent Initiative (no SDK changes needed)

The MS Agents SDK ActivityHandler is request-response, but initiative doesn't require proactive messaging. **Piggyback on every response:**

1. **Post-answer commentary:** "That's 3 in a row on ICE — leveling up." (Session 1)
2. **Session opening plan:** "Today: 5 overdue, 3 weak ICE, 2 new meta cards." (Session 2)
3. **Mid-session course correction:** "3 wrong in a row — switching to something you're stronger on." (Session 3)

All implemented within existing `on_message_activity` and `on_members_added_activity` methods. No new SDK features, no proactive messaging infrastructure.

#### Key Insight from Research

Nobody has built an adaptive quiz agent for card games. SRS tools (Anki, Mnemosyne) are pure tools. AI agents are chatbots. The intersection — an agent that autonomously manages domain-specific learning — barely exists. This is genuinely novel territory.

### Anti-Trap Constraint (from ideation phase)

**"No new infrastructure in sessions 1-3."**
- No schema changes. No new SQLite tables. No SDK refactoring. No new API calls.
- One JSON file, three pure functions, piggybacked onto existing response flow.
- Refactor into proper infrastructure in sessions 4+ AFTER the agent behavior works.

### Gate Check

If by end of session 3 the agent has not said something unprompted, **STOP and switch to Idea #8 (Standup Summarizer, git-only).**

### Session 1-3 Execution Plan

| Session | Build | Agent Behavior Gained |
|---------|-------|----------------------|
| **1** | `memory.json` + SM-2 function + wire into quiz handler | Agent remembers what you got right/wrong across sessions |
| **2** | `plan_session()` + session opening message | Agent greets you with a personalized plan |
| **3** | Post-answer initiative + mid-session course correction | Agent comments on performance and redirects |

### Next Steps

- Tuesday: Identify specific barriers to each session's deliverable
- Wednesday: Decompose into components with SMART success metrics

---

## Tuesday: Barriers

Reviewed the v1 codebase (`quiz_agent.py`, `card_data.py`, test fixtures) and identified 8 concrete barriers across the 3 session deliverables.

### Session 1: Memory (SM-2 + `memory.json`)

| # | Barrier | Why It Matters | Mitigation |
|---|---------|---------------|------------|
| 1 | No persistence hook — CLI loop has no save/load lifecycle | Ctrl+C exits lose all memory data | Write after every answer (cheap for JSON); `atexit` handler as safety net |
| 2 | `session_cards_seen` initialized but never used | Can't prevent within-session repeats; SM-2 tracks `(card_code, quiz_type)` pairs, not just card codes | Replace with SM-2's "already reviewed today" check |
| 3 | **Card selection is random** (60% meta-biased `random.choice`) | SM-2 needs priority-based selection: overdue → weak → unseen → reinforcement. Rewires the core quiz loop. | New `_select_next_card()` method that consults memory before `_generate_question()` |

**Barrier #3 is the critical path.** If card selection stays random, memory is write-only — tracked but never acted on.

### Session 2: Planning (`plan_session()` + opening message)

| # | Barrier | Why It Matters | Mitigation |
|---|---------|---------------|------------|
| 4 | No session boundary concept — `while True` with no start/end | Can't inject a session-opening plan before the first question | Add `_start_session()` called once before the loop; prints plan |
| 5 | Cold start — first session has no history to plan from | `plan_session()` returns empty when memory.json doesn't exist | Default to "meta staples intro" plan; explain to user what future plans will look like |

### Session 3: Initiative (commentary + course correction)

| # | Barrier | Why It Matters | Mitigation |
|---|---------|---------------|------------|
| 6 | `handle_message()` returns a single string | Initiative means extra commentary appended to quiz results | Response builder: accumulate parts, join at return |
| 7 | No rolling performance tracking | Course correction needs streaks and per-topic accuracy, not just cumulative totals | Add `_session_tracker` with recent-N window (last 5-10 answers) |

### Cross-cutting

| # | Barrier | Why It Matters | Mitigation |
|---|---------|---------------|------------|
| 8 | Test fixtures bypass `__init__` via `patch.object` | New init behavior (load memory, set up SM-2) silently skipped in tests | Update fixtures to either call real init with test memory file, or explicitly set up SM-2 state |

### Critical Path

```
Barrier #3 (card selection) → Barrier #1 (persistence) → Barrier #4 (session boundary) → Barrier #6 (response builder)
```

Everything else is additive. Solve #3 and sessions 2-3 are extensions of the same priority queue.

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
