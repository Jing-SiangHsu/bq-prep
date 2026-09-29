**Jing-Siang Hsu**
jingsianghsu@gmail.com • (213) 331-9601 • linkedin.com/in/jing-siang-hsu • github.com/Jing-Siang

# EDUCATION

**University of California, Los Angeles (UCLA)** | Los Angeles, CA
Master of Engineering in Artificial Intelligence | September 2026 – December 2027 (Expected)

**University of Twente** | Enschede, NL
Bachelor of Science in Technical Computer Science, Cum Laude, GPA: 4.0/4.0 (US equivalent) | September 2021 – July 2024

# SKILLS

**Programming & Frameworks:** Python, Java, Go, TypeScript, JavaScript, C#, React, Vue 2, Vue 3, AngularJS, Node.js, FastAPI
**Databases & APIs:** PostgreSQL, Firebase, MongoDB, gRPC, RESTful APIs, WebSocket, Kafka, Redis, Pinecone
**Cloud & DevOps:** Google Cloud Platform, Docker, Linux, Git, CI/CD, Distributed Systems
**ML & AI:** PyTorch, TensorFlow, HuggingFace Transformers, OpenAI API, MCP (Model Context Protocol)
**Certifications:** AI Engineer for Developers Associate (DataCamp)

# PROFESSIONAL EXPERIENCE

**Intrising Networks, Inc.** | Taipei, TW
Full-Stack Software Engineer | June 2025 – June 2026

**AutoPVT** *(Go, gRPC, PostgreSQL, Vue 3, Protocol Buffers)*

* Designed a **two-tier PostgreSQL locking model** as the sole engineer on AutoPVT, a concurrent test-editing platform for ~30 engineers, using exclusive locks with optimistic version checks for structural edits and shared locks with per-field merging for data entry, so testers editing different fields never block each other.

* Reduced load time on the 2,500-item test catalog from **3-5s to under 500ms** by collapsing 4 sequential SQL queries into a single JOIN, replacing nested-loop tree assembly with a hashmap-based build, and lazy-loading item details via RPC.

* Built most of AutoPVT's Vue 3 frontend, including a four-step test wizard, an admin app, and a results **autosave** that saves only the fields a tester edits, so concurrent testers' changes to different fields all survive.

**Switch Management Interface** *(AngularJS, TypeScript, Go, gRPC, CGO, PAM)*

* Led the **IEC 62443-4-2 SL3** security requirements for authentication and session control across four product lines: added TOTP-based MFA and enforced concurrent session limits per user and per interface across web, CLI, and Telnet via a shared internal gRPC service.

* Designed a **two-RPC authentication protocol** bridging PAM's interactive challenge-response model with stateless HTTP, adding TOTP as a second step without breaking existing single-factor accounts, via a custom PAM conversation handler that distinguished password from TOTP prompts.

* Reduced CI build time by **over 80%** by scoping asset compilation to per-product SVG allowlists, and forked the open-source gulp-multi-process to fail fast on the first error instead of continuing silently.

**InTriHub** *(Node.js, Firebase Realtime Database, Google Cloud Functions, Vue 2)*

* Cut Firebase costs from **$300/month to $0** by tracing the spike to a 70 MB collection bloated by an embedded log field and redundant per-page listeners, moving logs to a lazy-loaded node (70 MB to 5 MB) and consolidating to one global listener.

---

**Texim Europe B.V.** | Haaksbergen, NL
Full-Stack Software Engineer Intern | April 2023 – July 2023

* Eliminated Texim's reliance on an end-of-life ETL platform (SQL Server 2000 DTS) by building a **C# WinForms replacement** with a VBScript editor, a scheduler with daily/weekly/monthly recurrence, and a multithreaded Windows service, migrating the company's full script library without rewrites.

# PROJECTS

**Adaptive Ad Recommender** *(Python, FastAPI, Kafka, Debezium, Redis, PostgreSQL, Pinecone, OpenAI API, Google OIDC)*
Full-Stack Developer | July 2026 – Present

* Designed a **Kafka (KRaft) and Debezium change-data-capture pipeline** to replace a Postgres-to-Pinecone sync prone to dual-write drift, leveraging Kafka's per-partition ordering guarantee so eligibility updates apply in the correct order.

* Built an LLM-based campaign-review agent and onboarding chat, **red-teamed both with prompt-injection attacks** that successfully bypassed them, then closed the gaps with instruction-vs-data prompt framing where it held and a deterministic length-floor check where it didn't.

**Natural Language Processing for Journal Publication Selection** *(Team of 4)*
Machine Learning Engineer | September 2023 – November 2023

* Achieved **96.9% top-5 accuracy** across 378 journals by designing and training 27 models: 26 domain-specific SciBERT abstract classifiers and 1 subject-area classifier, combining their predictions with weights proportional to each model's validation accuracy, and validating the ensemble on a held-out test set of 3,000 abstracts.
