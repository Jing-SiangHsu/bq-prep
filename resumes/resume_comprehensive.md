**Jing-Siang Hsu**
Los Angeles, CA • jingsianghsu@gmail.com • (213) 331-9601 • linkedin.com/in/jing-siang-hsu • github.com/Jing-Siang

# EDUCATION

**University of California, Los Angeles (UCLA)** | Los Angeles, CA
Master of Engineering in Artificial Intelligence | September 2026 – December 2027 (Expected)

**University of Twente** | Enschede, NL
Bachelor's Degree in Technical Computer Science, Cum Laude, GPA: 4.0/4.0 (US equivalent) | September 2021 – July 2024

# SKILLS

**Programming & Frameworks:** Go, TypeScript, JavaScript, Python, C#, Java, React, Vue 2, Vue 3, Node.js, AngularJS, FastAPI
**Databases & APIs:** PostgreSQL, Firebase, MongoDB, gRPC, RESTful APIs, WebSocket, Kafka, Redis, Pinecone
**Cloud & DevOps:** Google Cloud Platform, Docker, Linux, Git, CI/CD, Distributed Systems
**ML & Data Science:** PyTorch, TensorFlow, HuggingFace Transformers, OpenAI API, MCP (Model Context Protocol)

# PROFESSIONAL EXPERIENCE

**Intrising Networks, Inc.** | Taipei, TW
Full-Stack Software Engineer | June 2025 – June 2026

**AutoPVT** *(Go, gRPC, grpc-gateway, PostgreSQL, Vue 3, Protocol Buffers)*

* Designed a two-tier PostgreSQL locking model as the sole engineer on AutoPVT, a concurrent test-editing platform serving ~30 engineers: exclusive advisory locks with compare-and-bump revision checks for structural writes, and shared locks with per-field merging for fills, eliminating blocking between concurrent testers editing different fields.
  * *GCS 6/8 | FAANG 6/6 | Total 12/14*

* Eliminated dirty-state broadcasts by enforcing commit-before-broadcast ordering: write and audit entry committed atomically in a single PostgreSQL transaction, with the WebSocket broadcast fired only on the gRPC success acknowledgment, ensuring the push channel could never deliver state from a transaction that subsequently rolled back.
  * *GCS 6/8 | FAANG 4/6 | Total 10/14*

* Delivered real-time firmware build status to concurrent subscribers by building a Postgres LISTEN/NOTIFY pipeline (chosen over Redis pubsub to avoid introducing a new infrastructure dependency) that fanned build events to gRPC server-streaming clients, with server-side filtering so each client received only events matching its subscription criteria.
  * *GCS 6/8 | FAANG 4/6 | Total 10/14*

* Resolved 3-5-second load delays on the 2,500-item test catalog, reducing load time to under 500ms by replacing 4 sequential SQL queries and nested-loop tree assembly with a single JOIN and a hashmap-based tree-building algorithm, lazy-loading item details via RPC, and prefetching the lightweight tree into the Vuex store at login.
  * *GCS 8/8 | FAANG 4/6 | Total 12/14*

**Switch Management Interface** *(AngularJS, TypeScript, Go, gRPC, CGO, PAM)*

* Led the authentication and session control requirements for IEC 62443-4-2 SL3 for four product lines: added TOTP-based MFA and enforced concurrent session limits per user and per interface across web, CLI, and Telnet via a shared internal gRPC service.
  * *GCS 6/8 | FAANG 5/6 | Total 11/14*

* Bridged PAM's interactive challenge-response model with HTTP's stateless request cycle by designing a two-RPC login protocol that preserved backward compatibility across all existing single-factor accounts: a password-only call returned an MFA-required signal and a password+TOTP call completed authentication, with a custom PAM conversation handler that distinguished password from TOTP prompts at the protocol level.
  * *GCS 6/8 | FAANG 6/6 | Total 12/14*

* Diagnosed and fixed a config-save bug that silently corrupted credentials across 5 configuration domains: display masking was applied before the final config-read step, causing exported configs to record placeholder masks instead of real values, rendering device configurations unrestorable. Fixed by relocating masking to the outermost gateway egress layer, and resolved a related SSH key-matching bug by switching from index-based to fingerprint-based identity, closing a second silent failure mode in the same subsystem.
  * *GCS 7/8 | FAANG 5/6 | Total 12/14*

* Reduced CI build time by over 80% through scoping asset compilation to per-product SVG allowlists, and forked the open-source gulp-multi-process task runner to add fail-fast behavior, terminating all remaining workers on the first error instead of continuing silently after a failure.
  * *GCS 8/8 | FAANG 3/6 | Total 11/14*

**InTriHub** *(Node.js, Firebase Realtime Database, Google Cloud Functions, PostgreSQL, Vue 2)*

