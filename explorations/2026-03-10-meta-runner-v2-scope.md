# Meta-runner v2: Scoping Document

**Date:** 2026-03-10
**Status:** Draft
**Repo:** [nlscng/meta-runner](https://github.com/nlscng/meta-runner)

---

## Vision Shift: From Popularity to Context

Meta-runner v1 defined "meta" as **raw card popularity** — how many decklists include a card. This is shallow. A card appearing in 500 decks tells you it's popular, not *why*, *where*, or *with what*.

Meta-runner v2 redefines "meta" as **card-in-context:**

| v1 "Meta" | v2 "Meta" |
|-----------|-----------|
| Card X is in 508 decks | Card X is a glacier staple, core to PD and Asa builds |
| Card Y is more played than Card Z | Card Y synergizes with A, B, C — together they form the asset-spam econ package |
| Top runner card by deck count | This card appears in 3 distinct archetypes with different roles in each |
| No awareness of format legality | Card is banned in Standard, legal in Startup, restricted in Eternal |

---

## What Exists (v1 Foundation)

| Component | Status | LOC |
|-----------|--------|-----|
| Card data layer (NRDB v2 API → SQLite) | ✅ 2,460 cards | 267 |
| Decklist data (1,245 lists, 52 tournament) | ✅ Stored with descriptions | — |
| Meta stats (flat popularity counts) | ✅ 1,876 card stats | — |
| Quiz agent (6 types, CLI, MS Agents SDK) | ✅ Working | 372 |
| Test suite (67 tests, in-memory SQLite) | ✅ 100% pass | 787 |
| **Total** | | **1,426** |

**Key v1 data already available but underutilized:**
- `decklists.cards_json` — flat `{code: count}` blob. Contains all card↔deck relationships but not queryable.
- `decklists.description` — HTML, user-written. Many contain archetype names, strategy discussion, card choice rationale. Not parsed.
- `decklists.mwl_code` — 562 on `standard-ban-list-25-12`, 90 on `26-03`. Legality data exists but isn't used.
- `cards.raw_json` — full NRDB card data including rotation info. Not extracted.

---

## New Data Sources

### AlwaysBeRunning.net (ABR) — ✅ Public API, no auth

| Endpoint | Data | Meta Value |
|----------|------|------------|
| `GET /api/tournaments` | All events: format, cardpool, date, player count, type | Tournament landscape over time |
| `GET /api/tournaments?concluded=1` | Completed events with winners | Which identities/decks win |
| `GET /api/entries?id={id}` | Per-tournament: standings, identity picks, **NRDB decklist URLs** | Performance data per deck |

**What ABR uniquely provides:** Identity → placement mapping across events. "PD went 4-1 at 3 events this month" is meta intelligence that NRDB alone can't give.

### NetrunnerDB — Additional Endpoints (beyond what v1 uses)

| Endpoint | Data | Meta Value |
|----------|------|------------|
| `/public/reviews` | Card reviews: user analysis, votes, comments | Community-written synergy/archetype discussion |
| `/public/mwl` | Full MWL/ban list history with card codes | Format legality at any point in time |
| `/public/rulings` | Official card rulings, NSG-verified | Interaction clarifications |

**Reviews are the richest source of "card-in-context" data.** Community members write about synergies, archetypes, meta positioning, and card evaluation. Vote counts provide quality signal.

### Null Signal Games (NSG) — Prose only, no API

- Format definitions (rotation, legality rules)
- Ban list reasoning (blog posts explaining *why* cards were banned — meta context)
- Would require scraping for structured data. Lower priority.

### YouTube Channels — Qualitative meta discussion

| Channel | Content Type |
|---------|-------------|
| Metropole Grid | Tournament coverage, meta analysis, competitive commentary |
| Neon Static | Deck techs, card reviews, meta discussions |
| Aksu | Gameplay, deck techs, tournament organizing |
| DullBulb | Deck techs, gameplay, meta discussion |

**How to leverage:**
- **Low effort:** Title + description scraping via YouTube Data API v3 → topic signals, trending decks/cards
- **Medium effort:** Auto-captions/transcripts (yt-dlp) → NLP-based meta sentiment
- **High effort:** Full video analysis → not realistic at 20 min/day

YouTube is a **Phase 3+** data source. Titles alone are surprisingly signal-rich ("Is Hoshiko still good?", "Startup meta breakdown March 2026").

---

## Deck Archetypes

### Common Archetypes

**Corp:**
| Archetype | Strategy | Detection Signal |
|-----------|----------|-----------------|
| Glacier | Tall servers, expensive ice, score behind them | ice_count > 18, avg_ice_cost > 4 |
| Rush | Score fast before runner sets up | Low-cost ice, small agendas, fast-advance |
| Fast Advance | Score from hand without installing | Biotic Labor, SanSan, Seamless Launch |
| Asset Spam | Flood board with assets, overwhelm clicks | asset_count > 10, low ice count |
| Kill/Flatline | Win by lethal damage | End of the Line, Neurospike, Punitive, traps |
| Combo | Assemble specific win condition | Specific card combos |

**Runner:**
| Archetype | Strategy | Detection Signal |
|-----------|----------|-----------------|
| Reg [faction] | Balanced econ + efficient breakers | Full breaker suite, moderate multiaccess |
| Big Rig | Powerful late-game setup | Expensive breakers, heavy tutoring |
| Aggro/Tempo | Early pressure, deny corp tempo | Cheap breakers, run events |
| Econ Denial | Attack corp credits | Diversion of Funds, Hippo |
| Deep Dive/Lock | Repeated central multi-access | Conduit, Deep Dive, Maker's Eye |

**Naming convention:** Identity + Strategy → "Glacier PD", "Reg Zahya", "Kill PE", "Asset Spam Au Co"

**Archetype detection approaches:**
1. **Heuristic rules** (identity + key card signatures) — fast, interpretable, ~70% accuracy
2. **LLM extraction from deck descriptions** — higher accuracy, requires API calls
3. **Card composition clustering** — unsupervised, discovers archetypes rather than matching known ones

---

## Card Synergies

Synergy data is **already implicit in v1's decklist data** — it just needs to be extracted.

**Approach:** Pointwise Mutual Information (PMI) on card co-occurrence.

```
PMI(a, b) = log2( P(a,b) / (P(a) * P(b)) )
```

High PMI = cards appear together more than chance predicts. This filters out staple noise — Hedge Fund appears in everything, so its PMI with any specific card is low. But Punitive Counterstrike + Ontological Dependency would have high PMI (genuine synergy).

**Schema:**
```sql
CREATE TABLE card_cooccurrence (
    card_a TEXT, card_b TEXT,
    shared_decks INTEGER,
    pmi REAL,
    PRIMARY KEY (card_a, card_b)
);
```

---

## Format Legality

NRDB's `/public/mwl` endpoint provides full ban/restriction history:
- `deck_limit: 0` → banned
- `is_restricted: 1` → restricted (max 1 restricted card per deck)
- Versioned by date, mapped to format

**Schema:**
```sql
CREATE TABLE card_legality (
    card_code TEXT,
    format TEXT,         -- 'standard', 'startup', 'eternal'
    status TEXT,         -- 'legal', 'banned', 'restricted', 'rotated'
    effective_date TEXT,
    PRIMARY KEY (card_code, format)
);
```

Quiz questions should **default to format-legal cards only** (Standard), with option to include rotated/banned for historical knowledge.

---

## New Quiz Types (enabled by richer meta)

| Quiz Type | Question | Data Source |
|-----------|----------|-------------|
| **archetype_quiz** | "What archetype runs this card?" | deck_archetypes |
| **synergy_quiz** | "Name a card that synergizes with X" | card_cooccurrence (high PMI) |
| **deck_context** | "This card is key in [archetype]. What role does it play?" | reviews + deck descriptions |
| **legality_quiz** | "Is this card legal in Standard?" | card_legality |
| **archetype_diff** | "Card X appears in both Glacier PD and Rush PD. What's different about its role?" | deck descriptions + reviews |
| **meta_shift** | "This card was in 80 decks last month, now 20. Why?" | temporal meta stats + ban lists |
| **counter_quiz** | "You're facing [archetype]. What cards counter it?" | community knowledge from reviews |

---

## Migration Path

### Phase 1: Schema Enrichment (no new APIs)

Denormalize what v1 already has:

```sql
-- Junction table from existing cards_json blobs
CREATE TABLE deck_cards (
    deck_id INTEGER, card_code TEXT, copies INTEGER,
    PRIMARY KEY (deck_id, card_code)
);

-- Co-occurrence computed from deck_cards
CREATE TABLE card_cooccurrence (
    card_a TEXT, card_b TEXT, shared_decks INTEGER, pmi REAL,
    PRIMARY KEY (card_a, card_b)
);
```

This unlocks: "which decks run this card?", "what cards co-occur?", synergy detection.

### Phase 2: Archetype Classification

- Extract identity from each decklist's `cards_json`
- Heuristic rules: identity + key card signatures → archetype label
- Store in `deck_archetypes(deck_id, archetype TEXT, confidence REAL)`
- Optional: LLM-parse descriptions for richer labels

### Phase 3: New Data Sources

- NRDB reviews endpoint → `card_reviews(card_code, user, text, votes)`
- NRDB MWL endpoint → `card_legality` table
- ABR tournaments + entries → `tournaments`, `tournament_entries` tables

### Phase 4: Quiz Enrichment

- New question types using archetype, synergy, legality data
- Replace flat popularity with contextual meta knowledge
- "This card is a glacier staple" vs "this card is in 508 decks"

### Phase 5: YouTube / NSG (stretch)

- YouTube Data API for title/description scraping (topic signals)
- NSG blog scraping for ban reasoning
- Transcript analysis for qualitative meta discussion

---

## Tension: Infrastructure vs Agent Behavior

**This must be called out honestly.**

The Cycle 2 anti-trap constraint was: "No new infrastructure in sessions 1-3. Agent behavior first."

This scoping document describes **significant new infrastructure** — new tables, new APIs, new data sources, new quiz types. That's the exact trap Cycle 1 fell into.

**Resolution: Two parallel tracks.**

| Track | Focus | Sessions |
|-------|-------|----------|
| **Track A: Agent Behavior** | Memory (SM-2), session planning, initiative — on the EXISTING v1 data | Sessions 1-3 (priority) |
| **Track B: Meta Enrichment** | Richer data, new sources, new quiz types — per this scoping doc | Sessions 4+ (after gate check passes) |

Track A uses v1's flat popularity data. It's not the ideal "meta" definition, but it's enough to build and prove agent behavior. The agent that says "you're weak on ICE" using popularity data is just as agentic as one that says it using archetype data.

Track B transforms the data layer after the agent is proven. The richer meta makes the agent *smarter*, but the agent must exist first.

**Gate check still applies:** If no unprompted agent behavior by session 3, switch projects. Don't let Track B pull resources from Track A.

---

## Carried Forward from v1

| Artifact | Reuse |
|----------|-------|
| Card data pipeline (API → SQLite) | Direct reuse, extend with new tables |
| Quiz generation framework | Extend with new quiz types |
| Fuzzy answer matching | Reuse for all text-based answers |
| Test fixtures (in-memory SQLite) | Extend with new tables |
| Meta stats computation | Replace with richer computation in Phase 1 |
| Decklist data (1,245 lists with descriptions) | Mine for archetypes, synergies |

---

## Summary

Meta-runner v2 is two things:

1. **An agent** — memory, planning, initiative (the primary learning goal)
2. **A smarter quiz tool** — richer meta definition with archetypes, synergies, legality, community knowledge

Build #1 first. Then make it smarter with #2. Don't reverse the order.
