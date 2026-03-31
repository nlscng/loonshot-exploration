# Design: Meta-Runner v2 — Conversational Coaching Agent

**Date:** 2026-03-30
**Status:** Draft
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

## Next Steps

1. Design the intent classifier and response generators
2. Decide on conversation state data model
3. Implement alongside existing quiz mode (not replacing it)
4. Add to Track A Phase 2/3 implementation
