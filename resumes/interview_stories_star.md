# Interview Stories (STAR Framework)

---

## Framework Note

**STAR** is the universally recognized behavioral interview format:

- **S — Situation:** Background, scene-setting, what was happening
- **T — Task:** YOUR specific role or responsibility — what you were asked to do or owned
- **A — Action:** The steps you took; technical decisions and why
- **R — Result:** Measurable outcome; optionally close with one sentence of reflection

**Difference from CART:**
- CART merges Situation + Task into "Context." STAR keeps them separate, which makes your personal ownership explicit — stronger for FAANG interviewers who score on ownership.
- CART adds a "Takeaway" (growth/reflection). STAR drops it, but you can add a one-sentence close to Result if the interviewer doesn't follow up.
- Use STAR when the interviewer explicitly says "STAR method." Use CART when the question is open-ended ("tell me about a time…") and you want to end with a growth signal.

Each story ends with **Probes** — same as the CART file.

---

## AutoPVT

*One-time context:*
AutoPVT is an in-house concurrent test-automation platform I built as the **sole engineer** at Intrising Networks. Around 30 QA engineers use it simultaneously to test the company's industrial Ethernet switches. Stack: Go/gRPC backend, PostgreSQL for persistence, Vue 3 frontend, running across a cloud API and local per-device box instances.

---

### AP-1: Two-tier PostgreSQL locking model

**S — Situation:**
AutoPVT is a concurrent test-editing platform where up to 30 engineers edit the same test objects simultaneously. Without concurrency control, two engineers writing to the same object at the same time would silently overwrite each other's changes, or produce a corrupted object where state is half from one write and half from another. There was no existing strategy.

**T — Task:**
As the sole engineer, I was responsible for designing the entire concurrency model from scratch — choosing the right primitives, mapping the full space of concurrent write combinations, and implementing it without blocking engineers who are working on different objects.

**A — Action:**
I identified two fundamentally different write types that needed different treatment. Structural writes (creating a test, editing the item list, finalizing, generating a report) must serialize because a second writer might invalidate the first. For these I used Postgres exclusive advisory locks keyed by test object ID, so two engineers on different objects never block each other, combined with a compare-and-bump revision check inside the lock: if the revision number has moved on since the client fetched the record, the write is rejected with "please reload." Result fills (engineers recording pass/fail for individual test items) are fine-grained: two engineers filling different items in the same test should never block each other. For these I used shared advisory locks with per-field merge semantics. Before writing code, I mapped out the full pairwise operation matrix (every write type against every other) to confirm the right lock type for each combination.

**R — Result:**
The platform has been in daily use by ~30 engineers with zero data corruption incidents from concurrent writes. Structural operations take turns; fills run in parallel. The granularity is correct.

**Probes:**

- *"Why advisory locks instead of row-level locks?"*
  Row-level locks are tied to specific rows and released at transaction end. Advisory locks are application-level, keyed by integer (I use the test object ID). They map cleanly onto "exactly one structural edit per test object at a time" across an operation that touches multiple tables.

- *"What about the start-new-test race?"*
  Two engineers clicking "start" on the same box device within milliseconds would both succeed without protection, creating two active tests on one device. I handle this with a per-box-MAC advisory lock during create: check if an active test already exists for that box; if so, reject the second with "reload to join the existing test."

- *"How would you scale this beyond one Postgres instance?"*
  There are three separate scaling problems. First, the advisory locks. The lock table lives in memory on the single Postgres node, so if I add a second database node and two writes for the same test object land on different nodes, each node thinks it holds the lock. Neither knows about the other's. Both writes go through and the data gets corrupted. The fix is to split data across database nodes by test object ID, so all operations for object #42 always route to the same node and the lock comparison always happens in the same place. Second, the broadcast. Right now the server that processes the write also holds the WebSocket connections and broadcasts directly. With multiple app servers, the write server might not be the one holding the engineer's WebSocket connection, so it has no way to reach that client. Moving to a shared message channel like Kafka or Redis solves this: the write server publishes "test changed" to the channel, and every app server is subscribed to it and broadcasts to its own connected clients. Third, the catalog prefetch loads 2,500 items and is read-heavy. Writes like adding a new test item always go to the main database. But heavy reads can go to read-only copies of the database that stay in sync with the main one automatically. That offloads the read traffic so the main database stays focused on handling writes.

