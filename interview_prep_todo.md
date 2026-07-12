# Interview Prep TODO

## System Design (Scale Questions)

These will come up whenever an interviewer digs into your resume. The pattern is:
"You built X for 30 users — how would you redesign it for 10,000 concurrent users?"

- [ ] **AutoPVT locking model at scale**
  - What breaks first? Advisory locks are per-PostgreSQL-instance — horizontal scaling breaks them.
  - How to shard: lock by test object ID, route writes to the shard owning that object.
  - Broadcast at scale: replace gRPC-response-coupled broadcast with a message queue (Kafka/Redis Pub/Sub) so the gateway is decoupled from the mutation path.
  - Read replicas for the 2,500-item catalog prefetch — at scale, prefetch becomes a problem.

- [ ] **AutoPVT concurrent editing at scale (CRDT vs. locks)**
  - Know the tradeoff: advisory locks work for 30 users; at 10,000 you'd explore OT or CRDT for fills.
  - Be able to explain why per-field merge under shared locks is essentially last-write-wins and what that means.

- [ ] **InTriHub Firebase architecture at scale**
  - How would you redesign the firmware collection for 1,000 products instead of hundreds?
  - When does the lazy-loaded sibling node pattern break?

- [ ] **Switch Management session limits at scale**
  - Current: in-memory session state in the gateway. What happens with multiple gateway instances?
  - Answer: distributed session store (Redis), or sticky sessions, or centralized session service.

---

## Technical Deep Dives (Resume Bullet Defense)

Every bullet on the resume is fair game. Be able to explain each one cold.

- [ ] **Two-tier PostgreSQL locking model**
  - Explain advisory locks vs. row-level locks — why advisory?
  - What is compare-and-bump? Walk through a race condition scenario.
  - What happens if two people edit the same field simultaneously? (last-write-wins under shared lock)
  - What is the "pairwise operation matrix" — can you draw it?

- [ ] **Commit-before-broadcast ordering**
  - Walk through the exact call sequence: tx.Begin → write → recordHistory → tx.Commit → gRPC response → gateway → broadcastTestProgress.
  - Why does committing before returning the gRPC response guarantee no dirty broadcast?
  - What happens if the WebSocket broadcast fails after the gRPC ack? (the write is committed; clients will see it on next fetch or reconnect — this is a known gap)

- [ ] **CGO-bridged PAM conversation handler**
  - What is PAM? What is a conversation handler?
  - Why CGO instead of shelling out to a binary? (can't distinguish which step failed)
  - Walk through the two-RPC login protocol state machine.
  - Why does web need two-RPC but SSH/Telnet don't? (HTTP is stateless; terminal sessions are already interactive)

- [ ] **IEC 62443-4-2 SL3 — what other requirements exist beyond what you implemented?**
  - CR 1.7 password strength, CR 2.1 RBAC, CR 3.1 communication integrity (TLS), CR 4.1 data confidentiality, CR 6.1 audit log, CR 6.2 continuous monitoring.
  - Be ready to explain that you owned CR 1.1 (TOTP) and CR 2.5 (session limits); others handled the rest.

- [ ] **Firebase cost debugging**
  - What is Firebase pricing model? Why does read size matter?
  - Walk through: embedded log field → 70 MB per read → 4 listeners per page → cost explosion.
  - Why did you reject Firestore/Algolia migration? (root cause fix is cheaper and doesn't risk data migration)

- [ ] **Fail-fast orchestrator**
  - What is gulp-multi-process? What was the bug in the original?
  - What does "kill all remaining workers on first non-zero exit" look like in code?
  - How did you verify the new pipeline produced equivalent output?

- [ ] **SciBERT ensemble**
  - Why 26 separate classifiers instead of one model over all 378 journals?
  - What is the [CLS] token and why use it?
  - What is the accuracy-weighted ensemble formula? Why is it dynamic?
  - Why freeze SciBERT weights? (dataset size; fine-tuning would overfit on 35k samples)

---

## Behavioral Questions (BQ)

FAANG uses structured behavioral interviews (Amazon: Leadership Principles; Google: Googleyness + leadership; Meta: similar). Prepare stories for each.

- [ ] **Ownership / Sole engineer on AutoPVT**
  - Story: designed, built, and shipped a production system used by ~30 engineers, solo.
  - Angle: what tradeoffs did you make under time pressure? What would you do differently?

- [ ] **Delivering results under constraints**
  - Story: Firebase cost spike — $300/month → free tier, rejected the easy but expensive migration.

- [ ] **Earn trust / technical correctness**
  - Story: fail-fast orchestrator — found that CI was silently passing broken builds, fixed it without being asked.

- [ ] **Invent and simplify**
  - Story: commit-before-broadcast ordering — identified a consistency hazard and enforced it at the architecture level.

- [ ] **Dive deep**
  - Story: Firebase debugging — traced cost to data model flaw, 4 independent listeners, fixed at root cause.

- [ ] **Conflict / disagreement**
  - Prepare a story about pushing back on a design decision or approach (any project).

- [ ] **Failure / what went wrong**
  - Prepare a story about something that broke in production or a mistake you made and what you learned.

---

## Coding / LeetCode

- [ ] LeetCode Medium: arrays, strings, hashmaps, sliding window, binary search
- [ ] LeetCode Medium/Hard: trees, graphs (BFS/DFS), dynamic programming
- [ ] Practice: concurrency problems (relevant given AutoPVT locking background)
- [ ] Practice: SQL (JOINs, window functions — relevant given PostgreSQL experience)
- [ ] Target: 2 mediums per day minimum until interviews start

---

## ML / AI (for AI-targeted roles)

- [ ] Be able to explain SciBERT project end-to-end including architecture decisions
- [ ] Review: transformer architecture, attention mechanism, fine-tuning vs. feature extraction
- [ ] Review: PyTorch basics — can you implement a simple classifier from scratch?
- [ ] UCLA MEng AI curriculum — know what you'll be studying and why it matters
- [ ] System design for ML: how would you scale the journal recommendation system to 1M users?

---

## Company-Specific Prep

- [ ] **Google**: focus on coding + system design. Study Spanner, Bigtable, MapReduce papers.
- [ ] **Meta**: behavioral around "move fast" and "impact." System design focus on news feed, real-time systems.
- [ ] **Amazon**: Leadership Principles — prepare one story per principle (16 principles). STAR format.
- [ ] **Apple**: correctness and craftsmanship. Deep system-level questions. Privacy awareness.
- [ ] **Netflix**: operational maturity. Study chaos engineering, failure-mode reasoning.

---

## Open Questions to Research

- [ ] How would you redesign AutoPVT's locking model for 10,000 concurrent editors?
- [ ] What breaks in the commit-before-broadcast pattern under network partition?
- [ ] How does Postgres advisory lock behavior change under connection pooling (PgBouncer)?
