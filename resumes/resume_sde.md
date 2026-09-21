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
**Certifications:** AI Engineer for Developers Associate (DataCamp)

# PROFESSIONAL EXPERIENCE

**Intrising Networks, Inc.** | Taipei, TW
Full-Stack Software Engineer | June 2025 – June 2026

**AutoPVT** *(Go, gRPC, PostgreSQL, Vue 3, Protocol Buffers)*

* Designed a **two-tier PostgreSQL locking model** as the sole engineer on AutoPVT, a concurrent test-editing platform serving ~30 engineers: exclusive advisory locks with compare-and-bump revision checks for structural writes, and shared locks with per-field merging for fills, eliminating blocking between concurrent testers editing different fields.

* Resolved 3-5-second load delays on the 2,500-item test catalog, reducing load time to **under 500ms** by replacing 4 sequential SQL queries and nested-loop tree assembly with a single JOIN and a hashmap-based tree-building algorithm, lazy-loading item details via RPC, and prefetching the lightweight tree into the Vuex store at login.

**Switch Management Interface** *(AngularJS, TypeScript, Go, gRPC, CGO, PAM)*

* Led the **IEC 62443-4-2 SL3** security requirements for authentication and session control across four product lines: added TOTP-based MFA and enforced concurrent session limits per user and per interface across web, CLI, and Telnet via a shared internal gRPC service.

* Bridged PAM's interactive challenge-response model with HTTP's stateless request cycle by designing a **two-RPC authentication protocol** that preserved backward compatibility across all existing single-factor accounts: a password-only call returned an MFA-required signal and a password+TOTP call completed authentication, with a custom PAM conversation handler that distinguished password from TOTP prompts at the protocol level.

* Reduced CI build time by **over 80%** through scoping asset compilation to per-product SVG allowlists, and forked the open-source gulp-multi-process task runner to add fail-fast behavior, terminating all remaining workers on the first error instead of continuing silently after a failure.

**InTriHub** *(Node.js, Firebase Realtime Database, Google Cloud Functions, Vue 2)*

* Cut Firebase costs from **$300/month to within Firebase's free tier** by tracing the spike to a 70 MB firmware collection inflated by an embedded log field and 3 redundant per-page listeners re-fetching it on every navigation, restructuring it into a lazy-loaded sibling node (70 MB to 5 MB, 93% reduction) and consolidating to one global Vuex listener.

---

**Texim Europe B.V.** | Haaksbergen, NL
Full-Stack Software Engineer Intern | April 2023 – July 2023

* Eliminated Texim's reliance on an end-of-life ETL platform (SQL Server 2000 DTS, unable to receive security patches) by building a **C# WinForms replacement** with a VBScript editor, a scheduler with daily/weekly/monthly recurrence, and a multithreaded Windows service, migrating the company's full script library without rewrites.

# PROJECTS

**Adaptive Ad Recommender** *(Python, FastAPI, Kafka, Debezium, Redis, PostgreSQL, Pinecone, OpenAI API, Google OIDC)*
Full-Stack Developer | July 2026 – Present

* Designed a **Kafka (KRaft) and Debezium change-data-capture pipeline** to replace a Postgres-to-Pinecone sync prone to dual-write drift, choosing Kafka's per-partition ordering guarantee so eligibility updates apply in the correct order.

* Built an LLM-based campaign-review agent and onboarding chat, then **red-teamed them with real prompt-injection attacks** and found they were both bypassable. Closed the gap with prompt-level instruction-vs-data framing where it held, and with a deterministic length-floor check where it didn't.

**Natural Language Processing for Journal Publication Selection** *(Team of 4)*
Machine Learning Engineer | September 2023 – November 2023

* Achieved **96.9% top-5 accuracy** across 378 journals by designing and training 27 models: 26 domain-specific SciBERT abstract classifiers and 1 subject-area classifier, combining their predictions with weights proportional to each model's validation accuracy, and validating the ensemble on a held-out test set of 3,000 abstracts.