---

### AP-2: Commit-before-broadcast ordering

**S — Situation:**
AutoPVT broadcasts real-time updates to all connected engineers when test state changes. If the broadcast fires before the transaction commits, clients receive state from a transaction that subsequently rolled back — a ghost update showing data that was never actually persisted.

**T — Task:**
I was responsible for designing the broadcast system so it was structurally impossible for clients to receive uncommitted state, without adding latency to the write path.

**A — Action:**
I enforced commit-before-broadcast as a strict structural invariant. The sequence is: (1) write state change and audit entry in a single Postgres transaction, (2) commit, (3) gRPC returns success acknowledgment, (4) only after receiving that ack does the sentry layer fire the WebSocket broadcast. The broadcast payload comes from the gRPC response message, which was only generated from committed data, so it's structurally impossible for the broadcast to carry uncommitted state. The audit entry is in the same transaction as the state write, not written after the fact, so the audit trail is always atomically consistent with live state.

**R — Result:**
The broadcast system has run without any dirty-state incidents. Every push clients receive is guaranteed committed, and the audit history is always complete.

**Probes:**

- *"What if the WebSocket broadcast fails after the gRPC ack?"*
  The write is committed and correct; the client misses the push notification. This is at-most-once delivery, an accepted trade-off. On the next mutation or reconnect they'll see current state. For guaranteed delivery I'd add a message queue with consumer acknowledgment.

- *"Why put the audit write in the same transaction?"*
  A separate post-fact write leaves a window where data changed but the audit didn't record it, or the audit write fails entirely. Same transaction means both succeed or both fail atomically.

---

### AP-3: LISTEN/NOTIFY firmware event pipeline

**S — Situation:**
AutoPVT's autotesting flow needs to notify machine clients the moment a new firmware build lands in the database. Polling was the existing approach. The architecture had a complication: firmware arrives via hub-action-server (a separate Node.js service) which has no gRPC or HTTP connection to autopvt-api — the shared Postgres database is the only integration point.

**T — Task:**
I was responsible for designing a real-time event pipeline that delivered firmware build notifications to gRPC server-streaming clients, without adding new infrastructure dependencies and without direct coupling between the two services.

**A — Action:**
Because hub-action-server and autopvt-api only share a database, the API can't receive a direct call when a firmware arrives. A Postgres database trigger fires on INSERT or UPDATE to the firmwares table, but only when the md5 checksum actually changes — that guard prevents spurious fires when hub-db-sync rewrites a row with the same content. The trigger posts a NOTIFY on the firmware_event channel with a minimal payload (firmware ID, operation type, platform, customization) to stay under the 8KB NOTIFY limit; the receiver re-fetches the full row on receipt. On the autopvt-api side, a firmwareEventHub runs a single pgx goroutine listening on that channel. When an event arrives, it fans out to all registered server-streaming gRPC clients, filtering by each client's platform and customization subscription server-side. I chose Postgres LISTEN/NOTIFY over Redis Pub/Sub to avoid introducing a third service to deploy, monitor, and operate just to bridge two services that already shared a database.

**R — Result:**
Autotesting clients receive firmware availability notifications within seconds of a build landing, with no polling overhead and no direct coupling between hub-action-server and autopvt-api.

**Probes:**

- *"What happens if a client disconnects mid-stream?"*
  Their subscriber is removed from the hub. Events during the disconnect are missed with no replay buffer. Acceptable for this use case; the client reconnects quickly. Guaranteed delivery would require a persistent queue.

