# Resume Assessment Rubric
Based on UCLA GCS Resume + Cover Letter Guide (Winter 2026) + FAANG+ hiring signals

---

## Part 1 — UCLA GCS Criteria (max 8 pts per bullet)

| Criterion | Description | Scale |
|-----------|-------------|-------|
| **V — Verb-first** | Opens with an action verb | 0 = no / 1 = yes |
| **A — Achievement-led** | Outcome-focused, not responsibility-focused | 0 = pure Z (what you built) / 1 = mixed (Z-first, impact later) / 2 = achievement-first |
| **M — Metric** | Contribution is quantified | 0 = none / 1 = technical/scale metric (counts, sizes) / 2 = business metric ($, %, time) |
| **Z — Action clear** | What you personally did is specific and concrete | 0 = vague / 1 = concrete |
| **X — Impact explicit** | Outcome is clearly stated | 0 = not stated / 1 = implied / 2 = explicit |

**Guide thresholds:** 7–8 = strong | 5–6 = good | 3–4 = needs rework | 0–2 = rewrite

---

## Part 2 — FAANG+ Criteria (max 6 pts per bullet)

| Criterion | Description | Scale |
|-----------|-------------|-------|
| **S — Scale** | Does the bullet communicate the size of the system or impact? FAANG cares whether you worked on something real and large. | 0 = no scale signal / 1 = internal/team scale (N engineers, N product lines, N items) / 2 = production/external scale (user count, traffic volume, data size, enterprise customers) |
| **O — Ownership** | Is your personal contribution clearly distinguished from the team's? | 0 = ambiguous, could be anyone / 1 = your role is implied / 2 = explicit ("sole engineer", "Led", "designed X while others...") |
| **D — Difficulty** | Does the bullet signal a genuinely hard engineering problem? | 0 = routine, any mid-level engineer would do this / 1 = intermediate complexity, some constraint visible / 2 = clearly hard: concurrency, security protocol design, distributed correctness, performance at scale |
| **R — Reasoning** | Does the bullet hint at WHY a technical decision was made — tradeoffs, constraints, non-obvious design choices? | 0 = only states what was built / 1 = implies or states the constraint or design choice that drove the decision |

**FAANG thresholds:** 5–6 = strong | 3–4 = good | 1–2 = weak | 0 = invisible to senior screeners

---

## Assessment — resume_comprehensive.md

### Intrising — AutoPVT

| # | Bullet (summary) | V | A | M | Z | X | GCS | S | O | D | R | FAANG | Total |
|---|-----------------|---|---|---|---|---|-----|---|---|---|---|-------|-------|
| 1 | Two-tier locking model, sole engineer, ~30 engineers | 1 | 1 | 1 | 1 | 2 | **6** | 1 | 2 | 2 | 1 | **6** | **12** |
| 2 | Eliminated dirty-state broadcasts, commit-before-broadcast ordering | 1 | 2 | 0 | 1 | 2 | **6** | 0 | 1 | 2 | 1 | **4** | **10** |
| 3 | LISTEN/NOTIFY pipeline, Redis rationale, firmware build status | 1 | 2 | 0 | 1 | 2 | **6** | 0 | 1 | 2 | 1 | **4** | **10** |
| 4 | 3-5s → under 500ms, N+1 eliminated, 2,500-item catalog | 1 | 2 | 2 | 1 | 2 | **8** | 1 | 1 | 1 | 1 | **4** | **12** |

**Notes:**
- #1: "Sole engineer" is the strongest ownership signal in the resume. Concurrency design scores max on difficulty.
- #2: No metric. Architectural correctness bullets are hard to quantify — acceptable trade-off. Convergence clause ("all connected clients always converge on committed state") added for clarity but does not change rubric scores.
- #3: Added Redis rationale — explicit trade-off reasoning ("chosen over Redis pubsub to avoid introducing a new infrastructure dependency") lifts D from 1→2. Still no scale metric (S=0).
- #4: Added before/after time metric (3-5s → under 500ms) — lifts M from 1→2 (time qualifies as business metric), pushing GCS to 8/8.

