# Career Notes — Framing the AutoPVT (Issue #80) work & transitioning to AI Engineer

---

## Part 1 — How to phrase the work for an interviewer

Interview framing is a different genre. For an interview/resume you drop the feature
checklist and lead with **impact, scale, and the hard technical decisions**.

### Résumé bullets (impact-first, quantifiable where possible)

- Designed and shipped **multi-user concurrent editing** for a hardware test-automation
  platform, letting several QA engineers work the same device-under-test simultaneously —
  replacing a single-user-locked workflow across a 5-service stack (PostgreSQL, Go gRPC
  gateway, two Vue 3 SPAs).
- Built the conflict-resolution layer with **optimistic concurrency (per-record revision
  tokens) + Postgres advisory locks**, plus **field-level 3-way merge** so simultaneous
  edits to the same record never silently overwrite each other.
- Added **real-time collaboration** via a WebSocket event-broadcast system with per-tester
  attribution ("updated by X") and automatic state convergence.
- Cut UI render time on large test plans by **lazy-loading detail payloads and
  batch-rendering rows**, eliminating multi-second freezes (diagnosed via Chrome DevTools
  flame charts).
- Implemented a **persistent audit trail** (who changed what, when) surfaced to supervisors
  through a history viewer.

### The verbal version (a STAR story they can dig into)

- **Situation/Task:** "Our test stations were single-user — one engineer locked the whole
  session. The team wanted several people testing one device at once, which meant solving
  concurrent writes to shared state."
- **Action:** "I went with optimistic concurrency rather than pessimistic locking — each
  record carries a revision token, the server rejects stale writes, and the client merges at
  the *field* level so two people editing different columns of the same row both win. I
  layered Postgres advisory locks for the few truly serial operations, and added a WebSocket
  broadcast so everyone sees changes live."
- **Result:** "First-writer-wins with no lost updates, live awareness, and a full audit
  trail — and along the way I killed a class of UI freezes by making payloads lazy."

### Things interviewers will probe — have answers ready

- **"Why optimistic over pessimistic locking?"** → Reads dominate, conflicts are rare, and
  you avoid holding locks across human think-time.
- **"How do you resolve a real conflict?"** → field-level 3-way merge (server = theirs,
  baseline = common ancestor, form = mine).
- **"What was the subtlest bug?"** → the stale "start new test" wiping another tester's
  freshly-created session — a **compare-and-swap on the box→session binding** fixed it.
- **"How did you know the UI was slow?"** → profiled it, found ~1,400 widget initializations
  dominating; chose batched rendering. (Measure before optimizing.)

### Meta-tip

For an interviewer, **pick one or two threads and go deep** rather than reciting all seven
feature areas. "Concurrency model" and "the performance investigation" are the strongest
stories — they show judgment and trade-off reasoning. The breadth (5 repos, full-stack) is
worth one sentence to establish scope, then dive.

---

## Part 2 — What job title to update to (Frontend → AI Engineer, US market)

Honest answer: **a title change alone won't move you into AI Engineer roles — but the right
title removes a ceiling, and "Frontend Engineer" is a ceiling for AI work.**

### What to put now

**"Software Engineer"** or **"Full-Stack Engineer"** — not "Frontend Engineer," and not
"AI Engineer" yet.

- **Why drop "Frontend":** AI Engineer roles in the US are ~80% backend/systems/integration
  work. A "Frontend Engineer" title makes a recruiter's filter pass you over before a human
  reads anything. And this project earns the broader title — you worked across Go gRPC
  services, Postgres schema, proto/type contracts, and distributed-concurrency design, not
  just UI.
- **Why not "AI Engineer" yet:** if the work behind the title isn't AI, it gets exposed in
  the first screen and damages credibility. Claiming it hollow is worse than not claiming it.

"Full-Stack Engineer" is probably the strongest current label: truthful given what you did,
and it signals the backend breadth that AI-eng hiring cares about.

### Why your background genuinely transfers

"AI Engineer" (vs. ML Engineer) in the US right now ≈ **building production LLM
applications**: RAG, agents, tool-calling, evals, prompt/orchestration, vector search, plus
the unglamorous 80% — API integration, latency/cost/observability, retries, streaming,
state. That is *software engineering*, not model training. Direct mappings from this project:

- real-time/WebSocket streaming → streaming LLM responses, agent event loops
- gRPC/proto API design → tool/function-calling contracts
- optimistic concurrency, retries, conflict resolution → agent state & idempotency
- performance profiling (the 1,400-widget bottleneck) → token/latency/cost optimization

So the pitch is "full-stack engineer with distributed-systems and real-time chops, now
applying them to LLM systems."

### The bridge (what actually unblocks the job search)

A title gets you past filters; **shipped AI projects get you the interview.** Sequence:

1. **Now:** title → *Full-Stack / Software Engineer.*
2. **Next 1–3 months:** build 2 substantive LLM projects — e.g. a RAG system with real
   evals, and an agent that calls tools — deployed, with a writeup on latency/cost/eval
   trade-offs (not a toy notebook).
3. **Then:** evolve title to **"Software Engineer, AI"** or **"AI Engineer"**, or for
   startups, **"Member of Technical Staff"** (common AI-startup framing that sidesteps the
   ML-vs-AI label).

### Reframe this project for AI-target resumes

Lead with the transferable systems story, not the test-automation domain: "real-time
multi-user system with optimistic concurrency, event broadcasting, and full-stack API design
across a service-oriented architecture" — then one line that you're now applying it to
LLM/agent systems.

### Bottom line

Change to **Full-Stack / Software Engineer** today (credible, removes the frontend ceiling),
build demonstrable LLM projects, then claim **AI Engineer** once it's true.
