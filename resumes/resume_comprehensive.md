**Jing-Siang Hsu**
Los Angeles, CA • j35201887910228@gmail.com • linkedin.com/in/jing-siang-hsu

# EDUCATION

**University of California, Los Angeles (UCLA)** | Los Angeles, CA
Master of Engineering in Artificial Intelligence | September 2026 – December 2027 (Expected)

**University of Twente** | Enschede, NL
Bachelor's Degree in Technical Computer Science, Cum Laude, GPA: 4.0/4.0 (US equivalent) | September 2021 – July 2024

# SKILLS

**Programming & Frameworks:** Go, TypeScript, JavaScript, Python, C#, React, Vue 2, Vue 3, Node.js, AngularJS
**Databases & APIs:** PostgreSQL, Firebase, MongoDB, gRPC, Protocol Buffers, RESTful APIs, WebSocket, Microservices
**Cloud & DevOps:** Google Cloud Platform, Docker, Linux, Git, CI/CD, Distributed Systems
**ML & Data Science:** PyTorch, TensorFlow, HuggingFace Transformers

# PROFESSIONAL EXPERIENCE

**Intrising Networks, Inc.** | Taipei, TW
Full-Stack Software Engineer | June 2025 – June 2026

**AutoPVT** *(Go, gRPC, grpc-gateway, PostgreSQL, Vue 3, Protocol Buffers)*

* Designed a two-tier PostgreSQL locking model as the sole engineer on AutoPVT, a concurrent test-editing platform used by ~30 engineers: exclusive advisory locks with compare-and-bump revision checks for structural writes, and shared locks with per-field merging for fills, so concurrent testers editing different fields never blocked each other.
  * *GCS 6/8 | FAANG 6/6 | Total 12/14*

* Eliminated dirty-state broadcasts by enforcing commit-before-broadcast ordering: write and audit entry committed atomically in a single PostgreSQL transaction, with the WebSocket broadcast fired only on the gRPC success acknowledgment, ensuring the push channel could never deliver state from a transaction that subsequently rolled back.
  * *GCS 6/8 | FAANG 4/6 | Total 10/14*

* Delivered real-time firmware build status to concurrent subscribers by building a Postgres LISTEN/NOTIFY pipeline (chosen over Redis pubsub to avoid introducing a new infrastructure dependency) that fanned build events to gRPC server-streaming clients, with server-side filtering so each client received only events matching its subscription criteria.
  * *GCS 6/8 | FAANG 4/6 | Total 10/14*

* Resolved 3-5 second load delays on the 2,500-item test catalog, reducing load time to under 500ms, by rewriting the server-side catalog fetch from 4 sequential SQL queries with nested loop tree assembly to a single JOIN with hashmap-based tree building, splitting item detail into a lazy-loaded RPC, and prefetching the lightweight tree into the Vuex store at login.
  * *GCS 8/8 | FAANG 4/6 | Total 12/14*

**Switch Management Interface** *(AngularJS, TypeScript, Go, gRPC, CGO, PAM)*

* Led the authentication and session control requirements for IEC 62443-4-2 SL3 for four product lines: added TOTP-based MFA and enforced per-user per-interface concurrent session limits across web, CLI, and Telnet via a shared internal gRPC service.
  * *GCS 6/8 | FAANG 5/6 | Total 11/14*

* Bridged PAM's interactive challenge-response model with HTTP's stateless request cycle by designing a two-RPC login protocol that preserved backward compatibility across all existing single-factor accounts: a password-only call returned an MFA-required signal and a password+TOTP call completed authentication, with a custom PAM conversation handler that distinguished password from TOTP prompts at the protocol level.
  * *GCS 6/8 | FAANG 6/6 | Total 12/14*

* Diagnosed and fixed a config-save bug that silently corrupted credentials across 5 configuration domains: display masking was applied before the final config-read step, causing exported configs to record placeholder masks instead of real values, rendering device configurations unrestorable. Fixed by relocating masking to the outermost gateway egress layer, and resolved a related SSH key-matching bug by switching from index-based to fingerprint-based identity, closing a second silent failure mode in the same subsystem.
  * *GCS 7/8 | FAANG 5/6 | Total 12/14*

* Reduced CI build time by over 80% through scoping asset compilation to per-product SVG allowlists, and replaced a parallel task runner that continued silently despite worker failures with a fail-fast orchestrator that terminated all remaining workers on first error, surfacing failures immediately.
  * *GCS 8/8 | FAANG 3/6 | Total 11/14*

**InTriHub** *(Node.js, Firebase Realtime Database, Google Cloud Functions, PostgreSQL, Vue 2)*

* Cut Firebase costs from $300/month to within Firebase's free tier by tracing the spike to a 70 MB firmware collection inflated by an embedded log field and 4 redundant per-page listeners re-fetching it on every navigation, restructuring it into a lazy-loaded sibling node (70 MB to 5 MB, 93% reduction) and consolidating to one global Vuex listener.
  * *GCS 8/8 | FAANG 5/6 | Total 13/14*

---

**Texim Europe B.V.** | Haaksbergen, NL
Full-Stack Software Engineer Intern | April 2023 – July 2023

* Eliminated Texim's reliance on an end-of-life ETL platform (SQL Server 2000 DTS, unable to receive security patches) by building a C# WinForms replacement with a VBScript editor, scheduler with daily/weekly/monthly recurrence, and multithreaded Windows service, migrating the company's full script library without rewrites.
  * *GCS 6/8 | FAANG 4/6 | Total 10/14*
* Implemented SQL Server Query Notifications to deliver push-based ETL job status updates, giving clients immediate visibility into package completion as each run finished.
  * *GCS 4/8 | FAANG 3/6 | Total 7/14*

# PROJECTS

**Natural Language Processing for Journal Publication Selection** *(Team of 4)*
Machine Learning Engineer | September 2023 – November 2023

* Achieved 96.9% top-5 accuracy across 378 journals by designing and training 27 models: 26 domain-specific SciBERT abstract classifiers and 1 subject area classifier, whose predictions were combined with weights proportional to each model's validation accuracy, validated on a held-out test set of 3,000 abstracts.
  * *GCS 8/8 | FAANG 5/6 | Total 13/14*

**Rosenxt Group - Underground Water Pipeline Inspection Application** *(Team of 5)*
Front-End Software Engineer | January 2024 – April 2024

* Led frontend development across the majority of the application in React, independently owning the login, profile, user management, marker management, and map settings pages, with collaborative implementation of pipeline management and sidebar components.
  * *GCS 5/8 | FAANG 3/6 | Total 8/14*
* Secured multi-user access by implementing Role-Based Access Control with session management across three permission levels, restricting feature visibility and data access by stakeholder role.
  * *GCS 7/8 | FAANG 3/6 | Total 10/14*
