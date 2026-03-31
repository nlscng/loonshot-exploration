# Design: Meta-Runner v2 — Conversational Coaching Agent

**Date:** 2026-03-30
**Status:** Reviewed (4-round critique applied)
**Context:** Pivoting from quiz-only agent to conversational coaching agent

---

## The Shift

Meta-runner v2 currently operates as a **quiz agent** — it picks a concept, asks a question, the user self-evaluates, and SM-2 schedules the next review. This is fine for memorisation but wrong for the actual goal: **learning the Netrunner metagame**.

Learning meta isn't about answering quiz questions. It's about understanding:
- Why glacier is strong in the current format
- What cards make a PD deck tick and what counters it
- How to read a decklist and understand its game plan
- Why a card that was unplayed six months ago is now everywhere

A **conversational coach** fits this better than a quiz engine. The agent should feel like sitting down with a seasoned player who walks you through the meta, answers your questions with real data, and challenges your understanding when appropriate.

---

## Interaction Model

### Current (Quiz)
```
Agent: [archetype] Glacier — Why does glacier prefer expensive ice?
       (Answer in your own words, then self-evaluate: 0-5)
User:  Because it taxes the runner more per run
Agent: 📝 Self-evaluate: 0=wrong, 3=hard, 4=good, 5=easy
User:  4
Agent: ✅ Strong understanding! Score: 1/1 (100%)
```

### Proposed (Conversational Coaching)
```
Agent: Welcome back! Last session you explored glacier and rush.
       Your understanding of glacier is solid, but you haven't
       looked at how glacier performs in the current meta yet.

       Want to explore that? Or is there something else on your mind?

User:  Yeah, what's the glacier matchup landscape look like?

Agent: In the current Standard format, glacier is primarily a PD
       (Haas-Bioroid) archetype. The key cards are:
       - Border Control (hard stop, forces the runner to spend again)
       - Ansel 1.0 (strong while face-checking, taxing once broken)
       - Manegarm Skunkworks (makes runs extremely expensive)

       The main runner counterplay is econ denial — cards like
       Diversion of Funds that attack the corp's credit pool before
       they can rez their ice suite.

       Based on recent tournament results from ABR, PD glacier has
       a ~55% win rate, but drops to ~40% against Hoshiko econ
       denial builds.

       Want me to walk through a specific PD decklist? Or explore
       what makes the Hoshiko matchup bad for glacier?

User:  Show me a PD decklist

Agent: Here's "Glacial Precision" by NetDecker (1st at Melbourne SC):
       [card list with key cards highlighted]

       The game plan is...
       Notice how the ice suite is all 4+ rez cost — that's the
       glacier tax. But look at the agenda composition...

       What do you think the weakness of this deck is?
```

### Key Differences

| Aspect | Quiz Mode | Coaching Mode |
|--------|-----------|---------------|
| **Who drives** | Agent picks topic | Both — agent suggests, user can redirect |
| **Response format** | Closed (grade 0-5) | Open conversation |
| **Knowledge depth** | Sample questions from catalog | Deep, data-grounded explanations |
| **Data usage** | Concepts only | Cards, decklists, reviews, tournament data |
| **Learning signal** | Self-evaluation grades | Conversation flow, questions asked, topics explored |
| **Agent personality** | Flashcard machine | Knowledgeable mentor |

---

## What Changes, What Stays

### Stays (reusable infrastructure)

| Component | Why It Survives |
|-----------|----------------|
| **Meta concepts catalog** (17 concepts) | Still the backbone of what the agent knows about |
| **SM-2 scheduling** | Repurposed: instead of scheduling quiz questions, schedules which concepts to proactively surface ("You haven't explored kill decks in 2 weeks — want to revisit?") |
| **Concept memory** | Still tracks what the user has explored and how well |
| **Session planning** | Still opens with a personalised plan |
| **NRDB reviews** | Directly quoted as meta knowledge during coaching |
| **Card data layer** | Grounds conversations in real card data |
| **Cold start / returning user** | Same session lifecycle |
| **atexit persistence** | Same |

### Changes

