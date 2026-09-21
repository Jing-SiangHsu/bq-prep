# Interview Stories (CART Framework)

---

## Framework Note

**CART** (from UCLA GCS guide) is the structure used here for each story:

- **C — Context:** Your role, the project, what the situation was (1–2 sentences)
- **A — Action:** The steps you took; technical decisions and why (2–4 sentences)
- **R — Result:** Measurable impact on the team, product, or organization (1–2 sentences)
- **T — Takeaway:** What this demonstrates about you; why it's relevant to the role (1–2 sentences)

Each story ends with a **Probes** section with prepared answers to likely follow-up questions.

CART differs from STAR in two ways: "Context" is more concise than "Situation+Task," and "Takeaway" explicitly connects the story to the role. STAR leaves that inference to the interviewer. For behavioral questions, always end with the Takeaway so the interviewer doesn't have to guess why the story matters.

---

## AutoPVT

*One-time context to set at the start of any AutoPVT story:*
AutoPVT is an in-house concurrent test-automation platform I built as the **sole engineer** at Intrising Networks. Around 30 QA engineers use it simultaneously to test the company's industrial Ethernet switches. Stack: Go/gRPC backend, PostgreSQL for persistence, Vue 3 frontend, running across a cloud API and local per-device box instances.

---

### AP-1: Two-tier PostgreSQL locking model

**C — Context:**
As the sole engineer on AutoPVT, I designed the concurrency model for a platform where up to 30 engineers edit the same test objects simultaneously. Without any concurrency control, two engineers writing to the same test object at the same time would silently overwrite each other's changes, or worse, produce a corrupted object where half the state is from one write and half is from another. There was no existing strategy; I built this from scratch.

**A — Action:**
I identified two fundamentally different write types that needed different treatment. Structural writes (creating a test, editing the item list, finalizing, generating a report) must serialize because a second writer might invalidate the first. For these I used Postgres exclusive advisory locks keyed by test object ID, so two engineers on different objects never block each other, combined with a compare-and-bump revision check inside the lock: if the revision number has moved on since the client fetched the record, the write is rejected with "please reload." Result fills (engineers recording pass/fail for individual test items) are fine-grained: two engineers filling different items in the same test should never block each other. For these I used shared advisory locks with per-field merge semantics. Before writing code, I mapped out the full pairwise operation matrix (every write type against every other) to confirm the right lock type for each combination.

**R — Result:**
The platform has been in daily use by ~30 engineers with zero data corruption incidents from concurrent writes. Structural operations take turns; fills run in parallel. The granularity is correct.

**T — Takeaway:**
This shows I design concurrent systems from first principles, building the operation matrix before writing code, rather than applying a generic locking pattern and hoping it holds.

**Probes:**

- *"Why advisory locks instead of row-level locks?"*
  Row-level locks are tied to specific rows and released at transaction end. Advisory locks are application-level, keyed by integer (I use the test object ID). They map cleanly onto "exactly one structural edit per test object at a time" across an operation that touches multiple tables.

- *"What about the start-new-test race?"*
  Two engineers clicking "start" on the same box device within milliseconds would both succeed without protection, creating two active tests on one device. I handle this with a per-box-MAC advisory lock during create: check if an active test already exists for that box; if so, reject the second with "reload to join the existing test."

- *"How would you scale this beyond one Postgres instance?"*
  There are three separate scaling problems. First, the advisory locks. The lock table lives in memory on the single Postgres node, so if I add a second database node and two writes for the same test object land on different nodes, each node thinks it holds the lock. Neither knows about the other's. Both writes go through and the data gets corrupted. The fix is to split data across database nodes by test object ID, so all operations for object #42 always route to the same node and the lock comparison always happens in the same place. Second, the broadcast. Right now the server that processes the write also holds the WebSocket connections and broadcasts directly. With multiple app servers, the write server might not be the one holding the engineer's WebSocket connection, so it has no way to reach that client. Moving to a shared message channel like Kafka or Redis solves this: the write server publishes "test changed" to the channel, and every app server is subscribed to it and broadcasts to its own connected clients. Third, the catalog prefetch loads 2,500 items and is read-heavy. Writes like adding a new test item always go to the main database. But heavy reads can go to read-only copies of the database that stay in sync with the main one automatically. That offloads the read traffic so the main database stays focused on handling writes.

---

### AP-2: Commit-before-broadcast ordering

**C — Context:**
AutoPVT broadcasts real-time updates to all connected engineers when test state changes. The risk: if the broadcast fires before the transaction commits, clients see state from a transaction that subsequently rolled back. I call this a ghost update.

**A — Action:**
I enforced commit-before-broadcast as a strict structural invariant. The sequence is: (1) write state change and audit entry in a single Postgres transaction, (2) commit, (3) gRPC returns success acknowledgment, (4) only after receiving that ack does the sentry layer fire the WebSocket broadcast. The broadcast payload comes from the gRPC response message, which was only generated from committed data, so it's structurally impossible for the broadcast to carry uncommitted state. The audit entry is in the same transaction as the state write, not written after the fact, so the audit trail is always atomically consistent with live state.

**R — Result:**
The broadcast system has run without any dirty-state incidents. Every push clients receive is guaranteed committed, and the audit history is always complete.

**T — Takeaway:**
This demonstrates attention to distributed ordering properties. The commit-before-publish pattern is critical whenever a write path is coupled to a notification path, and getting it wrong is the kind of bug that's very hard to reproduce.

**Probes:**

- *"What if the WebSocket broadcast fails after the gRPC ack?"*
  The write is committed and correct; the client misses the push notification. This is at-most-once delivery, an accepted trade-off. On the next mutation or reconnect they'll see current state. For guaranteed delivery I'd add a message queue with consumer acknowledgment.

- *"Why put the audit write in the same transaction?"*
  A separate post-fact write leaves a window where data changed but the audit didn't record it, or the audit write fails entirely. Same transaction means both succeed or both fail atomically.

---

### AP-3: LISTEN/NOTIFY firmware event pipeline

**C — Context:**
AutoPVT's autotesting flow needs to notify machine clients the moment a new firmware build lands in the database so automated test runs can pick it up immediately. Polling was the existing approach.

