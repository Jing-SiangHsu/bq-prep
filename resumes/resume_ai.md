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

**AutoPVT** *(Go, gRPC, PostgreSQL, Vue 3, Protocol Buffers)*

* Designed a two-tier PostgreSQL locking model as the sole engineer on AutoPVT, a concurrent test-editing platform used by ~30 engineers: exclusive advisory locks with compare-and-bump revision checks for structural writes, and shared locks with per-field merging for fills, so concurrent testers editing different fields never blocked each other.

* Resolved 3-5 second load delays on the 2,500-item test catalog, reducing load time to under 500ms, by rewriting the server-side catalog fetch from 4 sequential SQL queries with nested loop tree assembly to a single JOIN with hashmap-based tree building, splitting item detail into a lazy-loaded RPC, and prefetching the lightweight tree into the Vuex store at login.

**Switch Management Interface** *(AngularJS, TypeScript, Go, gRPC)*

* Led the authentication and session control requirements for IEC 62443-4-2 SL3 for four product lines: added TOTP-based MFA and enforced per-user per-interface concurrent session limits across web, CLI, and Telnet via a shared internal gRPC service.

* Diagnosed and fixed a config-save bug that silently corrupted credentials across 5 configuration domains: display masking was applied before the final config-read step, causing exported configs to record placeholder masks instead of real values, rendering device configurations unrestorable. Fixed by relocating masking to the outermost gateway egress layer, and resolved a related SSH key-matching bug by switching from index-based to fingerprint-based identity, closing a second silent failure mode in the same subsystem.

**InTriHub** *(Node.js, Firebase Realtime Database, Google Cloud Functions, Vue 2)*

* Cut Firebase costs from $300/month to within Firebase's free tier by tracing the spike to a 70 MB firmware collection inflated by an embedded log field and 4 redundant per-page listeners re-fetching it on every navigation, restructuring it into a lazy-loaded sibling node (70 MB to 5 MB, 93% reduction) and consolidating to one global Vuex listener.

# PROJECTS

**Natural Language Processing for Journal Publication Selection** *(Team of 4)*
Machine Learning Engineer | September 2023 – November 2023

* Achieved 96.9% top-5 accuracy across 378 journals by designing and training 27 models: 26 domain-specific SciBERT abstract classifiers and 1 subject area classifier, whose predictions were combined with weights proportional to each model's validation accuracy, validated on a held-out test set of 3,000 abstracts.
