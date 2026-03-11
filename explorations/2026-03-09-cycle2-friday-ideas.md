# Cycle 2 Friday Ideation - 2026-03-09

**Day of Week:** Monday (running Friday phase early)
**Framework Phase:** Idea Generation + AI Narrowing
**Time Spent:** ~30 min

---

## 11 Ideas

| # | Idea | Source |
|---|------|--------|
| 1 | Meta-runner v2: make it truly agentic | Cycle 1 continuation |
| 2 | Agentic Terraform/IaC tutor | New |
| 3 | Build an MCP server | Cycle 1 unused (#2) |
| 4 | Synthetic PII agent for Haymaker | Cycle 1 unused (#3) |
| 5 | Tenant resource scrubber agent | Cycle 1 unused (#4) |
| 6 | Learn IaC by reverse-engineering | Cycle 1 unused (#8) |
| 7 | Agent that agents: meta-agent builder | New, from learnings |
| 8 | Daily standup summarizer agent | New |
| 9 | Code review agent with memory | New |
| 10 | Personal knowledge graph builder | New |
| 11 | A2A (Agent-to-Agent protocol) project | New (added 2026-03-10) |

---

## Analysis Method

Three rounds of review, each from a different perspective:

1. **Round 1 — Engineer/Architect:** Deep analysis of scope, feasibility, architecture, design, learning value, fun, career value, and meta-runner trap risk for each idea.
2. **Round 2 — Devil's Advocate:** Critical counter-review challenging Round 1's assumptions, finding blind spots, stress-testing feasibility, steelmanning and objecting.
3. **Round 3 — Decision Advisor:** Synthesized both rounds into adjusted scores, behavioral tests, and definitive ranking.

---

## Idea-by-Idea Analysis

### Idea 1: Meta-runner v2 — Make It Truly Agentic

**What:** Add learning memory (spaced repetition, per-user state), session planning (agent proposes focus areas and executes), and autonomous initiative (agent decides when to review, hint, switch topics) to the existing meta-runner quiz tool.

**Scope:** ~300-400 new LOC on existing 639 LOC foundation. Components: Learning Memory store, SM-2 spaced repetition engine, Session Planner, Initiative Engine. Realistic for 20 min/day — no new APIs, no new frameworks.

**Feasibility:** Low risk overall. Infrastructure is done. SM-2 is well-documented. The hard problem is making the MS Agents SDK ActivityHandler "take initiative" when it's fundamentally request-response — may need a loop/scheduler. The critical review flagged this as underestimated: the SDK constraint is the core design challenge, not a footnote.

**Architecture:**
```
[Existing Card Data] → [Quiz Tool (existing)]
                              ↓
[Learning Memory (NEW)] ← [Session Planner (NEW)]
                              ↓
[Initiative Engine (NEW)] → [CLI Output]
```

**Learning value:** HIGH. Directly addresses Cycle 1 gap: persistent state, decision-making logic, autonomous behavior in a request-response framework.

**Fun:** MEDIUM. Extending existing project loses novelty, but "fixing a known failure" has motivational power.

**Career value:** HIGH. Agent memory patterns, spaced repetition, autonomous decision-making — all transferable.

**Trap risk:** LOW. The remaining work IS the agent behavior.

**Critical review highlights:**
- SM-2 tuning for card games (not language flashcards) will consume sessions if not hardcoded.
- Schema extension IS infrastructure even if the analysis says "trivial."
- Cold-start problem: need card history data to test planning meaningfully.
- Comfort zone objection: learning in familiar domain misses navigation skills.
- Counter: 639 LOC foundation means observable agent behavior in 2-3 sessions — no other idea offers this.

**Behavioral test:** The agent opens a session and says "I notice you're weak on ICE subtypes — let's focus there today" *without the user requesting a topic.*

| Scope Fit | Feasibility | Agent Learning | Career Value | Fun | Trap Risk (10=worst) |
|:---------:|:-----------:|:--------------:|:------------:|:---:|:--------------------:|
| 8 | 7 | 8 | 6 | 5 | 3 |

---

### Idea 2: Agentic Terraform/IaC Tutor

**What:** An agent that teaches Terraform through exercises, reviews .tf files, quizzes on resource types, adapts to user level. Focus on agent tutoring behavior from day one.

**Scope:** Building from scratch: knowledge base, exercise generator, .tf file reviewer, quiz engine, adaptive difficulty, agent tutoring behavior. TIGHT for 20 min/day — more components than meta-runner v1 started with.

**Feasibility:** Medium risk. Terraform docs are vast — scoping "which concepts" is a curriculum design project. python-hcl2 parser is limited. Need to decide: real execution vs dry-run? Who validates tutor correctness if you're learning simultaneously?

**Critical review highlights:**
- "Double learning" (Terraform + agents) is actually divided attention: 0.5x each, not 2x.
- Agent tutor NEEDS content to tutor about — can't build behavior first, unlike Idea 1.
- At 20 min/day this is a 3-cycle project pretending to be 1 cycle.
- Counter: transferable pattern — agentic tutor for Domain X is reusable template. Career-critical IaC knowledge compounds.

**Behavioral test:** The agent says "You've mastered resource blocks but keep misusing `depends_on` — here's a targeted exercise" *without being told what to practice.*

| Scope Fit | Feasibility | Agent Learning | Career Value | Fun | Trap Risk |
|:---------:|:-----------:|:--------------:|:------------:|:---:|:---------:|
| 3 | 4 | 5 | 7 | 6 | 8 |

---

### Idea 3: Build an MCP Server

**What:** Create a Model Context Protocol server to extend AI tool capabilities.

**Scope:** Minimal server is ~100 LOC. Purpose-dependent — the protocol is simple, the question is what to expose.

**Feasibility:** Low risk technically. The risk is purpose — no compelling use case was proposed.

**Critical review highlights:**
- This is a technology choice in search of a problem.
- MCP is literally infrastructure — a protocol adapter. It never initiates anything.
- Counter: ecosystem timing gives disproportionate career value. Could be a multiplier if combined with another idea (expose meta-runner as MCP tools). But that's a Cycle 4 idea.
- "The procrastination project. Pencil-sharpening before writing."

**Behavioral test:** None exists. An MCP server responds to requests. It never initiates. **Disqualifying.**

| Scope Fit | Feasibility | Agent Learning | Career Value | Fun | Trap Risk |
|:---------:|:-----------:|:--------------:|:------------:|:---:|:---------:|
| 7 | 8 | 2 | 5 | 3 | 9 |

---

### Idea 4: Synthetic PII Agent for Haymaker

**What:** Agent that generates realistic synthetic personal info for benign telemetry testing.

**Scope:** PII generation engine (Faker), realism scorer, schema interpreter. Feasible for 20 min/day basics.

**Feasibility:** Low-medium risk. Faker handles most generation. "Realistic enough" is an undefined quality bar.

**Critical review highlights:**
- Agent behavior is thin — the "PlannerAgent" is a for-loop reading a schema and calling generators.
- 90% Faker configuration, 10% agent behavior.
- Counter: only idea with immediate professional value. Schema-driven generation is a common real-world agent task.
- "This isn't a side project, it's a work task disguised as learning. Use sprint time."

**Behavioral test:** The agent says "Your schema has a phone field next to a country field — I'll generate regionally-correct formats" *without being told fields are related.*

| Scope Fit | Feasibility | Agent Learning | Career Value | Fun | Trap Risk |
|:---------:|:-----------:|:--------------:|:------------:|:---:|:---------:|
| 6 | 7 | 3 | 5 | 4 | 8 |

---

### Idea 5: Tenant Resource Scrubber Agent

**What:** Generate new IDs/resource names for copied Azure tenants while scrubbing originals.

**Scope:** ARM template parsing + name generation + cross-reference resolution. ARM templates are adversarially complex JSON.

**Feasibility:** Medium-high risk. Nested dependsOn, copy loops, reference() functions, linked templates. "80% done" means "20% through the real complexity."

**Critical review highlights:**
- "A JSON transformation pipeline with an agent label stapled to it."
- 95% infrastructure. Worst learning-to-effort ratio.
- Counter: radically descoped (any-JSON PII scrubber with learning from corrections) might be interesting. But the rated version is a non-starter.

**Behavioral test:** The agent says "These three resources reference the same key vault — I'll generate consistent replacements" *without being told about cross-references.* (Would never reach this.)

| Scope Fit | Feasibility | Agent Learning | Career Value | Fun | Trap Risk |
|:---------:|:-----------:|:--------------:|:------------:|:---:|:---------:|
| 4 | 3 | 2 | 4 | 2 | 9 |

---

### Idea 6: Learn IaC by Reverse-Engineering

**What:** Take existing Azure resources, export to Terraform/Bicep/ARM, compare trade-offs.

**Scope:** Not a build project — a structured learning exercise. No software to design.

**Feasibility:** Low risk, but needs Azure subscription with representative resources. terraform import is notoriously painful.

**Critical review highlights:**
- Wrong goal entirely. HIGH learning value for IaC, ZERO for agent building.
- "Procrastination disguised as productivity. Choosing comfort over growth."
- Counter: if IaC is the actual career bottleneck, this is rational optimization.

**Behavioral test:** None. Not an agent. **Formally eliminated.**

| Scope Fit | Feasibility | Agent Learning | Career Value | Fun | Trap Risk |
|:---------:|:-----------:|:--------------:|:------------:|:---:|:---------:|
| 5 | 6 | 0 | 8 | 2 | N/A |

---

### Idea 7: Agent that Agents — Meta-Agent Builder

**What:** An agent that helps scaffold and iterate on other agents. Template library + behavior classifier + scaffold generator + iteration advisor.

**Scope:** Very tight for 20 min/day. Defining agent patterns formally is hard — it's judgment, not rules.

**Feasibility:** High risk. Chicken-and-egg: need agent expertise to build this, but this is supposed to teach agent expertise. The LLM backbone IS the product — your code is just a prompt wrapper.

**Critical review highlights:**
- "Abstraction addiction. After failing to build one concrete agent, the response is to build an abstract meta-agent?"
- Template library is infrastructure that blocks the interesting work.
- Counter: formalizing patterns forces deep study. Sometimes building the wrong thing teaches the right lesson.
- "Build a real agent first. Earn the right to go meta."

**Behavioral test:** The agent says "Your description sounds like a planning agent — here's a scaffold with memory and goal decomposition" *without being told the agent type.* (Can't build this classifier without the experience you're trying to gain.)

| Scope Fit | Feasibility | Agent Learning | Career Value | Fun | Trap Risk |
|:---------:|:-----------:|:--------------:|:------------:|:---:|:---------:|
| 3 | 3 | 4 | 5 | 3 | 7 |

---

### Idea 8: Daily Standup Summarizer Agent

**What:** Reads git commits, PRs, work items, calendar → autonomously drafts daily standup. Adapts to your style over time.

**Scope:** Multiple API integrations (git, GitHub, ADO, Calendar) + LLM summarizer + style adapter. Tight with all sources. **With git-only hard constraint: very feasible.**

**Feasibility:** Split. Git-only: low risk (git log is 10 LOC). Full version: medium-high risk (four APIs with different auth models).

**Architecture (git-only version):**
```
[Git Log (last 24h)] → [LLM Summarizer] → [Style Memory] → [Standup Draft]
```

**Critical review highlights:**
- WITH git-only constraint: passes meta-runner trap. Into summarization and style learning by day 2.
- WITHOUT constraint: fails. Four API integrations is pure infrastructure.
- Only idea with daily utility DURING the build cycle — tight feedback loop.
- Counter: core loop (read → LLM → format) is a pipeline, not an agent. The "agentic" parts are real but secondary.
- Cold-start on style learning: only ~5 standups in one cycle.

**Behavioral test:** The agent says "You committed a lot to auth/ this week but your standup never mentions it — want me to include it?" *without being told what's important.*

| Scope Fit | Feasibility | Agent Learning | Career Value | Fun | Trap Risk |
|:---------:|:-----------:|:--------------:|:------------:|:---:|:---------:|
| 7 | 7 | 6 | 6 | 8 | 4 |

*Scores assume git-only constraint.*

---

### Idea 9: Code Review Agent with Memory

**What:** Reviews PRs, remembers past feedback patterns, stops repeating itself, adapts to your style. Persistent memory across sessions.

**Scope:** PR diff reader + LLM review + memory store + style learner + dedup + prioritizer. Tight for one cycle.

**Feasibility:** Medium risk. LLM review noise is the central challenge (competing with well-funded products). Memory feedback loop needs weeks of data — 3-5 reviews in one cycle isn't enough.

**Critical review highlights:**
- Memory-augmented LLM with feedback loops is THE foundational agent pattern. Highest agent-learning ceiling.
- But: insufficient data in one cycle for memory to differentiate from noise.
- LLM review setup eats sessions 1-4 before you reach the interesting parts.
- Counter: the memory/adaptation pattern is everything. Domain barely matters.
- **Strong Cycle 3 candidate** after building memory patterns in Cycle 2.

**Behavioral test:** The agent says "You rejected a similar `try/except: pass` pattern last week — flagging this as high priority" *without being told about the previous review.*

| Scope Fit | Feasibility | Agent Learning | Career Value | Fun | Trap Risk |
|:---------:|:-----------:|:--------------:|:------------:|:---:|:---------:|
| 5 | 5 | 9 | 8 | 7 | 5 |

---

### Idea 10: Personal Knowledge Graph Builder

**What:** Reads notes/journals/repos, builds queryable knowledge graph with autonomous connection finding.

**Scope:** Document ingester + entity extractor + relationship mapper + graph storage + query + connection finder. NO — this is a research project at 20 min/day.

**Feasibility:** High risk. Entity extraction from unstructured text is research-grade NLP. No clear MVP — a knowledge graph with 10 entities is useless.

**Critical review highlights:**
- Four infrastructure layers before first agent behavior. CATASTROPHICALLY fails meta-runner trap.
- Counter: brutally scoped (git commits only, tool/decision/reason triples) might work.
- "Solution in search of a problem. Will build graph and never query it."

**Behavioral test:** The agent says "I found a connection between your consensus notes and your CAP theorem paper" *without being asked.* (Would be debugging Neo4j for the entire cycle.)

| Scope Fit | Feasibility | Agent Learning | Career Value | Fun | Trap Risk |
|:---------:|:-----------:|:--------------:|:------------:|:---:|:---------:|
| 2 | 2 | 4 | 5 | 5 | 10 |

---

## Idea #11: A2A (Agent-to-Agent Protocol) Project — Added 2026-03-10

**What:** Build a multi-agent system using Google's A2A (Agent-to-Agent) protocol — an open protocol (now Linux Foundation) for agent-to-agent communication. JSON-RPC 2.0 over HTTP, Agent Cards for discovery, task lifecycle management. Distinct from MCP: MCP is vertical (model→tools), A2A is horizontal (agent↔agent). Python SDK at v0.3.25, pre-1.0, ~11 months old.

**Scope:** Three possible shapes:
- Shape A: "Netrunner Debate Club" — two agents argue card strategy (~330 LOC, 5-6 sessions)
- Shape B: "Draft Advisor" — scout + evaluator pipeline via A2A (~410 LOC, 6-7 sessions)
- Shape C: "Hiring Board" — dynamic discovery orchestrator with 3+ agents (~500 LOC, 7-8 sessions, tight)

**Architecture (Shape B — most balanced):**
```
                    USER (CLI)
                       │
                       ▼
              ORCHESTRATOR (A2A Client)
              1. Fetch Agent Cards
              2. Delegate to Scout
              3. Pass results to Evaluator
              4. Present synthesis
               ╱              ╲
          A2A/HTTP          A2A/HTTP
             ╱                  ╲
    ┌──────────────┐    ┌──────────────────┐
    │ SCOUT AGENT  │    │ EVALUATOR AGENT  │
    │ :8001        │    │ :8002            │
    │ [Card DB]    │    │ [LLM + Meta KB]  │
    └──────────────┘    └──────────────────┘
```

**Feasibility:** SDK works, samples exist, docs are decent. But pre-1.0 ecosystem — expect thin Stack Overflow coverage, SDK bugs, and async/multi-process friction. Protocol debugging will consume time.

**Learning value:** Teaches *coordination between agents* — discovery, delegation, task lifecycle. Does NOT teach the core agent skills that failed in Cycle 1 (memory, planning, initiative). The protocol is about the pipes, not about what makes something an agent.

**Fun:** Medium (6/10). The "wow" of two agents talking across processes is real but comes late (session 3-4) after significant boilerplate. More technical than experiential.

**Career value:** High (7/10). A2A is positioned as the standard for agent interop. Google + IBM + Microsoft backing. But pre-1.0 means implementation-specific knowledge depreciates in ~12 months. The *concepts* transfer; the *SDK calls* won't.

### Three-Round Analysis

**Round 1 — Engineer/Architect:**
- A2A is a legitimate protocol solving a real problem (agent collaboration as peers, not tool invocation)
- 40-50% of time on protocol plumbing in sessions 1-3, dropping to 20% in sessions 4+
- Cannot write a behavioral test as compelling as Meta-runner v2's — "the protocol adds ceremony, not capability"
- Recommended as Cycle 3-4 project: "Build the conversationalist before building the telephone network"

**Round 2 — Devil's Advocate:**
- "This isn't an idea. It's Idea #3 wearing a different protocol's t-shirt." The word "something" is doing all the work in the sentence.
- MCP was rejected as "technology choice in search of a problem." A2A has the same sin — both are protocols, both are infrastructure.
- Key difference: A2A connects agents that *can* have initiative, MCP connects tools that *can't*. But you have ZERO working agents with initiative — "building a highway between two cities that don't exist."
- Estimated 60-70% protocol plumbing. Probability of repeating Cycle 1 failure: 70-80%.
- Kill shot: "You can't build a highway between two cities that don't exist yet."

**Round 3 — Decision Advisor:**
- Feasibility score disagreement (Round 1: 7, Round 2: 4) resolved to **5** — you can build it, you'll learn the wrong things.
- Career value disagreement (Round 1: 7, Round 2: 6) resolved to **7** — value is real, timing is wrong.
- Agent Learning confirmed at **3** — protocol work dominates, core agent skills untouched.
- Behavioral test assessment: weak/partially disqualifying. Best attempt: "Agent B remembers prior critique and adjusts intensity" — but that tests the *agent*, not A2A. Strip out A2A, replace with function calls, test is identical.

**Behavioral test:** Agent B receives a deck critique request, remembers what it criticized in the previous exchange, and adjusts critique intensity based on whether Agent A addressed the prior feedback — *without being told about the history.* **Verdict: Weak.** The interesting behaviors (memory, adaptation) exist independent of A2A. The protocol adds transport, not capability.

| Scope Fit | Feasibility | Agent Learning | Career Value | Fun | Trap Risk (10=worst) |
|:---------:|:-----------:|:--------------:|:------------:|:---:|:--------------------:|
| 4 | 5 | 3 | 7 | 6 | 8 |

---

## Updated Final Ranking (11 Ideas)

| Rank | Idea | Justification |
|:----:|------|---------------|
| **1** | **#1 Meta-runner v2** | Only idea where infrastructure is DONE and remaining work IS agent behavior. Observable autonomy in 2-3 sessions. "Comfort zone" objection overridden: learning agent patterns in a familiar domain is better than learning neither in a new one. |
| **2** | **#8 Standup Summarizer** (git-only) | Highest fun score, daily utility during build, tight scope with constraint. Loses to #1 because core loop is a pipeline — "agentic" parts are real but secondary. |
| **3** | **#9 Code Review Agent** | Best agent-learning ceiling. Memory + feedback loops + style adaptation is exactly the pattern to master. Ranked 3rd because insufficient data in one cycle for memory to matter. **Strong Cycle 3 candidate.** |
| **4** | **#2 Terraform Tutor** | Good transferable pattern, wrong timing. Domain knowledge overhead kills agent learning at 20 min/day. Revisit after building one successful agent AND learning Terraform separately. |
| **5** | **#4 Synthetic PII** | Practical but wrong venue. Agent surface area too thin. Do on company time. |
| **6** | **#7 Meta-Agent Builder** | Right idea, wrong sequence. Build a real agent first, earn the right to go meta. |
| **7** | **#11 A2A Protocol Project** | Real technology, real career value, wrong cycle. Protocol plumbing ≠ agent behavior. Slots above MCP because multi-agent coordination is a harder, rarer skill — but still in the "real value, wrong timing" tier. **Strong Cycle 4 candidate.** |
| **8** | **#3 MCP Server** | Infrastructure by definition. Protocol knowledge, not agent knowledge. |
| **9** | **#10 Knowledge Graph** | Research project cosplaying as a side project. |
| **10** | **#5 Tenant Scrubber** | JSON pipeline in an agent trenchcoat. |
| **11** | **#6 IaC Reverse-Eng** | Eliminated. Wrong goal entirely. |

---

## Combination Opportunities

- **#1 → #9 (Sequential, not combined):** Build agent memory + planning in meta-runner (Cycle 2). Port those patterns to code review where they have higher career value and more data (Cycle 3). Memory system from Cycle 2 becomes reusable foundation.
- **#3 as distribution for #1:** Expose meta-runner as MCP server so it can be invoked from your editor. But this is Cycle 4 — build the agent first, distribute later.
- **#8 + #1:** Don't combine. Different domains, different data. Combining dilutes both.
- **#11 as evolution of #1 (Cycle 4):** Build meta-runner v2 with real agency (Cycle 2). Build a second agent (Cycle 3). Connect them via A2A (Cycle 4). By then both agents have memory/planning/initiative, and A2A has something worth carrying. The behavioral test becomes: "Does the meta-runner know *when* to delegate and *what context* to send?"

---

## Recommendation

**Pick #1: Meta-runner v2.** The cure for "built infrastructure, never reached agent behavior" is NOT "build infrastructure in a new domain." It's: reach agent behavior. Meta-runner v2 is the only idea where Day 1 work is agent work.

### Anti-Trap Constraint (non-negotiable)

**"No new infrastructure in sessions 1-3."**

- No schema changes. No new database tables. No SDK refactoring. No new API calls.
- Sessions 1-3 produce agent behavior on the EXISTING schema, even if ugly.
- Hardcode SM-2 parameters. Use a JSON file for memory. Write planning logic as a pure function.
- Refactor into proper infrastructure in sessions 4-7 AFTER the agent behavior works.

### Gate Check

If by end of session 3 the agent has not said something unprompted, **STOP and switch to #8 (Standup Summarizer, git-only).** Don't sink-cost a second cycle.

---

## Next Session

**Monday's Phase:** Select & Frame (if choosing #1, framing is mostly done — refine the four sentences for v2)

Pick up by reading this entry and the Cycle 1 retrospective (2026-03-09.md).