- *"How does authentication work for server-streaming RPCs?"*
  The gRPC-gateway only handles unary RPCs, so server streaming requires a native gRPC connection to port 49000. Auth is handled in-handler at stream start: JWT from request metadata is validated before the client is added to the hub. I also added gRPC keepalive pings (30s interval) so the long-lived connection isn't reaped by proxy idle timeouts.

---

### AP-4: 3-5 second load delays to under 500ms

**S — Situation:**
The test catalog has 2,500 items organized in a four-level tree with embedded topology diagrams. Loading this view took 3-5 seconds — visible daily friction for 30 engineers on a tool they used constantly.

**T — Task:**
I was tasked with diagnosing and fixing the load time. No single bottleneck had been identified; I needed to trace the full stack.

**A — Action:**
Three separate problems were stacking. First: the server-side catalog fetch was doing 4 separate SQL queries, one per tree level, then assembling the tree with four nested loops in Go — O(categories × suites × groups × items) work plus 4 database round trips. I rewrote it as a single SQL JOIN across all four tables, assembled in one pass using three hashmaps keyed by ID. Each row from the JOIN is processed in O(1). 4 round trips became 1; tree assembly became O(n). Second: even with a faster backend, the payload was still huge because every item included its full detail and an embedded base64 topology diagram, pushing responses past the 4MB gRPC message limit. I split the fetch into a lightweight list (IDs, names, structure only) and a separate lazy fetch per item on click. Third: with a lightweight response, prefetching the whole tree into Vuex at login became cheap. By the time an engineer navigates to the item selection step, the tree is already in the store and navigation is instant.

**R — Result:**
Load time dropped from 3-5 seconds to under 500ms.

**Probes:**

- *"Why not just add a database index?"*
  The bottleneck was not query speed on individual tables — it was 4 separate round trips and the O(categories × suites × groups × items) in-memory assembly. An index on each table would still require 4 queries. The JOIN moves all the work into one database operation.

- *"Why prefetch at login instead of on first navigation?"*
  Login is a natural initialization boundary. Prefetching on first-step navigation delays the engineer exactly when they want to act. By the time they've set up a test and navigated to item selection, the prefetch is almost certainly complete.

- *"Why split the fetch rather than just compressing the payload?"*
  Compression reduces transfer size but the server still serializes 2,500 full records and the client still deserializes them. The lazy detail fetch means that work never happens at all for items the engineer doesn't click — which is most of them.

---

## Switch Management Interface (SMI)

*One-time context:*
The Switch Management Interface is the web UI and backend gateway for Intrising's industrial Ethernet switches, used in factory floors and critical infrastructure. I owned the Go gateway layer; core firmware was written in C by other engineers. I worked across four product lines.

---

### SMI-1: IEC 62443-4-2 SL3 — session limits + TOTP

**S — Situation:**
Intrising's switch products needed IEC 62443-4-2 Security Level 3 certification — an industrial cybersecurity standard. Key requirements: CR 1.1 (TOTP-based MFA) and CR 2.5 (per-user per-interface concurrent session limits) across all login paths: web, CLI, and Telnet.

**T — Task:**
I led the session control and authentication requirements across four product lines — responsible for the gateway and web UI portions, and for designing the interface by which core firmware would integrate session checking without owning that code.

**A — Action:**
CR 2.5 requires per-user per-interface session limits across all login paths. The web path was natural: my gateway already controls JWT-based sessions, so I added a per-user per-interface counter and enforced the limit at JWT issuance. For CLI and Telnet, those authenticate via PAM in the core firmware. I designed a CheckSessionLimit RPC in the gateway's InternalService: IS-Roger's auth code calls this after a successful PAM login to both record the new session and check if the limit is exceeded. The gateway becomes the single authoritative session store across all three interfaces. I also implemented proactive termination: when an admin disables a user's interface access, the gateway immediately invalidates existing JWTs. This works via a stateful JWT pattern: every issued JWT gets a `jti` claim, and the gateway keeps an in-memory map of all active sessions keyed by `jti`. Every authenticated request calls `CheckToken`, which only passes if that `jti` exists with state active. To invalidate, `RuinToken` flips the state flag. The JWT signature is still cryptographically valid, but the state check rejects it on the next request. Tradeoff: the session map lives in memory, so a gateway restart logs everyone out.