| Component | Current | Proposed |
|-----------|---------|----------|
| **handle_message()** | Command parser (`quiz`, `help`, `score`, grade) | Intent classifier + response generator |
| **Question generation** | Pick concept → random sample question | Context-aware topic exploration |
| **Answer evaluation** | User self-grades 0-5 | Agent tracks engagement implicitly (topics explored, depth reached, questions asked) |
| **Response format** | Single-line quiz result | Multi-paragraph explanations with card references |
| **Course correction** | 3-wrong/3-easy triggers | Conversation-aware: "You keep asking about corp — want to explore the runner side?" |
| **Score** | correct/wrong ratio | Concepts explored, depth reached, sessions completed |

### New Capabilities Needed

1. **Intent classification** — Understand what the user is asking:
   - Topic exploration ("tell me about glacier")
   - Card lookup ("what does Border Control do?")
   - Decklist analysis ("show me a PD deck")
   - Matchup question ("what beats glacier?")
   - Meta question ("what's strong right now?")
   - Redirect ("let's talk about something else")
   - Quiz request ("quiz me on this") — keep as an option

2. **Grounded response generation** — Pull relevant data:
   - Card DB for card details and stats
   - Decklists for real examples
   - Reviews for community perspective
   - Concept catalog for structured knowledge
   - Memory for personalisation
   - **Tournament results from ABR** for performance context