---

### Intrising — Switch Management Interface

| # | Bullet (summary) | V | A | M | Z | X | GCS | S | O | D | R | FAANG | Total |
|---|-----------------|---|---|---|---|---|-----|---|---|---|---|-------|-------|
| 1 | IEC 62443-4-2 SL3, four product lines, session limits + TOTP | 1 | 2 | 1 | 1 | 1 | **6** | 1 | 1 | 2 | 1 | **5** | **11** |
| 2 | Bridged PAM/HTTP impedance mismatch, two-RPC protocol, backward compat | 1 | 2 | 0 | 1 | 2 | **6** | 1 | 1 | 2 | 1 | **5** | **11** |
| 3 | Config-save bug, unrestorable configs, two silent failure modes closed | 1 | 2 | 1 | 1 | 2 | **7** | 1 | 1 | 2 | 1 | **5** | **12** |
| 4 | Cut CI build time 80%+, fail-fast orchestrator | 1 | 2 | 2 | 1 | 2 | **8** | 0 | 1 | 1 | 1 | **3** | **11** |

**Notes:**
- #1: "Led" signals ownership but it was partial (session control + auth only). X=1 because impact is risk mitigation, not a business number.
- #2: Rewritten to lead with PAM/HTTP impedance mismatch ("Bridged PAM's interactive challenge-response model with HTTP's stateless request cycle"), naming the engineering problem explicitly and lifting FAANG S from 0→1. Backward compat preserved in the protocol clause. CGO removed from bullet (jargon to non-Go screeners); "custom PAM conversation handler" carries the technical signal instead. No metric is the remaining gap.
- #3: One of the two best individual bullets in the resume. Root-cause + fix + cross-domain scope signals debugging depth FAANG values.
- #4: First 8/8 guide score. FAANG lower because no scale signal and CI optimization reads as intermediate difficulty.

---

### Intrising — InTriHub

| # | Bullet (summary) | V | A | M | Z | X | GCS | S | O | D | R | FAANG | Total |
|---|-----------------|---|---|---|---|---|-----|---|---|---|---|-------|-------|
| 1 | Cut $300/month Firebase to free tier, 70 MB → 5 MB, 93% | 1 | 2 | 2 | 1 | 2 | **8** | 2 | 1 | 1 | 1 | **5** | **13** |
| 2 | Access-control gap: firmware log exposed to non-admin roles | 1 | 0 | 0 | 1 | 1 | **3** | 2 | 1 | 1 | 0 | **4** | **7** |

**Notes:**
- #1: Tied for highest combined score. Business metric up front ($300/month → free tier) plus 93% size reduction. M=2. Two fixes named: lazy-loaded sibling node and listener consolidation. R=1 (listener stacking implies architecture insight). S=2 because cloud cost is production/external. Only gap: O=1 (role implied, not stated as "sole").
- #2: 5/5 independent FAANG-persona reviewers recommended **cutting from the 1-page resume** — textbook "found and fixed a single bug in my own code" (surfaced incidentally while restructuring data for #1, not from a deliberate security-testing practice). M=0 (no metric) and R=0 (no stated design reasoning) confirm it structurally. S=2 only because InTriHub itself has external partner users, not because the fix is hard. Kept in `resume_comprehensive.md` as interview-prep material, removed from `resume_sde.md`.

---

### Texim Europe

| # | Bullet (summary) | V | A | M | Z | X | GCS | S | O | D | R | FAANG | Total |
|---|-----------------|---|---|---|---|---|-----|---|---|---|---|-------|-------|
| 1 | Eliminated reliance on end-of-life ETL platform | 1 | 2 | 0 | 1 | 2 | **6** | 1 | 1 | 1 | 1 | **4** | **10** |
| 2 | SQL Server Query Notifications, push-based ETL status updates | 1 | 1 | 0 | 1 | 1 | **4** | 0 | 1 | 1 | 1 | **3** | **7** |