**R — Result:**
Four product lines achieved IEC 62443-4-2 SL3 compliance on session control and authentication. Session limits are enforced in real time across all login paths from one source of truth.

**Probes:**

- *"What was your scope within IEC 62443?"*
  CR 1.1 (TOTP MFA) and CR 2.5 (session limits). Other requirements were already implemented or handled by other engineers.

- *"How do you handle timeout vs. explicit logout?"*
  Both go through `RuinToken`. Explicit logout sets state to logged-out immediately. Expiry is lazy: the state is flipped to expired the next time that token hits the auth middleware and fails the timeout check. Admin termination sets state to session-terminated. All three paths are just different state values on the same in-memory `tokenDB` entry.

---

### SMI-2: Two-RPC protocol, PAM conversation handler, backward compatibility

**S — Situation:**
Adding TOTP MFA to the web login required bridging two incompatible models: PAM's interactive challenge-response protocol (multi-round conversation: send prompt, receive response, repeat) and HTTP's stateless single request-response. There is no way to pause a PAM conversation mid-flight and resume it when a second HTTP request arrives.

**T — Task:**
My task was to design a login protocol that added MFA support for PAM-authenticated users while preserving backward compatibility for all existing single-factor accounts — no user-side changes for non-MFA users.

**A — Action:**
PAM knows per-user whether TOTP is required: that config lives in PAM's own per-user system. So PAM only sends a TOTP prompt to users who have TOTP configured; non-MFA users get only the password prompt and succeed in a single PAM run. The problem is that for MFA users, PAM sends the TOTP prompt during that same PAM run, immediately after the password, with no way for the gateway to pause and ask the browser for a code. I solved this with a two-RPC protocol backed by a CGO-bridged PAM conversation handler. The handler intercepts each PAM prompt individually in Go. First RPC: password only. For non-MFA users, only the password prompt appears, PAM succeeds, JWT issued. For MFA users, the password prompt succeeds, then the TOTP prompt appears — the handler responds with a sentinel value it knows PAM will reject. PAM fails at the TOTP step. The handler surfaces "MFA required" to the gateway, which returns it to the browser as a signal to show the TOTP entry page. Second RPC: password + real TOTP. The handler provides the real code to the TOTP prompt, PAM succeeds, JWT issued.

**R — Result:**
TOTP MFA shipped across all login interfaces. All existing single-factor accounts continued to work with no user-side changes.

**Probes:**

- *"Why CGO instead of shelling out to the existing PAMLogin binary?"*
  The binary returned only a final success/fail exit code, with no visibility into which step failed. With the CGO conversation handler, each PAM prompt is intercepted individually. I can detect exactly where failure occurred: wrong password vs. wrong TOTP vs. account disabled. That granularity is required for the two-RPC protocol to work.

- *"Walk me through the full MFA login flow."*
  Step 1: browser sends username + password. Gateway calls PAM with just the password. Handler intercepts prompts. PAM validates password (correct), then sends a TOTP prompt. Handler responds with sentinel. PAM fails at TOTP. Handler reports "TOTP prompt reached, failed." Gateway returns MFA-required. Browser shows TOTP entry page. Step 2: user enters 6-digit code, browser sends username + password + TOTP. Gateway calls PAM again. Handler provides real TOTP to the TOTP prompt. PAM validates both, succeeds. Gateway issues JWT.

- *"How is the TOTP secret stored?"*
  AES-256-GCM encrypted in the user's gateway record. Users can enable MFA on their own account; only the admin can disable someone else's or reset their secret. Setup generates an otpauth:// URI for QR code scanning.

---

### SMI-3: Config-save bug — silent credential corruption