* Cut Firebase costs from $300/month to within Firebase's free tier by tracing the spike to a 70 MB firmware collection inflated by an embedded log field and 3 redundant per-page listeners re-fetching it on every navigation, restructuring it into a lazy-loaded sibling node (70 MB to 5 MB, 93% reduction) and consolidating to one global Vuex listener.
  * *GCS 8/8 | FAANG 5/6 | Total 13/14*

* Fixed an access-control gap where an internal firmware log on InTriHub was readable by non-admin roles, including vendor, salesperson, and partner accounts, instead of staying admin-only. (Role-based over-exposure, not cross-tenant leakage between partner orgs, InTriHub's per-partner data isolation elsewhere was unaffected.)
  * *GCS 3/8 | FAANG 4/6 | Total 7/14 — 5/5 independent FAANG-persona reviewers recommended cutting from the 1-page resume: single bug found incidentally during the cost-optimization work above, not from a deliberate security-testing practice. Kept here as interview-prep material only.*
  * Source: commit `8e9d5c6` (hub-cloud-function) — tightened RTDB rules restricting `firmwareInternalLog` reads to admin only (was leaking to vendor/salesperson/partner via the firmware node).

---

**Texim Europe B.V.** | Haaksbergen, NL
Full-Stack Software Engineer Intern | April 2023 – July 2023

* Eliminated Texim's reliance on an end-of-life ETL platform (SQL Server 2000 DTS, unable to receive security patches) by building a C# WinForms replacement with a VBScript editor, a scheduler with daily/weekly/monthly recurrence, and a multithreaded Windows service, migrating the company's full script library without rewrites.
  * *GCS 6/8 | FAANG 4/6 | Total 10/14*
* Implemented SQL Server Query Notifications to deliver push-based ETL job status updates, giving clients immediate visibility into package completion as each run finished.
  * *GCS 4/8 | FAANG 3/6 | Total 7/14*

# PROJECTS

**Adaptive Ad Recommender** *(Python, FastAPI, Kafka, Debezium, Redis, PostgreSQL, Pinecone, OpenAI API, Google OIDC)*
Full-Stack Developer | July 2026 – Present

* Designed a Kafka (KRaft) and Debezium change-data-capture pipeline to replace a Postgres-to-Pinecone sync prone to dual-write drift, choosing Kafka's per-partition ordering guarantee so eligibility updates apply in the correct order.
  * *GCS 4/8 | FAANG 4/6 | Total 8/14*

* Protected concurrent profile-vector updates with a Postgres advisory lock, then diagnosed a leak where SQLAlchemy's connection pooling let a mid-transaction commit return the lock-holding connection to the pool, silently stranding the lock and deadlocking every later request for that user, fixed by holding one dedicated connection for the entire critical section.
  * *GCS 6/8 | FAANG 5/6 | Total 11/14*

* Authenticated users via Google OIDC, issuing short-lived JWT access tokens alongside Redis-tracked refresh tokens so sessions could be revoked and rotated, keeping the refresh token in an httpOnly cookie out of reach of client-side scripts.
  * *GCS 4/8 | FAANG 5/6 | Total 9/14*

* Red-teamed every LLM call site with real, unmocked prompt-injection attacks, finding that injected override text could bypass the policy reviewer and the onboarding judge alike, then proved instruction-vs-data framing alone fully closed the gap on one surface (5/5 clean re-runs) while it failed on the other (7/7 still exploitable), so replaced the fragile prompt-only fix there with a deterministic length-floor backstop instead.
  * *GCS 7/8 | FAANG 5/6 | Total 12/14*

**Natural Language Processing for Journal Publication Selection** *(Team of 4)*
Machine Learning Engineer | September 2023 – November 2023

* Achieved 96.9% top-5 accuracy across 378 journals by designing and training 27 models: 26 domain-specific SciBERT abstract classifiers and 1 subject-area classifier, combining their predictions with weights proportional to each model's validation accuracy, and validating the ensemble on a held-out test set of 3,000 abstracts.
  * *GCS 8/8 | FAANG 5/6 | Total 13/14*

**Rosenxt Group - Underground Water Pipeline Inspection Application** *(Team of 5)*
Front-End Software Engineer | January 2024 – April 2024

* Led frontend development across the majority of the application in React, independently owning the login, profile, user management, marker management, and map settings pages, with collaborative implementation of pipeline management and sidebar components.
  * *GCS 5/8 | FAANG 3/6 | Total 8/14*
* Secured multi-user access by implementing Role-Based Access Control with session management across three permission levels, restricting feature visibility and data access by stakeholder role.
  * *GCS 7/8 | FAANG 3/6 | Total 10/14*