**A — Action:**
The full pipeline has two distinct stages, and the design decision I made connects them cleanly.

In the first stage, firmware images are stored in Firebase Realtime Database. A separate Node.js service called hub-action-server, managed by PM2, runs independently of the main Go API, watching Firebase with RTDB listeners (child_added, child_changed, child_removed). When a build is added or updated in Firebase, hub-db-sync (a component of hub-action-server) writes that change into the Postgres firmwares table with INSERT or UPDATE. Critically, hub-action-server has no gRPC or HTTP connection to autopvt-api at all. The shared Postgres database is the only integration point between the two services.

That's exactly what made LISTEN/NOTIFY the right tool for the second stage. Because hub-action-server and autopvt-api only share a database, the API can't receive a direct call from the hub when a firmware arrives. Instead, a Postgres database trigger fires on INSERT or UPDATE to the firmwares table, but only when the md5 checksum actually changes. That guard prevents the trigger from firing spuriously when hub-db-sync rewrites a row with the same content. The trigger posts a NOTIFY on the firmware_event channel with a small payload: firmware ID, operation type, platform, and customization. The 8KB NOTIFY limit makes a minimal payload necessary, so the receiver re-fetches the full row on receipt. On the autopvt-api side, a firmwareEventHub runs a single pgx goroutine listening on that channel. When an event arrives, it fans out to all registered server-streaming gRPC clients, filtering by each client's platform and customization subscription server-side. I chose Postgres LISTEN/NOTIFY over Redis Pub/Sub to avoid introducing a new infrastructure dependency. Redis would have been a third service to deploy, monitor, and operate just to bridge two services that already shared a database.

**R — Result:**
Autotesting clients receive firmware availability notifications within seconds of a build landing in Firebase, with no polling overhead and no direct coupling between hub-action-server and autopvt-api.

**T — Takeaway:**
When two services already share a database, the database itself can be the integration bus. LISTEN/NOTIFY was the right fit here precisely because the architecture had no direct channel between hub and API, and adding one (a message queue, a webhook) would have increased coupling without adding capability.

**Probes:**

- *"What happens if a client disconnects mid-stream?"*
  Their subscriber is removed from the hub. Events during the disconnect are missed with no replay buffer. Acceptable for this use case; the client reconnects quickly. Guaranteed delivery would require a persistent queue.

- *"How does authentication work for server-streaming RPCs?"*
  The gRPC-gateway only handles unary RPCs, so server streaming requires a native gRPC connection to port 49000. Auth is handled in-handler at stream start: JWT from request metadata is validated before the client is added to the hub. I also added gRPC keepalive pings (30s interval) so the long-lived connection isn't reaped by proxy idle timeouts.

---

### AP-4: 3-5 second load delays to under 500ms

**C — Context:**
The test catalog has 2,500 items organized in a tree with embedded topology diagrams. Loading this view took 3-5 seconds, a visible daily friction for 30 engineers.

**A — Action:**
Three separate problems were stacking. First: the server-side catalog fetch was doing 4 separate SQL queries, one per tree level (categories, suites, groups, items), then assembling the tree with four nested loops in Go. The outermost loop iterated over categories; inside that it scanned all suites to find matches; inside that it scanned all groups; and at the innermost level it scanned all 2,500 items again for every group to find which ones belonged there. So for every group in the tree, the code walked the entire item list. That is O(categories x suites x groups x items) work just to assemble the tree, plus 4 database round trips. I rewrote it as a single SQL JOIN across all four tables, then assembled the tree in one pass using three hashmaps keyed by ID. Each row from the JOIN is processed in O(1): look up or create the category node, look up or create the suite node under it, look up or create the group node under that, append the item. 4 round trips became 1, and tree assembly became O(n). Second: even with a faster backend, the payload was still huge because every item included its full detail, purpose, procedure, expected behavior, and an embedded base64 topology diagram, pushing responses past the 4MB gRPC message size limit. I split the catalog fetch into a lightweight list (IDs, names, structure only) and a separate lazy fetch per item on click. Third: with a lightweight response, prefetching the whole tree into Vuex at login became cheap. By the time an engineer navigates to the item selection step, the tree is already in the store and navigation is instant.

**R — Result:**
Load time dropped from 3-5 seconds to under 500ms.

**T — Takeaway:**
Each fix enabled the next one: making the backend efficient made the payload small enough to prefetch, which made the UI instant. Layered performance problems need to be traced to the root, not patched at the surface.

**Probes:**

- *"Why not just add a database index?"*
  The bottleneck was not query speed on individual tables, it was 4 separate round trips and the O(categories x suites x groups x items) in-memory assembly. An index on each table would still require 4 queries. The JOIN moves all the work into one database operation with its own optimized execution plan.

- *"Why prefetch at login instead of on first navigation?"*
  Login is a natural initialization boundary. Prefetching on first step navigation delays the engineer exactly when they want to act. By the time they have set up a test and navigated to item selection, the prefetch is almost certainly complete.

- *"Why split the fetch rather than just compressing the payload?"*
  Compression reduces transfer size but the server still has to serialize 2,500 full records and the client still deserializes them. The lazy detail fetch means that work never happens at all for items the engineer does not click, which is most of them in any session.

---

## Switch Management Interface (SMI)

*One-time context:*
The Switch Management Interface is the web UI and backend gateway for Intrising's industrial Ethernet switches, used in factory floors and critical infrastructure. I owned the Go gateway layer; core firmware was written in C by other engineers. I worked across four product lines.

---

### SMI-1: IEC 62443-4-2 SL3 — session limits + TOTP

**C — Context:**
Intrising's switch products needed IEC 62443-4-2 Security Level 3 compliance, an industrial cybersecurity standard. I led the session control and authentication requirements (CR 1.1: TOTP-based MFA; CR 2.5: concurrent session limits) across four product lines.