**S — Situation:**
A bug report: after saving and restoring the device configuration, the device couldn't log in. Credentials were corrupted. Nothing looked wrong during normal operation — the bug was completely invisible until the device rebooted from a saved config.

**T — Task:**
I was responsible for diagnosing the root cause of the credential corruption and fixing it across however many domains it affected — with no repro path available other than the full export/reboot/restore cycle.

**A — Action:**
To understand the bug I had to trace where in the stack masking was being applied. The system has a Protocol Service layer in the firmware below the gateway — it handles config read/write for each domain (SNMP, user permissions, system, log, main config). Masking had been placed there: the Protocol Service applied `*****` substitution on all GET responses. The problem is that the device's config-save procedure reads from that same Protocol Service layer. So when the device saved its running config, it read already-masked values and wrote `*****` to the config file. On reboot the device loaded `*****` as the real credential. It was invisible during normal operation because everything looked correct on screen — the corruption only surfaced on restore. This affected five domains because each had its own Protocol Service handler applying the same wrong pattern. The fix: move masking out of Protocol Service entirely and into the gateway egress layer. Each GET handler now masks on the way out to the browser; Protocol Service always returns real values. On the SET side, when the browser echoes back a placeholder, the gateway detects it, fetches the real value, and substitutes it before forwarding. In the same subsystem I found a related SSH key-matching bug: keys were identified by index position, so reordering keys matched the wrong key. I switched to fingerprint-based matching.

**R — Result:**
Five configuration domains fixed and two silent failure modes closed in the same subsystem. Saved and restored device configurations are now reliable.

**Probes:**

- *"How did this get through normal testing?"*
  Normal usage never triggers it. The device runs fine; the config looks correct on screen. Only when you export and restore — an infrequent operation — does the device load placeholder strings as real credentials. It wasn't caught because the test cycle almost never included restore-from-config.

- *"How did you test the fix?"*
  Unit tests for the masking and restore functions across all five domains, plus integration test of the full cycle: configure → export → restore from config → verify login succeeds.

---

### SMI-4: CI build time reduction

**S — Situation:**
The SMI CI pipeline had two problems. A correctness problem: Gulp's `parallel()` doesn't propagate worker exit codes — if any worker exited non-zero, the remaining workers kept running and the overall build reported green. Broken builds were invisible. A performance problem: the pipeline compiled SVG diagrams for all switch models even when most were irrelevant to the current product.

**T — Task:**
I was responsible for fixing CI correctness — broken builds must fail visibly — and for reducing build time across four product-line repos.

**A — Action:**
I addressed correctness first. I wrote `gulp-multi-process-fail-fast.js`, a Node.js script that spawns all build worker processes, monitors their exit codes, and on the first non-zero exit kills every remaining worker and exits with failure, replacing Gulp's `parallel()`. CI now surfaces any build failure immediately. For performance: I introduced an `included_svg_models.txt` allowlist per product. Only the listed models get compiled for that product's build; everything else is skipped. SVG compilation was the dominant build cost, so scoping the compiled set was the high-leverage change. Both changes shipped across four product-line repos.

**R — Result:**
Build time dropped over 80%. CI now correctly fails on any worker error rather than silently passing.

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

**S — Situation:**
Starting around July 2025, InTriHub's Firebase bill spiked to $300/month. My supervisor flagged it. Firebase bills for data egress, so something was downloading a large amount of data frequently — but nothing had obviously changed in the code around that time, and before July the project had zero Firebase cost.

**T — Task:**
My task was to investigate the root cause of the cost spike and bring costs back to the free tier, without a full database migration if possible.

**A — Action:**
My first step was to trace the git history for anything that changed around July — I even used Claude Code to help search through commits — but found nothing suspicious. That pushed my initial hypothesis toward a structural Firebase limitation: RTDB has no compound queries, so every page was fetching the full firmware collection and filtering client-side. But I had a nagging sense something must have actually changed, because if it were just the query model we would have been paying from day one.