3. **Conversation state** — Track:
   - Current topic (concept being explored)
   - Depth level (surface → intermediate → deep)
   - Related concepts mentioned
   - Cards and decklists referenced
   - User questions (what they're curious about)

4. **Proactive suggestions** — End responses with:
   - Related topics to explore
   - Questions to check understanding (optional quiz moments)
   - "Want to see a decklist?" / "Want to explore the matchup?"

---

## Data Sources

### Current: NetrunnerDB v2 API

Already integrated (card_data.py + review_index.py). Provides cards, decklists, community reviews, MWL/ban data.

### New: AlwaysBeRunning.net (ABR) API

Public JSON API, no auth. Verified live and returning data.

**Why ABR matters:** NRDB tells you what cards exist and what decks people build. ABR tells you **what wins** — which decks placed at tournaments, how many players showed up, what the winner played. This is the difference between "this deck is popular" and "this deck is competitive."

| Endpoint | Data | Meta Value |
|----------|------|------------|
| `GET /api/tournaments?concluded=1` | All concluded events: format, cardpool, MWL, player count, winner identity codes | Tournament landscape over time |
| `GET /api/entries?id={id}` | Per-tournament standings: rank, player, runner/corp identity+faction, **NRDB decklist URLs** | Performance data per deck, links to full deck lists |

**ABR response shape (verified):**

Tournament:
```json
{"id": 5431, "title": "Standard AMT - March 29th (EMEA)", "format": "standard",
 "cardpool": "Vantage Point", "mwl": "Standard Ban List 26.03",
 "players_count": 21, "winner_runner_identity": "34066",
 "winner_corp_identity": "35069", "matchdata": true}
```

Entry (standing):
```json
{"user_name": "34Witches", "rank_swiss": 1, "rank_top": 1,
 "runner_deck_title": "Sebaasy (4-0, 1st at March EMEA AMT)",
 "runner_deck_identity_id": "34066", "runner_deck_identity_title": "Sebastião Souza Pessoa",
 "runner_deck_url": "https://netrunnerdb.com/en/decklist/95178",
 "corp_deck_identity_id": "35069", "corp_deck_identity_title": "The Zwicky Group",
 "corp_deck_url": "https://netrunnerdb.com/en/decklist/95177"}
```

**Key bridge:** ABR entries include `runner_deck_url` / `corp_deck_url` that link to NRDB decklists. This connects tournament performance to full deck composition — the agent can say "this deck won 3 tournaments" and then walk through the actual card list.

### Future: YouTube / Podcast Meta Commentary

Channels: Metropole Grid, Neon Static, Aksu, Dullbulb. These provide qualitative meta analysis — format predictions, card evaluations, matchup discussions. Integration deferred — requires solving the video/audio → text extraction problem. Will revisit when thinking about how the agent keeps up with evolving meta over time.

---

## Architecture Options

### Option A: Rule-Based Intent + Template Responses

Classify intent via keyword matching, generate responses from templates filled with real data.

**Pros:** No LLM dependency, deterministic, fast, testable
**Cons:** Rigid, can't handle nuanced questions, responses feel formulaic

### Option B: LLM-Powered Conversation with Data Grounding

Use an LLM (local or API) for intent understanding and response generation. Feed it relevant card data, reviews, and concept knowledge as context.

**Pros:** Natural conversation, handles any question, deep explanations
**Cons:** LLM dependency, cost, latency, harder to test, hallucination risk

### Option C: Hybrid — Rule-Based Routing + LLM Responses

Classify intent with rules (or a small classifier), retrieve relevant data, then use an LLM to compose the response from that data.

**Pros:** Best of both — fast routing, natural responses, grounded in data
**Cons:** Still needs LLM, but with constrained scope (response generation only)

### Option D: Rule-Based Routing + Rich Template Responses (No LLM)

Similar to A but with much richer templates — multi-paragraph explanations, card highlighting, decklist walkthroughs — all pre-structured from data.

**Pros:** Zero external dependencies, fully testable, works offline, fast
**Cons:** Responses less natural than LLM, limited to pre-designed interaction patterns

### Recommendation

**Start with Option D** (rule-based routing + rich templates). Reasons:

1. **Anti-infrastructure-trap:** Adding an LLM is a major new dependency and distraction from building agent *behavior*. The track A lesson was clear — agent behavior first, infra later.
2. **Testability:** Templates are deterministic and testable. LLM responses are not.
3. **Data is the value:** The interesting part is connecting real card data, decklists, reviews, and tournament results into coherent explanations. The *data retrieval and composition* is the hard problem, not the natural language generation.
4. **LLM can come later:** If/when the template responses feel too rigid, adding an LLM response layer on top of the same data retrieval pipeline is straightforward.

---

## Coexistence with Quiz Mode

The quiz mode doesn't need to die. It becomes one interaction pattern within the coaching agent:

```
User:  quiz me on glacier
Agent: [generates a concept question from the glacier catalog]
       Why does glacier prefer expensive, multi-sub ice?
       (Think about it, then tell me your answer or type 'show' for the explanation)

User:  Because each subroutine is an additional tax
Agent: That's the right direction. More specifically: each subroutine the
       runner has to break costs them credits. Ice like Tollbooth (rez 8,
       break ~6-8 depending on breaker) creates a credit differential
       that glacier exploits over many runs.

       The key insight: glacier doesn't need to *stop* the runner, just
       make each run cost more than the corp spent to create the server.

       Grade yourself: 0=didn't know, 3=had the idea, 4=good, 5=nailed it
```

The quiz becomes an embedded tool, not the whole experience.

---

## Learning Signal Without Self-Evaluation

With quiz mode, SM-2 gets a grade (0-5) from the user. In coaching mode, the agent needs a different signal. Options:

1. **Explicit check-ins:** Periodically ask "Does that make sense?" or "Want me to go deeper?"
2. **Topic coverage tracking:** Mark concepts as "explored at surface/intermediate/deep" based on conversation depth
3. **Question sophistication:** User asking "what does this card do?" = surface. "How does this card change the matchup?" = deep. Track the depth of questions per concept.
4. **Keep optional self-eval:** After a teaching moment, offer "How well did you know that? (0-5)" — but make it optional, not mandatory.

**Recommendation:** Combine 2 + 4. Track topic depth automatically, offer optional self-eval after key teaching moments. SM-2 still works — it just gets grades less frequently, so intervals are longer.

---

## Proposed Interaction Commands

| Command | Action |
|---------|--------|
| `explore <topic>` | Start/continue exploring a meta concept |
| `card <name>` | Look up a specific card with meta context |
| `deck <id or search>` | Analyze a decklist |
| `matchup <A> vs <B>` | Explore a matchup |
| `meta` | Overview of current format landscape |
| `quiz [topic]` | Enter quiz mode (optional topic) |
| `plan` | Show session plan |
| `concepts` | List all concepts with exploration depth |
| `help` | Show commands |
| (free text) | Agent interprets as question/topic exploration |

---

## Impact on Track A Goals

The Track A goal was: "≥3 unprompted meta-grounded observations per session."

Coaching mode makes this **easier, not harder**:
- Every response can include meta-grounded observations
- Proactive suggestions at the end of each response are naturally unprompted
- Course correction becomes "I notice you've been exploring corp archetypes — the runner side has interesting counterplay, want to explore that?"

The gate check criteria still apply. If anything, coaching mode raises the bar: the agent needs to be genuinely knowledgeable, not just rotate through sample questions.

---

## Four-Round Design Critique

### Round 1: Engineer

**The good:** Data retrieval pipeline is the right focus. Separating intent classification from response generation gives clean module boundaries. Reusing SM-2 for topic surfacing instead of quiz scheduling is a clever repurposing — the algorithm is sound for "what's overdue" regardless of whether the interaction is a quiz or a conversation. Option D is the right starting architecture.

**Concerns:**

1. **Intent classification scope is underspecified.** The doc lists 7 intent categories but doesn't define how free-text maps to them. "What beats PD?" requires matching "PD" to an identity, "beats" to a matchup query, and composing from both the concept catalog and card data. Keyword matching alone won't parse compound queries — you need at least pattern matching with entity extraction (card names, archetype names, identity names).

2. **Conversation state model is too vague.** "Track current topic and depth" doesn't specify the data structure. If the user asks about glacier, then Border Control, then "what deck uses this?" — are those three separate topics or one topic at increasing depth? The state model needs to distinguish between topic threads and tangent threads so proactive suggestions are coherent.

3. **Template authoring cost is hidden.** Option D says "rich templates" but the proposed interaction example has ~200 words of coherent, grounded meta-analysis. Writing that for every concept × intent combination is a content creation project, not an engineering project. With 17 concepts × ~7 intents = 119 response templates. That's a lot of manual writing.

4. **ABR data is powerful but adds ingestion complexity.** The tournament endpoint returns identity codes, not deck compositions. You need to follow `deck_url` links to NRDB to get actual card lists. That's a two-hop data pipeline (ABR → NRDB decklist) that needs caching, error handling, and rate limiting. The entries endpoint also returns variable data — some players don't link decks.

**Proposed solutions:**

- (1) Define a lightweight entity extraction layer: maintain lookup tables for card names, identity names, and archetype aliases. Pattern-match user input against these before intent classification.
- (2) Specify conversation state as a stack-based model: `[{concept: "glacier", depth: "intermediate", entered_via: "user_question"}, {concept: "border-control", depth: "surface", entered_via: "tangent"}]`. Pop when user redirects.
- (3) Don't write 119 templates. Instead, write **response composers** — functions that assemble responses from data fragments. A "concept explorer" composer pulls concept description + key cards + related reviews + tournament stats and assembles them. The data *is* the content.
- (4) ABR ingestion should be a separate module (`tournament_data.py`) with its own fetch/cache cycle, independent of card_data.py. Cache tournament results + entries as JSON, similar to how reviews are cached.

### Round 2: Devil's Advocate

**The pivot itself might be the problem.** The quiz agent was *supposed* to become agentic through Track A (memory, planning, initiative). That work isn't done yet — we're in Phase 2/3. Now we're redesigning the interaction model *before proving the agent behaviors work*. This is scope creep dressed as a pivot.

**Specific challenges:**

1. **"Conversational" without an LLM is a contradiction.** The proposed interaction shows fluid, natural dialogue. Templates can't produce that — they produce recognisably templated output. The user will quickly learn the patterns and it'll feel like navigating a phone tree, not talking to a mentor. The doc acknowledges this ("responses less natural than LLM") but underestimates how much it matters for a "coaching" experience.

2. **The template approach has a content ceiling.** The agent can only teach what's been explicitly authored into templates. A quiz agent can generate infinite questions from a concept catalog + card DB. A template-based coach is bounded by the templates. When the user asks something outside the authored paths, the agent has nothing to say.

3. **Topic depth tracking is easy to specify, hard to validate.** How does the agent know a user went from "surface" to "intermediate" on glacier? By counting exchanges? By keyword analysis? This is actually a harder problem than SM-2 grading because there's no explicit signal.

4. **ABR data staleness.** Tournament results are a snapshot. If data was fetched 2 months ago, the "current meta" responses are outdated. The design doesn't address refresh cadence.

**The honest question:** Are you building a conversational agent, or are you building an interactive Netrunner encyclopedia? Those are different products with different architectures.

**Concession:** The *direction* is right — coaching is more valuable than quizzing for meta learning. But Option D (pure templates) won't deliver the experience shown in the interaction example. Either accept more rigid interactions, or plan for LLM integration sooner than "later."

### Round 3: Product / UX

**The interaction model is promising but the command interface undermines it.** Having `explore`, `card`, `deck`, `matchup`, `meta` as separate commands creates a tool interface, not a coaching interface. A real mentor doesn't need you to prefix your question with a command — they understand what you're asking.

**UX issues:**

1. **Command-first vs conversation-first.** The proposed commands (explore, card, deck, matchup) are a power-user interface. A new user's first instinct is to type "what's a good corp deck?" not "explore glacier." The free-text handler needs to be the *primary* path, with commands as shortcuts.

2. **Session pacing is undefined.** How long is a session? When does the agent suggest wrapping up? The quiz agent had a natural rhythm (question → answer → grade → next). Coaching doesn't have that structure. Without pacing, sessions either end awkwardly or drag.

3. **No discovery affordance.** A new user faces "what do I even ask?" The agent needs to proactively offer topics, not wait. The cold start should be: "Here are the 4 main corp archetypes in Standard right now. Which one interests you?" — not "Type 'help' for commands."

4. **Depth transitions need signalling.** When the agent goes deeper on a topic, the user should know it. "Let me dig into this..." or "At a deeper level..." helps the user track where they are in the learning journey.

5. **ABR tournament data makes a great entry point.** "Yesterday's EMEA AMT was won by Sebastião / Zwicky Group. Want to explore what those decks do?" — this is timely, specific, and naturally leads to archetype exploration. The agent should be able to open with real, recent results.

### Round 4: Decision Advisor

All three rounds raise valid points. Here's how they reconcile:

**The Engineer is right** that response composers (not static templates) are the answer to the template scaling problem. Functions that assemble data fragments into coherent responses are both maintainable and produce varied output. This also addresses the Devil's Advocate's "content ceiling" concern — the ceiling is the data, not the templates.

**The Devil's Advocate is right** that pure templates won't deliver the proposed interaction quality — but wrong that this means we need an LLM immediately. Response composers with good data produce surprisingly readable output when the data is inherently interesting (which card game meta data is). The interaction example is aspirational; the MVP can be more structured and still be valuable.

**The Product person is right** that commands are a crutch. The interaction should be conversation-first with commands as shortcuts. This means the intent classifier is actually the most important component, not the response generators.

**Concrete resolutions:**

1. **Architecture stays at Option D** but with response composers, not static templates. Accept that early output will be more structured/formatted than the aspirational interaction example. Natural language polish is a later concern.

2. **Intent classifier is priority #1 of implementation.** Before any response generators, build the entity extraction + intent classification layer. If this works well, the coaching experience follows. If it doesn't, the whole thing breaks.

3. **Conversation state: use a simple model.** Current concept + exploration depth (surface/intermediate/deep) + list of concepts touched this session. Don't over-engineer — a stack model sounds elegant but adds complexity that isn't needed for a single-user CLI.

4. **ABR tournament data is a first-class data source.** Add `tournament_data.py` for fetching and caching, with `data/tournaments.json` alongside `data/reviews.json`. ABR entries with NRDB deck URLs become the agent's "what's winning" knowledge. Refresh cadence: manual for now (same as reviews), with a note that automated refresh is a future concern alongside YouTube/podcast integration.

5. **Accept the LLM question is deferred, not answered.** Design response composers with a clean `(intent, entities, data) → response_text` interface so an LLM can slot in later as an alternative response generator. Don't optimise for LLM-free forever — just don't block on LLM availability now.

6. **Session pacing:** Define a session as "until the user quits" but add a soft prompt after 15+ minutes or 10+ exchanges: "Good session — we've covered X. Want to keep going or save this for next time?" This mirrors the quiz agent's natural ending without imposing a rigid structure.

7. **Cold start should use real data.** If ABR tournament data is cached, open with "The recent [tournament name] was won with [identity]. Want to explore that archetype?" If no tournament data, fall back to "Here are the main archetypes — which interests you?"

---

## Revised Next Steps

Based on the critique, implementation order is:

1. **ABR tournament data module** (`tournament_data.py`) — fetch + cache tournaments and entries, resolve deck URLs to NRDB decklists. This is the "current meta" backbone.
2. **Entity extraction layer** — lookup tables for card names, identity names, archetype aliases. Fuzzy matching.
3. **Intent classifier** — free-text → (intent, entities) mapping. The most critical new component.
4. **Response composers** — data-driven functions for each intent type (explore, card, deck, matchup, meta overview, quiz).
5. **Conversation state** — simple model: current concept, depth, session history.
6. **Integrate into agent** — new `handle_message()` that routes through intent classifier → data retrieval → response composer.
7. **Keep quiz as embedded mode** — accessible via "quiz me" intent or explicit command.
