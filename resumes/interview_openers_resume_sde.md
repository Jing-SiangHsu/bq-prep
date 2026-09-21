# Interview Openers for resume_sde.md

Short spoken openers (about 30 to 40 seconds each) for the 10 stories behind the bullets on resume_sde.md, in resume order. Say the opener, then stop and let the interviewer pick where to dig. The full STAR versions and follow-up answers are in `star_stories_resume_sde.md`, same story numbers.

For pure behavioral questions ("tell me about a time..."), expand the opener to roughly a minute and a half instead of stopping at four sentences.

---

## 1. AutoPVT: two-tier locking model

AutoPVT is a test-editing platform where about 30 engineers edit the same test objects at once, and I was the sole engineer designing its concurrency model. I split writes into two kinds: structural changes like editing the item list take an exclusive Postgres advisory lock with a revision check that rejects stale writes, while filling in results takes a shared lock so engineers on different items never block each other. I mapped every write type against every other before writing any code. There were no data corruption incidents from concurrent writes while I was there.

## 2. AutoPVT: catalog load time

Loading the 2,500-item test catalog took 3 to 5 seconds, and about 30 engineers hit it every day. It turned out to be three problems stacked together: four sequential SQL queries with nested-loop tree assembly, a payload heavy enough to push past the gRPC message limit, and no prefetching. I replaced the queries with a single JOIN and hashmap assembly, split out a lightweight list with a lazy fetch per item, and prefetched the tree at login. Load time dropped to under 500 milliseconds.

## 3. Switch Management Interface: IEC 62443 session limits and MFA

Our switch products had to meet IEC 62443-4-2 Security Level 3, and I led the authentication and session control requirements across four product lines, on the gateway and web UI side. That meant adding TOTP-based MFA and limiting concurrent sessions per user and per interface across web, CLI, and Telnet. Since CLI and Telnet log in through PAM in the core firmware, I designed an RPC in the gateway that the firmware calls after a successful login, which makes the gateway the single source of truth for sessions. All four product lines met SL3 on those requirements.

## 4. Switch Management Interface: two-RPC login protocol

Adding TOTP to the web login was tricky because PAM's login is one interactive conversation, but HTTP is stateless, so you can't pause PAM to ask the browser for a code. I designed a two-call protocol: the first call sends only the password, PAM reaches the TOTP prompt and fails on an empty code, and the gateway returns a specific "MFA required" error. The second call sends the password and the real code and finishes the login. Accounts without MFA kept working with no changes, and a CGO-bridged handler let me tell which prompt PAM was on.

## 5. Switch Management Interface: CI build time

Our web UI CI had two problems. The package we used to run build tasks in parallel waited for every worker to finish before reporting a failure, so broken builds sat there silently, and we were compiling SVGs for all 290 switch models when a given product only needed about 48. I forked the package so it kills the remaining workers on the first failure, and added a per-product allowlist for SVG compilation. That cut build time by over 80% across four product-line repos.

## 6. InTriHub: Firebase cost spike

InTriHub's Firebase bill jumped to $300 a month, and the git history didn't explain it. While evaluating a move to Firestore and Algolia, I hit a record size limit that exposed the real cause: a changelog field embedded in every firmware record had grown the collection to about 70 MB, and three pages were each attaching their own listener that re-downloaded it on every navigation. I moved the changelog into a lazy-loaded sibling node, which took the collection down to about 5 MB, and consolidated the listeners into one. Costs dropped back into the free tier, and I skipped the migration entirely.

## 7. Texim Europe: ETL platform replacement

At Texim Europe, the company's ETL ran on SQL Server 2000 DTS, which Microsoft had discontinued and which couldn't get security patches. I built a C# WinForms replacement with a script editor, a scheduler, and a Windows service that runs jobs in the background. The hard constraint was that their whole library of VBScript scripts had to run without rewrites, so I routed execution through cscript.exe, the same engine that had always run them. The full library migrated unchanged.

## 8. Adaptive Ad Recommender: Kafka and Debezium pipeline

In my Adaptive Ad Recommender, campaign eligibility lives in Postgres but ad retrieval runs on Pinecone, and a dual write between them can silently drift, so a budget-exhausted campaign could keep getting served. I replaced it with a change-data-capture pipeline: the app only writes to Postgres, Debezium reads the write-ahead log, and a Kafka consumer updates Pinecone. I chose Kafka because keying messages by campaign ID keeps each campaign's events in order, which the app's Redis queue couldn't guarantee. Postgres stays the single source of truth, and Pinecone converges from the log.

## 9. Adaptive Ad Recommender: prompt-injection testing

I built two LLM features in the same project, a campaign policy reviewer and an onboarding judge, and since both take attacker-controlled text, I red-teamed them with real, unmocked calls. Both were bypassable. Adding instruction-versus-data framing to the prompt fully fixed the policy reviewer, but the same fix didn't work on the onboarding judge, so I added a deterministic length check before anything gets embedded. One remaining issue there I left open on purpose, because fixing it would need server-side state that conflicts with the app's stateless design.

## 10. NLP project: 27-model journal recommender

In a four-person team project, I was the ML engineer for a journal recommendation system covering 378 journals across 26 subject areas. Instead of one big model, I trained 26 domain-specific classifiers on frozen SciBERT embeddings plus a subject-area classifier, and combined their predictions weighted by each model's validation accuracy. It reached 96.9% top-5 accuracy on a held-out test set of 3,000 abstracts.