**A — Action:**
CR 2.5 requires per-user per-interface concurrent session limits across all login paths: web, CLI, and Telnet. The web path was natural: my gateway already controls JWT-based sessions, so I added a per-user per-interface counter and enforced the limit at JWT issuance. For CLI and Telnet, those authenticate via PAM in the core firmware (IS-Roger's code). I designed a CheckSessionLimit RPC in the gateway's InternalService: IS-Roger's auth code calls this after a successful PAM login to both record the new session and check if the limit is exceeded. The gateway becomes the single authoritative session store across all three interfaces. I also implemented proactive termination: when an admin disables a user's interface access, the gateway immediately invalidates existing JWTs rather than waiting for timeout. This works because the gateway uses a stateful JWT pattern: every issued JWT gets a `jti` claim (a random ID), and the gateway keeps an in-memory map of all active sessions keyed by `jti`. Every authenticated request calls `CheckToken`, which only passes if that `jti` exists in the map with state active. To invalidate, I call `RuinToken`, which flips the state flag for that entry. The JWT signature is still cryptographically valid, but the state check rejects it on the next request. The tradeoff is that the session map lives only in memory, so a gateway restart logs everyone out.

**R — Result:**
Four product lines achieved IEC 62443-4-2 SL3 compliance on the session control and authentication requirements. Session limits are enforced in real time across all login paths from one source of truth.

**T — Takeaway:**
Making the gateway the session authority, rather than distributing session tracking across each firmware component, is the kind of architectural decision that prevents future inconsistencies as new login paths get added.

**Probes:**

- *"What was your scope within IEC 62443?"*
  CR 1.1 (TOTP MFA) and CR 2.5 (session limits). Other requirements (CR 1.7 password strength, CR 2.1 RBAC, CR 3.1 TLS, CR 6.1 audit logging) were already implemented or handled by other engineers.

- *"How do you handle timeout vs. explicit logout?"*
  Both go through `RuinToken`. Explicit logout sets state to logged-out immediately. Expiry is lazy: the state is flipped to expired the next time that token hits the auth middleware and fails the timeout check. Admin session termination sets state to session-terminated, same immediate effect. All three paths are just different state values on the same in-memory `tokenDB` entry.

---

### SMI-2: Two-RPC protocol, PAM conversation handler, backward compatibility

**C — Context:**
Adding TOTP MFA to the web login required bridging two incompatible models: PAM's interactive challenge-response protocol and HTTP's stateless request-response. The hard constraint: all existing single-factor accounts must continue to work without any changes.

**A — Action:**
The core challenge is that PAM's interactive conversation model (send prompt, receive response, send next prompt, receive response, then decide) cannot be mapped onto a single stateless HTTP POST. There is no way to pause a PAM conversation mid-flight and resume it when a second HTTP request arrives. PAM knows per-user whether TOTP is required: that config lives in PAM's own per-user system, not the gateway. So PAM only sends a TOTP prompt to users who have TOTP configured; non-MFA users get only the password prompt and their login succeeds in a single PAM run. The problem is that for MFA users, PAM sends the TOTP prompt during that same PAM run, immediately after the password, with no way for the gateway to pause and ask the browser for a TOTP code. I solved this with a two-RPC login protocol backed by a CGO-bridged PAM conversation handler. The conversation handler intercepts each individual PAM prompt in Go and inspects its text. First RPC: password only. For non-MFA users, only the password prompt appears, PAM succeeds, and the gateway issues a JWT. For MFA users, the password prompt appears and succeeds, then the TOTP prompt appears. The handler responds with a sentinel value it knows PAM will reject. PAM fails at the TOTP step. The handler sees exactly which step failed and surfaces "MFA required" to the gateway, which returns it to the browser as a signal to show the TOTP entry page. Second RPC: password + real TOTP, used only when the first call returned MFA-required. The handler provides the real code to the TOTP prompt, PAM succeeds, gateway issues JWT. One operational requirement: TOTP is time-based, so the switch must be configured as an NTP client. A time-skewed device will reject valid TOTP codes.

**R — Result:**
TOTP MFA shipped across all login interfaces. All existing single-factor accounts continued to work with no user-side changes.

**T — Takeaway:**
This shows I can bridge authentication protocols that weren't designed to work together, a common challenge when adding auth features to existing systems, while preserving backward compatibility throughout.

**Probes:**

- *"Why CGO instead of shelling out to the existing PAMLogin binary?"*
  The binary returned only a final success/fail exit code, with no visibility into which step failed. With the CGO conversation handler, each PAM prompt is intercepted individually in Go. I can detect exactly where the failure occurred: wrong password vs. wrong TOTP vs. account disabled. That granularity is required for the two-RPC protocol to work correctly.

- *"Walk me through the full MFA login flow."*
  Step 1: browser sends username + password. Gateway calls PAM with just the password. Conversation handler intercepts prompts. PAM validates password (correct), then sends a TOTP prompt. Handler responds with a sentinel value it knows PAM will reject. PAM fails at the TOTP step. Handler reports "TOTP prompt reached, failed" to the gateway. Gateway returns MFA-required signal. Browser shows TOTP entry page. Step 2: user enters 6-digit code, browser sends username + password + TOTP. Gateway calls PAM again. Handler provides real TOTP to the TOTP prompt. PAM validates both, succeeds. Gateway issues JWT.

- *"How is the TOTP secret stored?"*
  AES-256-GCM encrypted in the user's gateway record. Users can enable MFA on their own account; only the admin can disable someone else's or reset their secret. Setup generates an otpauth:// URI for QR code scanning.

---

### SMI-3: Config-save bug — silent credential corruption

**C — Context:**
A bug report: after saving and restoring the device configuration, the device couldn't log in. The credentials were corrupted. Nothing looked wrong during normal operation. The bug was completely invisible until the device rebooted from a saved config.

**A — Action:**
To understand the bug I had to trace where in the stack masking was being applied. The system has a Protocol Service layer in the firmware below the gateway — this layer handles the actual config read/write for each domain (SNMP, user permissions, system, log, main config). Masking had been placed there: the Protocol Service applied `*****` substitution on all GET responses. The problem is that the device's config-save procedure reads from that same Protocol Service layer. So when the device saved its running config to disk, it read the already-masked values and wrote `*****` to the config file. On reboot the device loaded `*****` as the real credential. It was completely invisible during normal operation because everything looked correct on screen — the corruption only surfaced when you restored from that saved config. This affected five domains because each had its own Protocol Service handler applying the same wrong pattern. The fix was to move masking out of the Protocol Service layer entirely and into the gateway egress layer: each GET handler in the gateway now applies masking on the way out to the browser, so the Protocol Service always returns real values to any caller. On the SET side, when the browser echoes back a placeholder for an unchanged field, the gateway detects it, fetches the real value from the Protocol Service, and substitutes it before forwarding — so IS-Roger always receives real values. In the same subsystem I found a related SSH key-matching bug: the code identified public keys by index position, so if an admin reordered keys, the wrong key was matched. I switched it to fingerprint-based matching.

**R — Result:**
Five configuration domains and two silent failure modes closed in the same subsystem. Saved and restored device configurations are now reliable.

**T — Takeaway:**
Silent data corruption bugs that only manifest under infrequent operations are the hardest class to catch in testing, and the most dangerous in production. Tracing the full data-flow pipeline to find where "display value" and "persistence value" had been conflated required reading the code end-to-end, not just the surface symptom.

**Probes:**

- *"How did this get through normal testing?"*
  Normal usage never triggers it. The device runs fine; the config looks correct on screen. Only when you export and restore (an infrequent operation) does the device load the placeholder strings as real credentials. It wasn't caught because the test cycle almost never included restore-from-config.

- *"How did you test the fix?"*
  Unit tests for the masking and restore functions across all five domains, plus integration test of the full cycle: configure → export → restore from config → verify login succeeds.

---

### SMI-4: CI build time reduction

**C — Context:**
The SMI CI pipeline had a correctness problem: the build used Gulp's built-in `parallel()` to run worker processes, but Gulp's `parallel()` does not propagate worker exit codes — if any worker exited non-zero, the remaining workers kept running and the overall build still reported green. Broken builds were invisible. Additionally, build time was long because the pipeline compiled SVG diagrams for all switch models even when most models were not needed for a given product.

**A — Action:**
I addressed correctness first. I wrote `gulp-multi-process-fail-fast.js`, a Node.js script that spawns all build worker processes, monitors their exit codes, and on the first non-zero exit kills every remaining worker and exits with failure, replacing Gulp's `parallel()`. CI now surfaces any build failure immediately. For performance: I introduced an `included_svg_models.txt` allowlist per product. Only the listed models get compiled for that product's build; everything else is skipped. SVG compilation was the dominant build cost, so cutting the compiled set was the high-leverage change. Both changes shipped across four product-line repos.

**R — Result:**
Build time dropped over 80%. CI now correctly fails on any worker error rather than silently passing.

**T — Takeaway:**
The correctness fix mattered more than the performance fix. A CI pipeline that silently passes broken builds is worse than a slow one. Silent failures erode confidence in CI and hide real problems.

**Probes:**

- *"How did you verify the new orchestrator produced identical output for successful builds?"*
  Ran both implementations on the same input and diffed the output artifacts. The new orchestrator only changes failure propagation, not the build logic itself. Each subprocess is unchanged.

- *"Why an allowlist file rather than per-product build config?"*
  Simpler to review in PRs; adding a new device model is one line. Distributing the list across multiple build configs means updating multiple repos when a new model launches.

---

## InTriHub

*One-time context:*
InTriHub is an internal web portal for Intrising managing firmware builds, product catalogs, and license distribution. Node.js backend, Firebase Realtime Database, Google Cloud Functions, Vue 2 frontend.

---

### IH-1: Firebase cost spike — $300/month to free tier

**C — Context:**
Starting around July 2025, InTriHub's Firebase bill spiked to $300/month. My supervisor flagged it. Firebase bills for data egress, so something was downloading a large amount of data frequently — but nothing had obviously changed in the code around that time, and before July the project had zero Firebase cost.

**A — Action:**
My first step was to trace the git history to see if anything had changed around the time costs started rising — I even used Claude Code to help search through commits — but found nothing suspicious. That pushed my initial hypothesis toward a structural Firebase limitation: RTDB has no compound queries, sorting, or pagination, so every page was fetching the full firmware collection and filtering client-side. That seemed like the likely cause of the egress. But I had a nagging sense that something must have actually changed, because before July we had zero Firebase cost on this project — if it were just RTDB's query model, we would have been paying from day one.

So I explored migrating to Firestore for structured querying and Algolia for full-text search. When I started importing the firmware collection into Algolia to test it, I immediately hit Algolia's record size limit: over 70 documents exceeded 100KB, with several approaching 1MB. That's when the actual root cause became visible. I investigated the data and found an `internalLog` field embedded in each firmware record — it stored the full version changelog going back to initial release — which had been growing steadily as more firmware versions were released. By the time of the analysis it was 36MB; by the fix it had grown to 64MB, with `internalLog` alone accounting for 59MB of that. The cost spike in July wasn't random: it was the point where the collection crossed a size threshold that made every full fetch expensive. Every page load was downloading 60+ MB of changelog content that most users couldn't even view — visibility was gated by a UI `v-if` only, not at the RTDB rule level.

I wrote a full evaluation of the Firestore + Algolia migration: scope, effort, operational cost, failure modes. It came out to 8–11 engineer-weeks with a 2–3× increase in operational surface — two databases, five Algolia sync pipelines, a secured-key mint function, a new auth path. The decision was to stay on RTDB and fix the data model instead. The problem wasn't Firebase's query capability; it was a single overloaded field.

I split `internalLog` into a lazy sibling node `firmwareInternalLog/{pt,mt}/<id>`, keeping only a `hasInternalLog` boolean on the firmware record. The log is now fetched on demand with a `.once('value')` call only when an admin explicitly expands a row, and only if the flag is set. I also tightened the RTDB rule so admin-only is enforced at the database layer rather than relying on UI hiding. Existing records were migrated with a one-off two-phase admin script (non-destructive copy pass, then a separate destructive strip pass, batched to stay under Firebase's 4MB/500-entry update limit), not a live Cloud Function, with the old path kept readable during the rollout.

Separately I found a listener pattern problem: three pages (Firmware, Products, and Viewer) each attached their own `.on('value')` live listener to the full firmware node when the user navigated there. Each `.on()` call triggers an immediate download of the full collection snapshot — so navigating to any of these pages cost a full re-fetch of the entire 70MB, up to three times in a single session. None of the three called `.off()` in a cleanup hook, so old listeners also accumulated silently across navigations. I replaced all three per-page listeners with a single global `subscribeFirmware()` in `src/index.js` that runs once after login and tears down on logout via the `onAuthStateChanged` hook. All three pages now read from a shared Vuex `firmwareList` state.

**R — Result:**
Cost went from $300/month to within the free tier, with no RTDB line item. The firmware collection shrank from 70 MB to 5 MB (93% reduction).

**T — Takeaway:**
Cost problems are usually data model or access pattern problems. I resisted migrating platforms and traced the root cause instead, a far smaller change that fully solved it with no new infrastructure risk.

**Probes:**

- *"Why not Firestore or Algolia?"*
  I actually started down that path — Algolia for search, Firestore for structured querying. It was while importing firmware data into Algolia that I hit the record size limit and discovered `internalLog` was the real culprit. Once I knew the root cause was a single overloaded field rather than a query capability gap, I wrote a full migration evaluation: 8–11 engineer-weeks, 2–3× operational surface, new infrastructure and billing relationships. Splitting one field and fixing the listener pattern was a fraction of that cost with no new infrastructure risk.

- *"How did you handle the migration without downtime?"*
  A two-phase admin script: copy internalLog to `firmwareInternalLog/{pt,mt}/<id>` first, verify, then strip the field from the original record in a separate pass, batched under Firebase's per-update payload limit. I kept the old path readable during migration until both the migration and UI update deployed. Transparent to users.

---

## Adaptive Ad Recommender

*One-time context to set at the start of any Adaptive Ad Recommender story:*
Adaptive Ad Recommender is a personal project I've been building since July 2026, a two-sided ad-recommendation system where advertisers submit campaigns that go through LLM-based policy review, and the system serves the most relevant eligible campaign to a user based on their profile, learning from feedback over time. Stack: FastAPI/Python backend, Postgres as the source of truth, Kafka+Debezium for change-data-capture into Pinecone (vector search), Redis for job queueing and rate limiting, OpenAI API for the LLM surfaces, Google OIDC for auth. Solo project, no team.

---

### AAR-1: Kafka + Debezium CDC pipeline

**C — Context:**
Campaign eligibility (status, budget) lives in Postgres, but serving needs to filter and rank eligible campaigns via Pinecone's vector search. Keeping those two systems in sync with a naive dual-write (write to Postgres, then separately write to Pinecone) means two independent points of failure: if the Postgres write succeeds but the Pinecone write silently fails, Pinecone quietly drifts out of sync, and could keep serving a campaign that's actually already budget-exhausted.

**A — Action:**
I built a Kafka (KRaft mode, no separate ZooKeeper cluster) and Debezium change-data-capture pipeline instead: the app only ever writes to Postgres, Debezium watches Postgres's own change log, and a Kafka consumer propagates every change into Pinecone. I chose Kafka specifically for its per-partition ordering guarantee, partitioned by campaign_id, because eligibility updates have to apply in the order they happened. Two budget-debit events on the same campaign crossing the exhaustion threshold, applied out of order, could leave a depleted campaign servable. Redis/RQ (already used for campaign review) has no such ordering guarantee, so it wasn't a fit for this specific job even though it's a perfectly good task queue elsewhere in the same app. While building this I also found and fixed two smaller issues with real data behind them: a Pinecone client object was being reconstructed on every single call instead of cached, a one-line fix that measured a 2.4-2.9x speedup on non-embedding writes; and a defensive oversampling multiplier on retrieval turned out to be unnecessary, something I only trusted after my first stress test (an unrealistic 49% catalog churn burst at the wrong batch size) gave a misleading answer, and I reran it at a realistic churn rate before removing the multiplier.

**R — Result:**
Postgres stays the single source of truth for eligibility; Pinecone converges from the CDC log rather than a second manual write, removing the class of bug where the two silently disagree.

**T — Takeaway:**
This shows I pick infrastructure for a specific correctness property it provides, not because it's popular, and that I don't trust a first benchmark result without checking whether the test itself was representative.

**Probes:**

- *"Why not just add a database index or optimize the dual write?"*
  The problem isn't write speed, it's that two independent writes have two independent chances to fail, and there's no way to guarantee "both succeeded or neither did" across two different databases without a distributed transaction, which neither Postgres nor Pinecone supports natively. CDC sidesteps this entirely: there's only ever one write, to Postgres, and Pinecone converges from a durable, ordered log of what changed.

- *"What if the Kafka consumer falls behind?"*
  There's a dead-letter topic for malformed events so a single bad message doesn't block the whole partition, and consumer-group lag is self-logged so a growing backlog is visible rather than silent.

- *"Walk me through why the first oversample test was misleading."*
  I tested at 49% catalog churn in one burst, an unrealistic spike, at the wrong batch size for how the app actually queries. It looked like it justified keeping a 3x oversample safety margin. Rerunning at realistic churn and the real production batch size showed the safety margin caught nothing, zero trims across the test run, so I removed it rather than keep "free insurance" that wasn't actually free or necessary.

---

### AAR-2: Postgres advisory-lock leak from ORM connection churn

**C — Context:**
Two concurrent reactions from the same user (liking two different ads back to back) could both read the same starting profile vector before either write landed, so the second write would silently clobber the first's update. Pinecone has no atomic read-modify-write primitive, so I needed external coordination.

**A — Action:**
I used a Postgres session-level advisory lock keyed on user_id to serialize the fetch-and-write. That fixed the original race, verified with a real two-thread test that I confirmed actually caught the race by temporarily removing the lock and watching the test fail. But the fix itself had a bug: the locked region contained a database commit partway through, and SQLAlchemy's ORM session releases its connection back to the pool on commit, checking out a connection, not necessarily the same one, for the next statement. Postgres advisory-lock release is scoped to the specific connection that acquired it, so if the unlock call landed on a different connection than the one that acquired the lock, it silently no-opped, and the original connection went back into the pool still holding the lock, invisibly. I caught this because a full test-suite run reproducibly hung forever on a second call for the same user, even against a provably clean Postgres instance I'd confirmed had zero advisory locks right before the run. Checking pg_locks showed the exact signature: one idle connection last-queried COMMIT, still holding the lock, blocking the next acquire. I fixed it with a dedicated context manager that acquires and releases the lock on one connection held open for the entire critical section, independent of whatever the ORM does with its own connection pool in between.

**R — Result:**
The full test suite went from hanging indefinitely to completing all 101 tests in 23 seconds, with zero locks left behind afterward, verified both in the test suite and live through the real API.

**T — Takeaway:**
This is the kind of bug that doesn't throw an error, it silently leaks a resource, so I had to reason from a hang and a database system table back to the exact mechanism, not from a stack trace. It also reinforced a lesson about fixing a fix: adding a lock doesn't finish the job if the surrounding framework can move state out from under you.

**Probes:**

- *"How did you know it was a connection issue and not a real deadlock?"*
  I checked pg_locks directly and confirmed the lock was still held by a connection whose last query was COMMIT, not an active query. A real deadlock would show two connections each waiting on the other; this showed one connection that had already finished its work but never actually released the lock.

- *"Why not just use a lock with a timeout as a safety net?"*
  A timeout would have masked the bug rather than fixed it. Requests would eventually stop hanging, but they'd fail or retry instead of succeeding, and the underlying leak would still be there, waiting to cause a different symptom somewhere else.

- *"Would this bug happen with row-level locks instead of advisory locks?"*
  No, row-level locks are tied to the transaction and release automatically on commit or rollback, no matter which connection issues the commit. This bug is specific to advisory locks because Postgres deliberately makes them independent of transactions, which is exactly the property that made them the right tool for the original race, but also what made this failure mode possible.

---

### AAR-3: Google OIDC auth

**C — Context:**
The app started with zero authentication, every endpoint trusted a caller-supplied user_id with no verification, which meant any request could impersonate any user or approve/reject any campaign with no access control.

**A — Action:**
I added Google OIDC login: the frontend gets a Google-signed ID token via Google Identity Services, and the backend verifies it against Google's public keys and confirms the aud claim matches this app specifically. After verifying identity, the backend issues its own short-lived JWT access token rather than trusting Google's token directly, so the app controls its own session lifetime and role claims. Alongside that, a longer-lived refresh token is tracked server-side in Redis, since a signed JWT can't be un-issued once it exists, Redis is what actually makes logout or rotation revoke something real. The access token lives in localStorage since it's short-lived and low-risk if stolen; the refresh token lives in an httpOnly cookie, out of reach of client-side JavaScript, since it's longer-lived and more sensitive.

**R — Result:**
Every endpoint now requires a verified identity, and roles (end_user, advertiser, moderator) are checked separately from authentication, so "who is this" and "what can they do" are two distinct, composable checks rather than one conflated one.

**T — Takeaway:**
Authentication and authorization are genuinely different problems, and keeping them as separate layers (OIDC answers identity, a role column answers permission) made the system easier to reason about as new endpoints got added.

**Probes:**

- *"Why not just use Google's token directly instead of issuing your own JWT?"*
  Google's token is scoped to Google's own session semantics, not this app's. Issuing my own JWT means I control expiry, what claims are on it, and what happens on logout, independent of anything Google does on their side.

- *"Why split access and refresh tokens instead of one long-lived token?"*
  A single long-lived token that's stolen is a long-lived vulnerability with no way to revoke it short of rotating the signing secret for everyone. Splitting them means the thing that's actually dangerous if leaked (the refresh token) is the one kept out of JavaScript's reach and tracked server-side so it can be individually revoked.

---

### AAR-4: Adversarial prompt-injection testing

**C — Context:**
I'd built two LLM-facing surfaces, a campaign policy reviewer and an onboarding checkpoint judge, both taking attacker-controlled text as input (an advertiser's ad copy, a user's chat message). A mocked test proves nothing about whether the real model actually resists a real attack, so I built a real, unmocked adversarial test suite to find out.

**A — Action:**
I wrote crafted injection attempts against both surfaces: fake "SYSTEM OVERRIDE" instructions, claims of pre-approval, fabricated conversation history claiming a step already happened. The first run found real, reproducible vulnerabilities on both surfaces. For the policy reviewer, two different injections (forced approval, suppressed exclusions) succeeded non-deterministically; adding explicit instruction-vs-data framing to the system prompt fixed both, clean across every follow-up run. For the onboarding checkpoint judge, the same class of fix did not work: an injected override on vague input and a fabricated fake history turn both still succeeded every single run, even after adding the equivalent framing. Since prompt hardening alone wasn't sufficient there, I added a real deterministic backstop instead of chasing more prompt wording: a length floor before any interest summary gets embedded and persisted as a real profile vector, closing the concrete harm even though the judge's raw output is still technically manipulable. One of the two known issues on that surface still has no fix, deliberately: fabricated history can make the judge think onboarding already finished, but fixing it properly would require adding server-side session state to verify a round actually happened, which conflicts with the app's intentionally stateless design. I left it open and tracked rather than rush a fragile patch.

**R — Result:**
Two of the two tested policy-review injection paths are fully closed. One onboarding vulnerability has a real deterministic backstop closing its concrete harm. One remains open by deliberate choice, tracked with an explicit reason rather than hidden.

**T — Takeaway:**
Testing against the real model, not a mock, found vulnerabilities that would have looked fine in unit tests with mocked LLM responses. And knowing when not to patch something, because the proper fix has a real architectural cost, is as much a signal of engineering judgment as finding and fixing the bug in the first place.

**Probes:**

- *"How do you know your test suite itself isn't unreliable, since it's calling a real LLM?"*
  Where the check is a literal field value (did the outcome flip to approved, is a category present in a list), the assertion is deterministic, no LLM judging involved. Only the two checks that are inherently semantic, does a reply leak the system prompt, is a summary contaminated by injected content, route through a small LLM-as-judge helper, and that helper is itself hardened against the same injected text it's evaluating.

- *"Is the campaign reviewer an actual agent?"*
  Only in the tool-calling loop added afterward, where it can query a real Postgres table for an advertiser's history on borderline cases, that's custom code my own process executes across multiple turns. The earlier web-search-only version is not an agent by any meaningful definition, since OpenAI runs that tool server-side within a single call, there's no loop my own code drives.

- *"Why leave the fabricated-history vulnerability open instead of fixing it?"*
  Because the real fix isn't a quick patch, it requires adding server-side session state to verify a prior round genuinely happened, and this app is deliberately stateless today. Adding that state is an architecture decision with real trade-offs, not something to bolt on hastily just to close a test. The actual harm from leaving it open is also low severity, onboarding ends a little early with friendly messaging, not any real data corruption, so the risk didn't justify rushing a fragile fix.

---

## Texim Europe

*One-time context:*
Summer 2023, internship at Texim Europe, a Dutch electronics distributor. Their ETL pipeline ran on SQL Server 2000 DTS, which was end of support and unable to receive security patches.

---

### TX-1: ETL platform replacement

**C — Context:**
Texim's ETL pipeline ran on SQL Server 2000 DTS — a data transformation service that Microsoft discontinued with SQL Server 2005 and replaced with SSIS. Texim had been running it for years past end-of-life, which meant no security patches and no vendor support. The business constraint driving the replacement was security compliance, not functionality: the tool still worked, but it was an unpatched surface. The hard migration constraint from the start: the replacement must run Texim's full existing VBScript ETL library without any script rewrites. The entire pipeline was in VBScript, and rewriting it was not in scope.

**A — Action:**
The VBScript constraint drove the architecture. Rather than embedding a VBScript interpreter, I routed execution through Windows Script Host (wscript.exe) — the same engine that had always run these scripts. That meant existing scripts ran exactly as before with no compatibility work at all. The replacement is a C# WinForms application with three components:

The editor GUI lets engineers author and organize packages: list all scripts, edit them in-tool, and run individual scripts manually. The scheduler handles daily, weekly, and monthly recurrence with email alerts on completion or failure. Schedule state persists in a SQL Server table so configuration survives reboots. The third component is PackageRunService, a multithreaded Windows service that runs in the background and executes scheduled scripts concurrently — the key requirement here was that scripts must run on schedule even when no one has the UI open, which rules out executing directly from the WinForms process.

**R — Result:**
Texim's full ETL script library migrated to the new platform without any script rewrites. They replaced an end-of-life, unpatched system with a maintainable C# application on a supported stack, with no disruption to the existing pipeline.

**T — Takeaway:**
Greenfield replacement of a legacy system succeeds or fails on the migration constraint. "No script rewrites" wasn't a nice-to-have — it was the condition under which the replacement was actually usable. The wscript.exe routing was the key insight: instead of trying to replicate VBScript execution, I reused the existing execution engine entirely.

**Probes:**

- *"Why not migrate to SSIS, Microsoft's official DTS replacement?"*
  SSIS is the right answer for teams with SQL Server expertise and budget for it. For Texim, the requirement was zero script rewrites and fast delivery. SSIS would have required migrating every DTS package to a new format. Routing through wscript.exe meant existing scripts ran unchanged from day one.

- *"Why WinForms and not a web app?"*
  Internal tool, all users on Windows, the Windows service model for background execution fits naturally with a Windows-native app, and WinForms was the fastest path for a summer internship timeline.

- *"How did you handle script errors and output?"*
  wscript.exe returns an exit code — non-zero means failure. Output is captured from stdout/stderr. The scheduler logs both exit code and output per run, and sends an email alert on failure.

---

### TX-2: SQL Server Query Notifications

**C — Context:**
After building the ETL replacement, engineers could run and schedule scripts, but had no way to know when a running job finished except by refreshing the UI manually. The tool needed real-time status updates. I considered three options: polling on a timer (simple but adds load and has latency proportional to the interval), SignalR (push-based but requires a web server, unnecessary infrastructure for a desktop tool), and SQL Server Query Notifications — a push mechanism built on Service Broker that's already part of SQL Server.

**A — Action:**
SQL Server Query Notifications work by registering a query with SQL Server: the server monitors whether the result set of that query would change, and fires a notification to the .NET application via Service Broker when it does. I implemented three SqlDependency subscriptions: one on the running packages table (to detect when a job starts or finishes), one on the scheduled packages table (to reflect schedule changes in the UI), and one on the packages list (to detect additions or deletions). The subscription is single-fire by design — after it fires, it expires. The OnChange handler re-registers the dependency before re-querying, so there's no window between registration and query where an update could be missed.

**R — Result:**
Engineers see immediate status updates when jobs complete, with no polling overhead and no artificial latency. The UI updates the moment SQL Server commits the status change.

**T — Takeaway:**
This is structurally the same push-based pattern as Postgres LISTEN/NOTIFY — the database monitors state server-side and pushes a notification to the application, which then re-queries for current state. The specific mechanism differs but the architecture is identical. It's a pattern worth reaching for whenever you need database-driven real-time updates without adding a message broker.

**Probes:**

- *"What are SqlDependency's constraints?"*
  The subscribed query must use schema-qualified table names, no aggregates, no subqueries, no SELECT *. Service Broker must be enabled on the database. It fires on any change to the result set — not just the change you care about — so the OnChange handler always discards the notification payload and re-queries for current state. That's fine; the notification is a wake-up signal, not a data carrier.

- *"Why not just poll on a timer?"*
  Polling works but has two costs: unnecessary database load when nothing has changed, and latency equal to the polling interval. For a tool that engineers watch while waiting for a job to finish, even 5-second polling feels sluggish. Query Notifications fire within milliseconds of the commit.

---

## NLP Project

*One-time context:*
Team project of four at the University of Twente, November 2023. Goal: given a research paper abstract and subject areas, recommend the top 5 journals to submit to. I was the ML engineer: designed the architecture, trained the models, built the ensemble.

---

### NLP-1: 96.9% top-5 accuracy, 27-model SciBERT ensemble

**C — Context:**
As the ML engineer on a four-person team, I was responsible for designing and training the model architecture for a journal recommendation system covering 378 journals across 26 subject areas.

**A — Action:**
The core architectural decision: one model over all 378 journals, or 26 domain-specific models? I chose 26 separate models. "Which chemistry journal?" and "which math journal?" are fundamentally different classification problems. A single model would have to learn unrelated decision boundaries simultaneously and would likely underperform on all of them. The architecture: 26 SciBERT-based abstract classifiers, one per subject area, plus 1 subject area classifier operating on subject area label strings. SciBERT is Allen AI's transformer pretrained on 1.14 million scientific papers. I used it for feature extraction only (weights frozen, not fine-tuned) because with 35,370 training articles across 26 domains, averaging about 1,350 per domain, fine-tuning a 110M-parameter model would overfit badly. I feed each abstract through the frozen SciBERT transformer, extract the [CLS] token embedding (a 768-dimensional dense summary vector), and pass it to a small softmax classifier I actually train. The subject area classifier uses CountVectorizer + softmax regression: its inputs are short label strings, not abstracts, so contextual embeddings aren't needed. The ensemble is accuracy-weighted and dynamic: when a user selects N subject areas, N+1 models run, weights proportional to each model's validation accuracy.

**R — Result:**
96.9% top-5 accuracy on a held-out test set of 3,000 abstracts across 378 journals. Top-1: 76.8%, top-3: 93.5%. Random baseline: 0.26%.

**T — Takeaway:**
The domain-specific architecture was the decisive design choice. Brute-forcing one model over 378 classes with limited data would have underperformed significantly. Knowing when to decompose a problem is more valuable than knowing which model to use.

**Probes:**

- *"What's the [CLS] token?"*
  In BERT-style models, a special [CLS] token is prepended to the input with no real-word meaning. Because it carries no semantic content of its own, the model accumulates a representation of the full input in that position over the attention layers. After the forward pass, its 768-dimensional vector is a dense summary of the whole abstract. That's what I pass to the downstream classifier.

- *"Why freeze SciBERT rather than fine-tune?"*
  Three reasons: dataset size (average ~1,350 samples per domain for 110M parameters, far too small), compute (no GPU budget for extended fine-tuning), and catastrophic forgetting risk. Frozen feature extraction is fast, cheap, and avoids all three failure modes. For this dataset size it was clearly the right call.

- *"Why accuracy-weighted rather than equal weights?"*
  Equal weighting over-relies on classifiers for small subject areas (NURS: 2 journals, near-100% accuracy) and under-relies on classifiers for large, difficult areas (BIOC: 73 journals, lower accuracy). Accuracy-proportional weighting lets more reliable classifiers contribute more while still incorporating domain-specific signals from all active models.

- *"How would you scale to 1 million users?"*
  The SciBERT embedding step is the bottleneck, O(seq²) attention on CPU in our setup. At scale: cache embeddings for abstracts already seen, batch inference across users, deploy SciBERT on GPU with TorchServe or a managed endpoint, and separate the feature extraction tier from the classification tier so they scale independently.

---

## Behavioral Stories (BQ — Not on Resume)

*These are pure behavioral stories, not tied to a specific resume bullet, but grounded in real incidents. Use for questions like: "Tell me about a conflict with a teammate," "Tell me about a time you made a mistake," "Tell me about a time you took ownership."*

---

### BQ-1: Conflict Resolution — Missing API (getCloudToken)

Use for: Earn Trust, Ownership, Conflict with a teammate

**C — Context:**
Near the end of my time as de-facto lead on AutoPVT, my manager was taking over the project and working to finalize it for production cloud deployment. He discovered that getCloudToken, a critical API the core firmware calls to obtain a cloud-scoped JWT, was missing from the implementation. Without it, field devices cannot authenticate to the cloud.

**A — Action:**
I was confident the API had been implemented previously, because the demo a month earlier could not have functioned without it. I had a Slack thread showing I had asked IS-Roger to implement it and he had replied "Done." When I raised this, Roger's initial response was "I don't remember implementing that, maybe you asked someone else" and "Isn't this part of your side?" I was frustrated. The protobuf definition for the method was in Roger's own codebase, visible in the git history. But I recognized that winning the attribution argument wasn't my job; unblocking the project was. I set aside the blame question and re-explained clearly what getCloudToken does, when it's called in the flow, and what it returns, then asked Roger to implement it again. In parallel I informed my manager that I had the Slack evidence, framed informally and without escalating it as a formal dispute, so the record was clear without creating conflict.

**R — Result:**
Roger re-implemented the method. My manager was able to verify the cloud deployment. The project shipped unblocked, and the team relationship stayed intact.

**T — Takeaway:**
When attribution becomes a distraction from the actual problem, the right move is to separate them. I protected my track record (I shared the evidence) but prioritized outcome over being right. That's what ownership looks like when a project is at stake.

---

### BQ-2: Mistake & Recovery — Cloud Function Backward Compatibility

Use for: Ownership, Bias for Action, "Tell me about a time you made a mistake"

**C — Context:**
While implementing new InTriHub cloud functions, I noticed that existing functions of the same type were inconsistently structured. I decided to standardize them while I was in the codebase.

**A — Action:**
In the process of adding consistency, I added a new required parameter, "limit", to the getBootloader function. I didn't think carefully enough about backward compatibility: some functions had been in production for years and were consumed by external software that was no longer actively maintained. A week after my change, a bug report came in: RenewX, a customer-facing tool, was throwing errors when calling getBootloader. Because I had maintained a changelog documenting what I changed in each cloud function and when, I was immediately able to identify my "limit" field addition as the cause. I changed "limit" from required to optional with a sensible default value, deployed the fix, and confirmed RenewX was working again.

**R — Result:**
The customer-facing tool was restored quickly. Time-to-resolution was short specifically because of the changelog. Without it, tracing the change across functions with no code ownership markings would have been much slower.

**T — Takeaway:**
Two lessons: treat any public API field addition as potentially breaking, default to optional-with-default, not required. And the value of a changelog isn't obvious until you need it. I now maintain changelogs for shared APIs because this incident proved they pay off precisely when the cost of not having one is highest.
