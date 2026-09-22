# Interview Stories for resume_sde.md (STAR Framework)

Only the 10 stories behind bullets that are actually on the 1-page resume, in the same order they appear there. Full versions of every story (including the ones cut from this resume) live in `interview_stories_star.md` / `interview_stories_cart.md`.

---

## 1. AutoPVT — Two-tier PostgreSQL locking model

**Resume bullet:** *Designed a two-tier PostgreSQL locking model as the sole engineer on AutoPVT, a concurrent test-editing platform serving ~30 engineers: exclusive advisory locks with compare-and-bump revision checks for structural writes, and shared locks with per-field merging for fills, eliminating blocking between concurrent testers editing different fields.*

**S — Situation:**
AutoPVT is a concurrent test-editing platform where up to 30 engineers edit the same test objects simultaneously. Without concurrency control, two engineers writing to the same object at the same time would silently overwrite each other's changes, or produce a corrupted object where state is half from one write and half from another. There was no existing strategy.

**T — Task:**
As the sole engineer, I was responsible for designing the entire concurrency model from scratch, choosing the right primitives, mapping the full space of concurrent write combinations, and implementing it without blocking engineers who are working on different objects.

**A — Action:**
I identified two fundamentally different write types that needed different treatment. Structural writes (creating a test, editing the item list, finalizing, generating a report) must serialize because a second writer might invalidate the first. For these I used Postgres exclusive advisory locks keyed by test object ID, so two engineers on different objects never block each other, combined with a compare-and-bump revision check inside the lock: if the revision number has moved on since the client fetched the record, the write is rejected with "please reload." Result fills (engineers recording pass/fail for individual test items) are fine-grained: two engineers filling different items in the same test should never block each other. For these I used shared advisory locks with per-field merge semantics. Before writing code, I mapped out the full pairwise operation matrix (every write type against every other) to confirm the right lock type for each combination.

**R — Result:**
The platform has been in daily use by ~30 engineers with zero data corruption incidents from concurrent writes. Structural operations take turns; fills run in parallel.