**Notes:**
- Both improved to A=2 (achievement-first). Remaining weakness is no metric — add a number (N jobs automated, time saved per week) if you can recall one.
- #2 FAANG S=0 because "scheduled ETL jobs" has no stated count or scale.

---

### NLP Project

| # | Bullet (summary) | V | A | M | Z | X | GCS | S | O | D | R | FAANG | Total |
|---|-----------------|---|---|---|---|---|-----|---|---|---|---|-------|-------|
| 1 | Achieved 96.9% top-5 accuracy, 378 journals, "by designing and training" explicit | 1 | 2 | 2 | 1 | 2 | **8** | 1 | 1 | 2 | 1 | **5** | **13** |
| 2 | Flask app, parallel inference, 378 journals, top-5 | 1 | 1 | 1 | 1 | 2 | **6** | 1 | 1 | 1 | 1 | **4** | **10** |

**Notes:**
- #1: "Achieved 96.9%..." leads with metric and attributes it to "designing and training" — lifts O from 0 to 1. Ties InTriHub for highest combined score.
- #2: A=1 (mixed — "Built a Flask web application" is still Z-first). Acceptable; the serving logic is secondary to the training result above it.

---

### Rosenxt Project

| # | Bullet (summary) | V | A | M | Z | X | GCS | S | O | D | R | FAANG | Total |
|---|-----------------|---|---|---|---|---|-----|---|---|---|---|-------|-------|
| 1 | Frontend breadth, 5 pages independently owned | 1 | 1 | 1 | 1 | 1 | **5** | 1 | 1 | 0 | 1 | **3** | **8** |
| 2 | RBAC, session management, three permission levels | 1 | 2 | 1 | 1 | 2 | **7** | 1 | 0 | 1 | 1 | **3** | **10** |

**Notes:**
- #1: A=1 (activity-focused, no concrete outcome). D=0 (routine React pages). O=1 because "independently owning" is stated. Weakest bullet on the resume — pending optional improvement (lead with "5 of the application's core pages" instead of "majority").
- #2: RBAC bullet is solid on guide criteria. O=0 because team of 5 makes ownership unclear. Pending candidate input: role names would lift Z.

---

## Summary by Combined Score

| Score | Bullets |
|-------|---------|
| **13 (best)** | InTriHub, NLP #1 |
| **12** | AutoPVT #1, AutoPVT #4, SM #3 |
| **11** | SM #1, SM #2, SM #4 |
| **10** | AutoPVT #2, AutoPVT #3, Texim #1, NLP #2, Rosenxt #2 (RBAC) |
| **8** | Rosenxt #1 (frontend) |
| **7** | Texim #2, InTriHub #2 (access-control, cut from 1-pager) |

---

## Remaining Gaps

1. **AutoPVT #2 and #3 (no metric, S=0)** — architectural correctness bullets with no quantification. Acceptable trade-off; the work is genuinely hard to quantify without fabricating numbers.

2. **SM #2 TOTP (no metric, X=1)** — hard to improve without a confirmed user/device count at time of migration. Leave as-is unless that number is available.

3. **SM #4 CI build time** — "over 80%" is solid but absolute before/after times (e.g., "from ~20 min to under 4 min") would push M from 2→2 (already maxed) and add interview talking points. Pending candidate input.

4. **Texim #1 (no script count)** — "full script library" is unquantified. Even a rough count would lift M. Pending candidate input.

5. **Texim #2 (weakest bullet, Total 7)** — needs a before-state for context. IMPORTANT: SQL Server QN was the original solution, not a replacement of polling. Confirm what (if anything) existed before before editing.

6. **Rosenxt #1 (frontend breadth)** — optional: replace "majority of the application" with explicit "5 of the application's core pages" (no new data needed).

7. **Rosenxt #2 (RBAC ownership, O=0)** — role names (3 permission levels) would add specificity. Pending candidate input.