So I explored migrating to Firestore and Algolia. When I started importing the firmware collection into Algolia, I immediately hit Algolia's record size limit: over 70 documents exceeded 100KB, with several approaching 1MB. That's when the actual root cause became visible: an `internalLog` field embedded in each firmware record, storing the full version changelog going back to initial release, had been growing steadily as more firmware versions were released. By the fix it was 64MB total with `internalLog` alone at 59MB. The July spike wasn't random — it was when the collection crossed the size threshold that made every full fetch expensive. Every page load was downloading 60+ MB of changelog content most users couldn't even view, since visibility was gated by UI `v-if` only, not at the RTDB rule level.

I wrote a full evaluation of the Firestore + Algolia migration: 8–11 engineer-weeks, 2–3× operational surface. The problem wasn't query capability — it was one overloaded field. I split `internalLog` into a lazy sibling node `firmwareInternalLog/{fid}`, keeping only a `hasInternalLog` boolean on the firmware record. The log is now fetched on demand only when an admin explicitly expands a row. RTDB rules now enforce admin-only at the database layer. A one-time Cloud Function migration script backfilled existing records idempotently.

Separately: four pages each attached their own `.on('value')` live listener to the full firmware node on every navigation. Each `.on()` call triggers an immediate full collection download. Neither page had a `beforeDestroy` cleanup hook — old listeners accumulated silently, and each page visit added another. In a normal session, up to four active listeners stacked on the same node. I replaced all four with a single global `subscribeFirmware()` in `src/index.js`, attached once after login and torn down on logout.

**R — Result:**
Cost went from $300/month to within the free tier, with no RTDB line item. The firmware collection shrank from 70 MB to 5 MB (93% reduction).

**Probes:**

- *"Why not Firestore or Algolia?"*
  I actually started down that path — it was while importing firmware into Algolia that I hit the record size limit and discovered `internalLog` was the real culprit. Once I knew the root cause was a single overloaded field rather than a query capability gap, I wrote a full migration evaluation: 8–11 engineer-weeks, 2–3× operational surface, new infrastructure and billing relationships. Splitting one field was a fraction of that cost with no new infrastructure risk.

- *"How did you handle the migration without downtime?"*
  The migration script ran as a batched Cloud Function: copy internalLog to firmwareInternalLog/{fid}, then clear the field from the original record. I kept the old path readable during migration until both the migration and UI update deployed. Transparent to users.

---

## Texim Europe

*One-time context:*
Summer 2023, internship at Texim Europe, a Dutch electronics distributor. Their ETL pipeline ran on SQL Server 2000 DTS, which was end of support and unable to receive security patches.

---

### TX-1: ETL platform replacement

**S — Situation:**
Texim's ETL pipeline ran on SQL Server 2000 DTS — discontinued by Microsoft in 2005 and unable to receive security patches. The business driver was security compliance. The entire existing pipeline was in VBScript.

**T — Task:**
My task was to replace the platform with a maintainable, patchable tool. The hard constraint: the replacement must run Texim's full existing VBScript ETL library without any script rewrites.

**A — Action:**
The VBScript constraint drove the architecture. Rather than embedding a VBScript interpreter, I routed execution through Windows Script Host (wscript.exe) — the same engine that had always run these scripts. Existing scripts ran exactly as before with no compatibility work. The replacement is a C# WinForms application with three components: an editor GUI for authoring and organizing packages; a scheduler with daily, weekly, and monthly recurrence plus email alerts, with schedule state persisted to SQL Server so configuration survives reboots; and PackageRunService, a multithreaded Windows service that runs in the background and executes scheduled scripts concurrently — required because scripts must run on schedule even when no one has the UI open.

**R — Result:**
Texim's full ETL script library migrated to the new platform without any script rewrites. They replaced an end-of-life, unpatched system with a maintainable C# application on a supported stack, with no disruption to the existing pipeline.

**Probes:**