**Probes:**
- *"Why advisory locks instead of row-level locks?"* Row-level locks are tied to specific rows and released at transaction end. Advisory locks are application-level, keyed by integer (I use the test object ID). They map cleanly onto "exactly one structural edit per test object at a time" across an operation that touches multiple tables.
- *"What happens if two testers fill the same field?"* Last-write-wins, deliberately, not a merge. A single result field can only hold one value, so there's nothing to reconcile; the exact rejection message on a stale structural write is "This test was changed by someone else. Refresh and try again."
- *"What about the start-new-test race?"* Two engineers clicking "start" on the same box device within milliseconds would both succeed without protection, creating two active tests on one device. I handle this with a per-box-MAC advisory lock during create.
- *"How would you scale this beyond one Postgres instance?"* Three separate scaling problems: the advisory locks (fix: shard data by test object ID so a given object's operations always route to the same node), the broadcast (fix: a shared message channel like Kafka or Redis instead of the write server broadcasting directly), and the read-heavy catalog prefetch (fix: read-only replicas).

---

## 2. AutoPVT — Catalog load time, 3-5s to under 500ms

**Resume bullet:** *Resolved 3-5-second load delays on the 2,500-item test catalog, reducing load time to under 500ms by replacing 4 sequential SQL queries and nested-loop tree assembly with a single JOIN and a hashmap-based tree-building algorithm, lazy-loading item details via RPC, and prefetching the lightweight tree into the Vuex store at login.*

**S — Situation:**
The test catalog has 2,500 items organized in a four-level tree with embedded topology diagrams. Loading this view took 3-5 seconds, visible daily friction for 30 engineers on a tool they used constantly.

**T — Task:**
I was tasked with diagnosing and fixing the load time. No single bottleneck had been identified; I needed to trace the full stack.

**A — Action:**
Three separate problems were stacking. First: the server-side catalog fetch was doing 4 separate SQL queries, one per tree level, then assembling the tree with four nested loops in Go, O(categories × suites × groups × items) work plus 4 database round trips. I rewrote it as a single SQL JOIN across all four tables, assembled in one pass using three hashmaps keyed by ID. 4 round trips became 1; tree assembly became O(n). Second: even with a faster backend, the payload was still huge because every item included its full detail and an embedded base64 topology diagram, pushing responses past the 4MB gRPC message limit. I split the fetch into a lightweight list and a separate lazy fetch per item on click. Third: with a lightweight response, prefetching the whole tree into Vuex at login became cheap, so navigation to item selection is instant.

**R — Result:**
Load time dropped from 3-5 seconds to under 500ms.

**Probes:**
- *"Why not just add a database index?"* The bottleneck wasn't query speed, it was 4 round trips plus O(categories × suites × groups × items) in-memory assembly. The JOIN moves all the work into one database operation.
- *"Why prefetch at login instead of on first navigation?"* Login is a natural initialization boundary; by the time an engineer sets up a test and navigates to item selection, the prefetch is almost certainly complete.
- *"Why split the fetch rather than just compressing the payload?"* Compression still means the server serializes and the client deserializes all 2,500 full records. The lazy fetch means that work never happens for items the engineer doesn't click, most of them in any session.
- *"How bad did it get before the fix?"* Worse than just slow: on the largest catalogs, the old query path could silently blow past the WS-bridge's 20-second default timeout, so it didn't degrade to "slow," it looked indistinguishable from a hung or broken request.

---

## 3. Switch Management Interface — IEC 62443-4-2 SL3, session limits + TOTP

**Resume bullet:** *Led the IEC 62443-4-2 SL3 security requirements for authentication and session control across four product lines: added TOTP-based MFA and enforced concurrent session limits per user and per interface across web, CLI, and Telnet via a shared internal gRPC service.*

**S — Situation:**
Intrising's switch products needed IEC 62443-4-2 Security Level 3 certification, an industrial cybersecurity standard. Key requirements: CR 1.1 (TOTP-based MFA) and CR 2.5 (per-user per-interface concurrent session limits) across all login paths: web, CLI, and Telnet.

**T — Task:**
I led the session control and authentication requirements across four product lines, responsible for the gateway and web UI portions, and for designing the interface by which core firmware would integrate session checking without owning that code.

**A — Action:**
CR 2.5 requires per-user per-interface session limits across all login paths. The web path was natural: my gateway already controls JWT-based sessions, so I added a per-user per-interface counter and enforced the limit at JWT issuance. For CLI and Telnet, which authenticate via PAM in the core firmware, I designed a `CheckSessionLimit` RPC in the gateway's InternalService that the firmware's auth code calls after a successful PAM login, making the gateway the single authoritative session store across all three interfaces. I also implemented proactive termination: when an admin disables a user's interface access, the gateway immediately invalidates existing JWTs via a stateful `jti`-keyed session map, checked on every request and flipped on invalidation.

**R — Result:**
The session control and authentication requirements are implemented across four product lines, as part of an SL3 effort that is still in progress. Session limits are enforced in real time across all login paths from one source of truth.

**Probes:**
- *"What was your scope within IEC 62443?"* CR 1.1 (TOTP MFA) and CR 2.5 (session limits). Other requirements were already implemented or handled by other engineers.
- *"How do you handle timeout vs. explicit logout?"* Both flip the same in-memory session entry's state; explicit logout is immediate, expiry is checked lazily on the next request, admin termination is the same mechanism.

---

## 4. Switch Management Interface — Two-RPC protocol, PAM conversation handler

**Resume bullet:** *Bridged PAM's interactive challenge-response model with HTTP's stateless request cycle by designing a two-RPC authentication protocol that preserved backward compatibility across all existing single-factor accounts: a password-only call returned an MFA-required signal and a password+TOTP call completed authentication, with a custom PAM conversation handler that distinguished password from TOTP prompts at the protocol level.*

**S — Situation:**
Adding TOTP MFA to the web login required bridging two incompatible models: PAM's interactive challenge-response protocol and HTTP's stateless single request-response. There is no way to pause a PAM conversation mid-flight and resume it when a second HTTP request arrives.

**T — Task:**
Design a login protocol that added MFA support for PAM-authenticated users while preserving backward compatibility for all existing single-factor accounts, no user-side changes for non-MFA users.

**A — Action:**
PAM only sends a TOTP prompt to users who have it configured; non-MFA users succeed in a single PAM run on the password prompt alone. For MFA users, PAM sends the TOTP prompt immediately after the password in that same run, with no way for the gateway to pause and ask the browser for a code. I solved this with a two-RPC protocol backed by a CGO-bridged PAM conversation handler that detects the TOTP prompt by matching PAM's message text, then intercepts each prompt individually in Go. First RPC (password only, endpoint `Login`): the handler calls the shared PAM-authenticate function with an empty TOTP code; for MFA users the TOTP prompt appears and naturally fails on that empty value, and the handler returns a distinct MFA-required error instead of a generic auth failure. Second RPC (password + real TOTP, endpoint `MFALogin`): the same shared function runs again, this time with the real code, PAM succeeds, JWT issued. Both endpoints call one shared internal authenticate function with a boolean flag that only changes the error-handling branch, rather than duplicating the PAM conversation and environment-parsing logic twice.

**R — Result:**
TOTP MFA shipped across all login interfaces. All existing single-factor accounts continued to work with no user-side changes.

**Probes:**
- *"Why CGO instead of shelling out to the existing PAMLogin binary?"* The binary returned only a final success/fail exit code with no visibility into which step failed. The CGO handler intercepts each prompt individually, so it can tell wrong password from wrong TOTP from account disabled, granularity the two-RPC protocol needs.
- *"Walk me through the full MFA login flow."* Step 1 (`Login`): password only, empty TOTP code; the shared authenticate function calls PAM, the TOTP prompt appears and fails on the empty value, the handler returns a distinct MFA-required error instead of a generic failure. Step 2 (`MFALogin`): the same shared function runs again with the real password and TOTP code, PAM succeeds, JWT issued.
- *"How is the TOTP secret stored?"* AES-256-GCM encrypted in the user's gateway record; users can enable their own MFA, only an admin can disable someone else's or reset a secret.

---

## 5. Switch Management Interface — CI build time reduction

**Resume bullet:** *Reduced CI build time by over 80% through scoping asset compilation to per-product SVG allowlists, and forked the open-source gulp-multi-process task runner to add fail-fast behavior, terminating all remaining workers on the first error instead of continuing silently after a failure.*

**S — Situation:**
The SMI CI pipeline had two problems: a correctness problem (the `gulp-multi-process` package we used always waited for every parallel build worker to finish before reporting failure, so a broken build sat silently until the last worker exited, and reviewers watching CI often saw a long run and assumed it was still healthy) and a performance problem (SVG diagrams were compiled for all switch models, 290 available, even when only a fraction were relevant to the current product's build).

**T — Task:**
Fix CI correctness, broken builds must fail fast and visibly, and reduce build time across four product-line repos.

**A — Action:**
I addressed correctness first: forked `gulp-multi-process` into a local fail-fast replacement (`gulp-multi-process-fail-fast.js`). It launches each build task (scripts, views, less-customs, less-common, devices-svg) as its own child process via `child_process.spawn`, tracks every worker's handle in an array, and attaches an exit listener to each. On the first non-zero exit, a guard flag prevents double-handling, it calls `.kill()` (SIGTERM) on every other still-running worker, and immediately returns the failing exit code to Gulp instead of waiting for the rest to finish. For performance, I replaced the unconditional per-model SVG compile step with a per-product allowlist file (`included_svg_models.txt`) plus an `--include-models-by-file` flag on the SVG conversion script, only the ~48 models listed for that product compile, the other ~240+ are skipped. Both changes shipped together in the same commit across four product-line repos.

**R — Result:**
CI now fails fast on any worker error instead of silently continuing to completion. Build time dropped significantly by scoping SVG compilation from 290 models down to the ~48 a given product actually needs.

**Probes:**
- *"How did you verify the new orchestrator produced identical output for successful builds?"* Ran both implementations on the same input and diffed the output artifacts; the change only affects failure propagation, not the build logic itself.
- *"Why an allowlist file rather than per-product build config?"* Simpler to review in PRs, adding a new device model is one line, versus updating multiple repos' configs.
- *"Why SIGTERM and not SIGKILL when killing remaining workers?"* SIGTERM lets each worker's own process (Node running a gulp task) exit cleanly rather than being killed mid-write; none of these tasks needed a harder kill.

---

## 6. InTriHub — Firebase cost spike, $300/month to free tier

**Resume bullet:** *Cut Firebase costs from $300/month to within Firebase's free tier by tracing the spike to a 70 MB firmware collection inflated by an embedded log field and 3 redundant per-page listeners re-fetching it on every navigation, restructuring it into a lazy-loaded sibling node (70 MB to 5 MB, 93% reduction) and consolidating to one global Vuex listener.*

**S — Situation:**
Starting around July 2025, InTriHub's Firebase bill spiked to $300/month. Nothing had obviously changed in the code around that time, and before July the project had zero Firebase cost.

**T — Task:**
Investigate the root cause of the cost spike and bring costs back to the free tier, without a full database migration if possible.

**A — Action:**
Tracing git history around the spike found nothing suspicious. Exploring a Firestore + Algolia migration, I hit Algolia's record size limit while importing the firmware collection, which surfaced the real root cause: an `internalLog` field embedded in each firmware record, storing the full version changelog, was the bulk of each record's size. Every page load was downloading that changelog content along with the rest of the record, most users couldn't even view it. I wrote a full migration evaluation and decided the problem was a single overloaded field, not a query capability gap. I split `internalLog` into a lazy sibling node (`firmwareInternalLog/{pt,mt}/<id>`, admin-read-restricted) fetched only when an admin explicitly expands a row, gated behind a `hasInternalLog` boolean left on the original record. Migrating existing data used a one-off two-phase admin script (non-destructive copy first, then a separate destructive strip pass, batched to stay under Firebase's 4MB/500-entry update limit) rather than a live Cloud Function. Separately, three pages, Firmware, Products, and Viewer, each attached their own live listener to the full firmware node on every navigation with no cleanup hook, stacking up duplicate active listeners per session. I hoisted all three into one global Vuex slice (in the app's central store) subscribed once on login and torn down once on logout.

**R — Result:**
Firebase costs dropped back to within the free tier, and the firmware collection's per-record payload shrank substantially once `internalLog` moved out of the default read path. *(Exact before/after cost and collection-size figures live in Firebase console billing history, not in git, confirm precise numbers there before quoting them in an interview.)*

**Probes:**
- *"Why not Firestore or Algolia?"* Once the root cause turned out to be one overloaded field rather than a query capability gap, the full migration (new infrastructure, a second billing relationship, real engineering weeks) wasn't justified against a much smaller, targeted fix.
- *"How did you handle the migration without downtime?"* A two-phase admin script, copy the log to the new node first, verify, then strip the old field in a separate pass, batched under Firebase's per-update payload limit, with the old path still readable until both the migration and the UI update were live.
- *"How many pages had the duplicate-listener bug?"* Three: Firmware, Products, and Viewer. Each independently subscribed to the same top-level firmware ref with no teardown; consolidated into one Vuex-managed subscription lifecycle tied to auth state.

---

## 7. Texim Europe — ETL platform replacement

**Resume bullet:** *Eliminated Texim's reliance on an end-of-life ETL platform (SQL Server 2000 DTS, unable to receive security patches) by building a C# WinForms replacement with a VBScript editor, a scheduler with daily/weekly/monthly recurrence, and a multithreaded Windows service, migrating the company's full script library without rewrites.*

**S — Situation:**
Texim's ETL pipeline ran on SQL Server 2000 DTS, discontinued by Microsoft in 2005 and unable to receive security patches. The entire existing pipeline was in VBScript.

**T — Task:**
Replace the platform with a maintainable, patchable tool. Hard constraint: the replacement must run Texim's full existing VBScript ETL library without any script rewrites.

**A — Action:**
Rather than embedding a VBScript interpreter, I routed execution through `cscript.exe`, the console-mode Windows Script Host engine that had always run these scripts, so existing scripts ran unchanged, no rewrites, no reformatting. The replacement is a C# WinForms application with three components: an editor GUI, a scheduler, and a Windows service. The scheduler supports daily/weekly/monthly recurrence, plus an Nth-weekday mode (e.g. "third Tuesday of every month"), with email alerts and state persisted to SQL Server. The Windows service polls the database every 60 seconds via a timer to detect newly due jobs, then spawns a dedicated thread per job (tracked in a lock-protected thread dictionary with a per-thread cancellation token for stop support) so scripts run concurrently in the background without the editor UI open.

**R — Result:**
Texim's full ETL script library migrated to the new platform without any script rewrites, replacing an end-of-life, unpatched system with a maintainable, supported one.

**Probes:**
- *"Why not migrate to SSIS, Microsoft's official DTS replacement?"* SSIS would require migrating every DTS package to a new format, violating the no-rewrites constraint. Routing through `cscript.exe` meant scripts ran unchanged from day one.
- *"Why WinForms and not a web app?"* Internal tool, all users on Windows, the Windows service model fits natively, and it was the fastest path for a summer internship timeline.
- *"Why a thread per job instead of a thread pool or async/await?"* Job count and concurrency were small and bounded (internal ETL scheduling, not a high-throughput service), so a dedicated thread per job with cancellation-token-based stop support was simple and predictable; a thread pool would have added complexity without a real throughput problem to solve.

---

## 8. Adaptive Ad Recommender — Kafka + Debezium CDC pipeline

**Resume bullet:** *Designed a Kafka (KRaft) and Debezium change-data-capture pipeline to replace a Postgres-to-Pinecone sync prone to dual-write drift, choosing Kafka's per-partition ordering guarantee so eligibility updates apply in the correct order.*

**S — Situation:**
Campaign eligibility (status, budget) lives in Postgres, but serving needs to filter and rank eligible campaigns via Pinecone's vector search. A naive dual-write has two independent points of failure: if the Postgres write succeeds but the Pinecone write silently fails, Pinecone drifts out of sync and could keep serving an already budget-exhausted campaign.

**T — Task:**
Keep Pinecone's eligibility data converged with Postgres without a second manual write path, and specifically preserve event order, since two budget-debit events on the same campaign applied out of order could leave a depleted campaign servable.

**A — Action:**
I built a Kafka (KRaft mode) and Debezium change-data-capture pipeline: the app only ever writes to Postgres (with `REPLICA IDENTITY FULL` enabled so Debezium's `pgoutput` plugin captures full row state), Debezium watches Postgres's own write-ahead log, and a Kafka consumer propagates every change into Pinecone. I chose Kafka specifically for its ordering guarantee, messages are keyed by campaign_id so all of one campaign's events stay in order, Redis/RQ (used elsewhere in the app) has no such guarantee. While building this I also fixed two smaller issues with real measurements: a Pinecone client being reconstructed on every call instead of cached (2.4-2.9x speedup once fixed), and a defensive 3x oversampling multiplier on retrieval that turned out unnecessary once tested at realistic churn (~2.8% instead of the first test's misleading ~49%-in-one-burst) instead of an unrealistic first stress test.

**R — Result:**
Postgres stays the single source of truth for eligibility; Pinecone converges from the CDC log rather than a second manual write, removing the class of bug where the two silently disagree.

**Probes:**
- *"Why not just optimize the dual write?"* Two independent writes have two independent chances to fail, with no way to guarantee both succeeded across two different databases. CDC means there's only one write, to Postgres.
- *"What if the Kafka consumer falls behind?"* A dead-letter topic isolates malformed events, and consumer-group lag is self-logged so a growing backlog is visible rather than silent.
- *"Walk me through why the first oversample test was misleading."* It tested 49% catalog churn in one burst at the wrong batch size, which looked like it justified a 3x oversample margin. Rerunning at realistic churn and the real batch size showed the margin caught nothing, zero trims across the run, so it was removed.
- *"Is the topic actually running with multiple partitions today?"* No, it's deployed with a single partition since traffic doesn't need more yet. The ordering guarantee is about the keying design, campaign_id as the message key, which is what makes it safe to add partitions later without breaking per-campaign order, not about partition count today.

---

## 9. Adaptive Ad Recommender — Adversarial prompt-injection testing

**Resume bullet:** *Built an LLM-based campaign-review agent and onboarding chat, then red-teamed them with real prompt-injection attacks and found they were both bypassable. Closed the gap with prompt-level instruction-vs-data framing where it held, and with a deterministic length-floor check where it didn't.*

**S — Situation:**
I'd built two LLM-facing surfaces, a campaign policy reviewer and an onboarding checkpoint judge, both taking attacker-controlled text as input. A mocked test proves nothing about whether the real model resists a real attack.

**T — Task:**
Actually find out whether these surfaces could be manipulated by crafted input, and fix whatever I found, using real, unmocked calls to the model rather than assumptions.

**A — Action:**
I wrote crafted injection attempts against both surfaces: fake "SYSTEM OVERRIDE" instructions, claims of pre-approval, fabricated conversation history. The first run found real, reproducible vulnerabilities on both. For the policy reviewer, two different injections succeeded non-deterministically; adding explicit instruction-vs-data framing to the system prompt fixed both, clean across every follow-up run. For the onboarding checkpoint judge, the same class of fix did not work, an injected override and a fabricated fake history both still succeeded every run even with equivalent framing. Since prompt hardening alone wasn't sufficient there, I added a deterministic backstop instead: a length floor before any interest summary gets embedded and persisted as a profile vector, closing the concrete harm even though the judge's raw output is still technically manipulable. One of the two known issues on that surface still has no fix, deliberately, a real fix would need server-side session state that conflicts with the app's intentionally stateless design, so I left it open and tracked rather than rush a fragile patch.

**R — Result:**
Two of two tested policy-review injection paths are fully closed. One onboarding vulnerability has a real deterministic backstop closing its concrete harm. One remains open by deliberate choice, tracked with an explicit written reason rather than hidden.

**Probes:**
- *"How do you know your test suite itself isn't unreliable, since it's calling a real LLM?"* Where the check is a literal field value, the assertion is deterministic, no LLM judging involved. Only the two checks that are inherently semantic route through a small LLM-as-judge helper, and that helper is itself hardened against the same injected text it's evaluating.
- *"Is the campaign reviewer an actual agent?"* Only in the tool-calling loop added afterward, where it queries a real Postgres table for an advertiser's history on borderline cases, that's custom code my own process executes across multiple turns. The earlier web-search-only version isn't an agent by any meaningful definition, since OpenAI runs that tool server-side within a single call.
- *"Why leave the fabricated-history vulnerability open instead of fixing it?"* The real fix needs server-side session state to verify a prior round genuinely happened, and the app is deliberately stateless today. The harm from leaving it open is also low severity, onboarding ends a little early with friendly messaging, not data corruption, so the risk didn't justify a rushed fix.

---

## 10. NLP Project — 96.9% top-5 accuracy, 27-model SciBERT ensemble

**Resume bullet:** *Achieved 96.9% top-5 accuracy across 378 journals by designing and training 27 models: 26 domain-specific SciBERT abstract classifiers and 1 subject-area classifier, combining their predictions with weights proportional to each model's validation accuracy, and validating the ensemble on a held-out test set of 3,000 abstracts.*

**S — Situation:**
A four-person team project: build a journal recommendation system covering 378 journals across 26 subject areas. Dataset: 35,370 scientific paper abstracts after cleaning, with domain sizes varying widely, from roughly 100 documents in the smallest subject area (nursing) up to several thousand in the largest (biochemistry). No prior architecture existed.

**T — Task:**
As the ML engineer, I was responsible for designing the model architecture, training all models, and building the ensemble that produced final recommendations.

**A — Action:**
The core decision: one model over all 378 journals, or 26 domain-specific models? I chose 26 separate models, "which chemistry journal" and "which math journal" are fundamentally different classification problems. Architecture: 26 SciBERT-based abstract classifiers (one per subject area) plus 1 subject-area classifier. I used SciBERT for feature extraction only, weights frozen, not fine-tuned, because several domains had only around 100 samples, badly overfit territory for a 110M-parameter model if fine-tuned. Each abstract goes through frozen SciBERT, the [CLS] token embedding is extracted, and a small softmax classifier I actually train runs on top. The subject-area classifier uses CountVectorizer + softmax regression instead, since its inputs are short label strings, not abstracts. The ensemble is accuracy-weighted and dynamic: when a user selects N subject areas, N+1 models run, weighted proportional to each model's validation accuracy.

**R — Result:**
96.9% top-5 accuracy on a held-out test set of 3,000 abstracts across 378 journals. Top-1: 76.8%, top-3: 93.4%. Random baseline: 0.26%.

**Probes:**
- *"What's the [CLS] token?"* A special token prepended to BERT-style input with no inherent meaning; because it carries no content of its own, the model accumulates a summary of the whole input there over the attention layers.
- *"Why freeze SciBERT rather than fine-tune?"* Domain sizes varied hugely, some subject areas as small as ~100 samples, badly overfit territory for a 110M-parameter model if fine-tuned; no GPU budget for extended fine-tuning either, and catastrophic forgetting risk on the small domains.
- *"Why accuracy-weighted rather than equal weights?"* Equal weighting over-relies on tiny, easy subject areas and under-relies on large, hard ones; accuracy-proportional weighting lets more reliable classifiers contribute more.
- *"How would you scale to 1 million users?"* The SciBERT embedding step is the bottleneck; at scale, cache embeddings already seen, batch inference, deploy on GPU, and separate the feature-extraction tier from the classification tier.
- *"Which parts did you personally build, versus your three teammates?"* Architecture design, all 27 models, and the ensemble were my responsibility on the team; this is a 4-person team project (not solo work) and there's no per-member commit history to point to, credit here rests on team roles as agreed, not a git trail.
