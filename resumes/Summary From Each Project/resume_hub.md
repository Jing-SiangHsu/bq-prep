# Hub Suite — Resume Content

**Role:** Full-Stack Engineer (sync-service architect + primary hub-ui owner)
**Stack:** Node.js (PM2 services), Firebase (Realtime Database + Cloud Functions), PostgreSQL, Vue 2, Vuex
**Scope:** 184 commits across 3 repos (hub-action-server, hub-cloud-function, hub-ui) over ~1 year, 10 GitHub issues
**What it is:** "Hub" (InTriHub) is a Firebase-backed portal for managing firmware/bootloader/MIB/product metadata across Intrising's hardware lines, used internally and by partners/vendors. Jason built the sync bridge connecting it to the AutoPVT system and owned most of the portal's frontend.

> Honest framing: still not AI/ML work, but two pieces here are unusually relevant: a real, **cost-driven evaluation of a search-vendor migration (Algolia) that he investigated deeply and then rejected** in favor of a cheaper data-model fix — exactly the kind of build-vs-buy, cost-aware reasoning AI infra teams do constantly — and a body of recent commits where he's visibly **directing an LLM coding assistant** (Claude) on real production refactors, reviewing and shaping its output rather than just accepting it.

---

## Resume Bullets

- **Designed and built a real-time, idempotent data-sync service** (Firebase Realtime Database → PostgreSQL) using RTDB's `child_added`/`child_changed`/`child_removed` listeners for both initial full-sync and live updates, with a generic upsert-on-conflict write path so replayed events converge safely without duplicate or stale records.
- **Investigated a cloud-cost regression end-to-end**: traced a database size increase from ~36MB to 64MB to a single oversized field being included in every full-collection read by every page that subscribed to it, regardless of whether the viewing role had permission to see that field.
- **Evaluated and rejected a full search-infrastructure migration (Firestore + Algolia)** after writing a complete technical assessment — page-by-page routing plan, secured-API-key design with per-role scoping, and an explicit effort/benefit/regression tradeoff table — then implemented a cheaper, smaller fix (splitting the oversized field into a lazily-loaded sibling node with tightened access rules) that solved the actual root cause without the migration's cost and complexity.
- **Diagnosed and fixed a tokenization/relevance bug class in a full-text search feature**, reasoning explicitly about token-and-sequence matching semantics (why version strings like "V5.01" were matching "V5.02.1007_01") and fixing it by adjusting separator/tokenization configuration rather than patching individual false-positive cases.
- **Found and eliminated duplicate real-time subscriptions** across multiple frontend pages (each independently opening its own live listener with near-identical role-based filtering logic), consolidating into a single shared, session-scoped subscription — cutting N redundant listeners down to one and catching a latent state-management API misuse in the process.
- **Designed a least-privilege access model** for the sync service's database role (connection-pool size capped explicitly below the role's connection limit) and for the portal's internal-use-only data (tightened realtime-database security rules after finding UI-only `v-if` hiding was the sole protection on sensitive fields, not the actual access-control layer).
- **Drove a breaking API schema change through a multi-consumer system safely**: split a single overloaded status enum into per-resource enums, communicated the breaking change via a public changelog comment ahead of rollout, and unified divergent response shapes across multiple endpoints behind one shared formatter.
- **Used an LLM coding assistant (Claude) as a collaborator on real production refactors** — search-utility extraction, dead-code removal, response-shape unification — while remaining the reviewer and technical owner of every change: writing the root-cause analysis, verifying the diff, and authoring the PR/issue documentation himself.

## Supporting talking points (for interviews)

**The Algolia decision (strongest story in this set):** Cloud costs were rising, and the obvious move — since search felt slow — was "migrate to a real search engine." Before doing that, I dug into where the size was actually going and found 92% of one collection's growth was a single internal-log text field that almost nobody needed to read. I still wrote out the full Algolia/Firestore migration plan seriously (index routing, secured-key design, cost model) so the option was fairly evaluated, not dismissed — and then I rejected it once the cheaper, more targeted fix (splitting that one field out and lazy-loading it) addressed the actual root cause. That's the build-vs-buy judgment call I want to highlight: I can produce a real implementation plan for the "exciting" option and still recommend against it when the data says otherwise.

**Search relevance debugging:** A search box was returning version-number false positives (querying "V5.01" matched "V5.02.1007_01"). The cause was in how the tokenizer split on punctuation — without forcing `.` and `_` into the separator set, the engine treated the query as "contains these tokens somewhere" rather than "contains this sequence." This is the same class of problem that shows up in any retrieval system — token boundaries and sequence-vs-bag-of-tokens matching directly determine relevance quality.

**Working with an AI coding assistant in production:** Several real refactors (search-mixin extraction, dead-code removal) were done with Claude as a co-author, visible directly in the commit trail. I didn't just accept the output — I wrote the root-cause and side-effects analysis for each change myself and reviewed the diffs before they shipped, the same way I would for a human collaborator's PR.

---

**Why this maps to AI engineering:** Evaluating and rejecting a vendor search migration after a rigorous cost/tradeoff writeup is the same skill as evaluating whether to buy a managed vector-DB/retrieval service versus building a targeted fix — and tokenization/relevance debugging is directly transferable to retrieval quality work in RAG systems. The AI-assisted-development pattern is also literal, current experience directing an LLM on real code, not a hypothetical.