- *"Why not migrate to SSIS, Microsoft's official DTS replacement?"*
  SSIS would require migrating every DTS package to a new format — violates the "no script rewrites" constraint. Routing through wscript.exe meant existing scripts ran unchanged from day one.

- *"Why WinForms and not a web app?"*
  Internal tool, all users on Windows, the Windows service model fits naturally with a Windows-native app, and WinForms was the fastest path for a summer internship timeline.

- *"How did you handle script errors and output?"*
  wscript.exe returns an exit code — non-zero means failure. Output is captured from stdout/stderr. The scheduler logs both per run and sends an email alert on failure.

---

### TX-2: SQL Server Query Notifications

**S — Situation:**
After building the ETL replacement, engineers could run and schedule scripts, but had no way to know when a running job finished except by refreshing the UI manually.

**T — Task:**
I needed to add real-time job status updates to the desktop tool. Options: polling on a timer, SignalR (push-based but requires a web server), or SQL Server Query Notifications — a push mechanism built on Service Broker that's already part of SQL Server with no new infrastructure needed.

**A — Action:**
SQL Server Query Notifications work by registering a query with SQL Server: the server monitors whether the result set would change and fires a notification to the .NET application via Service Broker when it does. I implemented three SqlDependency subscriptions: one on the running packages table (job start/finish), one on the scheduled packages table (schedule changes), and one on the packages list (additions/deletions). The subscription is single-fire by design — after it fires, it expires. The OnChange handler re-registers the dependency before re-querying, so there's no window between registration and query where an update could be missed.

**R — Result:**
Engineers see immediate status updates when jobs complete, with no polling overhead and no artificial latency. The UI updates the moment SQL Server commits the status change.

**Probes:**

- *"What are SqlDependency's constraints?"*
  The subscribed query must use schema-qualified table names, no aggregates, no subqueries, no SELECT *. Service Broker must be enabled. It fires on any change to the result set — not just the one you care about — so the OnChange handler always discards the notification payload and re-queries for current state. The notification is a wake-up signal, not a data carrier.

- *"Why not just poll on a timer?"*
  Polling has two costs: unnecessary load when nothing has changed, and latency equal to the polling interval. For a tool engineers watch while waiting for a job to finish, even 5-second polling feels sluggish. Query Notifications fire within milliseconds of the commit.

---

## NLP Project

*One-time context:*
Team project of four at the University of Twente, November 2023. Goal: given a research paper abstract and subject areas, recommend the top 5 journals to submit to. I was the ML engineer: designed the architecture, trained the models, built the ensemble.

---

### NLP-1: 96.9% top-5 accuracy, 27-model SciBERT ensemble

**S — Situation:**
A four-person team project: build a journal recommendation system covering 378 journals across 26 subject areas. Dataset: 35,370 scientific paper abstracts after cleaning, averaging ~1,350 per domain. No prior architecture existed.

**T — Task:**
As the ML engineer, I was responsible for designing the model architecture, training all models, and building the ensemble that produced final recommendations.

**A — Action:**
The core architectural decision: one model over all 378 journals, or 26 domain-specific models? I chose 26 separate models — "which chemistry journal?" and "which math journal?" are fundamentally different classification problems, and a single model would have to learn unrelated decision boundaries simultaneously. The architecture: 26 SciBERT-based abstract classifiers (one per subject area) plus 1 subject area classifier. SciBERT is Allen AI's transformer pretrained on 1.14M scientific papers. I used it for feature extraction only — weights frozen, not fine-tuned — because with ~1,350 samples per domain, fine-tuning a 110M-parameter model would overfit badly. I pass each abstract through frozen SciBERT, extract the [CLS] token embedding (768-dimensional dense summary), and pass it to a small softmax classifier I actually train. The subject area classifier uses CountVectorizer + softmax regression (not SciBERT — inputs are short label strings, contextual embeddings aren't needed). The ensemble is accuracy-weighted and dynamic: when a user selects N subject areas, N+1 models run, weights proportional to each model's validation accuracy.

**R — Result:**
96.9% top-5 accuracy on a held-out test set of 3,000 abstracts across 378 journals. Top-1: 76.8%, top-3: 93.5%. Random baseline: 0.26%.

**Probes:**

- *"What's the [CLS] token?"*
  In BERT-style models, a special [CLS] token is prepended to the input with no real-word meaning. Because it carries no semantic content of its own, the model accumulates a representation of the full input in that position over the attention layers. After the forward pass, its 768-dimensional vector is a dense summary of the whole abstract.

- *"Why freeze SciBERT rather than fine-tune?"*
  Three reasons: dataset size (~1,350 samples per domain for 110M parameters), no GPU budget for extended fine-tuning, and catastrophic forgetting risk. Frozen feature extraction is fast, cheap, and avoids all three failure modes.

- *"Why accuracy-weighted rather than equal weights?"*
  Equal weighting over-relies on classifiers for small subject areas (near-100% accuracy, few journals) and under-relies on classifiers for large difficult areas. Accuracy-proportional weighting lets more reliable classifiers contribute more while still incorporating domain-specific signals from all active models.

- *"How would you scale to 1 million users?"*
  The SciBERT embedding step is the bottleneck (O(seq²) attention on CPU). At scale: cache embeddings for abstracts already seen, batch inference across users, deploy SciBERT on GPU with TorchServe or a managed endpoint, and separate the feature extraction tier from the classification tier so they scale independently.

---

## Behavioral Stories (BQ — Not on Resume)

*Pure behavioral stories, not tied to a specific resume bullet. Use for: "Tell me about a conflict," "Tell me about a mistake," "Tell me about a time you took ownership."*

---

### BQ-1: Conflict Resolution — Missing API (getCloudToken)

Use for: Earn Trust, Ownership, Conflict with a teammate

**S — Situation:**
Near the end of my time as de-facto lead on AutoPVT, my manager discovered that getCloudToken — a critical API the core firmware calls to obtain a cloud-scoped JWT — was missing. Without it, field devices cannot authenticate to the cloud. The engineer who was supposed to have implemented it denied remembering it.

**T — Task:**
I had evidence (a Slack thread showing I had asked IS-Roger to implement it and he had replied "Done"), but my primary task was to unblock the production deployment — not to win an attribution dispute.

**A — Action:**
I was frustrated. The protobuf definition for the method was in Roger's own codebase in the git history. But I recognized that winning the attribution argument wasn't my job; unblocking the project was. I set aside the blame question and re-explained clearly what getCloudToken does, when it's called, and what it returns, then asked Roger to implement it again. In parallel I informed my manager that I had the Slack evidence, framed informally and without escalating it as a formal dispute, so the record was clear without creating conflict.

**R — Result:**
Roger re-implemented the method. My manager verified the cloud deployment. The project shipped unblocked and the team relationship stayed intact.

---

### BQ-2: Mistake & Recovery — Cloud Function Backward Compatibility

Use for: Ownership, Bias for Action, "Tell me about a time you made a mistake"

**S — Situation:**
While adding new InTriHub cloud functions, I noticed inconsistencies in existing functions and decided to standardize them while I was in the codebase. In doing so, I added a new required parameter ("limit") to getBootloader — an API that had been in production for years and was consumed by external software no longer actively maintained.

**T — Task:**
A week later a bug report came in: RenewX, a customer-facing tool, was throwing errors when calling getBootloader. I needed to diagnose the cause, fix it, and minimize time-to-resolution.

**A — Action:**
Because I had maintained a changelog documenting what I changed in each cloud function and when, I was immediately able to identify my "limit" field addition as the cause. I changed "limit" from required to optional with a sensible default value, deployed the fix, and confirmed RenewX was working again.

**R — Result:**
The customer-facing tool was restored quickly. Time-to-resolution was short specifically because of the changelog — without it, tracing the change across functions with no code ownership markings would have been much slower.
